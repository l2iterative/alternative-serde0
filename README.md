## Alternative serialization for RISC Zero

<img src="title.png" align="right" alt="a group of people in a very crowded train" width="300"/>

This repository presents an alternative serialization for RISC Zero that handles `Vec<u8>` as well as some tuples in a 
more compact format.

### Motivation

RISC Zero serializes the input to the zkVM into a vector of `u32`, through the `serde` framework. 
This, however, means that it suffers from one of the major open problems in Rust.

When `serde` derives `Serialize` and `Deserialize` for Rust data structures, it has the following implementation:
```rust
impl<T> Serialize for Vec<T> where T: Serialize
{
    #[inline]
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error> 
        where S: Serializer,
    {
        serializer.collect_seq(self)
    }
}
```
And `collect_seq` has the following default implementation, meaning that the element would be serialized one after the 
other, as a concatenation.
```rust
pub trait Serializer: Sized {
    fn collect_seq<I>(self, iter: I) -> Result<Self::Ok, Self::Error>
        where I: IntoIterator, <I as IntoIterator>::Item: Serialize,
    {
        let mut iter = iter.into_iter();
        let mut serializer = tri!(self.serialize_seq(iterator_len_hint(&iter)));
        tri!(iter.try_for_each(|item| serializer.serialize_element(&item)));
        serializer.end()
    }
}
```

This would impact RISC Zero because now, to serialize a byte array `Vec<u8>`, each number would be converted into u32, and an 
array of `Vec<u32>` are to be serialized, which leads to 4x storage overhead. 

