---
title: "Long-Term Validation for JSON Web Signature (LTV-JWS)"
abbrev: "LTV-JWS"
docname: "draft-miyachi-ltv-jws-latest"
category: info
submissiontype: IETF
author:
  - name: Naoto Miyachi
    organization: Lang Edge Ltd.
---

# Abstract

This specification extends JSON Web Signature (JWS, RFC 7515) and defines LTV-JWS, a lightweight long-term signature format for Long-Term Validation (LTV).

LTV-JWS adds signature extension elements, timestamps, validation information (certificates, CRLs, and OCSP responses), and archive structures as minimal extensions, thereby enabling the validity of signatures to be verified over extended periods of time. In addition, archive timestamps enable continued validation even after the obsolescence or compromise of cryptographic algorithms.

LTV-JWS preserves the simple structure and concept of JWS while progressively adding timestamps and validation information. It also enables more general-purpose signing use cases through indirect signatures using external references (refs).

# Status of This Memo

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79.

# Introduction

JSON Web Signatures (JWS) [RFC7515], a JSON-based signature format, are widely used to ensure the authenticity of target data. When data and JWS objects are stored over long periods of time, issues arise such as the expiration of signing certificates and the obsolescence or compromise of cryptographic algorithms.

As an approach to Long-Term Validation (LTV), this specification defines four signature levels: the base signature level (SIG-B), the signature timestamp level (SIG-T), the long-term validation signature level containing validation information (SIG-LTV), and the long-term archive timestamp signature level (SIG-LTA). Furthermore, the continued addition of validation information and archive timestamps enables continued verification of signature validity, even after the obsolescence or compromise of cryptographic algorithms.

Long-Term Validation for JSON Web Signature (LTV-JWS) is a JWS extension specification that defines a long-term signature format based on JWS JSON Serialization and a long-term validation approach similar to XAdES (XML Advanced Electronic Signature) for XML signatures.

In addition, this specification supports an indirect signing model using external references (refs), allowing multiple arbitrary files, including non-JSON data, to be used as indirect signature targets. This indirect signing mechanism enables JWS to be used as a more general-purpose signature.

LTV-JWS is designed for practical use over the Internet by enabling lightweight implementation and operation through the addition of only minimal structures and attributes to JWS.
In particular, signing inputs and hash inputs follow the JWS signing input model, in which BASE64URL-encoded elements are concatenated using the period "." character, thereby simplifying implementation and improving interoperability.

By using LTV-JWS, various types of JSON data and arbitrary data formats used on the Internet can be verified for authenticity over extended periods of time.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL",
"SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",
"NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
[RFC2119] [RFC8174] when, and only when, they appear in
all capitals, as shown here.

- **LTV (Long-Term Validation)**: a validation model that preserves the ability to verify the validity of a signature over extended periods of time by using timestamps, validation information, and renewable archive protection.
- **SIG-B (Signature Base level)**: The base signature level of LTV-JWS consisting of a JWS containing a protected signing certificate hash.
- **SIG-T (Signature Timestamp level)**: A signature level extending SIG-B by adding a timestamp used to provide trusted proof that the signature existed at a specific point in time.
- **SIG-LTV (Signature Long-Term Validation level)**: A signature level extending SIG-T by adding all validation information required for long-term validation of the signature.
- **SIG-LTA (Signature Long-Term Archive timestamp level)**: A signature level extending SIG-LTV by adding an archive timestamp used to preserve the entire validation state of the signature.
- **External Reference (refs)**: An indirect signing model for external data using the "refs" array containing external references with hash values. Verification of this indirect signing model requires hash validation in addition to signature verification.
- **Chained Signing**: A signing model in which one JWS includes another JWS as an external reference signing target, thereby constructing an ordered signature chain across multiple JWS objects.
- **Valid**: A validation result indicating that the signature value, hash values, and applicable PKIX validation have successfully validated the signature.
- **Invalid**: A validation result indicating that the signature has failed validation due to signature value mismatch, hash value mismatch, certificate revocation, or other validation failure.
- **Indeterminate**: A validation result indicating that the available information required for PKIX validation, such as certificates, revocation information, trust anchors, timestamps, or other validation-related information, is insufficient to determine whether the signature is Valid or Invalid.

# Overview

## Diagram

### LTV Diagram

