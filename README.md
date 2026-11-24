# curves

Prime fields, elliptic curves and pairings in Fe, for native programs:
BLS12-381, BN254, secp256k1, P-256, and a KZG point-evaluation proof check.
Extracted from [fevm](https://github.com/fe-lang/fevm)'s precompiles with its
history intact; fevm keeps the precompiles' input encodings and gas.

The arithmetic runs in variable time. Use it to verify, not to sign or to
handle secrets.

| Module | Contents |
| --- | --- |
| `montgomery` | `Mont<M>` (prime field below 2^256 in Montgomery form, chosen by a `Modulus` type), `Field`, and the carry-chain word ops `adc`, `sbb`, `mac`, `mul_wide` |
| `inversion` | Bernstein-Yang safegcd inverses: `invert`, `invert_256`, `invert_384` |
| `bls_field`, `bls_tower` | BLS12-381's `Fp` (six 64-bit Montgomery limbs), `Field`, `Fp2`, `Fp6`, `Fp12` |
| `bls_curve` | `ORDER`, Jacobian `Point<F>` and `Affine<F>` for G1 and G2, `g1_generator`, `g2_generator`, `on_g1`, `on_g2`, `in_subgroup`, `clear_cofactor`, `multiply_g1`, `multiply_g2`, `multiply_two`, `msm`, `decompress_g1` |
| `bls_map` | RFC 9380's `map_to_g1` and `map_to_g2` (field element to subgroup point; includes isogeny and cofactor clearing) |
| `bls_pairing` | `miller_loop`, `miller_loop_lines` for fixed G2 points, `final_exponentiation`, `pairing_product_is_one` |
| `kzg`, `kzg_lines` | `decode_kzg_g1`, `verify_proof`, and precomputed lines `NEG_G2_LINES`, `TAU_G2_LINES` for the mainnet trusted setup |
| `bn254_field`, `bn254_curve`, `bn254_pairing`, `bn254` | BN254's `MODULUS`, `ORDER`, `Fp`–`Fp12` tower, `Point<F>`, `on_curve`, `multiply_g1`, `in_g2`, `Pair`, `CHUNK`, `miller_loop`, `final_exponentiation`, `twist_b`, `frobenius` |
| `secp256k1` | `P`, `N`, `Base`, `Order`, `PublicKey`, `recover` |
| `p256` | `P`, `N`, `Base`, `Order`, `verify` |

Add an immutable git revision to your ingot's `fe.toml`:

```toml
[dependencies]
curves = { source = "https://github.com/fe-lang/curves", rev = "<full commit SHA>" }
```

Byte inputs (`bls_field::Fp::from_be_bytes`, `bls_curve::decompress_g1`,
`kzg::decode_kzg_g1`) are read through `keccak::ByteSource` (implemented for
arrays, `core::MemBuffer` and `core::MemSpan`), so a caller pins the same
[keccak](https://github.com/fe-lang/keccak) revision as this ingot's `fe.toml`.
All three read 48 bytes from `offset`, which the source must hold.

```fe
use curves::bls_curve::{g1_generator, g2_generator}
use curves::bls_pairing::pairing_product_is_one
use curves::kzg::{decode_kzg_g1, verify_proof}
use curves::p256
use curves::secp256k1

let g1 = g1_generator()
let g2 = g2_generator()
// e([5]G1, G2) * e(-G1, [5]G2) = 1.
let holds = pairing_product_is_one(g1.multiply(5).to_affine(), g2.to_affine(),
                                   g1.neg().to_affine(), g2.multiply(5).to_affine())

let commitment = decode_kzg_g1(commitment_bytes, offset: 0).unwrap()
let proof = decode_kzg_g1(proof_bytes, offset: 0).unwrap()
let ok = verify_proof(commitment, z, y, proof)

let key = secp256k1::recover(hash, recovery_id: 0, r, s)
let valid = p256::verify(hash, r, s, x, y)
```

## Preconditions and behavior

- **Fields and inversion.** `Mont<M>` and `bls_field::Fp` store elements in
  Montgomery form as little-endian 64-bit limbs. `Mont::new`,
  `bn254_field::fp`, and `Fp::from_canonical` assert the input is below the
  modulus and panic otherwise; `Fp::from_montgomery` is unchecked and expects
  already-reduced Montgomery limbs. `montgomery::Field` (re-exported by
  `bn254_field::Field`) names inversion `inverse`; `bls_field::Field` names it
  `invert`. Both map zero to zero. `inversion::{invert, invert_256, invert_384}`
  require an odd modulus in signed 62-bit limbs, `inv62 = modulus^-1 mod 2^62`,
  and `0 <= x < modulus` coprime to the modulus (zero maps to zero; non-coprime
  inputs are unchecked).
- **BLS12-381 curve and pairing.** Point arithmetic, `bls_pairing`, and
  `kzg::verify_proof` do not check curve or subgroup membership on their inputs.
  `Point<Fp>::in_subgroup` and `Point<Fp2>::in_subgroup` assume the point is
  already on the curve (`on_g1(x:, y:)` / `on_g2(x:, y:)`).
  `decompress_g1(source, offset:)` validates the 48-byte compressed encoding
  and curve equation (rejecting `x = 0` without the infinity flag), but does not
  check subgroup membership; use `kzg::decode_kzg_g1` to check both (accepting
  infinity). `Point::multiply` and `Affine::multiply` take any point and any
  scalar. `multiply_g1(point, scalar)` requires `point` in G1 (its
  endomorphism is invalid off G1) and reduces `scalar` modulo `ORDER`;
  `multiply_g2(point, scalar)` requires `point` in G2 (`psi` is `[x]` only
  there) and reduces `scalar` modulo `ORDER`.
  `multiply_two(a, s, b, t)` requires `a` and `b` in G1 and scalars below
  `ORDER`. `msm(points, scalars)`
  panics unless `points.len() == scalars.len()` and every scalar is below
  2^255 (reduce modulo `ORDER`), and returns `Option::None` if bucket
  allocation fails. `miller_loop` and `miller_loop_lines` take two pairs and ignore any
  pair containing infinity; multiply `Fp12` loop outputs for more pairs, then
  call `final_exponentiation` once on a nonzero result. `map_to_g1` and
  `map_to_g2` take field elements (`Fp`, `Fp2`), not message bytes.
- **KZG.** `kzg::verify_proof(commitment:, z:, y:, proof:)` returns `false`
  when `z >= ORDER` or `y >= ORDER`, assumes `commitment` and `proof` are
  already validated G1 points (from `decode_kzg_g1`), and does not check any
  versioned hash.
- **BN254.** `bn254_curve::Point::to_affine` returns `(0, 0)` for infinity;
  `Point::add_affine(x2:, y2:)` expects a finite point.
  `bn254::multiply_g1(x:, y:, scalar)` takes the affine coordinates of a
  point of G1 and reduces `scalar % ORDER`. `bn254::in_g2(x:, y:)` takes the
  affine coordinates of a twist point already checked with
  `on_curve(x:, y:, b: twist_b())`. Neither takes infinity, which is in both
  groups.
  `bn254_pairing::miller_loop(pairs, count:)` requires `count <= 4` (`CHUNK`)
  and finite, validated G1 and G2 points in `pairs[0..count]` (filter out
  infinity pairs beforehand). `bn254_pairing::final_exponentiation` raises `f`
  to `m * (p^12 - 1) / r` for a fixed `m` coprime to `r`, so `.equals(Fp12::one())`
  checks whether a pairing product is one while non-identity values differ by
  the power `m` from `f^((p^12 - 1) / r)`. `Fp6::frobenius(k)` and
  `Fp12::frobenius(k)` panic unless `k` is in `1..=3`.
- **secp256k1 and P-256.** `secp256k1::recover(hash:, recovery_id:, r:, s:)`
  returns `Option::None` unless `recovery_id <= 1`, `0 < r < N` and `0 < s < N`
  (`recovery_id` is the y-parity `0` or `1`, not `27`/`28`), and when `r` is
  not a curve x-coordinate or the recovered point is infinity;
  `PublicKey { x, y }` holds canonical coordinates below `P`. `p256::verify(hash:, r:, s:, x:, y:)` checks all ranges
  and curve membership and returns `false` on invalid inputs without panicking.
  Both reduce `hash` modulo `N` internally and accept both `s` and `N - s`
  (callers requiring low-`s` must check `s <= N / 2`).

## Checks

- `fe test --backend native .` runs the tests. They pass at `-O 0`, `-O 1` and
  `-O 2` with Fe `6ccb3ca8e`, the compiler fevm pins.
- `python3 gen_kzg_lines.py | diff - src/kzg_lines.fe` checks the precomputed
  Miller loop lines of `-G2` and `[tau]G2` against their generator.

## License

Licensed under the [Apache License, Version 2.0](./LICENSE).
