# pkcs12-nv

PKCS #12, specified in [RFC 7292](https://www.rfc-editor.org/rfc/rfc7292), is
the file in which a private key and the certificates that go with it are kept
under a password: the `.p12` or `.pfx` file that a browser exports, that a
Java keystore imports and that `openssl pkcs12 -export` writes. This package
reads and writes it in novo-lang, over the CMS layer of
[cms-nv](https://novo-lang.org/packages/cms-nv). It checks the file's MAC,
opens its safes, and hands out the keys and the certificates; the
certificates themselves are read by
[x509-nv](https://novo-lang.org/packages/x509-nv).

## What a PFX is

A **PFX** has a version (3), an `authSafe` and a **MAC**. The authSafe is a
CMS ContentInfo of type data whose octets are an **AuthenticatedSafe**, a
sequence of ContentInfos called **safes**. A safe is `data`, whose octets are a
SafeContents in the clear, or `encryptedData`, whose ciphertext decrypts to a
SafeContents under a password. A SafeContents is a sequence of **SafeBags**.
A bag holds one of:

| Bag | Value |
| --- | --- |
| `keyBag` | a PKCS #8 PrivateKeyInfo in the clear |
| `pkcs8ShroudedKeyBag` | a PKCS #8 EncryptedPrivateKeyInfo, encrypted under the password |
| `certBag` | a certificate: an X.509 certificate or an SDSI certificate |
| `crlBag` | a certificate revocation list |
| `secretBag` | an arbitrary secret, with a type |
| `safeContentsBag` | another SafeContents |

and has attributes: the `friendlyName`, a BMPString, and the `localKeyId`,
octets that tie a key to the certificate that goes with it.

The **MAC** is HMAC over the octets of the AuthenticatedSafe. Its key is not
the password. It is derived from the password, a salt and an iteration count
by the function of RFC 7292 appendix B, with the password written as a
**BMPString**: UTF-16 big-endian followed by two zero octets. The empty
password is the two zero octets.

Two families of password-based encryption are in the files in the world.
**PBES2** (RFC 8018) derives the key with PBKDF2 from the UTF-8 octets of the
password and encrypts with AES in CBC mode; OpenSSL 3 and every recent writer
use it. The **legacy schemes** of RFC 7292 appendix C, such as
`pbeWithSHAAnd3-KeyTripleDES-CBC` and `pbeWithSHAAnd40BitRC2-CBC`, derive the
key with the appendix B function and use TripleDES or RC2.

## Install

```
novo pkg add pkcs12-nv
```

## Example

```novo
use std.bytes
use pkcs12

// The number of certificates in a file, or the reason it could not be read.
// 100000 is the largest PBKDF2 iteration count the caller will run.
fn certificate_count(file: Bytes, password: Str) -> Str
    match pkcs12.read(file, password, 100000, true)
        Err(e)  => e.message()
        Ok(pfx) => "${list.len(pkcs12.certificates(pfx))}"
```

Build and test with:

```
novo pkg build               # type- and effect-check the package
novo test tests --isolate    # the OpenSSL files and the refusals
```

## What the package contains

| Module | Contents |
| --- | --- |
| `pkcs12` | The PFX: `read`, `write`, the MAC, the safes and bags, and the lists of certificates and private keys. |
| `p12pbe` | Password-based encryption: PBES2 read, decrypted and written; a legacy scheme named. |
| `p12kdf` | The password as a BMPString, and the key derivation of RFC 7292 appendix B. |

The DER layer, the digests and AES-CBC are cms-nv's. Errors are cms-nv's
`CmsError`. No function reads a clock, draws randomness or performs input or
output.

## How to choose an entry point

**To read a file,** call `pkcs12.read` with the file, the password, the
largest iteration count to accept, and whether a missing MAC is refused. Then
`certificates` and `private_keys`, or `bags_of_kind` for the attributes.

**To write one,** call `pkcs12.write` with the certificates, the keys, the
password and the salts, IVs and counts. A key is written as a shrouded key bag
in a data safe and the certificates in an encrypted safe, as OpenSSL 3 lays it
out.

## The rules a user needs

1. **The MAC is checked first.** A wrong password and a damaged file are both
   `CmsAuthFailed`. A PFX without a MAC is read only when `require_mac` is
   false, and then nothing authenticates the safes that are not encrypted.
2. **The password is two different byte strings.** The MAC and the legacy
   schemes take the BMPString (`p12kdf.bmp_password`); PBES2 takes the UTF-8
   octets (`p12kdf.utf8`). A password that is not well-formed UTF-8 is
   refused.
3. **The iteration counts are the writer's,** and the caller bounds them with
   `max_iterations`: a large count is a cheap way to occupy a server.
4. **A scheme that is not PBES2 is not opened.** A safe or a shrouded key bag
   that uses TripleDES or RC2 is returned with `opened` false and `algorithm`
   naming the scheme, and the ciphertext is kept. Its MAC was still checked,
   and the other safes were still read.
5. **A key and its certificate share a `localKeyId`.** Match them by it; the
   order of the bags means nothing.
6. **Nothing here draws random numbers.** The salts and the IVs are arguments
   and must be fresh for every file and every key.
7. **The values are handed out as DER.** A key is a PrivateKeyInfo and a
   certificate is a Certificate; the bytes are as the file held them.

## Algorithms

| Algorithm | Read | Written |
| --- | --- | --- |
| MAC: HMAC-SHA-1, -SHA-256, -SHA-384, -SHA-512 | yes | HMAC-SHA-256 |
| Key derivation of RFC 7292 appendix B, SHA-1, SHA-256, SHA-384, SHA-512 | yes | yes |
| PBES2: PBKDF2 with HMAC-SHA-1, -SHA-256, -SHA-512 and AES-128, -192, -256 in CBC mode | yes | PBKDF2-HMAC-SHA-256 and AES-256-CBC |
| `pbeWithSHAAnd3-KeyTripleDES-CBC` and the other legacy schemes | named, left encrypted | no |

## What is not included

- **TripleDES and RC2.** The registry has neither cipher, so the legacy
  schemes `pbeWithSHAAnd3-KeyTripleDES-CBC`,
  `pbeWithSHAAnd2-KeyTripleDES-CBC`, `pbeWithSHAAnd128BitRC2-CBC` and
  `pbeWithSHAAnd40BitRC2-CBC` are stated as not implemented. Their bags are
  read and left encrypted with the algorithm named; the key and IV they would
  use can be derived with `p12kdf.derive`.
- **Public-key privacy and integrity** (RFC 7292 sections 4.1 and 5.2), where
  the safes are enveloped to a key or signed. A safe that is not `data` or
  `encryptedData` is refused as the wrong content type.
- **Writing the legacy schemes, a key bag in the clear, a CRL or a secret
  bag,** and a MAC other than HMAC-SHA-256. They are read.
- **SDSI certificates** are read as their IA5String octets and not
  interpreted.
- **Reading the PKCS #8 key.** The value is the PrivateKeyInfo; asn1-nv's
  `asn1pkix.parse_pkcs8` reads it.
- **A clock and a random number generator.**
- **The embedded tier.** A file is read into lists of bags and decrypted with
  `Bytes`, which are heap objects, and a PBKDF2 over thousands of iterations is
  not work for a device; the package has no module that checks at
  `@tier(embedded)`.

## Tests

`tests/p12_modern.p12`, `tests/p12_3des.p12` and `tests/p12_rc2.p12` were made
on 2026-11-11 with OpenSSL 3.0.13 and the password `secret`; the commands are
in the header of `tests/pkcs12_tests.nv`. The first is what OpenSSL 3 writes:
PBES2 with PBKDF2-HMAC-SHA-256 and AES-256-CBC, 2048 iterations, and a
HMAC-SHA-256 MAC. The others are the legacy schemes under a HMAC-SHA-1 MAC;
their MACs verify, which is the check of the appendix B derivation against an
implementation that is not this one. RFC 7292 prints no example bytes. The
derivation is also checked against the published vector of the PKCS #12 test
suites and against an independent implementation, for every hash. In the
other direction, `openssl pkcs12` opens and prints the files this package
writes. `tests/pkcs12_tests.nv` holds the vectors and `tests/pkcs12_refusal_tests.nv`
the crafted refusals. `tests/coverage.sh` measures the line coverage of `src/`.

## Licence

Apache-2.0. See `LICENSE`.