```text
  +---------------------------------------------+
  | SIG-B (Create Base Signature)               |
  | Signature Base level                        |
  +-------------------+-------------------------+
                      |
                      v
  +---------------------------------------------+
  | SIG-T (Add "timestamp")                     |
  | Signature Timestamp level                   |
  +-------------------+-------------------------+
                      |
                      |    +------------------------------+
                      |    |                              |
                      v    v                              |
  +---------------------------------------------+         |
  | SIG-LTV (Add "validations")                 |         |
  | Signature Long-Term Validation level        |         |
  +---------------------+-----------------------+         |
                        |                                 | renew archive
                        v                                 |
  +---------------------------------------------+         |
  | SIG-LTA (Add "archive.timestamp")           |         |
  | Signature Long-Term Archive Timestamp level |         |
  +---------------------+-----------------------+         |
                        |                                 |
                        +---------------------------------+
```
Figure 1: LTV-JWS Level Flow

### External Reference Diagram

```text
  +---------------------------------------------------------+
  | signature:                                              |
  +---------------------------+-----------------------------+
                              |           +-----------------+
                              |           | protected:      |
                   (JWS sign) +---------->|   +---------+   |
                              |           |   | "ES256" |   | <= sign alg
                              |           |   +---------+   |
                              v           +-----------------+
  +---------------------------------------------------------+
  | payload:                                                |
  |      refs[0]          refs[1]          refs[2]      ... | <= refs array
  |  +-------------+  +-------------+  +-------------+      |
  |  | "data1.bin" |  | "file.json" |  | "data2.txt" |  ... | <= uri
  |  +-------------+  +-------------+  +-------------+      |
  |  |   "S256"    |  |   "S256"    |  |   "S256"    |  ... | <= hash alg
  |  +-------------+  +-------------+  +-------------+      |
  |  |   "hash1"   |  |   "hash2"   |  |   "hash3"   |  ... | <= hash value
  |  +------+------+  +------+------+  +------+------+      |
  +---------|----------------|----------------|-------------+
            | (hash1)        | (hash2)        | (hash3)
            v                v                v
     +-------------+  +-------------+  +-------------+
     |  data1.bin  |  |  file.json  |  |  data2.txt  |  ...
     +-------------+  +-------------+  +-------------+
```
Figure 2: External Reference Model

### Chained Signing Diagram

```text

  +---------------------+        +---------------------+        +-------+
  | JWS-A (LTV-JWS)     |    +-->| JWS-B (LTV-JWS)     |    +-->| JWS-C |
  | +-----------------+ |    |   | +-----------------+ |    |   |       |
  | | refs[n]         | |    |   | | refs[n]         | |    |   |       |
  | | +-------------+ | |    |   | | +-------------+ | |    |   +-------+
  | | | uri="JWS-B" +--------+   | | | uri="JWS-C" +--------+
  | | +-------------+ | | (hash) | | +-------------+ | | (hash)
  | | | type="jws"  | | |        | | | type="jws"  | | |
  | | +-------------+ | |        | | +-------------+ | |
  | +-----------------+ |        | +-----------------+ |
  +---------------------+        +---------------------+
```
Figure 3: Chained Signing Model

## Goals

The goals of LTV-JWS are as follows:

- To enable long-term validation of JWS signatures over extended periods of time.
- To preserve the simple structure and processing model of JWS while adding long-term validation capabilities.
- To enable lightweight implementation and interoperability by avoiding JSON canonicalization and by reusing the JWS signing input model.
- To support the continued verification of signatures even after certificate expiration or cryptographic algorithm obsolescence through timestamps, validation information, and archive timestamps.
- To support indirect signing of multiple arbitrary external files, including non-JSON data, through external references (refs).
- To allow validation information and archive timestamps to be incrementally added after signing without modifying the original signature value.
- To maintain compatibility with existing JOSE and JWS processing models as much as possible.
- To simplify future long-term validation by preserving validation-related information used during validation.

## Design Principles

LTV-JWS is designed based on the following principles:

