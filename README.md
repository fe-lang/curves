# curves

Prime fields, elliptic curves and pairings in Fe, for native programs:
BLS12-381, BN254, secp256k1 and P-256. Extracted from
[fevm](https://github.com/fe-lang/fevm)'s precompiles with its history intact;
fevm keeps the precompiles' input encodings and gas.

The arithmetic runs in variable time. Use it to verify, not to sign or to
handle secrets.

| Module | Contents |
| --- | --- |
| `montgomery` | `Mont<M>`, the field of a prime below 2^256 in Montgomery form, chosen by a `Modulus` type; the `Field` trait; the carry-chain word operations `adc`, `sbb`, `mac` and `mul_wide` |
| `inversion` | Bernstein-Yang safegcd inverses: `invert_256`, `invert_384` |
| `bls_field`, `bls_tower` | BLS12-381's `Fp` (six 64-bit limbs), `Fp2`, `Fp6` and `Fp12` |
| `bls_curve` | Jacobian `Point<F>` and `Affine<F>` for G1 and G2, subgroup checks, `msm`, `multiply_two`, `decompress_g1` |
| `bls_map` | RFC 9380's `map_to_g1` and `map_to_g2` |
| `bls_pairing` | `miller_loop`, `miller_loop_lines` for fixed G2 points, `final_exponentiation` |
| `kzg`, `kzg_lines` | `decode_kzg_g1` and `verify_proof` for the mainnet trusted setup |
| `bn254_field`, `bn254_curve`, `bn254_pairing`, `bn254` | BN254's tower, points and pairing, `multiply_g1`, `in_g2` |
| `secp256k1` | `recover`, a public key from a signature |
| `p256` | `verify`, an ECDSA signature |

Add an immutable git revision to your ingot's `fe.toml`:

```toml
[dependencies]
curves = { source = "https://github.com/fe-lang/curves", rev = "<full commit SHA>" }
```

```fe
use curves::bls_curve::{g1_generator, g2_generator}
use curves::bls_pairing::pairing_product_is_one

let g1 = g1_generator()
let g2 = g2_generator()
// e([5]G1, G2) * e(-G1, [5]G2) = 1.
let holds = pairing_product_is_one(g1.multiply(5).to_affine(), g2.to_affine(),
                                   g1.neg().to_affine(), g2.multiply(5).to_affine())
```

Byte inputs are read through `keccak::ByteSource`, so a caller pins the same
[keccak](https://github.com/fe-lang/keccak) revision as this ingot's `fe.toml`.

## Checks

- `fe test --backend native .` runs the tests. They pass at `-O 0`, `1` and
  `2` with Fe `6ccb3ca8e`, the compiler fevm pins.
- `python3 gen_kzg_lines.py | diff - src/kzg_lines.fe` checks the precomputed
  Miller loop lines of `-G2` and `[tau]G2` against their generator.

## License

Licensed under the [Apache License, Version 2.0](./LICENSE).
