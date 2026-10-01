# The Nexum key log

This is the public list of every key Nexum has ever used to sign an
examination, mirrored outside the house so that a signature can be checked
without asking Nexum for anything. **A signature made before a key's
revocation date remains valid: revoking a key ends its use from that moment
on, and does not invalidate anything it signed before.**

The `main` branch is the house's key log; the `staging` branch holds test
keys only.

5 keys issued, 3 live.
Canonical copy: <https://nexum.fund/keys>. Format `nexum-key-log/1.0`.

Each entry carries the key's identifier, its role (the house, or an
examiner), the public key in raw and PEM form, and the dates it was issued and
revoked. An examiner signs as analyst on the examinations they produce and as
second reviewer on those they review, with the same key; which capacity a
signature was made in is stated by the examination itself, not by the key. The
log does not carry who holds a key — publishing that would publish who
examined whom — nor why any key was revoked.

## Verifying an examination

`/verify` on nexum.fund does all of this for you. These are the commands for
doing it yourself, with nothing from Nexum but the files you were given and
this key log. You need `openssl` 3.0 or later, `jq`, `xxd`, `curl` and
`git`, and, from the examination, four files named by its reference and
version: `<reference>.v<version>.canonical.json`, `.signatures.json`,
`.tsr`, and, where present, `.ots`. Each check prints `ok` or `FAILED`.

<!-- verify:begin -->
### 0 · The files, and the key log at a commit you can name

Set the reference and version of the examination you hold, and clone this key log. The commit printed is the copy you verified against; cite it.

```sh
REF=NX-FAR-2026-0001   # the reference on the examination
V=1                    # its version
git clone --quiet --branch staging https://github.com/nexum-fund/nexum-keys keylog
git -C keylog log -1 --format='key log at commit %H (%cI)'
KEYS=keylog/keys.json
```

### 1 · Integrity: recompute the digest

The digest is SHA-256 over the canonical JSON exactly as delivered — the bytes in the file, not a re-serialisation of them. It must equal the digest in the signature block, and the one published for that reference and version at <https://nexum.fund/deliveries>.

```sh
D=$(openssl dgst -sha256 -r "$REF.v$V.canonical.json" | cut -d" " -f1)
if [ "$D" = "$(jq -r .digest "$REF.v$V.signatures.json")" ]; then echo "ok      digest $D"; else echo "FAILED  the digest does not match the signature block"; fi
```

### 2 · Signatures: each one, against this log, in its slot

Signatures are Ed25519 **over the 32 raw bytes the digest decodes to**, not over its hex text.

Check the slot as well as the signature: the canonical JSON names, under `identity.people`, the key each capacity signs with — `analystKeyId`, `reviewerKeyId` and `houseKeyId` — and those names are inside the digest. Each signature's `keyId` must equal the one the record names for its slot, and the analyst's and the reviewer's must differ.

And the key must have been live when it signed: `issuedAt` before the signature's `signedAt`, and `revokedAt` absent or after it. A signature made before a key was revoked remains valid.

```sh
printf %s "$D" | xxd -r -p > digest.bin
for SLOT in analyst reviewer house; do
  K=$(jq -r ".$SLOT.keyId" "$REF.v$V.signatures.json")
  T=$(jq -r ".$SLOT.signedAt" "$REF.v$V.signatures.json")
  if [ "$K" = "$(jq -r ".identity.people.${SLOT}KeyId" "$REF.v$V.canonical.json")" ]; then echo "ok      $SLOT signs with $K, the key the record names"; else echo "FAILED  $SLOT: the record names another key"; fi
  jq -r --arg k "$K" '.keys[] | select(.keyId == $k) | .publicKeyPem' "$KEYS" > "$SLOT.pem"
  jq -r ".$SLOT.signature" "$REF.v$V.signatures.json" | base64 -d > "$SLOT.sig"
  if openssl pkeyutl -verify -pubin -inkey "$SLOT.pem" -rawin -in digest.bin -sigfile "$SLOT.sig" >/dev/null 2>&1; then echo "ok      $SLOT signature verifies"; else echo "FAILED  $SLOT signature"; fi
  jq -r --arg k "$K" --arg t "$T" '.keys[] | select(.keyId == $k) | if .issuedAt < $t and (.revokedAt == null or .revokedAt > $t) then "ok      \(.keyId) was live when it signed" else "FAILED  \(.keyId) was not live at \($t)" end' "$KEYS"
done
if [ "$(jq -r .analyst.keyId "$REF.v$V.signatures.json")" != "$(jq -r .reviewer.keyId "$REF.v$V.signatures.json")" ]; then echo "ok      analyst and reviewer keys differ"; else echo "FAILED  one key in both capacities"; fi
```

### 3 · Time: the timestamp, against the authority's own chain

The RFC 3161 token proves the digest existed at a time, which is what makes the revocation dates above meaningful. The roots are the authorities' own, fetched from them and not from Nexum: DigiCert's, and FreeTSA's, which is the fallback; the token names which one signed.

```sh
openssl ts -reply -in "$REF.v$V.tsr" -text 2>/dev/null | grep -E "Time stamp:"
curl -s https://cacerts.digicert.com/DigiCertTrustedRootG4.crt.pem >  tsa-roots.pem
curl -s https://freetsa.org/files/cacert.pem                      >> tsa-roots.pem
if openssl ts -verify -digest "$D" -in "$REF.v$V.tsr" -CAfile tsa-roots.pem >/dev/null 2>&1; then echo "ok      timestamp verifies against the authority's chain"; else echo "FAILED  timestamp"; fi
```
<!-- verify:end -->

### And the Bitcoin anchor

Where an OpenTimestamps proof is present, it anchors the same digest in
Bitcoin. The OpenTimestamps client (`pip install opentimestamps-client`)
checks it against a Bitcoin node of your own; a proof made in the last few
hours may not be anchored yet, and `ots upgrade` fetches the completed one.

```sh
ots verify -d "$(jq -r .digest "$REF.v$V.signatures.json")" "$REF.v$V.ots"
```

## What verification does and does not tell you

It confirms the document is what Nexum delivered, unaltered, at that version.
It does not confirm the company's claims and is not an endorsement.
