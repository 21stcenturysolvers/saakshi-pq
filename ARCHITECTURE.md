# Architecture

SAAKSHI runs entirely inside an air-gapped network. The design is complete; nothing shown here is implemented yet.

```mermaid
flowchart LR
  subgraph AG["AIR-GAPPED NETWORK: no cloud KMS, no public blockchain"]
    subgraph S["1. SENDER / AUTHORITY"]
      S1["Classified document"]
      S2["Encrypt once<br/>AES-256-GCM"]
      S3["HSM - PKCS#11<br/>document keys + per-session watermark key"]
      S4["Envelope Service<br/>ML-KEM-1024 key per recipient"]
    end
    subgraph R["2. RECIPIENT DEVICE: Secure Enclave TEE"]
      R1["Smart card / Secure element<br/>ML-DSA-87 + ML-KEM keys + session secret Ru"]
      R2["Sign PREPARE record<br/>ML-DSA-87"]
      R3["Unwrap key + Decrypt<br/>in protected memory"]
      R4["Render page - PDFium"]
      R5["Embed 3-layer watermark<br/>LOCATE + NOMINATE + CONFIRM"]
      R6["Self-verify mark<br/>fail = no release"]
      R7["Sign COMPLETE record"]
      R8["Visually identical,<br/>forensically unique copy"]
    end
    subgraph L["3. IMMUTABLE LEDGER"]
      L1["CometBFT BFT ledger<br/>4 validators, separate admins"]
      L2["WORM vault<br/>checkpoints + escrow packages"]
      L3["Index DB - PostgreSQL<br/>watermark ID to ledger record"]
    end
    subgraph E["4. ESCROW: anti-framing"]
      E1["Split Ru 3-of-5<br/>Shamir secret sharing"]
      E2["5 independent arbiters<br/>ML-KEM-encrypted shares"]
    end
    subgraph F["5. FORENSIC INVESTIGATION: enclave"]
      F1["Leaked copy<br/>screenshot / photo / scan / crop"]
      F2["Hash + open case on ledger"]
      F3["Identify document<br/>OCR + PDQ hash"]
      F4["Correct geometry<br/>perspective, rotation, scale"]
      F5["LOCATE<br/>extract 64-bit watermark ID"]
      F6["NOMINATE<br/>Tardos collusion scoring"]
      F7["Ledger lookup<br/>= decryption event"]
      F8["Warrant: 3 of 5 arbiters<br/>release Ru into enclave"]
      F9["CONFIRM<br/>detect Ru-keyed mark"]
      F10["Verify ML-DSA signatures<br/>+ ledger + checkpoint"]
      F11["Decision<br/>ATTRIBUTED / LEAD / NO ATTRIBUTION"]
      F12["Signed evidence package"]
    end
  end
  S1 --> S2
  S2 -->|one ciphertext for all| R3
  S3 --> S4
  R1 --> R2
  R2 -->|PREPARE| L1
  L1 -->|final: release key| S4
  S4 -->|ML-KEM envelope| R3
  S3 -.->|session watermark key| R5
  R3 --> R4 --> R5 --> R6 --> R7
  R7 -->|COMPLETE| L1
  R7 --> R8
  R1 -->|Ru| E1
  E1 --> E2
  E1 -.->|escrow package| L2
  L1 --> L2
  L1 --> L3
  R8 -.->|LEAK| F1
  F1 --> F2 --> F3 --> F4
  F4 --> F5
  F4 --> F6
  F5 --> F7
  F6 --> F7
  L3 --> F7
  F7 --> F8
  E2 --> F8
  F8 --> F9 --> F10
  L1 --> F10
  F10 --> F11 --> F12
```

## Zones

**1. Sender / authority.** The classified document is encrypted once with AES-256-GCM, so every recipient receives the same ciphertext. Document keys and a fresh per-session watermark key are held in an HSM (PKCS#11). The Envelope Service wraps the document key for each recipient with ML-KEM-1024, and releases it only after the ledger has finalised the recipient's signed record.

**2. Recipient device.** Keys live on a smart card or secure element and are non-exportable. Inside a secure enclave the device signs a PREPARE record, unwraps the key, decrypts in protected memory and renders the page. The three-layer watermark is embedded during rendering and self-verified; if verification fails, nothing is released. The device then signs a COMPLETE record. The resulting copy is visually identical to others but forensically unique.

**3. Immutable ledger.** A CometBFT ledger with 4 validators under separate administrators stores signed records. A write-once (WORM) vault holds ledger checkpoints and escrow packages, and a PostgreSQL index maps each watermark ID to its ledger record. The ledger stores hashes and signatures only.

**4. Escrow (anti-framing).** The recipient's session secret (Ru) is split 3-of-5 with Shamir secret sharing among 5 independent arbiters, each share encrypted with ML-KEM. No single administrator can reconstruct Ru, so no single administrator can forge the final proof against a recipient.

**5. Forensic investigation.** Inside an enclave, a leaked copy is hashed, the source document is identified, geometry is corrected, and the watermark is extracted and matched to a ledger record. The result is a signed evidence package with a decision of ATTRIBUTED, LEAD or NO ATTRIBUTION.

## Release flow

The ten steps map one-to-one to the PS 26237 end-to-end workflow.

1. **Encrypt and distribute.** The authority encrypts the document once and distributes one ciphertext to all recipients.
2. **Decrypt.** The recipient requests access; the device signs a PREPARE record with ML-DSA-87. Once the ledger has finalised it, the Envelope Service releases the ML-KEM envelope and the enclave unwraps the key and decrypts in protected memory.
3. **Watermark.** The page is rendered and the three-layer watermark (LOCATE, NOMINATE, CONFIRM) is embedded using the per-session watermark key.
4. **Sign.** The enclave self-verifies the mark (on failure nothing is released) and signs a COMPLETE record.
5. **Commit to ledger.** The COMPLETE record is committed to the BFT ledger, and the recipient's escrow package is stored in the WORM vault.
6. **Copy delivered.** The recipient views a visually identical, forensically unique copy.
7. **Extract.** After a leak, the watermark ID is extracted from the leaked copy.
8. **Match on ledger.** The watermark ID is looked up on the ledger to find the decryption event.
9. **Verify.** ML-DSA signatures, ledger proof and checkpoint are verified.
10. **Evidence record.** A signed evidence package is produced.

## Investigation flow

1. Hash the leaked copy and open a case on the ledger.
2. Identify the source document (OCR and PDQ hashing).
3. Correct perspective, rotation and scale using the keyed sync pattern.
4. LOCATE: extract the 64-bit watermark ID and look it up on the ledger to find the decryption event.
5. NOMINATE: score candidates with a Tardos code when colluding recipients are suspected.
6. Under a warrant, 3 of 5 arbiters release Ru into the enclave.
7. CONFIRM: detect the Ru-keyed mark.
8. Verify ML-DSA signatures, ledger inclusion and checkpoint.
9. Decide: ATTRIBUTED only if every check passes; otherwise LEAD or NO ATTRIBUTION.
10. Emit a signed evidence package.

## Trust boundaries

- Plaintext exists only inside the recipient enclave.
- Recipient keys are non-exportable and stay in hardware.
- The ledger stores hashes and signatures only, never documents or keys.
- Ru is never held by a single party; recovering it requires 3 of 5 arbiters and a warrant.

## Air-gap statement

The whole system runs on an isolated network. It uses no cloud KMS, no public blockchain and no external services. Cryptography follows NIST FIPS 203 and FIPS 204, with keys held in on-premise HSMs and recipient hardware.
