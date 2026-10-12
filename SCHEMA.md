# OOAS: Open Operator Attestation Standard

Version 0.1-draft. Specification license CC BY 4.0.

## 1. Purpose

A signed, expiring, revocable statement by a Solana validator operator about a control it claims to operate, verifiable by anyone with the operator's public key and no account with anyone.

## 2. Scope

Operational and security controls: key custody and rotation, backup and recovery, patching, monitoring and alerting, continuity, change control, legal identity, infrastructure location.

Out of scope: leader-slot conduct (ordering, MEV, extraction). That is the Operator Register's subject, and OOAS does not duplicate it.

## 3. Attestation document

JSON, canonicalized per RFC 8785 before signing.

```json
{
  "ooas_version": "0.1",
  "subject": {
    "vote_account": "<base58>",
    "identity": "<base58>"
  },
  "claims": [
    {
      "id": "<control id from the registry, section 5>",
      "assertion": "<short statement in plain words>",
      "evidence_hash": "<sha256 hex of the supporting document, or null>",
      "issued_at": "<RFC 3339 UTC>",
      "expires_at": "<RFC 3339 UTC>",
      "attester": {
        "kind": "self | third_party",
        "pubkey": "<base58, null when self>",
        "name": "<display name, null when self>",
        "signature": "<base58 Ed25519 over the canonical claim, null when self>"
      }
    }
  ],
  "issued_at": "<RFC 3339 UTC>",
  "signature": {
    "signer": "identity | withdraw_authority",
    "pubkey": "<base58>",
    "value": "<base58 Ed25519 over the canonical document minus this field>"
  }
}
```

Rules:
- A claim with `attester.kind = "self"` is self-attested. Verifiers must label it as such and must never present it as third-party verified.
- A claim with `attester.kind = "third_party"` carries the attester's own signature over the canonical claim, plus the operator's document signature. Both must verify.
- `expires_at` is required on every claim. A claim past expiry is reported as expired, never as valid.
- The document signature is by the validator identity key or the vote account's withdraw authority. The verifier confirms the key matches on-chain.

## 4. Revocation

A signed tombstone:

```json
{
  "ooas_version": "0.1",
  "revokes": "<sha256 hex of the canonical attestation document>",
  "claim_ids": ["<id>", "..."],
  "reason": "<optional>",
  "issued_at": "<RFC 3339 UTC>",
  "signature": { "signer": "...", "pubkey": "...", "value": "..." }
}
```

An empty `claim_ids` revokes the whole document.

## 5. Control registry

Control IDs come from the Perihelion Risk and Control Matrix. Format: `<domain>.<control>`, lowercase, dot-separated, stable once published. Adding a control is a versioned change to this file.

Control IDs are stable once published. `rcm_ref` points at the NORS Risk Control Matrix, Solana Native Edition (36 controls, categories A, C, D, ELC, I, K, S). Controls marked "none" are Perihelion additions with no NORS source.

| id | rcm_ref | meaning | typical evidence | self-attestable |
|---|---|---|---|---|
| infra.location_asn | none | primary node location, ASN, hosting provider as disclosed | gossip, ASN lookup | yes, on-chain verifiable |
| infra.client_version | C-1a | validator client and version; upgrades within the staggered window | gossip version, upgrade log | yes, on-chain verifiable |
| ops.vote_performance | A-1a, A-2a | skip rate and vote credits against the service level | epoch stats export | yes, on-chain verifiable |
| ops.monitoring_alerting | I-4a, S-1a | external monitoring with alert routing to a reachable person | status page URL, probe config hash | yes |
| ops.governance_participation | none | voted or abstained in every open protocol-change round | vote tx signatures | yes, on-chain verifiable |
| ops.patch_cadence | C-1a, S-5a | OS and client patch policy; no overdue critical patch at period end | runbook hash, patch log | yes |
| ops.incident_reporting | S-1c | delegator-affecting incidents reported within the agreed window | incident report hashes | yes |
| custody.withdraw_authority_model | K-1c, K-2a, K-2b | how the withdraw authority is held (hardware, multisig, custodian) | signed statement, multisig address | no, needs third party |
| custody.key_rotation | K-1a, K-1d | identity and vote key rotation policy and last rotation date | on-chain history | yes |
| continuity.incapacity_bridge | D-1a, S-1d | named backup operator and current handoff procedure | letter of instruction hash | no |
| continuity.config_backup | K-1e, S-1e | off-node config and script backup with restore test date | manifest hash, restore log | yes |
| commercial.commission_disclosure | none | commission and MEV revenue-share settings; change notice period | on-chain config, announcement URL | yes, on-chain verifiable |
| commercial.stake_source_disclosure | none | stake sources and related-party delegation | stake account list, signed statement | no |
| legal.entity | ELC-4a | legal entity, jurisdiction, signatory; bond or insurance in force | registry document hash | no |

## 6. Publication and discovery

Published at `https://<domain>/.well-known/solana-operator-attestation.json`. The domain is bound to the vote account by an `infra.domain` claim signed by the identity key, so a verifier can go from vote account to document with no central registry.

Optional notarization: the document's sha256 may be written to a Solana Attestation Service attestation with the canonical JSON on Arweave. This is the Operator Register's pattern; adopting it means an OOAS document can be referenced from a Register record and vice versa.

## 7. Verification

Given a vote account:
1. Read identity and withdraw authority from the vote account on-chain.
2. Resolve the domain claim, fetch the document.
3. Canonicalize, verify the document signature against the on-chain key.
4. For each claim: check expiry; verify the third-party signature if present; check for a revocation tombstone.
5. Return per-claim status: `valid`, `valid_self_attested`, `expired`, `revoked`, `signature_invalid`, `unsigned`.

Never a single pass/fail score.

## 8. On-chain registry mapping

The `allowlist` program in `Digitalenergyllc/permissioned-token` stores the same semantics on-chain: a Registry per subject with one attester, Entry per claimant with `expires_at` and `revoked`. An on-chain entry corresponds to one claim with `attester.kind = "third_party"` where the attester is the registry's attester key.

## 9. Prior art

- Operator Register (Extol): conduct attestations with oracle evidence, SAS + Arweave notarization. OOAS covers controls, not conduct, and uses the same notarization pattern so the two can reference each other.
- Solana Attestation Service: the notarization primitive.
- RFC 8785 JSON Canonicalization Scheme: signing input.
