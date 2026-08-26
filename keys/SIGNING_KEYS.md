# AetherAC signing keys

Two keys, with different jobs. Confusing them is the mistake this document exists to prevent.

## The identity key

```
ed25519/FC2581BC9A0BBCF7  [C]  certify only
CF33 9EF6 3B6D 6EBE 0053  0460 FC25 81BC 9A0B BCF7
uid  AetherAC <contact@abnt.it>
└─ ssb ed25519/1EE08A7DA4925B06  [S]  expires 2027-08-26
```

The primary key certifies and nothing else — it cannot sign data, which is deliberate: it is stored offline and
exists only to vouch for the signing subkey and for the model-signing key below. The subkey does the signing and
expires yearly, so a compromise is contained to one subkey and one year rather than to the project's identity.

This key is not anybody's personal key. That matters for three reasons that all show up later: a maintainer can
be added without sharing a personal identity, the key can be rotated without invalidating anyone's commit
signatures, and a leak revokes one project key rather than a person.

Public key: [`aetherac-identity.asc`](aetherac-identity.asc)

## The model-signing key

```
Ed25519 raw, 32 bytes
base64       79JYUo11ssRP6Xob/6gKp5VQWTUZc1TvNwB23j7LUOI=
fingerprint  EFD2:5852:8D75:B2C4:4FE9:7A1B:FFA8:0AA7:9550:5935:1973:54EF:3700:76DE:3ECB:50E2
```

Model bundles are signed with this key, and AetherAC verifies them against a copy embedded in its jar.

It is raw Ed25519 rather than OpenPGP because the JDK verifies Ed25519 natively. A 657 KB plugin would otherwise
have to carry a two-megabyte OpenPGP implementation to check one signature, for a feature that is off by default.

Public key: [`model-signing.pub.raw`](model-signing.pub.raw) — endorsed by the identity key in
[`model-signing.pub.raw.asc`](model-signing.pub.raw.asc)

## Why the copy in the jar is the one that verifies

AetherAC verifies model signatures against the key **inside its own jar**, never against this page. Fetching a
trust anchor over the network would let whoever controls that network hand the runtime a different anchor, at
which point a valid signature proves only that the attacker signed the model. The plan also requires a model to
be verified *before* it loads, which no round-trip can guarantee on a server with no outbound access.

This page exists so the two can be **compared**. That is the linkage:

```sh
# What the jar will trust
unzip -p aether-paper-mc<version>.jar \
  dev/aetherac/ml/runtime/signing/model-signing-ed25519.pub | base64

# What AetherAC published
curl -s https://raw.githubusercontent.com/AetherAC/.github/main/keys/model-signing.pub.raw | base64
```

Equal means the jar carries the key AetherAC published. **Different means the jar did not come from AetherAC**,
and no amount of valid-looking signatures inside it changes that conclusion.

## Verifying the whole chain

```sh
# 1. The identity key signed the model-signing key
gpg --import aetherac-identity.asc
gpg --verify model-signing.pub.raw.asc model-signing.pub.raw

# 2. The jar carries that same model-signing key
unzip -p aether-paper-mc<version>.jar \
  dev/aetherac/ml/runtime/signing/model-signing-ed25519.pub | sha256sum
sha256sum model-signing.pub.raw

# 3. AetherAC itself checks each model against the embedded key at registration.
#    /aether ai models reports signatureValid per model.
```

## Scope of a compromise

If the model-signing key leaked, an attacker could sign a model AetherAC would accept. The blast radius is
bounded by what a model is allowed to do, and the plan bounds it deliberately: inference is **off by default**,
**shadow-only** when enabled, capped at `HEURISTIC`, and **cannot punish a player on its own**. So a forged
model can produce misleading advisory signals; it cannot ban anybody. Core detection — movement, combat, world,
inventory, protocol — does not involve a model at all and is unaffected.

Report a suspected key compromise through the private channel in [`SECURITY.md`](SECURITY.md), not a public
issue.
