# S4 Publication — The 831 Company

This folder is the S4 publication: a signed statement binding the
sealed S4 envelope, three independent timestamps over it, and a
public transparency-log entry. Everything here can be checked
offline with `openssl` and `sha256sum`. Nothing here requires
trusting this page — run the commands.

## The numbers

- **Published number** (the sha256 of `statement.json`; this is
  the number published on the site, in this repository, and in
  the transparency log — all three must agree):

  `197db7aae5923185c9a1f9e58bae219b32e1c1cddc9524afca2b644261928706`

- **Envelope number** (`manifest_sha256` inside the statement;
  the finalized S4 envelope in `envelope/`):

  `627810370efca3a0f449f7edc421ad95d3b751cc2e54fd33f2c451d98e3260f4`

- **Pins sha256:** `78a3dcebc1938ba1b226c504c13fc8e7965b94776bf938708daa2d7d33af3068`
- **Head root:** `33dc0158c0b0fe74450b79d8be82cfbd18dc8bc7fd30f83af77b1de09c1e3715`
- **Statement issued (UTC):** 2026-11-11T17:23:01Z

## The key

`831-publication.pub.pem` — ECDSA P-256, the S4 publication key.
Its fingerprint (sha256 of the DER encoding) is:

`1b3b8d7bb17f6e303dd3fcc30d18f5c72d23fd68981ca8b13092ac39bbbc07d5`

The same fingerprint is printed wherever this publication is
announced. If a copy of the key hashes to anything else, it is
not this key.

## Check it yourself

The statement is the published number:

```
sha256sum statement.json
```

The signature verifies against the statement, under this key:

```
openssl dgst -sha256 -verify 831-publication.pub.pem -signature statement.json.sig statement.json
```

Expected: `Verified OK`.

The key is the key its fingerprint names:

```
openssl pkey -pubin -in 831-publication.pub.pem -outform DER | sha256sum
```

Expected: `1b3b8d7bb17f6e303dd3fcc30d18f5c72d23fd68981ca8b13092ac39bbbc07d5`.

Each timestamp token binds the published number (the tokens'
message imprint is the statement's sha256) at its own time, on
its own root:

```
openssl ts -verify -in timestamps/freetsa.tsr -queryfile timestamps/freetsa.tsq -CAfile timestamps/freetsa-cacert.pem -untrusted timestamps/freetsa-tsa.crt
openssl ts -verify -in timestamps/microsoft.tsr -queryfile timestamps/microsoft.tsq -CAfile timestamps/microsoft-root.pem
openssl ts -verify -in timestamps/digicert.tsr -queryfile timestamps/digicert.tsq -CApath /etc/ssl/certs
```

Expected, each time: `Verification: OK`. FreeTSA's root is
trusted here **by pin** — the CA file in this folder is the one
the verification above uses; its own sha256 is
`2151b61137ffa86bf664691ba67e7da0b19f98c758e3d228d5d8ebf27e044438`.
Microsoft's token is checked against the Microsoft Identity
Verification Root 2020 in this folder; DigiCert's against the
public trust store (`-CApath` names it on a Debian/Ubuntu
system; use your own platform's public root store path).

## The transparency log (C2)

The same number sits in Sigstore's public Rekor log as a
hashedrekord entry — hash `197db7aa…8706`, signed by the key
above:

- Entry: https://rekor.sigstore.dev/api/v1/log/entries/108e9186e8c5677a71722d79475d68cdb1c0b599c8b671d2b6761535fe1dbe7207416127ef3825fc
- Search: https://search.sigstore.dev/?hash=sha256:197db7aae5923185c9a1f9e58bae219b32e1c1cddc9524afca2b644261928706

`rekor/entry.json` in this folder is the entry exactly as the
log returned it when it was created.

## The envelope

`envelope/` is the finalized S4 envelope, byte for byte as it
was sealed. Its own `VERIFY.md` and `MANIFEST.sha256` say how to
check it.

— The 831 Company · S4 publication, Nov-11, 2026