- Preserve the core structure and processing model of JWS as defined in RFC 7515.
- Minimize additional structures and processing requirements in order to enable lightweight implementation and interoperability.
- LTV-JWS avoids JSON canonicalization by reusing deterministic BASE64URL-based JWS signing input and hash input construction models.
- Separate integrity-protected information from incrementally added long-term validation information by using protected and unprotected header structures appropriately.
- Enable incremental extension of validation information and archive timestamps after signing without modifying the original signature value.
- Support long-term preservation of signature validity through timestamps, validation information, and renewable archive protection.
- Support indirect signing of multiple arbitrary external files through external references (refs).
- Reuse existing JOSE and JWS algorithms, identifiers, and processing models as much as possible.
- Maintain compatibility with existing JWS JSON Serialization implementations whenever possible.
- Prioritize practical Internet deployment and operational simplicity over introducing complex processing models.

## Relationship to JWS

LTV-JWS is an extension specification for JSON Web Signature (JWS) defined in RFC 7515.

The relationship between LTV-JWS and JWS is summarized as follows:

- LTV-JWS preserves the basic JWS JSON Serialization structure consisting of protected headers, payloads, and signatures.
- LTV-JWS adds long-term validation capabilities through additional LTV-related objects.
- LTV-JWS reuses the existing JWS signing input model and JOSE algorithms without introducing new signature algorithms or new canonicalization mechanisms.
- Long-term validation information, timestamps, and archive structures are added incrementally through unprotected header extensions without modifying the original signature value.
- LTV-JWS extends the JWS payload usage model by supporting indirect signing of external data through external references (refs).
- Verification of indirect signatures requires both JWS signature validation and external reference hash validation.
- LTV-JWS is designed to maintain compatibility with existing JWS processing and implementations whenever possible.

BASE64 encoding is used for DER-encoded ASN.1 objects for compatibility with existing PKIX and RFC 7515 x5c processing models.

# Data Model

## JWS Structure

LTV-JWS is based on the JWS JSON Serialization defined in RFC 7515 and consists of protected headers, unprotected headers, payloads, and signatures.

LTV-JWS stores long-term validation related information as "ltv" objects distributed across these structures.

- The "ltv" object in the protected header contains integrity-protected information included in the JWS signature.
- The "ltv" object in the unprotected header contains timestamps, validation information, and archive information incrementally added after signing.
- The "ltv.refs" array in the payload contains external references used for indirect signing of external data.

This structure enables long-term validation information to be incrementally added without modifying the original signature value.

## Protected Header

In LTV-JWS, the protected header MUST contain the following elements:

- "alg": the signature algorithm
- "crit": an array containing "ltv"
- "ltv": the LTV extension object

By including "ltv" in the "crit" parameter, implementations that do not understand this specification MUST reject the JWS.

The "ltv" object in the protected header is included in the JWS signature input and its integrity is protected.

## Unprotected Header

In LTV-JWS, validation information and timestamps added after signing are stored in the unprotected header.

The "ltv" object in the unprotected header is not included in the JWS signature input and therefore MUST NOT be trusted independently.

Such information is indirectly protected through timestamps or archive structures.

## Payload

The payload contains the signed target data.

LTV-JWS supports the following payload models:

1. Attached payload model:
   The payload contains BASE64URL-encoded signed data.

2. Indirect signing model:
   The payload contains a JSON object including "ltv.refs" used for indirect signing of external data.

Detached payloads as defined in RFC 7515 are allowed but NOT RECOMMENDED for LTV-JWS, since long-term validation benefits from preserving payload or external reference information together with the signature.

## Signature

The signature contains the JWS signature value generated using the protected header and payload according to RFC 7515.

LTV-JWS does not define new signature algorithms and reuses existing JOSE signature algorithms.

# LTV Objects

## ltv Protected Header Object

The "ltv" object in the protected header contains integrity-protected information included in the JWS signature input. This object contains information bound to the original signature at signing time.

### "signingTime" Parameter

The "signingTime" parameter is OPTIONAL.

The "signingTime" parameter represents the local signing time asserted by the signer. The value of "signingTime" MUST be represented as an RFC 3339 date-time string.

The "signingTime" parameter is informational and does not by itself provide cryptographically protected proof of signing time. Trusted proof of signing time is generally established using timestamp structures described in this specification. However, under a policy that permits the use of trusted system time, the "signingTime" value MAY be used as trusted signing time.

### "signingCertHash" Object

The protected header "ltv" object MUST contain the "signingCertHash" object.

The "signingCertHash" object identifies the signing certificate associated with the JWS signature. This object binds the signature to a specific signing certificate by storing a hash value of the signing certificate.

