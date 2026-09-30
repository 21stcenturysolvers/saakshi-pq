# Architecture

This document describes the SAAKSHI design. Nothing described here is implemented yet; see [ROADMAP.md](ROADMAP.md) for the planned build order.

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

### 1. Sender / Authority

The document is encrypted once with AES-256-GCM, so one ciphertext serves every recipient. Document keys and a fresh per-session watermark key are held in an HSM (PKCS#11). An envelope service wraps the document key for each recipient with ML-KEM-1024, and releases it only after the ledger confirms the recipient's signed PREPARE record.

### 2. Recipient device (secure enclave)

The recipient's smart card or secure element holds the ML-DSA-87 and ML-KEM keys and a session secret, Ru. The device signs a PREPARE record, unwraps the key and decrypts in protected memory, renders the page, and embeds the three-layer watermark. It then self-verifies the mark (failure means no release) and signs a COMPLETE record. The result looks identical to other copies but is forensically unique.

### 3. Immutable ledger

A CometBFT ledger with 4 validators under separate administrators stores PREPARE and COMPLETE records. A write-once (WORM) vault holds ledger checkpoints and escrow packages. An index database maps a watermark ID to its ledger record.

### 4. Escrow (anti-framing)

Ru is split 3-of-5 with Shamir secret sharing. Each share is encrypted with ML-KEM to one of 5 independent arbiters. The escrow package is stored in the WORM vault. No single party, including an administrator, can reconstruct Ru.

### 5. Forensic investigation (enclave)

Investigators work inside an enclave on a leaked screenshot, photo, scan or crop. The pipeline identifies the document, corrects geometry, extracts the watermark, looks up the decryption event, and, under warrant, obtains Ru from 3 of 5 arbiters to check the CONFIRM layer. It verifies signatures, ledger inclusion and checkpoint, and outputs a decision and a signed evidence package.

## Release flow

Mapped to the PS 26237 end-to-end workflow:

1. **Encrypt and distribute.** The authority encrypts the document once and distributes the single ciphertext.
2. **Decrypt request.** The recipient's device requests the key and signs a PREPARE record with ML-DSA-87.
3. **Commit to ledger.** The PREPARE record is committed to the BFT ledger; once final, the envelope service releases the ML-KEM-wrapped key. No record, no plaintext.
4. **Decrypt.** The device unwraps the key and decrypts in protected memory.
5. **Watermark.** The page is rendered and the three-layer watermark is embedded using the per-session key; the device self-verifies the mark.
6. **Sign.** The device signs a COMPLETE record.
7. **Commit COMPLETE.** The COMPLETE record is committed to the ledger and indexed by watermark ID; Ru is escrowed 3-of-5.
8. **Copy delivered.** The recipient views a visually identical, forensically unique copy.
9. **Leak, extract and match.** When a copy leaks, the watermark ID is extracted and matched on the ledger to a decryption event.
10. **Verify and evidence record.** Signatures, ledger proof and checkpoint are verified, and a signed evidence package is produced.

## Investigation flow

1. Hash the leaked file and open a case on the ledger.
2. Identify the source document (OCR and PDQ hash).
3. Correct perspective, rotation and scale using the keyed sync pattern.
4. LOCATE: extract the 64-bit watermark ID. NOMINATE: score collusion candidates (Tardos code).
5. Look up the watermark ID on the ledger to find the decryption event.
6. Under warrant, 3 of 5 arbiters release Ru into the enclave.
7. CONFIRM: detect the Ru-keyed mark.
8. Verify ML-DSA signatures, ledger records and checkpoint.
9. Decide: ATTRIBUTED, LEAD or NO ATTRIBUTION. A decryption event is named only when every check passes.
10. Produce the signed evidence package.

## Trust boundaries

- Plaintext exists only inside the recipient enclave.
- Recipient keys are non-exportable.
- The ledger stores hashes and signatures only, never documents or keys.
- No single administrator can alter ledger records or forge evidence against a recipient.

## Air-gap

The whole system runs on an isolated network. It uses no cloud key-management service and no public blockchain.