Back to the serde discussion. The issue is that we may want to specify different rules for different `T`, particularly, if `T = u8`, for better 
efficiency. One solution is the [serde_bytes](https://docs.rs/serde_bytes/latest/serde_bytes/index.html) crate, which 
allows one to bypass the limitation through a customized serde function.
```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize)]
struct Efficient<'a> {
    #[serde(with = "serde_bytes")]
    bytes: &'a [u8],

    #[serde(with = "serde_bytes")]
    byte_buf: Vec<u8>,

    #[serde(with = "serde_bytes")]
    byte_array: [u8; 314],
}
```

The idea is to implement a different derivation strategy that does not derive `Vec<T>` for any serializable `T`, and 
ask the developer to switch between these two strategies. There are a few problems with this approach:
- It requires developers to modify the layers and layers of abstractions. If a developer uses four crates---A, B, C, 
D---A depends on B, B depends on C, C depends on D, and D uses `Vec<u8>`, the developer needs to go all the way down to 
D to change the definitions of its data structures. This can cause compatibility issues with the rest of the system. 
- It is very difficult to be comprehensive because `Vec<u8>` is extremely common in Rust data structures, and a developer 
would need to do a few passes of the code in order to clean this issue.
- If it requires significant patching, it would be very difficult to sync the code with their original repositories. 
This not only increases maintenance overhead, but in practice often leads to security issues.

People's hope for this problem rests on [specialization](https://rust-lang.github.io/rfcs/1210-impl-specialization.html),
but there are limited chances that this can become stable any time soon. Use nightly for production environment is heavily
discouraged, and would not be suitable for RISC Zero's zkVM because it is in the process of becoming part of Rust.

## Our solution

Prior discussion shows that a bottom-up approach can be a disaster. This repository suggests a top-down approach, or in 
other words, we do not require any modification to the existing data structures implemented in Rust, but instead, we 
present a serializer that converts RISC Zero input into `Vec<u32>`, with the corresponding deserialization.

The idea is to run a finite-state automata (called `ByteBufAutomata`) alongside the serialization, and when it 
observes the following, it temporarily modifies the way of serialization so that it resembles the padded-to-word approach 
that RISC Zero recommends. 

- A sequence whose *immediate* child is `u8`, no exception. This applies to `[u8]`, `Vec<u8>`.
- The starting segment of a tuple that consists of only `u8`, which applies to `[u8; 0..=32]` and 
`(u8, u8, u8, ..., u8, T, ...)`. Of course, one may prefer to support only the former but not the latter, but this is 
not possible as `serde` interprets the short `u8` vector as like tuple, so in order to support short `u8` vector, we 
have to allow the starting segment of a tuple.

The finite-state automata, throughout the process, is being activated and deactivated.
- **Activate:** The start of serializing a sequence or a tuple would activate it.
- **Deactivate:** Virtually all of remaining serialization steps other than serializing `u8` would deactivate it. When it 
is deactivated, it serializes the receiving `u8` into a compact and padded `Vec<u32>`. The end of serializing a sequence 
or a tuple would also deactivate it.

When the machine is active, serializing and deserializing over `u8` would be different.
- **Serialize:** The automata withholds the `u8` and would serialize them altogether with deactivated.
- **Deserialize:** To read a single `u8`, the automata reads an entire word, converts it into four bytes, and supplies 
these four bytes. This process repeats until the automata is deactivated, at which moment either there are not remaining 
bytes or all the remaining bytes are zeros.

Detail of the implementation can be found in the codebase. Below we summarize the main changes in the code.

### Serialization

The old code for `serialize_u8`, `serialize_seq`, and `serialize_struct` are good examples to compare the code before and after, 
other than the code for the new finite-state automata.

```rust
// Old
impl<'a, W: WordWrite> serde::ser::Serializer for &'a mut Serializer<W> {
    fn serialize_u8(self, v: u8) -> Result<()> {
        self.serialize_u32(v as u32)
    }

    fn serialize_seq(self, len: Option<usize>) -> Result<Self::SerializeSeq> {
        match len {
            Some(val) => {
                self.serialize_u32(val.try_into().unwrap())?;
                Ok(self)
            }
            None => Err(Error::NotSupported),
        }
    }

    fn serialize_struct(self, _name: &'static str, _len: usize) -> Result<Self::SerializeStruct> {
        Ok(self)
    }
}

// New
impl<'a, W: WordWrite> serde::ser::Serializer for &'a mut Serializer<W> {
    fn serialize_u8(self, v: u8) -> Result<()> {
        if self.byte_buf_automata.borrow_mut().take(v) {
            Ok(())
        } else {
            self.serialize_u32(v as u32)
        }
    }

    fn serialize_seq(self, len: Option<usize>) -> Result<Self::SerializeSeq> {
        match len {
            Some(val) => {
                self.serialize_u32(val.try_into().unwrap())?;
                activate_byte_buf_automata!(self);
                Ok(self)
            }
            None => Err(Error::NotSupported),
        }
    }


    fn serialize_struct(self, _name: &'static str, _len: usize) -> Result<Self::SerializeStruct> {
        deactivate_byte_buf_automata!(self);
        Ok(self)
    }
}
```

The main changes are `activate!(self)` and `deactivate!(self)`, which are helpful macros that activate or deactivate the 
automata.

### Deserialization

Similarly, the code change outside the automata is minimal, consisting of redirection on `deserialize_u8` and 
insertions of `activate!(self)` and `deactivate!(self)` macro calls.

```rust
// old
impl<'de, 'a, R: WordRead + 'de> serde::Deserializer<'de> for &'a mut Deserializer<'de, R> {
    fn deserialize_u8<V>(self, visitor: V) -> Result<V::Value>
        where
            V: Visitor<'de>,
    {
        visitor.visit_u32(self.try_take_word()?)
    }

    fn deserialize_seq<V>(self, visitor: V) -> Result<V::Value>
        where
            V: Visitor<'de>,
    {
        let len = self.try_take_word()? as usize;
        visitor.visit_seq(SeqAccess {
            deserializer: self,
            len,
        })
    }

    fn serialize_struct(self, _name: &'static str, _len: usize) -> Result<Self::SerializeStruct> {
        Ok(self)
    }
}

// new
impl<'de, 'a, R: WordRead + 'de> serde::Deserializer<'de> for &'a mut Deserializer<'de, R> {
    fn deserialize_u8<V>(self, visitor: V) -> Result<V::Value>
        where
            V: Visitor<'de>,
    {

        visitor.visit_u8(self.byte_buf_automata.borrow_mut().take(&mut self.reader)?)
    }

    fn deserialize_seq<V>(self, visitor: V) -> Result<V::Value>
        where
            V: Visitor<'de>,
    {
        let len = self.try_take_word()? as usize;
        activate_byte_buf_automata!(self);
        visitor.visit_seq(SeqAccess {
            deserializer: self,
            len,
        })
    }

    fn serialize_struct(self, _name: &'static str, _len: usize) -> Result<Self::SerializeStruct> {
        deactivate_byte_buf_automata!(self);
        Ok(self)
    }
}
```

### Result

The integration test [here](src/integration_test.rs) can provide more information. But in our example, we use a struct 
that has a member `Vec<u8>` and another member `Vec<Vec<u8>>`.
```rust
fn test() {
    // ...
    test_s.strings = b"Here is a string.".to_vec();
    test_s.stringv = vec![b"string a".to_vec(), b"34720471290497230".to_vec()];
    // ...
}
```

Our experiment shows that it can correctly serialize them into the compact format in `Vec<u32>`.

### Relation to our Bonsai PHP SDK

It is important to clarify that the [Bonsai PHP SDK](https://github.com/l2iterative/bonsai-sdk-php) repository implements 
the standard RISC Zero serialization, not the alternative one here. But the alternative serialization does have the 
benefit because now `Vec<u8>` in Rust is being treated similar to `string` in PHP, which is a binary-safe version of 
byte buffer.

We may update the Bonsai PHP SDK, but we warn that there are going to be overhead. The fact that short u8 vectors are 
being treated as tuples force us to serialize some tuples in the same way, and PHP needs to conform to these existing 
serde practice.

### License

The code largely comes from RISC Zero's implementation [here](https://github.com/risc0/risc0/tree/main/risc0/zkvm/src/serde), 
with modifications necessary to add the finite state automata. 

Since RISC Zero is under the Apache 2.0 license, this repository would also be Apache 2.0.
