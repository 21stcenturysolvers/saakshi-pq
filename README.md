# SAAKSHI

Post-quantum decryption provenance and leak attribution for multi-recipient classified documents.

Problem statement PS 26237 (Indian Navy WESEE, Ministry of Defence), Smart India Hackathon.

- Project site: https://21stcenturysolvers.github.io/saakshi-pq/
- Demo video: coming soon

## Problem

A classified document shared with many recipients decrypts to identical copies. When one copy leaks, every recipient becomes an equal suspect. Access logs can be altered by a privileged admin, and identical watermarks identify no one.

## Approach

SAAKSHI is an offline, post-quantum system that watermarks every copy at the moment of decryption and commits a signed decryption record to a tamper-evident ledger. Its three-layer watermark lets a leaked copy be traced back to its decryption event.

1. **Sign before decrypt.** The recipient signs a decryption record (ML-DSA-87); it is committed to an offline BFT ledger before any key is released. No record, no plaintext.
2. **Watermark at decryption.** The page is marked while it renders, so a clean copy never exists. Copies look identical but are forensically distinct.
3. **Trace from the leak.** Extract the watermark, look it up on the ledger, verify signature and ledger proof, and output a signed evidence package naming the decryption event.
4. **Fully air-gapped.** NIST FIPS 203/204 post-quantum cryptography, on-premise HSM, no cloud KMS, no public blockchain.

## Three-layer watermark

| Layer | Purpose |
| --- | --- |
| LOCATE | Finds the decryption event by extracting a 64-bit watermark ID. |
| NOMINATE | Resists colluding recipients (Tardos code). |
| CONFIRM | Unforgeable proof, opened only under warrant. |

Design properties (targets, not measured results):

- Framing-resistant evidence: the final proof depends on a recipient secret split among 5 independent arbiters (3 needed); no single admin can forge evidence against anyone.
- Built for real leaks: screenshots, phone photos of a full page, print-and-scan, crops and recompression. A keyed sync pattern corrects perspective, rotation and scale; an error-corrected ID is carried on every page; evidence is combined across pages and photos.
- Refuses to guess: a decryption event is named only when every check passes; otherwise the result is LEAD or NO ATTRIBUTION.

## What SAAKSHI claims / does not claim

Can establish:

- A leaked copy carries evidence of a specific decryption event, signed by a registered credential, recorded before the investigation began.

Does not claim:

- That a person intended to leak.
- That the watermark cannot be removed.
- That retyped or verbally relayed content is traceable (no visual carrier exists).

## Technology stack

| Technology | Purpose |
| --- | --- |
| ML-KEM-1024 + ML-DSA-87 (FIPS 203/204) | Post-quantum key exchange and recipient signatures |
| AES-256-GCM + SHA-384 | Encrypt once for all recipients; hash records and leaked files |
| HSM (PKCS#11) + recipient smart card | Keys stay in hardware; fresh watermark key per session |
| CometBFT ledger, 4 validators, offline | No single admin can alter or delete records (Hyperledger Fabric SmartBFT evaluated for production) |
| WORM storage | Write-once vault for ledger checkpoints and escrow packages |
| Shamir secret sharing (3-of-5) | Splits the recipient secret among 5 arbiters |
| Python + OpenCV | Watermark embedding/extraction, geometry correction, error correction |
| Tesseract OCR + PDQ hashing | Identifies which document a leaked photo came from |
| Rust | Memory-safe security-critical services |

## Status

Nothing is implemented yet. The table reflects the current state of the project.

| Area | Status |
| --- | --- |
| System design (internal specification) | Complete |
| Repository and project site | Live |
| P0 Protocol freeze | Next |
| P1 Reference path (crypto, ledger, release agent) | Planned |
| P2 Watermark engine | Planned |
| P3 Forensic pipeline and verifier | Planned |
| Demo video | Coming soon |

## Repository layout

```
.gitignore
README.md
```

## References

| Reference | Source | Year | Note |
| --- | --- | --- | --- |
| [FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard](https://csrc.nist.gov/pubs/fips/203/final) | NIST | 2024 | Standard for ML-KEM, used for post-quantum key exchange. |
| [FIPS 204: Module-Lattice-Based Digital Signature Standard](https://csrc.nist.gov/pubs/fips/204/final) | NIST | 2024 | Standard for ML-DSA, used for recipient signatures; records stay verifiable against "harvest now, decrypt later". |
| [OpenSSL 3.5 LTS release notes](https://openssl-library.org/news/openssl-3.5-notes/) | OpenSSL | 2025 | Long-term-support release with post-quantum algorithm support. |
| [StegaStamp: Invisible Hyperlinks in Physical Photographs](https://arxiv.org/abs/1904.05343) | CVPR | 2020 | Prior work on watermarks that survive being photographed. |
| [Tardos: Optimal Probabilistic Fingerprint Codes](https://doi.org/10.1145/1346330.1346335) | J. ACM | 2008 | Collusion-resistant fingerprint codes, basis of the NOMINATE layer. |
| [India's leaky submarines](https://eastasiaforum.org/2016/10/29/indias-leaky-submarines/) | East Asia Forum | 2016 | In 2016, 22,400 pages on the Indian Navy's Scorpene submarines leaked, and the source was only suspected, years later. |
| [The Hateful Eight piracy](https://variety.com/2015/film/news/hateful-eight-piracy-andrew-kosove-1201667001/) | Variety | 2015 | Example of a pre-release copy leaking from a limited distribution. |
| [2026 Cost of Insider Risks: Global](https://ponemonsullivanreport.com/2026/05/2026-cost-of-insider-risks-global/) | Ponemon Institute | 2026 | Insider risk costs organisations US$19.5 M a year on average. |
| [Cost of org-level data breach in India](https://cxotoday.com/cybersecurity/cost-of-org-level-data-breach-touches-its-highest-level-in-india-at-rs-25-5-crore-ibm/) | CXOToday, reporting IBM | 2026 | An average data breach in India costs Rs 25.5 crore. |

## Licence

All rights reserved.
