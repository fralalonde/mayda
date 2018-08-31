# mayda

[![Build Status](https://travis-ci.org/harharkh/mayda.svg)](https://travis-ci.org/fralalonde/mayda)
[![Latest Version](https://img.shields.io/crates/v/mayda.svg)](https://crates.io/crates/mayda)
![Works on nightly](https://img.shields.io/badge/works%20on-nightly-lightgrey.svg)

`mayda` is a Rust library to compress integer arrays (all primitive integer
types are supported). The design favors decompression speed and the ability to
index the compressed array over the compression ratio, on the principle that
the runtime penalty for using compressed arrays should be as small as possible.

This crate provides three variations on a single compression algorithm. The
[`Uniform`] type can decompress around six billion `u32`s per second, or 24
GiB/s of decompressed integers, on a 2.6 GHz Intel Core i7-6700HQ processor
(see below for specifics). The [`Monotone`] and [`Unimodal`] types decompress
at a little less than half the speed, but can have much better compression
ratios depending on the distribution of the integers. Overall performance is
comparable to the fastest (known) libraries in any language.

The basic approach is described in [Zukowski2006] and [Lemire2015].

### Documentation

The [module documentation][docs] provides further examples and some more
detailed descriptions of the algorithms involved.

### Usage

Add this to your `Cargo.toml`:

```toml
[dependencies]
mayda = "0.2"
```

and this to your crate root:

```rust
extern crate mayda;
```

### Example: encoding and decoding

The `Uniform` struct is recommended for general purpose integer compression.
Use the `Encode` trait to encode and decode the array.

```rust
extern crate mayda;

use mayda::{Uniform, Encode};

fn main() {
	// Integers in some interval of length 255, require one byte per integer
	let input: Vec<u32> = (0..128).map(|x| (x * 73) % 181 + 307).collect();

	let mut uniform = Uniform::new();
	uniform.encode(&input).unwrap();

	let i_bytes = std::mem::size_of_val(input.as_slice());
	let u_bytes = std::mem::size_of_val(uniform.storage());
	
	// 128 bytes for encoded entries, 12 bytes of overhead
	assert_eq!(i_bytes, 512);
	assert_eq!(u_bytes, 140);

	let output: Vec<u32> = uniform.decode();
	assert_eq!(input, output);
}
```

### Example: indexing

Use the `Access` and `AccessInto` traits to index the compressed array. This is
similar to `Index`, but returns a vector instead of a slice.

```rust
extern crate mayda;

use mayda::{Uniform, Encode, Access};

fn main() {
	// All primitive integer types supported
	let input: Vec<isize> = (-64..64).collect();

	let mut uniform = Uniform::new();
	uniform.encode(&input).unwrap();

	let val: isize = uniform.access(64);
	assert_eq!(val, 0);

	// Vector is returned to give ownership to the caller
	let range: Vec<isize> = uniform.access(..10);
	assert_eq!(range, (-64..-54).collect::<Vec<_>>());
}
```

### Performance

Consider the following three vectors:
```rust
extern crate rand;

use rand::distributions::{IndependentSample, Range, Normal};

let mut rng = rand::thread_rng();
let length: usize = 1024;

// `input1` contains random integers
let mut input1: Vec<u32> = Vec::with_capacity(length);
let range = Range::new(0, 1024);
for _ in 0..length {
  input1.push(range.ind_sample(&mut rng));
}

// `input2` contains monotone increasing integers
let mut input2: Vec<u32> = input1.clone();
input2.sort();

// `input3` contains Gaussian distributed integers with outliers
let mut input3: Vec<u32> = Vec::with_capacity(length);
let gaussian = Normal::new(4086., 128.);
for _ in 0..length {
  input3.push(gaussian.ind_sample(&mut rng) as u32);
}
let index = Range::new(0, length);
let outliers = Range::new(0, std::u32::MAX);
for _ in 0..(length / 16) {
  input3[index.ind_sample(&mut rng)] = outliers.ind_sample(&mut rng);
}
```

The performance of the [`Uniform`], [`Monotone`] and [`Unimodal`] types for
these three vectors on a 2.6 GHz Intel Core i7-6700HQ processor is given below.
Encoding and decoding speeds using `decode_into` are reported in billions of
integers per second, time required to index the last entry is reported in
nanoseconds, and compression is reported as bits per integer. Native encoding
and decoding speed measurements perform a memcpy. The Shannon entropy is a
reasonable target for the bits per integer.

For `input1` the Shannon entropy is 10.00. `Uniform` is preferrable in every
respect for the general case.

|          | Encode (BInt/s) | Decode (BInt/s) | Index (ns) | Bits/Int |
|----------|-----------------|-----------------|------------|----------|
| Uniform  |       1.28      |       5.75      |     26     |   10.63  |
| Monotone |       1.34      |       2.49      |     63     |   32.63  |
| Unimodal |       0.21      |       2.01      |     59     |   11.16  |
| Native   |      26.26      |      26.26      |      0     |    32    |

For `input2` the Shannon entropy is 10.00, but the additional structure is used
by `Monotone` to improve compression.

|          | Encode (BInt/s) | Decode (BInt/s) | Index (ns) | Bits/Int |
|----------|-----------------|-----------------|------------|----------|
| Uniform  |       1.23      |       5.88      |     26     |   8.00   |
| Monotone |       1.42      |       2.42      |     69     |   3.63   |
| Unimodal |       0.24      |       2.07      |     27     |   8.19   |
| Native   |      26.26      |      26.26      |      0     |    32    |

For `input3` the Shannon entropy is 12.46, but compression is difficult due to
the presence of outliers. `Unimodal` gives the most robust compression.

|          | Encode (BInt/s) | Decode (BInt/s) | Index (ns) | Bits/Int |
|----------|-----------------|-----------------|------------|----------|
| Uniform  |       1.26      |       6.10      |     26     |   32.63  |
| Monotone |       1.35      |       2.49      |     65     |   32.63  |
| Unimodal |       0.18      |       1.67      |     61     |   12.50  |
| Native   |      26.26      |      26.26      |      0     |    32    |

## License

`mayda` is licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.

### Contributors

[Jeremy Mason](https://github.com/harharkh) - original author
[Francis Lalonde](https://github.com/fralalonde) - current maintainer

[//]: #
   [`Uniform`]: <https://harharkh.github.io/mayda/mayda/uniform/index.html>
   [`Monotone`]: <https://harharkh.github.io/mayda/mayda/monotone/index.html>
   [`Unimodal`]: <https://harharkh.github.io/mayda/mayda/unimodal/index.html>
   [Zukowski2006]: <https://dx.doi.org/10.1109/ICDE.2006.150>
   [Lemire2015]: <https://arxiv.org/abs/1401.6399>
   [docs]: <https://harharkh.github.io/mayda/mayda/index.html>
   [Apache-2.0]: <https://www.apache.org/licenses/LICENSE-2.0>
   [MIT]: <https://opensource.org/licenses/MIT>
