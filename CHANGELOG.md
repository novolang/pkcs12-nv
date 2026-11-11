# Changelog

All notable changes to pkcs12-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

PKCS #12, RFC 7292, over cms-nv.

### Added

- The PFX: the version, the authSafe, the AuthenticatedSafe of data and
  encrypted-data safes, and the MAC (HMAC-SHA-1, -SHA-256, -SHA-384 and
  -SHA-512), checked before anything is decrypted.
- The SafeBags: key, shrouded key, certificate (X.509 and SDSI), CRL, secret
  and safe-contents, with the `friendlyName` as a BMPString and the
  `localKeyId`.
- The key derivation of RFC 7292 appendix B for SHA-1, SHA-256, SHA-384 and
  SHA-512, and the password as a BMPString.
- PBES2 of RFC 8018 read with PBKDF2-HMAC-SHA-1, -SHA-256 and -SHA-512 and
  AES-128, -192 and -256 in CBC mode, and written with PBKDF2-HMAC-SHA-256 and
  AES-256-CBC.
- Writing a PFX with the layout OpenSSL 3 uses: shrouded key bags in a data
  safe, certificates in an encrypted safe, and a HMAC-SHA-256 MAC.
- Tests: three files made with OpenSSL 3.0.13 (PBES2, TripleDES and RC2),
  checked in under `tests/`; the key derivation against the published vector
  and an independent implementation; crafted refusals. Files this package
  writes are opened by OpenSSL. Measured line coverage of `src/` is 100%.

### Known

- Not implemented: TripleDES and RC2 (the registry has neither; the legacy
  schemes are named and their bags left encrypted), the public-key privacy and
  integrity modes of RFC 7292, and writing the legacy schemes.
- `cms-nv` is a path dependency (`{ path = "../cms-nv" }`) until it is on the
  registry; the publish round switches it to the range.

This release requires novo 0.24.1 or later.