The "signingCertHash" object MUST contain both "hashAlg" and "hashValue" parameters. The "signingCertHash" object contains the following parameters:

- "hashAlg": the hash algorithm identifier
- "hashValue": the hash value of the signing certificate

The "signingCertHash" object is conceptually similar to the "x5t" and "x5t#S256" header parameters defined in RFC 7515, but provides algorithm agility and mandatory certificate binding semantics for long-term validation.

Unlike standalone certificate thumbprint parameters, the "signingCertHash" object is part of the protected "ltv" object together with other signing-time related information such as "signingTime".

#### "hashAlg" Parameter

The values "S256", "S384", and "S512" are shorthand identifiers for SHA-256, SHA-384, and SHA-512 respectively.

Additional hash algorithm identifiers MAY be defined by future specifications or IANA registrations.

#### "hashValue" Parameter

The "hashValue" parameter contains the BASE64URL-encoded hash value of the DER-encoded X.509 signing certificate calculated using the hash algorithm specified by the "hashAlg" parameter.

## ltv Unprotected Header Object

### "validations" Object

The "validations" object contains all validation information required to validate the target certificate associated with the signature or timestamp.

The purpose of the "validations" object is to preserve validation-related information used during signature validation so that future validation can be performed using this object alone.

Implementations MUST preserve all validation-related information used during validation, even if such information is duplicated.

However, if a certificate does not contain a corresponding "validations" object, or if validation using the corresponding "validations" object is insufficient, implementations MAY use validation information contained in other "validations" objects. Whether insufficient validation information results in validation failure depends on the applicable validation policy.

The "validations" object contains the following parameters.

#### "certs" Parameter

The "validations" object MUST contain the "certs" parameter.

The "certs" parameter contains an array of BASE64-encoded DER X.509 certificates required to validate the target certificate.

The array MUST contain the target certificate (signing certificate or TSA certificate), intermediate CA certificates, root certificates, and other certificates required for validation.

#### "crls" Parameter

The "crls" parameter is OPTIONAL.

The "crls" parameter contains an array of BASE64-encoded DER CRLs required to validate the target certificate.

If CRLs are used during validation, implementations MUST preserve the corresponding CRLs in the "crls" parameter.

#### "ocsps" Parameter

The "ocsps" parameter is OPTIONAL.

The "ocsps" parameter contains an array of BASE64-encoded DER OCSP responses required to validate the target certificate.

If OCSP responses are used during validation, implementations MUST preserve the corresponding OCSP responses in the "ocsps" parameter.

### "signing" Parameter

The "signing" parameter contains validation information
required to validate the signing certificate used for the
JWS signature.

The "signing" parameter is used to preserve PKIX validation
information for the signing certificate.

The value of the "signing" parameter is a BASE64URL-encoded
JSON object containing a "validations" object.

The "validations" object contained in the "signing"
parameter uses the same structure as the "validations"
object defined in "### "validations" Object".

The "signing" parameter is added to the "ltv" object in
the unprotected header.

Because the "signing" parameter is stored in the
unprotected header, it is not directly protected by the
JWS signature.

However, it is protected by subsequent archive timestamps.

### "timestamp" Object

The "timestamp" object contains timestamp information used to prove the existence of signatures, validation information, or archive information at a specific point in time.

Timestamps are used to prove that target data existed prior to a specific point in time and to preserve signature validity during long-term validation.

The structure of the "timestamp" object depends on the value of the "type" parameter.

#### "type" Parameter

The "type" parameter is OPTIONAL.

The "type" parameter identifies the format of the timestamp information.

The default value of "type" is "rfc3161".

This specification defines the following timestamp type identifier:

- "rfc3161": RFC 3161 TimeStampToken

Additional timestamp type identifiers MAY be defined by future specifications or IANA registrations.

#### "tst" Parameter

The "tst" parameter is OPTIONAL.

If "type" is "rfc3161", the "timestamp" object MUST contain the "tst" parameter.

The "tst" parameter contains a BASE64-encoded RFC 3161 TimeStampToken.

#### "validations" Object

The "validations" object under the "timestamp" object contains validation information required to validate the TSA certificate associated with the timestamp.

The "validations" object uses the same structure defined in the "validations" object described above.

### "archive" Object

The "archive" object contains archive timestamp information and renewed hash information used for long-term preservation.

