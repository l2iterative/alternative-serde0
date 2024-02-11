## Smaller serialization for RISC Zero

<img src="title.png" align="right" alt="a group of people in a very crowded train" width="300"/>

This repository implements a different algorithm for RISC Zero's serialization and has been 
circulated for discussion in https://github.com/risc0/risc0/pull/1303.

There is a chance that this algorithm may become the official serializer in RISC Zero, but there are 
issues pending to be resolved. 

In the meantime, developers can use this serializer in their own implementation for customized serialization and 
deserialization. 

### Motivation

RISC Zero serializes the input to the zkVM into a vector of `u32`, through the `serde` framework. 
This, however, means that it suffers from one of the major open problems in Rust.

When `serde` implements `Serialize` and `Deserialize` for Rust data structures, it has the following implementation:
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

The idea is to implement a different implementation strategy that does not apply a generic rule to `Vec<T>` (for any serializable `T`), and 
asks the developer to switch between these two strategies. There are a few problems with this approach:
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

Prior discussion shows that a bottom-up approach can be disastrous. This repository suggests a top-down approach, or in 
other words, we do not require any modification to the existing data structures implemented in Rust, but instead, we 
present a serializer that converts RISC Zero input into `Vec<u32>`, with the corresponding deserialization.

The idea is to have a side buffer, which we call `ByteBuffer`, managed by `ByteHandler`. When it observes several 
continuous u8 being serialized, it tries to put them together rather than having each of them occupying a word.

The byte buffer consists of four bytes. So when there are four bytes in the buffer, a word would be produced, and the 
buffer would be emptied. When something other than a byte is being serialized, the byte handler would immediately emit 
a word and clean up the buffer.

Detail of the implementation can be found in the codebase. Below we summarize the main changes in the code.

### Serialization

The old code for `serialize_u8` and `serialize_u32` are good examples to compare the code before and after, 
other than the code for the new finite-state automata.

```rust
// Old
impl<'a, W: WordWrite> serde::ser::Serializer for &'a mut Serializer<W> {
    fn serialize_u8(self, v: u8) -> Result<()> {
        self.serialize_u32(v as u32)
    }

    fn serialize_u32(self, v: u32) -> Result<()> {
        self.stream.write_words(&[v]);
        Ok(())
    }
}

// New
impl<'a, W: WordWrite> serde::ser::Serializer for &'a mut Serializer<W> {
    fn serialize_u8(self, v: u8) -> Result<()> {
        self.byte_handler.handle(&mut self.stream, v)
    }

    fn serialize_u32(self, v: u32) -> Result<()> {
        self.byte_handler.reset(&mut self.stream)?;
        let res = self.stream.write_words(&[v]);

        if res.is_err() {
            return Err(Error::from(res.unwrap_err()));
        } else {
            return Ok(res.unwrap());
        }
    }
}
```

The main changes are `self.byte_handler.handle()` and `self.byte_handler.reset()`. 
- `Handle` passes over a byte to the byte handler so that this byte would be emitted together with other bytes into a 
word when appropriate.
- `Reset` tells the byte handler that something other than a byte is going to be serialized, and whatever in the buffer 
must be emitted, and the buffer needs to be emptied.

### Deserialization

Similarly, the code change consists of redirection on `deserialize_u8` and insertions of 
`activate_byte_buf_automata_and_take!(self)` and `deactivate_byte_buf_automata!(self)` macro calls.

```rust
// old
impl<'de, 'a, R: WordRead + 'de> serde::Deserializer<'de> for &'a mut Deserializer<'de, R> {
    fn deserialize_u8<V>(self, visitor: V) -> Result<V::Value>
        where
            V: Visitor<'de>,
    {
        visitor.visit_u32(self.try_take_word()?)
    }

    fn deserialize_u128<V>(self, visitor: V) -> Result<V::Value>
    where
        V: Visitor<'de>,
    {
        let mut bytes = [0u8; 16];
        self.reader.read_padded_bytes(&mut bytes)?;
        visitor.visit_u128(u128::from_le_bytes(bytes))
    }
}

// new
impl<'de, 'a, R: WordRead + 'de> serde::Deserializer<'de> for &'a mut Deserializer<'de, R> {
    fn deserialize_u8<V>(self, visitor: V) -> Result<V::Value>
        where
            V: Visitor<'de>,
    {   
        visitor.visit_u8(self.byte_handler.handle_byte(&mut self.reader)?)
    }

    fn deserialize_u128<V>(self, visitor: V) -> Result<V::Value>
    where
        V: Visitor<'de>,
    {
        self.byte_handler.reset()?;
        let mut bytes = [0u8; 16];
        self.reader.read_padded_bytes(&mut bytes)?;
        visitor.visit_u128(u128::from_le_bytes(bytes))
    }
}
```

