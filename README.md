# noble-bls12-381 ![Node CI](https://github.com/paulmillr/noble-secp256k1/workflows/Node%20CI/badge.svg) [![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)

**[Fastest](#speed)** JS implementation of BLS12-381. Auditable, secure, 0-dependency aggregated signatures & pairings.

The pairing-friendly Barreto-Lynn-Scott elliptic curve construction allows to:

- Construct [zk-SNARKs](https://z.cash/technology/zksnarks/) at the 128-bit security
- Use [threshold signatures](https://medium.com/snigirev.stepan/bls-signatures-better-than-schnorr-5a7fe30ea716),
  which allows a user to sign lots of messages with one signature and verify them swiftly in a batch,
  using Boneh-Lynn-Shacham signature scheme.

Compatible with Algorand, Chia, Dfinity, Ethereum, FIL, Zcash. Matches specs [pairing-curves-10](https://tools.ietf.org/html/draft-irtf-cfrg-pairing-friendly-curves-10), [bls-sigs-04](https://tools.ietf.org/html/draft-irtf-cfrg-bls-signature-04), [hash-to-curve-12](https://tools.ietf.org/html/draft-irtf-cfrg-hash-to-curve-12).

To learn more about internals, check out [BLS12-381 for the rest of us](https://hackmd.io/@benjaminion/bls12-381) & [key concepts of pairings](https://medium.com/@alonmuroch_65570/bls-signatures-part-2-key-concepts-of-pairings-27a8a9533d0c). To try it live, see [the online demo](https://paulmillr.com/ecc) & [threshold sigs demo](https://genthresh.com).

### This library belongs to *noble* crypto

> **noble-crypto** — high-security, easily auditable set of contained cryptographic libraries and tools.

- Just two files
- No dependencies
- Easily auditable TypeScript/JS code
- Supported in all major browsers and stable node.js versions
- All releases are signed with PGP keys
- Check out [homepage](https://paulmillr.com/noble/) & all libraries:
  [secp256k1](https://github.com/paulmillr/noble-secp256k1),
  [ed25519](https://github.com/paulmillr/noble-ed25519),
  [bls12-381](https://github.com/paulmillr/noble-bls12-381),
  [hashes](https://github.com/paulmillr/noble-hashes)

## Usage

Use NPM in node.js / browser, or include single file from
[GitHub's releases page](https://github.com/paulmillr/noble-bls12-381/releases):

> npm install @noble/bls12-381

```js
const bls = require('@noble/bls12-381');
// if you're using single file, use global variable nobleBls12381

// You can use Uint8Array, or hex string for readability
const privateKey = '67d53f170b908cabb9eb326c3c337762d59289a8fec79f7bc9254b584b73265c';
const privateKeys = [
  '18f020b98eb798752a50ed0563b079c125b0db5dd0b1060d1c1b47d4a193e1e4',
  'ed69a8c50cf8c9836be3b67c7eeff416612d45ba39a5c099d48fa668bf558c9c',
  '16ae669f3be7a2121e17d0c68c05a8f3d6bef21ec0f2315f1d7aec12484e4cf5'
];
const message = '64726e3da8';
const messages = ['d2', '0d98', '05caf3'];

(async () => {
  const publicKey = bls.getPublicKey(privateKey);
  const publicKeys = privateKeys.map(bls.getPublicKey);

  const signature = await bls.sign(message, privateKey);
  const isCorrect = await bls.verify(signature, message, publicKey);
  console.log('key', publicKey);
  console.log('signature', signature);
  console.log('is correct:', isCorrect);

  // Sign 1 msg with 3 keys
  const signatures2 = await Promise.all(privateKeys.map(p => bls.sign(message, p)));
  const aggPubKey2 = bls.aggregatePublicKeys(publicKeys);
  const aggSignature2 = bls.aggregateSignatures(signatures2);
  const isCorrect2 = await bls.verify(aggSignature2, message, aggPubKey2);
  console.log();
  console.log('signatures are', signatures2);
  console.log('merged to one signature', aggSignature2);
  console.log('is correct:', isCorrect2);

  // Sign 3 msgs with 3 keys
  const signatures3 = await Promise.all(privateKeys.map((p, i) => bls.sign(messages[i], p)));
  const aggSignature3 = bls.aggregateSignatures(signatures3);
  const isCorrect3 = await bls.verifyBatch(aggSignature3, messages, publicKeys);
  console.log();
  console.log('keys', publicKeys);
  console.log('signatures', signatures3);
  console.log('merged to one signature', aggSignature3);
  console.log('is correct:', isCorrect3);
})();
```

To use the module with [Deno](https://deno.land),
you will need [import map](https://deno.land/manual/linking_to_external_code/import_maps):

- `deno run --import-map=imports.json app.ts`

- `app.ts`

    ```typescript
    import * as bls from "https://deno.land/x/bls12_381/mod.ts";
    const publicKey = bls.getPublicKey(bls.utils.randomPrivateKey());
    console.log(publicKey);
    ```
- `imports.json`

    ```json
    {"imports": {"crypto": "https://deno.land/std@0.119.0/node/crypto.ts"}}
    ```

## API

- [`getPublicKey(privateKey)`](#getpublickeyprivatekey)
- [`sign(message, privateKey)`](#signmessage-privatekey)
- [`verify(signature, message, publicKey)`](#verifysignature-message-publickey)
- [`aggregatePublicKeys(publicKeys)`](#aggregatepublickeyspublickeys)
- [`aggregateSignatures(signatures)`](#aggregatesignaturessignatures)
- [`verifyBatch(signature, messages, publicKeys)`](#verifybatchsignature-messages-publickeys)
- [`pairing(G1Point, G2Point)`](#pairingg1point-g2point)
- [Utilities](#utilities)

##### `getPublicKey(privateKey)`
```typescript
function getPublicKey(privateKey: Uint8Array | string | bigint): Uint8Array;
```
- `privateKey: Uint8Array | string | bigint` will be used to generate public key.
  Public key is generated by executing scalar multiplication of a base Point(x, y) by a fixed
  integer. The result is another `Point(x, y)` which we will by default encode to hex Uint8Array.
- Returns `Uint8Array`: encoded publicKey for signature verification

**Note:** if you need spec-based `KeyGen`, use [paulmillr/bls12-381-keygen](https://github.com/paulmillr/bls12-381-keygen). It should work properly with ETH2 and FIL keys.

##### `sign(message, privateKey)`
```typescript
function sign(message: Uint8Array | string, privateKey: Uint8Array | string): Promise<Uint8Array>;
function sign(message: PointG2, privateKey: Uint8Array | string | bigint): Promise<PointG2>;
```
- `message: Uint8Array | string` - message which would be hashed & signed
- `privateKey: Uint8Array | string | bigint` - private key which will sign the hash
- Returns `Uint8Array | string | PointG2`: encoded signature

Check out [Utilities](#utilities) section on instructions about domain separation tag (DST).

##### `verify(signature, message, publicKey)`
```typescript
function verify(
  signature: Uint8Array | string | PointG2,
  message: Uint8Array | string | PointG2,
  publicKey: Uint8Array | string | PointG1
): Promise<boolean>
```
- `signature: Uint8Array | string` - object returned by the `sign` or `aggregateSignatures` function
- `message: Uint8Array | string` - message hash that needs to be verified
- `publicKey: Uint8Array | string` - e.g. that was generated from `privateKey` by `getPublicKey`
- Returns `Promise<boolean>`: `true` / `false` whether the signature matches hash

##### `aggregatePublicKeys(publicKeys)`
```typescript
function aggregatePublicKeys(publicKeys: (Uint8Array | string)[]): Uint8Array;
function aggregatePublicKeys(publicKeys: PointG1[]): PointG1;
```
- `publicKeys: (Uint8Array | string | PointG1)[]` - e.g. that have been generated from `privateKey` by `getPublicKey`
- Returns `Uint8Array | PointG1`: one aggregated public key which calculated from public keys

##### `aggregateSignatures(signatures)`
```typescript
function aggregateSignatures(signatures: (Uint8Array | string)[]): Uint8Array;
function aggregateSignatures(signatures: PointG2[]): PointG2;
```
- `signatures: (Uint8Array | string | PointG2)[]` - e.g. that have been generated by `sign`
- Returns `Uint8Array | PointG2`: one aggregated signature which calculated from signatures

##### `verifyBatch(signature, messages, publicKeys)`
```typescript
function verifyBatch(
  signature: Uint8Array | string | PointG2,
  messages: (Uint8Array | string | PointG2)[],
  publicKeys: (Uint8Array | string | PointG1)[]
): Promise<boolean>
```
- `signature: Uint8Array | string | PointG2` - object returned by the `aggregateSignatures` function
- `messages: (Uint8Array | string | PointG2)[]` - messages hashes that needs to be verified
- `publicKeys: (Uint8Array | string | PointG1)[]` - e.g. that were generated from `privateKeys` by `getPublicKey`
- Returns `Promise<boolean>`: `true` / `false` whether the signature matches hashes

##### `pairing(pointG1, pointG2)`
```typescript
function pairing(
  pointG1: PointG1,
  pointG2: PointG2,
  withFinalExponent: boolean = true
): Fp12
```
- `pointG1: PointG1` - simple point, `x, y` are bigints
- `pointG2: PointG2` - point over curve with complex numbers (`(x₁, x₂+i), (y₁, y₂+i)`) - pairs of bigints
- `withFinalExponent: boolean` - should the result be powered by curve order; very slow
- Returns `Fp12`: paired point over 12-degree extension field.

##### Utilities

The library uses G1 for public keys and G2 for signatures. Adding support for G1 signatures is planned.

- BLS Relies on Bilinear Pairing (expensive)
- Private Keys: 32 bytes
- Public Keys: 48 bytes: 381 bit affine x coordinate, encoded into 48 big-endian bytes.
- Signatures: 96 bytes: two 381 bit integers (affine x coordinate), encoded into two 48 big-endian byte arrays.
    - The signature is a point on the G2 subgroup, which is defined over a finite field
    with elements twice as big as the G1 curve (G2 is over Fp2 rather than Fp. Fp2 is analogous to the complex numbers).
- The 12 stands for the Embedding degree.

Formulas:

- `P = pk x G` - public keys
- `S = pk x H(m)` - signing
- `e(P, H(m)) == e(G, S)` - verification using pairings
- `e(G, S) = e(G, SUM(n)(Si)) = MUL(n)(e(G, Si))` - signature aggregation

The BLS parameters for the library are:

- `PK_IN` `G1`
- `HASH_OR_ENCODE` `true`
- `DST` `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_NUL_` - use `bls.utils.getDSTLabel()` & `bls.utils.setDSTLabel("...")` to read/change the Domain Separation Tag label
- `RAND_BITS` `64`

Filecoin uses little endian byte arrays for private keys - so ensure to reverse byte order if you'll use it with FIL.


Use `utils.bytesToHex(str)` to convert byte output to hexademical string.

```typescript
import { CURVE } from '@noble/bls12-381';
// characteristic; z + (z⁴ - z² + 1)(z - 1)²/3
CURVE.P // 0x1a0111ea397fe69a4b1ba7b6434bacd764774b84f38512bf6730d2a0f6b0f6241eabfffeb153ffffb9feffffffffaaab
CURVE.r // curve order; z⁴ − z² + 1, 0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001
curve.h // cofactor; (z - 1)²/3, 0x396c8c005555e1568c00aaab0000aaab
bls.CURVE.Gx, bls.CURVE.Gy   // G1 base point coordinates (x, y)
bls.CURVE.G2x, bls.CURVE.G2y // G2 base point coordinates (x₁, x₂+i), (y₁, y₂+i)

// Classes
bls.Fp   // field over Fp
bls.Fp2  // field over Fp₂
bls.Fp12 // finite extension field over irreducible polynominal

// projective point (xyz) at G1
class PointG1 extends ProjectivePoint<Fp> {
    constructor(x: Fp, y: Fp, z?: Fp);
    static BASE: PointG1;
    static ZERO: PointG1;
    static fromHex(bytes: Bytes): PointG1;
    static fromPrivateKey(privateKey: PrivateKey): PointG1;
    toRawBytes(isCompressed?: boolean): Uint8Array;
    toHex(isCompressed?: boolean): string;
    assertValidity(): this;
    millerLoop(P: PointG2): Fp12;
    clearCofactor(): PointG1;
}
// projective point (xyz) at G2
class PointG2 extends ProjectivePoint<Fp2> {
    constructor(x: Fp2, y: Fp2, z?: Fp2);
    static BASE: PointG2;
    static ZERO: PointG2;
    static hashToCurve(msg: Bytes): Promise<PointG2>;
    static fromSignature(hex: Bytes): PointG2;
    static fromHex(bytes: Bytes): PointG2;
    static fromPrivateKey(privateKey: PrivateKey): PointG2;
    toSignature(): Uint8Array;
    toRawBytes(isCompressed?: boolean): Uint8Array;
    toHex(isCompressed?: boolean): string;
    assertValidity(): this;
    clearCofactor(): PointG2;
}
```

## Speed

To achieve the best speed out of all JS / Python implementations, the library employs different optimizations:

- cyclotomic exponentation
- endomorphism for clearing cofactor
- pairing precomputation

Benchmarks measured with Apple M1:

```
getPublicKey x 737 ops/sec @ 1ms/op
sign x 37 ops/sec @ 26ms/op
verify x 28 ops/sec @ 34ms/op
pairing x 70 ops/sec @ 14ms/op
aggregatePublicKeys/8 x 102 ops/sec @ 9ms/op
aggregateSignatures/8 x 39 ops/sec @ 25ms/op

with compression / decompression disabled:
sign/nc x 52 ops/sec @ 18ms/op
verify/nc x 47 ops/sec @ 21ms/op
aggregatePublicKeys/32 x 1,014 ops/sec @ 986μs/op
aggregatePublicKeys/128 x 662 ops/sec @ 1ms/op
aggregatePublicKeys/512 x 278 ops/sec @ 3ms/op
aggregatePublicKeys/2048 x 84 ops/sec @ 11ms/op
aggregateSignatures/32 x 444 ops/sec @ 2ms/op
aggregateSignatures/128 x 236 ops/sec @ 4ms/op
aggregateSignatures/512 x 81 ops/sec @ 12ms/op
aggregateSignatures/2048 x 22 ops/sec @ 44ms/op
```

## Security

Noble is production-ready.

1. No public audits have been done yet. Our goal is to crowdfund the audit.
2. It was developed in a similar fashion to
[noble-secp256k1](https://github.com/paulmillr/noble-secp256k1), which **was** audited by a third-party firm.
3. It was fuzzed by [Guido Vranken's cryptofuzz](https://github.com/guidovranken/cryptofuzz),
no serious issues have been found. You can run the fuzzer by yourself to check it.

We're using built-in JS `BigInt`, which is "unsuitable for use in cryptography" as [per official spec](https://github.com/tc39/proposal-bigint#cryptography). This means that the lib is potentially vulnerable to [timing attacks](https://en.wikipedia.org/wiki/Timing_attack). But, *JIT-compiler* and *Garbage Collector* make "constant time" extremely hard to achieve in a scripting language. Which means *any other JS library doesn't use constant-time bigints*. Including bn.js or anything else. Even statically typed Rust, a language without GC, [makes it harder to achieve constant-time](https://www.chosenplaintext.ca/open-source/rust-timing-shield/security) for some cases. If your goal is absolute security, don't use any JS lib — including bindings to native ones. Use low-level libraries & languages.

We however consider infrastructure attacks like rogue NPM modules very important; that's why it's crucial to minimize the amount of 3rd-party dependencies & native bindings. If your app uses 500 dependencies, any dep could get hacked and you'll be downloading malware with every `npm install`. Our goal is to minimize this attack vector.

## Contributing

1. Clone the repository.
2. `npm install` to install build dependencies like TypeScript
3. `npm run build` to compile TypeScript code
4. `npm run test` to run jest on `test/index.ts`

Special thanks to [Roman Koblov](https://github.com/romankoblov), who have helped to improve pairing speed.

## License

MIT (c) 2019 Paul Miller [(https://paulmillr.com)](https://paulmillr.com), see LICENSE file.