The "archive" object is used to protect the entire validation state required for long-term validation and to enable the continuation and extension of signature validity even after cryptographic algorithm obsolescence or compromise.

#### "timestamp" Object

The "timestamp" object contains archive timestamp information associated with the "archive" object.

Archive timestamps are used to protect the entire set of signatures, timestamps, and validation information accumulated up to that point.

If a "rehashes" object exists within the same "archive" object, the "rehashes" object is protected by the corresponding archive timestamp.

The "timestamp" object uses the same structure as the "timestamp" object defined in the "ltv Unprotected Header Object".

The exact archive timestamp hash input construction is described in the "Archive Timestamp Input Construction" section.

#### "rehashes" Parameter (refs renew digests)

The "rehashes" parameter contains renewed hash information for external references ("refs").

The purpose of the "rehashes" parameter is to enable continued verification of externally referenced data after cryptographic algorithm updates.

The value of the "rehashes" parameter is a BASE64URL-encoded JSON object containing an "ltv" object with a "refs" array.

The "ltv.refs" array within the "rehashes" parameter MUST preserve the same external references and the same ordering as the payload "ltv.refs" array.

Each element in the "ltv.refs" array MUST modify only the hash-renewal-related "hashAlg" and "hashValue" parameters.

All other parameters MUST match the corresponding element in the payload "ltv.refs" array.

The number of elements in the "ltv.refs" array within the "rehashes" parameter MUST match the number of elements in the payload "ltv.refs" array.

The "ltv.refs" array within the "rehashes" parameter MUST NOT add, remove, reorder, or modify elements other than the "hashAlg" and "hashValue" parameters.

#### "archive" (recursive Object)

The "archive" object MAY recursively contain the next generation of "archive" objects.

Recursive "archive" objects are used to preserve additional archive timestamps and renewed hash information and to continuously extend signature validity.

## ltv Payload Object

The payload "ltv" object is OPTIONAL.

If the "ltv" object exists, it contains external reference information used for indirect signing of external data.

### "refs" Array

The "refs" array contains external references used for indirect signing of external data.

Each element in the "refs" array represents one external reference object.

The order of elements in the "refs" array MUST be preserved during signature generation and validation.

#### "uri" Parameter

The "uri" parameter is REQUIRED.

The "uri" parameter contains the URI identifying the externally referenced data.

The referenced external data MUST be hashed using the algorithm specified by the corresponding "hashAlg" parameter, and the resulting hash value MUST match the corresponding "hashValue" parameter.

The URI format is application-specific.

#### "hashAlg" Parameter

The "hashAlg" parameter is REQUIRED.

The "hashAlg" parameter contains the hash algorithm identifier used to calculate the hash value of the externally referenced data.

The "hashAlg" parameter uses the same identifier system as the "hashAlg" parameter in the "signingCertHash" object defined in the "ltv Protected Header Object".

#### "hashValue" Parameter

The "hashValue" parameter is REQUIRED.

The "hashValue" parameter contains the BASE64URL-encoded hash value calculated for the externally referenced data.

The "hashValue" parameter uses the same format as the "hashValue" parameter in the "signingCertHash" object defined in the "ltv Protected Header Object".

The hash input used to calculate the "hashValue" parameter depends on the value of the corresponding "type" parameter.

#### "type" Parameter

The "type" parameter is OPTIONAL.

The "type" parameter identifies the type of externally referenced data or the signing method.

The default value is "raw".

This specification defines the following "type" identifiers.

- "raw" (default):
  The entire binary data of the externally referenced data is used directly as the hash target.

- "jws":
  Indicates that the externally referenced data is another JWS object.

  If "type=jws", only the "protected", "payload", and "signature" of the externally referenced JWS are used as the hash target, and the exact hash input construction method is defined in the "External Reference Hash Input Construction" section.

  This enables chained signing in which one JWS includes another JWS as a signing target.

  Chained signing can be used for constructing ordered signature chains of multiple JWS objects, additional signatures, or multi-stage signing processes.

# Processing

## Validation Reference Time

If an immediately enclosing trusted timestamp or archive timestamp exists, its timestamp time is used as the validation reference time.

Otherwise, the current time is used as the validation reference time.

The exact determination of trusted timestamps depends on the applicable validation policy.

## SIG-B (Signature Base level)

SIG-B is the base signature level in LTV-JWS.