### Handling one corner case

The high-level plan described above has a limitation. Since the byte handler is withholding bytes, and that the serializer
would not notify the byte handler when it reaches the end of serialization, there is a situation when the byte handler 
is unable to emit the bytes into a word because the serialization has been completed. 

This is a tricky issue that requires special attention. Our strategy is to observe that, first of all, we can split all types in Rust into three groups.
- primitive types:
  * bool, u8, u16, u32, u64, u128, i8, i16, i32, i64, i128, f32, f64, char, str,
  * ()
  * unit struct aka "struct Nothing"
  * unit enum aka "enum{ A, B, C }" with no underlying variables
- "newtype", a virtually pseudo type defined over another type, which is specifically defined for efficiency
  * Option<T>, which can be considered as `enum { None, Some(T) }`
  * newtype struct, aka `struct(T)`
  * newtype variant, aka `enum { struct1(T1), struct2(T2), ... }`
- wrapper:
  * seq, including vec![], long slice
  * tuple aka `(T1, T2)`
  * tuple struct aka `struct(T1, T2)`
  * tuple variant aka `enum { struct1(T1, T2), struct2(T3, T4, T5) }`
  * map
  * struct aka `struct { a: A, b: B}`
  * struct variant aka `enum { struct1 {a: T1, b:T2}, struct2{a: T3, b: T4, c: T5) }`

Note that if the buffered bytes have not been fully written to the stream, it means that the last primitive type being 
written has to be either bool or u8 (we will just treat bool as u8 in the following). There are no primitive types after it, 
otherwise it would be written to the stream because the byte handler would be reset.

So, there are only two possibilities of this u8.
- the variable to be serialized is directly or indirectly inside some sort of a wrapper. By "indirectly", it means that `struct { Option<struct(Option<u8>)> }` will also be considered as "inside a wrapper".
- the variable to be serialized is not inside any wrapper. It could be `u8` or `Option<struct(Option<u8>)>`.

We handle both separately.

For the first case, we introduce a notion of "depth". When serializer enters a wrapper, the depth is increased by one, when it leaves the wrapper, the depth is decreased by one. When the depth is zero, it means that it must have left the last layer of meaningful wrapper, and there are no new bytes possible after it. It needs to be immediately written to the stream when the depth hits zero. The implementation looks like this:
```rust
    #[inline]
    fn decrease_depth<W: WordWrite>(&mut self, stream: &mut W) -> Result<()> {
        self.depth -= 1;
        if self.depth == 0 && self.status != 0 {
            stream.write_words(&[self.byte_holder])?;
            self.status = 0;
        }
        Ok(())
    }
```

For the second case, we notice that the last u8 is in depth 0, and therefore, this u8 is put into the buffer and immediately being written down into the stream, i.e., treating u8 as u32.
```rust
     fn handle<W: WordWrite>(&mut self, stream: &mut W, v: u8) -> Result<()> {
        if self.depth == 0 {
            stream.write_words(&[v as u32])?;
        } else {
            ......
        }
        Ok(())
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

### Updates

This algorithm has been completed changed several times to handle different corner cases.

It is necessary to credit @austinabell, @flaub, and @nategraf for the thought process of leading to this new algorithm.

Readers interested in the algorithm can check the pending PR: https://github.com/risc0/risc0/pull/1303.

### License

The code largely comes from RISC Zero's implementation [here](https://github.com/risc0/risc0/tree/main/risc0/zkvm/src/serde), 
with modifications necessary to add the finite state automata. 

Since RISC Zero is under the Apache 2.0 license, this repository would also be Apache 2.0.
