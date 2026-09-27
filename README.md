# The Nexum key log

This is the public list of every key Nexum has ever used to sign an
examination, mirrored outside the house so that a signature can be checked
without asking Nexum for anything. **A signature made before a key's
revocation date remains valid: revoking a key ends its use from that moment
on, and does not invalidate anything it signed before.**

2 keys issued, 2 live.
Canonical copy: <https://nexum.fund/keys>. Format `nexum-key-log/1.0`.

Each entry carries the key's identifier, the role it signs in (house, analyst
or reviewer), the public key in raw and PEM form, and the dates it was issued
and revoked. It does not carry who holds a key — publishing that would publish
who examined whom — nor why any key was revoked.

## Verifying an examination

You need three files from the examination: its canonical JSON, its signature
block, and its timestamp token. `/verify` on nexum.fund will do all of this
for you; these are the commands for doing it yourself.

### 1 · Recompute the digest

The digest is SHA-256 over the canonical JSON exactly as delivered — the bytes
in the file, not a re-serialisation of them.

```sh
openssl dgst -sha256 -hex NX-FAR-2026-0001.v1.canonical.json
```

It must equal the digest in the signature block, and the one published for
that reference and version at <https://nexum.fund/deliveries>.

### 2 · Verify a signature

Signatures are Ed25519 **over the 32 raw bytes the digest decodes to**, not
over its hex text. Take the signer's `publicKeyPem` from `keys.json` by its
`keyId`, and check that `issuedAt` is before the signature's `signedAt`
and that `revokedAt` is either absent or after it.

```sh
printf '%s' '<digest hex>'   | xxd -r -p    > digest.bin
printf '%s' '<signature b64>' | base64 -d   > sig.bin
printf '%s' '<publicKeyPem>'                > signer.pem

openssl pkeyutl -verify -pubin -inkey signer.pem -rawin -in digest.bin -sigfile sig.bin
```

Prints `Signature Verified Successfully`. Repeat for each of the three
signatures: the analyst's, the reviewer's and the house's.

### 3 · Verify the timestamp

The RFC 3161 token proves the digest existed at a time, which is what makes
the revocation dates above meaningful.

```sh
openssl ts -verify -digest '<digest hex>' -in NX-FAR-2026-0001.v1.tsr \
  -CAfile tsa-chain.pem
```

The chain is the timestamp authority's own, published by them and not by
Nexum. Where an OpenTimestamps proof is present, it anchors the same digest in
Bitcoin and is checked with the OTS client:

```sh
ots verify NX-FAR-2026-0001.v1.ots
```

## What verification does and does not tell you

It confirms the document is what Nexum delivered, unaltered, at that version.
It does not confirm the company's claims and is not an endorsement.