SIG-B binds signed target data and the signing certificate using a JWS signature and a signing certificate hash.

SIG-B can be used both as a normal JWS signature and as an indirect signing model using external references ("refs").

External references are OPTIONAL. If no external references exist, SIG-B is processed as a normal JWS signature.

Generation and validation of SIG-B use the JWS JSON Serialization processing model defined in RFC 7515.

### External Reference Hash Construction

When external references are used, hash values for each "refs" element MUST be generated.

The generated hash value is stored in the corresponding "hashValue" parameter.

The hash input construction method depends on the value of the corresponding "type" parameter.

#### type=raw (default)

If "type=raw", the entire binary data of the externally referenced data is used directly as the hash input.

The hash value MUST be calculated using the algorithm specified by the corresponding "hashAlg" parameter.

#### type=jws (Chained Signing)

If "type=jws", the "protected", "payload", and "signature" of the externally referenced JWS are used as the hash input.

The hash input MUST be constructed by concatenating BASE64URL-encoded elements in the following order using the period "." character.

```text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature)
```
Figure 4: Chained Signing Hash Input Construction

This hash input construction method uses the same concept as the JWS Signing Input construction model defined in RFC 7515.

The hash value MUST be calculated using the algorithm specified by the corresponding "hashAlg" parameter.

### Signature Input Construction

The signature input of LTV-JWS MUST be constructed using the same method as the JWS Signing Input defined in RFC 7515.

The signature input is constructed by concatenating BASE64URL-encoded elements in the following order using the period "." character.

```text
BASE64URL(protected) || "." ||
BASE64URL(payload)
```
Figure 5: JWS Signature (SIG-B) Input Construction

The "ltv" object in the protected header is included in the signature input.

The "ltv" object in the unprotected header is not included in the signature input.

### Signature Generation

During signature generation, implementations MUST use the JWS JSON Serialization signature generation process defined in RFC 7515.

The protected header "ltv" object MUST contain at least the "signingCertHash" object.

If the payload contains the "ltv.refs" array, all external reference hash values MUST be generated before signature generation.

The generated signature value is stored in the "signature" element.

### Signature Validation

During signature validation, implementations MUST reconstruct the signature input according to RFC 7515 and perform signature validation.

The protected header "ltv" object is protected by the signature and therefore can be trusted during validation.

Implementations MUST validate the "signingCertHash" object against the signing certificate used for signature validation.

The hash value calculated from the signing certificate MUST match the "hashValue" parameter contained in the "signingCertHash" object.

If the payload contains the "ltv.refs" array, hash validation for all external references MUST be performed.

Hash values MUST be recalculated using the hash input construction method defined for the corresponding "type" parameter, and MUST match the corresponding "hashValue" parameter.

Implementations MUST perform applicable PKIX certification path validation for the signing certificate.

If no trusted outer timestamp or archive timestamp exists, and if the protected header contains a trusted "signingTime" value permitted by the applicable validation policy, the "signingTime" value MAY be used as the validation reference time.

The exact validation policy depends on the application or operational policy.

If the signature value or external reference hash value does not match, the SIG-B validation result MUST be treated as Invalid.

If required validation information for PKIX certification path validation is insufficient, the SIG-B validation result MAY be treated as Indeterminate.

The exact determination criteria for validation results depend on the application or operational policy.

## SIG-T (Signature Timestamp level)

SIG-T is a signature level extending SIG-B by adding a signature timestamp.

SIG-T provides trusted proof that the signature existed prior to a specific point in time.

SIG-T is used to prove that the signature existed before expiration or revocation of the signing certificate.

The SIG-T timestamp is generated over a hash input constructed from the JWS "protected", "payload", and "signature".

### Signature Timestamp Input Construction

The hash input for the signature timestamp MUST be constructed using the JWS "protected", "payload", and "signature".

The hash input is constructed by concatenating BASE64URL-encoded elements in the following order using the period "." character.

```text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature)
```
Figure 6: Signature Timestamp Input Construction

This hash input construction method extends the JWS Signing Input construction model defined in RFC 7515 by additionally including the signature value.

### Signature Timestamp Generation

During signature timestamp generation, implementations MUST generate the timestamp request using the hash input defined in the "Signature Timestamp Input Construction" section.

