# Decryption records

Status: draft for P0 review. Nothing here describes implemented software.

A decryption event is represented on the ledger by two signed records: PREPARE, committed before any key is released, and COMPLETE, committed after the copy has been rendered and marked. This document defines what each record attests and how the pair is used. Encodings, sizes and field layouts are not part of the public documents.

## PREPARE

Signed by the recipient's credential (ML-DSA-87) on the recipient device, before decryption.

Attests that:

- a specific registered recipient requested access to a specific document;
- the request was made in a specific session;
- the recipient accepts the decryption being recorded.

Carries, conceptually: the record type, an identifier for the document, an identifier for the recipient credential, a session identifier, and the signature.

Effect: once the ledger has finalised a PREPARE record, the envelope service may release the recipient's wrapped document key. Without a finalised PREPARE record, no key is released.

## COMPLETE

Signed by the recipient's credential after the page has been rendered, the watermark embedded and the self-check passed.

Attests that:

- the copy for this session was marked and the mark verified before release;
- it refers to the PREPARE record of the same session.

Effect: the watermark ID recovered from a leaked copy can be matched through the ledger index to the record pair for one decryption event.

## Lifecycle

1. The recipient device signs PREPARE and submits it to the ledger.
2. The ledger finalises PREPARE. The envelope service releases the wrapped key.
3. The device decrypts in protected memory, renders, embeds the watermark and self-verifies. If self-verification fails, nothing is released and no COMPLETE record is signed.
4. The device signs COMPLETE and submits it to the ledger.
5. The ledger finalises COMPLETE. The index maps the watermark ID to the record pair.

## Properties the records must give

- A record cannot be altered or removed without detection, because it is finalised by a ledger of four validators under separate administrators.
- A record cannot be forged by an administrator, because it carries a signature made with a key that does not leave the recipient credential.
- The ledger holds hashes and signatures only, never documents or keys.

## Open questions

These are to be settled in the decision log before the freeze.

- How a PREPARE record with no matching COMPLETE record is reported, and how long it stays open.
- Which party resubmits a record if the ledger is unreachable at the moment of signing.
- How records are revoked or superseded when a recipient credential is withdrawn.