If RFC 3161 timestamps are used, the messageImprint of the timestamp request MUST use the hash value calculated from the signature timestamp hash input.

The generated RFC 3161 TimeStampToken is stored in the "ltv.timestamp.tst" parameter of the unprotected header.

If the "type" parameter of the timestamp is omitted, the default value is treated as "rfc3161".

### Signature Timestamp Validation

During signature timestamp validation, implementations MUST reconstruct the signature timestamp hash input using the method defined in the "Signature Timestamp Input Construction" section.

If RFC 3161 timestamps are used, the messageImprint of the TimeStampToken MUST match the recalculated hash value.

Implementations MUST perform signature validation of the RFC 3161 TimeStampToken.

Implementations MUST perform applicable PKIX certification path validation for the TSA certificate.

PKIX validation of the SIG-B signing certificate MUST be performed using the signature timestamp time as the validation reference time.

This allows verification that the signature existed before expiration or revocation of the signing certificate.

The exact validation policy depends on the application or operational policy.

If the signature timestamp hash value or TimeStampToken signature does not match, the SIG-T validation result MUST be treated as Invalid.

If required validation information for PKIX validation of the signing certificate or TSA certificate is insufficient, the SIG-T validation result MAY be treated as Indeterminate.

The exact determination criteria for validation results depend on the application or operational policy.

## SIG-LTV (Signature Long-Term Validation level)

SIG-LTV is a signature level that preserves validation information required for Long-Term Validation.

SIG-LTV MAY be constructed after validation of SIG-T or SIG-LTA.

SIG-LTV preserves validation information used for validation of signing certificates and timestamp TSA certificates in order to enable Long-Term Validation of signatures in the future without depending on external validation services or network access.

### Validation Information Embedding

All validation information is represented as "validations" objects.

A "validations" object contains certificates, CRLs, and OCSP responses required for PKIX validation of the associated certificate.

#### from SIG-T

When constructing SIG-LTV from SIG-T, implementations MUST preserve the validation information actually used during SIG-T validation.

The preserved validation information MUST include:

- Certificates, CRLs, and OCSP responses used for PKIX validation of the signing certificate
- Certificates, CRLs, and OCSP responses used for PKIX validation of the signature timestamp TSA certificate

Validation information related to the signing certificate MUST be stored as a BASE64URL-encoded "validations" object in the "ltv.signing" parameter of the unprotected header.

Validation information related to the signature timestamp TSA certificate MUST be stored in the "validations" object under the corresponding signature "timestamp" object.

#### from SIG-LTA

When constructing SIG-LTV from SIG-LTA, implementations MUST preserve the validation information actually used during SIG-LTA validation.

The preserved validation information MUST include:

- Certificates, CRLs, and OCSP responses used for PKIX validation of the latest archive timestamp TSA certificate

Validation information related to the latest archive timestamp TSA certificate MUST be stored in the "validations" object under the corresponding archive timestamp object.

### Validation Information Validation

During SIG-LTV validation, implementations MUST perform PKIX validation of the signing certificate and each timestamp TSA certificate using the preserved validation information.

If validation can be completed using only the preserved validation information, implementations MAY perform validation without accessing external certificates, CRLs, OCSP responses, or other validation information.

Implementations MAY additionally use external validation information if permitted by the applicable validation policy.

If required validation information is insufficient, the SIG-LTV validation result MAY be treated as Indeterminate.

If signature values, external reference hash values, or timestamp signatures are invalid, the SIG-LTV validation result MUST be treated as Invalid.

If multiple archive timestamps exist, validation may be performed from outer archive timestamps toward inner archive timestamps, or from inner archive timestamps toward outer archive timestamps, depending on implementation or validation policy.

For validation of inner signatures, timestamps, validation information, and archive structures, the timestamp time of the immediately enclosing signature timestamp or archive timestamp SHOULD be used as the validation reference time.

## SIG-LTA (Signature Long-Term Archive Timestamp level)

SIG-LTA is a signature level that continuously extends the
long-term validity of signatures by incrementally adding
archive timestamps to SIG-LTV.

When SIG-LTA is generated for the first time, the first
"archive" object is added as `header.ltv.archive`.

If an "archive" object already exists, long-term validity is
continuously preserved by recursively adding the next
"archive" object within the existing "archive" object.

A newly added "archive" object MAY contain "rehashes", which
stores renewed hash values for external references ("refs").

A newly added "archive" object also contains an archive
timestamp "timestamp" used to protect all signatures,
timestamps, validation information, and renewed hash
information accumulated up to that point.

### External Reference Hash Renewal (rehashes)

"rehashes" stores renewed hash values for external references
("refs").

If the hash algorithm used for externally referenced data
becomes obsolete or cryptographically weak, newly calculated
hash values using a new hash algorithm are added as
"rehashes".

If "rehashes" is used, it MUST be added within the same
"archive" object before generating the corresponding archive
timestamp.

### Archive Timestamp Input Construction

The hash input for archive timestamps consists of
BASE64URL-encoded elements concatenated using the "."
character.

The archive timestamp hash input is generated using the
following procedure.

1. Initialize the archive timestamp hash input as an empty
   string.

2. Append the value of JWS `protected`.

3. Append the value of `payload` prefixed with ".".

4. Append the value of JWS `signature` prefixed with ".".

5. Append the value of `header.ltv.timestamp`
   prefixed with ".".

6. Append the value of `header.ltv.signing`
   prefixed with ".".

7. If an `archive` object exists, process the contained
   elements.

8. If `rehashes` exists within the `archive` object,
   append its value prefixed with ".".

9. If `timestamp` exists within the `archive` object,
   append its value prefixed with ".".

10. If the next `archive` object exists within the current
    `archive` object, process the next `archive` object
    recursively and repeat from step 8.

11. Finally, calculate the hash value of the resulting hash
    input and use the calculated hash value as the
    `messageImprint` of the RFC 3161 timestamp request for the
    target `archive.timestamp`.

For example, the hash input for a first-generation archive
timestamp is as follows.

```text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing)
```
Figure 7: First-Generation Archive Timestamp Input

For example, the hash input for a second-generation archive
timestamp is as follows.
```text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing) || "." ||
BASE64URL(header.ltv.archive.timestamp)
```
Figure 8: Second-Generation Archive Timestamp Input

For example, the hash input for a third-generation archive
timestamp using "rehashes" is as follows.
```text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing) || "." ||
BASE64URL(header.ltv.archive.timestamp) || "." ||
BASE64URL(header.ltv.archive.archive.timestamp) || "." ||
BASE64URL(header.ltv.archive.archive.archive.rehashes)
```
Figure 9: Third-Generation Archive Timestamp Input
(rehashes included)

### Archive Timestamp Generation

An archive timestamp is generated by creating an RFC 3161
timestamp over the archive timestamp hash input.

The generated timestamp is stored as the corresponding
"archive.timestamp".

### Archive Timestamp Validation

Archive timestamp validation MUST perform SIG-B signature
validation, SIG-T timestamp validation, and validation using
`validations` before validating all generations of
`archive.timestamp`.

For each `archive.timestamp`, the corresponding archive
timestamp hash input MUST be reconstructed and verified to
match the `messageImprint` value contained in the timestamp
token.

If archive objects recursively exist, all
`archive.timestamp` values MUST be validated sequentially
from the outermost archive object toward the innermost
archive object.

### Next-Generation Archive Extension

To continuously preserve the long-term validity of signatures,
the next "archive" object is added within the existing
"archive" object.

If necessary, new "rehashes" are added and a new archive
timestamp is generated, thereby enabling continuous extension
of long-term validity.








# Security Considerations

## Trust Model of Unprotected Header Information
## Validation Requirements for External References
## Timestamp and TSA Validation
## Recursive Archive Validation
## Cryptographic Algorithm Agility
## Chained Signing Considerations
## Canonicalization and Deterministic Inputs
## Local Signing Time Considerations
## Validation Information Freshness

# IANA Considerations

## Registration of the "ltv" JOSE Header Parameter
## LTV-JWS External Reference Type Registry
## LTV-JWS Timestamp Type Registry
## Hash Algorithm Identifiers
## No New Cryptographic Algorithms

# References

## Normative References

- RFC 7515
- RFC 7518
- RFC 2119
- RFC 3161
- RFC 5280

## Informative References

TBD

# Appendix: LTV-JWS Example

## Example SIG-B
## Example SIG-T
## Example SIG-LTV
## Example SIG-LTA
## Example SIG-LTV 2nd generation
## Example SIG-LTA 2nd generation
## Example SIG-LTA refs renew digests


