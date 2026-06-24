---
title: "Long-Term Validation for JSON Web Signature (LTV-JWS)"
abbrev: "LTV-JWS"
docname: "draft-miyachi-ltv-jws-00"
category: info
ipr: trust200902
submissiontype: IETF
author:
  - name: Naoto Miyachi
    organization: LangEdge, Inc.
---

--- abstract

This specification extends JSON Web Signature (JWS, {{!RFC7515}}) and defines LTV-JWS, a lightweight long-term signature format for Long-Term Validation (LTV).

LTV-JWS adds signature extension elements, timestamps, validation information (certificates, CRLs, and OCSP responses), and archive structures as minimal extensions, thereby enabling the validity of signatures to be verified over extended periods of time. In addition, archive timestamps enable continued validation even after the obsolescence or compromise of cryptographic algorithms.

LTV-JWS preserves the simple structure and concept of JWS while progressively adding timestamps and validation information. It also enables more general-purpose signing use cases through indirect signatures using external references (refs).

--- middle

# Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

# Introduction

JSON Web Signatures (JWS) {{!RFC7515}}, a JSON-based signature format, are widely used to ensure the authenticity of target data. When data and JWS objects are stored over long periods of time, issues arise such as the expiration of signing certificates and the obsolescence or compromise of cryptographic algorithms.

As an approach to Long-Term Validation (LTV), this specification defines four signature levels: the base signature level (SIG-B), the signature timestamp level (SIG-T), the long-term validation signature level containing validation information (SIG-LTV), and the long-term archive timestamp signature level (SIG-LTA). Furthermore, the continued addition of validation information and archive timestamps enables continued verification of signature validity, even after the obsolescence or compromise of cryptographic algorithms.

Long-Term Validation for JSON Web Signature (LTV-JWS) is a JWS extension specification that defines a long-term signature format based on JWS JSON Serialization and a long-term validation approach similar to XAdES [ISO14533-2] for XML signatures.

In addition, this specification supports an indirect signing model using external references (refs), allowing multiple arbitrary files, including non-JSON data, to be used as indirect signature targets. This indirect signing mechanism enables JWS to be used as a more general-purpose signature.

LTV-JWS is designed for practical use over the Internet by enabling lightweight implementation and operation through the addition of only minimal structures and attributes to JWS.
In particular, signing inputs and hash inputs follow the JWS signing input model, in which BASE64URL-encoded elements are concatenated using the period "." character, thereby simplifying implementation and improving interoperability.

By using LTV-JWS, various types of JSON data and arbitrary data formats used on the Internet can be verified for authenticity over extended periods of time.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL",
"SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",
"NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
{{!RFC2119}} {{!RFC8174}} when, and only when, they appear in
all capitals, as shown here.

- **LTV (Long-Term Validation)**: a validation model that preserves the ability to verify the validity of a signature over extended periods of time by using timestamps, validation information, and renewable archive protection.
- **SIG-B (Signature Base level)**: The base signature level of LTV-JWS consisting of a JWS containing a protected signing certificate hash.
- **SIG-T (Signature Timestamp level)**: A signature level extending SIG-B by adding a signature timestamp used to provide trusted proof that the signature existed at a specific point in time.
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

~~~ ascii-art
  +---------------------------------------------+
  | SIG-B (Create Base Signature)               |
  | Signature Base level                        |
  +-------------------+-------------------------+
                      |
                      v
  +---------------------------------------------+
  | SIG-T (Add "signature timestamp")           |
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
  | SIG-LTA (Add "archive timestamp")           |         |
  | Signature Long-Term Archive Timestamp level |         |
  +---------------------+-----------------------+         |
                        |                                 |
                        +---------------------------------+
~~~
Figure 1: LTV-JWS Level Flow

### External Reference Diagram

~~~ ascii-art
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
~~~
Figure 2: External Reference Model

### Chained Signing Diagram

~~~ ascii-art
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
~~~
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

- Preserve the core structure and processing model of JWS as defined in {{!RFC7515}}.
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

LTV-JWS is an extension specification for JSON Web Signature (JWS) defined in {{!RFC7515}}.

The relationship between LTV-JWS and JWS is summarized as follows:

- LTV-JWS preserves the basic JWS JSON Serialization structure consisting of protected headers, payloads, and signatures.
- LTV-JWS adds long-term validation capabilities through additional LTV-related objects.
- LTV-JWS reuses the existing JWS signing input model and JOSE algorithms without introducing new signature algorithms or new canonicalization mechanisms.
- Long-term validation information, timestamps, and archive structures are added incrementally through unprotected header extensions without modifying the original signature value.
- LTV-JWS extends the JWS payload usage model by supporting indirect signing of external data through external references (refs).
- Verification of indirect signatures requires both JWS signature validation and external reference hash validation.
- LTV-JWS is designed to maintain compatibility with existing JWS processing and implementations whenever possible.

BASE64 encoding is used for DER-encoded ASN.1 objects for compatibility with existing PKIX and {{!RFC7515}} x5c processing models.

LTV-JWS uses both BASE64URL encoding and BASE64 encoding depending on the type of data being represented.

Values participating in JWS signing input construction, hash input construction, external reference hash processing, and hash-based identifiers use BASE64URL encoding consistent with {{!RFC7515}} JWS processing and JOSE thumbprint conventions.

DER-encoded PKIX and CMS related binary objects, including certificates, CRLs, OCSP responses, and {{!RFC3161}} TimeStampTokens, use BASE64 encoding consistent with the "x5c" header parameter defined in {{!RFC7515}}.

# Data Model

## JWS Structure

LTV-JWS is based on the JWS JSON Serialization defined in {{!RFC7515}} and consists of protected headers, unprotected headers, payloads, and signatures.

LTV-JWS stores long-term validation related information as "ltv" objects distributed across these structures.

- The "ltv" object in the protected header contains integrity-protected information included in the JWS signature.
- The "ltv" object in the unprotected header contains timestamps, validation information, and archive information incrementally added after signing.
- The "ltv.refs" array in the payload contains external references used for indirect signing of external data.

This structure enables long-term validation information to be incrementally added without modifying the original signature value.

LTV-JWS supports both Flattened JWS JSON Serialization and General JWS JSON Serialization defined in {{!RFC7515}}.

If the payload contains the "ltv.refs" array, all signatures associated with the same payload MUST be processed as LTV-JWS signatures.

In such cases, all corresponding protected headers MUST contain the "ltv" Header Parameter and MUST include "ltv" in the "crit" Header Parameter.

This requirement prevents inconsistent interpretation of external reference semantics among signatures sharing the same payload.

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

Detached payloads as defined in {{!RFC7515}} are allowed but NOT RECOMMENDED for LTV-JWS, since long-term validation benefits from preserving payload or external reference information together with the signature.

If detached payloads are used, implementations and operational environments SHOULD ensure long-term availability and consistent preservation of the detached payload data associated with the signature.

Loss, modification, or ambiguity of detached payload data may result in failure of signature validation, timestamp validation, archive validation, or long-term validation.

If the payload contains the "ltv.refs" array, the payload itself still participates in JWS signature input construction.

Externally referenced data identified by "ltv.refs" is validated separately through external reference hash validation.

## Signature

The signature contains the JWS signature value generated using the protected header and payload according to {{!RFC7515}}.

LTV-JWS does not define new signature algorithms and reuses existing JOSE signature algorithms.

# LTV Objects

## ltv Protected Header Object

The "ltv" object in the protected header contains integrity-protected information included in the JWS signature input. This object contains information bound to the original signature at signing time.

### "signingTime" Parameter

The "signingTime" parameter is OPTIONAL.

The "signingTime" parameter represents the local signing time asserted by the signer. The value of "signingTime" MUST be represented as an {{!RFC3339}} date-time string.

The "signingTime" parameter is informational and does not by itself provide cryptographically protected proof of signing time. Trusted proof of signing time is generally established using timestamp structures described in this specification. However, under a policy that permits the use of trusted system time, the "signingTime" value MAY be used as trusted signing time.

### "signingCertHash" Object

The protected header "ltv" object MUST contain the "signingCertHash" object.

The "signingCertHash" object identifies the signing certificate associated with the JWS signature. This object binds the signature to a specific signing certificate by storing a hash value of the signing certificate.

The "signingCertHash" object MUST contain both "hashAlg" and "hashValue" parameters. The "signingCertHash" object contains the following parameters:

- "hashAlg": the hash algorithm identifier
- "hashValue": the hash value of the signing certificate

The "signingCertHash" object is conceptually similar to the "x5t" and "x5t#S256" header parameters defined in {{!RFC7515}}, but provides algorithm agility and mandatory certificate binding semantics for long-term validation.

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

The array MUST contain the target certificate (signing certificate or TSA certificate), certificates forming the certification path, and any additional certificates required for validation, such as certificates required for validation of OCSP responses or CRLs.

#### "crls" Parameter

The "crls" parameter is OPTIONAL.

The "crls" parameter contains an array of BASE64-encoded DER CRLs required to validate the target certificate.

If CRLs are used during validation, implementations MUST preserve the corresponding CRLs in the "crls" parameter.

#### "ocsps" Parameter

The "ocsps" parameter is OPTIONAL.

The "ocsps" parameter contains an array of BASE64-encoded DER OCSP responses required to validate the target certificate.

If OCSP responses are used during validation, implementations MUST preserve the corresponding OCSP responses in the "ocsps" parameter.

### "signing" Parameter

The value of the "signing" parameter is a BASE64URL-encoded JSON object containing validation information required to validate the signing certificate used for the JWS signature.

The "signing" parameter is used to preserve PKIX validation information for the signing certificate.

The decoded JSON object contained in the "signing" parameter contains a "validations" object.

The "validations" object contained in the "signing" parameter uses the same structure as the "validations"
object defined in "validations" Object.

The "signing" parameter is added to the "ltv" object in the unprotected header.

Because the "signing" parameter is stored in the unprotected header, it is not directly protected by the JWS signature.

However, it can later be protected by archive timestamps.

The outer JSON object of the "signing" parameter is BASE64URL-encoded because it participates in archive timestamp hash input construction.

This avoids ambiguity caused by JSON reserialization and enables deterministic reconstruction of archive timestamp hash inputs without requiring JSON canonicalization.

However, DER-encoded PKIX-related binary objects contained within the embedded "validations" object use BASE64 encoding.

### "timestamp" Parameter

The value of the "timestamp" parameter is a BASE64URL-encoded JSON object containing timestamp information used to prove the existence of signatures, validation information, or archive information at a specific point in time.

Timestamps are used to prove that target data existed prior to a specific point in time and to preserve signature validity during long-term validation.

The structure of the decoded JSON object contained in the "timestamp" parameter depends on the value of the "type" parameter.

#### "type" Parameter

The "type" parameter is OPTIONAL.

The "type" parameter identifies the format of the timestamp information.

The default value of "type" is "rfc3161".

This specification defines the following timestamp type identifier:

- "rfc3161": {{!RFC3161}} TimeStampToken

Additional timestamp type identifiers MAY be defined by future specifications or IANA registrations.

#### "tst" Parameter

The "tst" parameter is OPTIONAL.

If "type" is "rfc3161", the decoded JSON object contained in the "timestamp" parameter MUST contain the "tst" parameter.

The "tst" parameter contains a BASE64-encoded {{!RFC3161}} TimeStampToken.

#### "validations" Object

The "validations" object within the decoded JSON object contained in the "timestamp" parameter contains validation information required to validate the TSA certificate associated with the timestamp.

The "validations" object uses the same structure defined in the "validations" object described above.

### "archive" Object

The "archive" object contains archive timestamp information and renewed hash information used for long-term preservation.

The "archive" object is used to protect the entire validation state required for long-term validation and to enable the continuation and extension of signature validity even after cryptographic algorithm obsolescence or compromise.

#### "timestamp" Parameter

The value of the "timestamp" parameter is a BASE64URL-encoded JSON object containing archive timestamp information associated with the "archive" object.

Archive timestamps are used to protect the entire set of signatures, timestamps, and validation information accumulated up to that point.

If a "rehashes" object exists within the same "archive" object, the "rehashes" object is protected by the corresponding archive timestamp.

The "timestamp" parameter uses the same structure as the "timestamp" parameter defined in the "ltv Unprotected Header Object".

The exact archive timestamp hash input construction is described in the "Archive Timestamp Input Construction" section.

#### "rehashes" Parameter (refs renew hashes)

The value of the "rehashes" parameter is a BASE64URL-encoded JSON object containing renewed hash information for external references ("refs").

The purpose of the "rehashes" parameter is to enable continued verification of externally referenced data after cryptographic algorithm updates.

The decoded JSON object contained in the "rehashes" parameter contains an "ltv" object with a "refs" array.

The "ltv.refs" array within the decoded JSON object contained in the "rehashes" parameter MUST preserve the same external references and the same ordering as the payload "ltv.refs" array.

Each element in the "ltv.refs" array MUST modify only the hash-renewal-related "hashAlg" and "hashValue" parameters.

All other parameters MUST match the corresponding element in the payload "ltv.refs" array.

The "ltv.refs" array within the decoded JSON object contained in the "rehashes" parameter MUST NOT add, remove, reorder, or modify elements other than the "hashAlg" and "hashValue" parameters.

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

# Creation Processing

## Signature Input and Timestamp Hash Inputs

LTV-JWS constructs signature inputs and timestamp hash inputs by concatenating BASE64URL-encoded values using the "." character.

The following simplified LTV-JWS structure illustrates how signature inputs and timestamp hash inputs are constructed across each signature level.

The values "AAAAAA" and similar values represent BASE64URL-encoded element values.

~~~json
{
  "payload": "AAAAAA",
  "protected": "BBBBBB",
  "signature": "CCCCCC",
  "header":
  {
    "ltv":
    {
      "timestamp": "DDDDDD",
      "signing": "EEEEEE",
      "archive":
      {
        "timestamp": "FFFFFF",
        "archive":
        {
          "rehashes": "GGGGGG",
          "timestamp": "HHHHHH"
        }
      }
    }
  }
}
~~~
Figure 4: Simplified LTV-JWS Structure

Where:

* "DDDDDD" is the signature timestamp with validations
* "EEEEEE" contains validations for the signing certificate
* "FFFFFF" is the 1st archive timestamp with validations
* "GGGGGG" contains renewal hashes for refs
* "HHHHHH" is the 2nd archive timestamp (with validations)

The following table shows the corresponding signature and timestamp hash inputs.

| Signature Level | Processing | Input Construction |
|---|---|---|
| SIG-B | Signature (JWS) | Signature over `"BBBBBB.AAAAAA"` |
| SIG-T | Signature Timestamp | Hash of `"BBBBBB.AAAAAA.CCCCCC"` |
| SIG-LTA (1st) | 1st Archive Timestamp | Hash of `"BBBBBB.AAAAAA.CCCCCC.DDDDDD.EEEEEE"` |
| SIG-LTA (2nd) | 2nd Archive Timestamp | Hash of `"BBBBBB.AAAAAA.CCCCCC.DDDDDD.EEEEEE.FFFFFF.GGGGGG"` |

Table 1: Signature Input and Timestamp Hash Input Construction

For timestamp generation, the resulting concatenated string is hashed
and used as the {{!RFC3161}} messageImprint value.

The strings above represent the exact concatenated BASE64URL-encoded values used as signature input or timestamp hash inputs.

## SIG-B (Signature Base level) Creation Processing

SIG-B is the initial signature level of LTV-JWS and represents the signature-only extension level of JWS.

SIG-B binds signed target data and the signing certificate using a JWS signature and a signing certificate hash.

SIG-B can be used both as a normal JWS signature and as an indirect signing model using external references ("refs").

External references are OPTIONAL. If no external references exist, SIG-B is processed as a normal JWS signature.

### Payload External References (ltv.refs) Creation

When external references are used, hash values for each "refs" element MUST be generated.

The generated hash value is stored in the corresponding "hashValue" parameter.

The hash input construction method depends on the value of the corresponding "type" parameter.

#### type=raw (default)

If "type=raw", the entire binary data of the externally referenced data is used directly as the hash input.

The hash value MUST be calculated using the algorithm specified by the corresponding "hashAlg" parameter.

#### type=jws (Chained Signing)

If "type=jws", the "protected", "payload", and "signature" of the externally referenced JWS are used as the hash input.

The hash input MUST be constructed by concatenating BASE64URL-encoded elements in the following order using the period "." character.

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature)
~~~
Figure 5: Chained Signing Hash Input Construction

This hash input construction method uses the same concept as the JWS Signing Input construction model defined in {{!RFC7515}}.

The hash value MUST be calculated using the algorithm specified by the corresponding "hashAlg" parameter.

### Protected Header ltv Object Creation

The protected header "ltv" object contains integrity-protected signing-related information associated with the JWS signature.

The protected header "ltv" object MUST contain the "signingCertHash" object.

The "signingCertHash" object binds the JWS signature to the corresponding signing certificate by storing a hash value of the signing certificate.

Implementations MAY optionally add the "signingTime" parameter as the signing time asserted by the signer.

The protected header "ltv" object is included in the JWS signature input and therefore its integrity is protected by the JWS signature.

### Signature Creation

The signature input of LTV-JWS MUST be constructed using the same method as the JWS Signing Input defined in {{!RFC7515}}.

The signature input is constructed by concatenating BASE64URL-encoded elements in the following order using the period "." character.

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload)
~~~
Figure 6: JWS Signature (SIG-B) Input Construction

During signature generation, implementations MUST use the JWS JSON Serialization signature generation process defined in {{!RFC7515}}.

The protected header "ltv" object MUST contain at least the "signingCertHash" object.

If the payload contains the "ltv.refs" array, all external reference hash values MUST be generated before signature generation.

The generated signature value is stored in the "signature" element.

Implementations SHOULD preserve the signing certificate when constructing SIG-B objects in order to simplify future validation processing and support offline validation and long-term preservation.

The signing certificate or certificate chain MAY be preserved within the optional "header.ltv.signing" parameter or by using the existing JWS "x5c" header parameter defined in {{!RFC7515}}.

When used in SIG-B, the optional "header.ltv.signing" parameter MAY contain only the signing certificate or certificate chain without additional revocation information such as CRLs or OCSP responses.

If the "x5c" parameter is used, it MAY be included in either the protected or unprotected header according to {{!RFC7515}}.

The "header.ltv.signing" parameter is stored in the unprotected header and therefore is not directly protected by the JWS signature. However, it can later be protected by archive timestamps.

## SIG-T (Signature Timestamp level) Creation Processing

SIG-T is the signature level that adds a signature timestamp to SIG-B.

SIG-T provides trusted proof that the signature existed prior to a specific point in time.

SIG-T is used to prove that the signature existed before expiration or revocation of the signing certificate.

The SIG-T signature timestamp is generated over a hash input constructed from the JWS "protected", "payload", and "signature".

### Signature Timestamp Embedding

The hash input for the signature timestamp MUST be constructed using the JWS "protected", "payload", and "signature".

The hash input is constructed by concatenating BASE64URL-encoded elements in the following order using the period "." character.

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature)
~~~
Figure 7: Signature Timestamp Input Construction

This hash input construction method extends the JWS Signing Input construction model defined in {{!RFC7515}} by additionally including the signature value.

During signature timestamp generation, implementations MUST generate the timestamp request using the hash input defined in the "Signature Timestamp Input Construction" section.

If {{!RFC3161}} timestamps are used, the messageImprint of the timestamp request MUST use the hash value calculated from the signature timestamp hash input.

The generated {{!RFC3161}} TimeStampToken is stored in the "tst" parameter within the decoded JSON object contained in "ltv.timestamp" in the unprotected header.

If the "type" parameter of the timestamp is omitted, the default value is treated as "rfc3161".

## SIG-LTV (Signature Long-Term Validation level) Creation Processing

SIG-LTV is the signature level that validates SIG-T or SIG-LTA and adds validation information.

SIG-LTV MAY be constructed after validation of SIG-T or SIG-LTA.

SIG-LTV preserves validation information used for validation of signing certificates and all timestamp TSA certificates in order to enable Long-Term Validation of signatures in the future without depending on external validation services or network access.

#### from SIG-T Validation Information Embedding

When constructing SIG-LTV from SIG-T, implementations MUST preserve the validation information actually used during SIG-T validation.

The preserved validation information MUST include:

- Certificates, CRLs, and OCSP responses used for PKIX validation of the signing certificate
- Certificates, CRLs, and OCSP responses used for PKIX validation of the signature timestamp TSA certificate

Validation information related to the signing certificate MUST be stored as a BASE64URL-encoded "validations" object in the "ltv.signing" parameter of the unprotected header.

Validation information related to the signature timestamp TSA certificate MUST be stored in the "validations" object within the decoded JSON object contained in the corresponding signature "timestamp" parameter.

#### from SIG-LTA Validation Information Embedding

When constructing SIG-LTV from SIG-LTA, implementations MUST preserve the validation information actually used during SIG-LTA validation.

The preserved validation information MUST include:

- Certificates, CRLs, and OCSP responses used for PKIX validation of the latest archive timestamp TSA certificate

Validation information related to the latest archive timestamp TSA certificate MUST be stored in the "validations" object within the decoded JSON object contained in the corresponding archive "timestamp" parameter.

## SIG-LTA (Signature Long-Term Archive Timestamp level) Creation Processing

SIG-LTA is the signature level that adds an archive timestamp to SIG-LTV.

When SIG-LTA is generated for the first time, the first "archive" object is added as `header.ltv.archive`.

If an "archive" object already exists, long-term validity is continuously preserved by recursively adding the next "archive" object within the existing "archive" object.

A newly added "archive" object MAY contain "rehashes", which stores renewed hash values for external references ("refs").

A newly added "archive" object also contains an archive timestamp "timestamp" parameter used to protect information contained within the existing archive structure.

### Archive Timestamp Embedding

The hash input for archive timestamps consists of BASE64URL-encoded elements concatenated using the "." character.

The archive timestamp hash input is generated using the following procedure.

1. Initialize the archive timestamp hash input as an empty string.

2. Append the value of JWS `protected`.

3. Append the value of `payload` prefixed with ".".

4. Append the value of JWS `signature` prefixed with ".".

5. Append the value of `header.ltv.timestamp` prefixed with ".".

6. Append the value of `header.ltv.signing` prefixed with ".".

7. If an `archive` object exists, process the contained elements.

8. If `rehashes` exists within the `archive` object,
   append its value prefixed with ".".

9. If `timestamp` exists within the `archive` object, append its value prefixed with ".".

10. If the next `archive` object exists within the current `archive` object, process the next `archive` object recursively and repeat from step 8.

11. Finally, calculate the hash value of the resulting hash input and use the calculated hash value as the `messageImprint` of the {{!RFC3161}} timestamp request for the target archive timestamp.

For example, the hash input for a first-generation archive timestamp is as follows.

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing)
~~~
Figure 8: First-Generation Archive Timestamp Input

For example, the hash input for a second-generation archive timestamp is as follows.

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing) || "." ||
BASE64URL(header.ltv.archive.timestamp)
~~~
Figure 9: Second-Generation Archive Timestamp Input

An archive timestamp is generated by creating an {{!RFC3161}} timestamp over the archive timestamp hash input.

The generated timestamp is stored as the corresponding archive timestamp.

### External Reference Hash Renewal (archive.rehashes) Embedding

"rehashes" is OPTIONAL and stores renewed hash values for external references ("refs").

If the hash algorithm used for externally referenced data becomes obsolete or cryptographically weak, newly calculated hash values using a new hash algorithm are added as "rehashes".

If "rehashes" is used, it MUST be added within the same "archive" object before generating the corresponding archive timestamp.

For example, the hash input for a second-generation archive timestamp using "rehashes" is as follows.

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing) || "." ||
BASE64URL(header.ltv.archive.timestamp) || "." ||
BASE64URL(header.ltv.archive.archive.rehashes)
~~~
Figure 10: Second-Generation Archive Timestamp Input (rehashes included)

### Next-Generation Archive Extension

To continuously preserve the long-term validity of signatures, the next "archive" object is added within the existing "archive" object.

If necessary, new "rehashes" are added and a new archive timestamp is generated, thereby enabling continuous extension of long-term validity.

# Validation Processing

## Validation Reference Time

LTV-JWS validation uses the validation reference time shown in the following table.

| Signature Level | Validation Reference Time |
|---|---|
| SIG-B | Signature Timestamp, if present |
| SIG-T / SIG-LTV / SIG-LTA | nearest protecting Archive Timestamp, if present |

Table 2: Validation Reference Time

If no corresponding timestamp exists, the current time is used as the validation reference time.

If SIG-B does not contain a valid Signature Timestamp and the applicable validation policy permits use of signingTime, signingTime MAY be used as the validation reference time.

The determination of validation reference time and validation results depends on the applicable validation policy.

## SIG-B Validation Processing

SIG-B is the base validation level that validates the JWS signature and the signing certificate.

SIG-B validation performs JWS signature verification, PKIX validation of the signing certificate, and external reference ("refs") hash validation.

The validation result of SIG-B is determined as Valid, Invalid, or Indeterminate.

### Signature Verification

Implementations MUST perform JWS signature verification using the JWS Signature Input construction defined in {{!RFC7515}}.

Implementations MUST verify the signature value using the signature algorithm specified by the JWS "alg" parameter.

If signature verification fails, the validation result MUST be treated as Invalid.

### Signing Certificate Validation

During signing certificate validation, implementations MUST verify that the signing certificate hash value contained in "ltv.signingCertHash" matches the signing certificate.

Implementations MUST perform PKIX validation of the signing certificate according to the applicable validation policy.

During SIG-B signing certificate validation, implementations MUST validate the certificate validity period, revocation status, and certificate chain using the Validation Reference Time.

If required validation information is unavailable, the validation result MAY be treated as Indeterminate.

### External Reference Verification

If the payload contains "ltv.refs", implementations MUST perform hash validation for all external references.

For each external reference, implementations MUST calculate the hash value using the corresponding "hashAlg" and verify that it matches the corresponding "hashValue".

If "type=raw", the entire binary data of the externally referenced data is used as the hash input.

If "type=jws", implementations MUST reconstruct the external reference hash input according to the chained signing hash input construction rules defined in the "Payload External References (ltv.refs) Creation" section.

Implementations MUST verify that the calculated hash value matches the corresponding "hashValue" parameter.

If external reference hash validation fails, the validation result MUST be treated as Invalid.

## SIG-T Validation Processing

During SIG-T validation, implementations MUST also validate SIG-B.

### Signature Timestamp Verification

During signature timestamp verification, implementations MUST reconstruct the signature timestamp hash input according to the signature timestamp input construction rules defined in the "Signature Timestamp Embedding" section.

Implementations MUST verify that the reconstructed hash input matches the messageImprint contained in the {{!RFC3161}} TimeStampToken.

Implementations MUST verify the signature value of the {{!RFC3161}} TimeStampToken.

If the messageImprint does not match or TimeStampToken signature verification fails, the validation result MUST be treated as Invalid.

### Signature Timestamp TSA Certificate Validation

Implementations MUST perform PKIX validation of the TSA certificate associated with the signature timestamp.

During TSA certificate validation, implementations MUST validate the certificate validity period, revocation status, and certificate chain using the Validation Reference Time.

If SIG-LTV or SIG-LTA exists, implementations MAY perform offline validation using validation information contained in the "validations" object within the decoded JSON object contained in the corresponding "timestamp" parameter.

If required validation information is unavailable, the validation result MAY be treated as Indeterminate.

## SIG-LTV Validation Processing

SIG-LTV validation MUST enable offline validation of SIG-B, SIG-T, and all existing SIG-LTA levels.

### Signing and All TSA Certificate Validation

SIG-LTV validation performs PKIX validation of the signing certificate and all TSA certificates associated with signature timestamps and archive timestamps.

Implementations SHOULD use validation information preserved in "ltv.signing" and the "validations" objects within all corresponding decoded JSON objects contained in "timestamp" parameters.

During validation of each certificate, implementations MUST validate the certificate validity period, revocation status, and certificate chain using the applicable Validation Reference Time.

Validation information preserved in SIG-LTV MAY include certificates, CRLs, OCSP responses, and other validation-related information required for offline validation. This may include certificates required for validation of OCSP responses or CRLs.

If the required validation information is insufficient to complete validation, the validation result MAY be treated as Indeterminate.

If any signing certificate or TSA certificate validation fails, the validation result MUST be treated as Invalid.

## SIG-LTA Validation Processing

During SIG-LTA validation, implementations MUST validate SIG-B, SIG-T, and all existing SIG-LTA levels.

### Archive Timestamp Verification

During archive timestamp verification, implementations MUST reconstruct the corresponding archive timestamp hash input according to the archive timestamp input construction rules defined in the "Archive Timestamp Embedding" section.

Implementations MUST verify that the reconstructed hash input matches the messageImprint contained in the {{!RFC3161}} TimeStampToken.

This includes reconstruction of recursive "archive" structures and any applicable "rehashes" elements.

Implementations MUST verify the signature value of the {{!RFC3161}} TimeStampToken.

If the messageImprint does not match or TimeStampToken signature verification fails, the validation result MUST be treated as Invalid.

### Archive Timestamp TSA Certificate Validation

Implementations MUST perform PKIX validation of the TSA certificate associated with the archive timestamp.

During TSA certificate validation, implementations MUST validate the certificate validity period, revocation status, and certificate chain using the Validation Reference Time.

Implementations MAY perform offline validation using validation information preserved in the "validations" object within the decoded JSON object contained in the corresponding "timestamp" parameter.

If required validation information is unavailable, the validation result MAY be treated as Indeterminate.

### External Reference Hash Renewal Verification (rehashes)

If "rehashes" exists, implementations MUST revalidate all external reference hash values using the renewed "hashAlg" and "hashValue" parameters.

Verification of "rehashes" does not replace validation of the original external references defined in SIG-B.

Implementations MUST verify that the "ltv.refs" array within "rehashes" preserves the same number of elements and the same ordering as the payload "ltv.refs" array.

Implementations MUST verify that no parameters other than "hashAlg" and "hashValue" have been modified.

If renewed external reference hash validation fails, the validation result MUST be treated as Invalid.

# Security Considerations

LTV-JWS preserves the processing model and security properties of JWS {{!RFC7515}}.

In addition, LTV-JWS adopts the long-term signature approach used in long-term signature formats such as XAdES [ISO14533-2].

In the long-term signature approach, validation information and timestamps are maintained separately from the JWS signature in the unprotected header, and long-term integrity and authenticity are preserved through independent cryptographic protection and the continuous addition of archive timestamps.

Once an archive timestamp has been added, information existing prior to that archive timestamp and covered by it is protected and SHOULD NOT be modified.

Implementations MUST carefully validate all cryptographic inputs, timestamp and external reference hash inputs, certificate validation information, and timestamp tokens themselves defined in this specification.

Improper validation processing, incorrect handling of unprotected header information, or incorrect interpretation of BASE64URL and BASE64 encoded values may result in incorrect validation results or loss of long-term signature validity.

## JSON Object Processing

Implementations MUST reject JSON objects containing duplicate member names.

This requirement applies to all JSON objects defined in this specification, including:

- protected headers
- unprotected headers
- payload "ltv" objects
- "validations" objects
- "archive" objects
- decoded JSON objects contained in "signing", "timestamp", and "rehashes" parameters

Duplicate member names may cause inconsistent interpretation between implementations and may result in incorrect signature validation, timestamp validation, archive validation, or external reference validation.

## Trust Model of Unprotected Header Information

LTV-JWS stores timestamps, validation information, and archive-related information in the unprotected header.

Information stored in the unprotected header is not directly protected by the JWS signature.

However, most validation information and timestamp-related information stored in the unprotected header are protected by existing PKIX/CMS signatures generated by CAs, OCSP responders, or TSAs.

In addition, unprotected header information is subject to long-term integrity protection through archive timestamps.

Implementations MUST NOT treat unprotected header information as trusted solely because such information is present in the JWS object.

If archive timestamp validation fails, the associated unprotected header information MUST be treated as untrusted.

## Detached Payload Considerations

Detached payload usage may complicate long-term preservation and validation because the payload data required for JWS signature validation is stored outside the JWS object.

Implementations and operational environments SHOULD ensure long-term preservation, availability, and integrity of detached payload data associated with LTV-JWS objects.

Loss or ambiguity of detached payload data may result in failure of signature validation, timestamp validation, archive validation, or long-term validation.

## Validation Requirements for External References

LTV-JWS supports indirect signing of externally referenced data through the "refs" array.

Implementations MUST validate all external reference hash values associated with the signature.

Successful JWS signature validation alone does not guarantee the integrity or authenticity of externally referenced data.

If the recalculated hash value of externally referenced data does not match the corresponding "hashValue" parameter, the validation result MUST be treated as Invalid.

LTV-JWS external references are limited to a single-level reference structure using the "refs" array within the payload, and recursive or multi-level external reference structures are not defined.

This restriction simplifies external reference validation processing and prevents reference loops, complex dependency relationships, and ambiguity regarding validation scope.

Implementations SHOULD carefully consider the trust model, retrieval method, and persistence of externally referenced data.

The security and availability of externally referenced data are outside the scope of this specification.

## External Reference Retrieval Security

Implementations SHOULD carefully control retrieval of externally referenced data identified by "uri" parameters.

Retrieval of external references may introduce security and operational risks including:

- Server-Side Request Forgery (SSRF)
- unintended access to internal network resources
- loopback or local file access
- retrieval of mutable or replaced content
- excessive resource consumption
- denial-of-service conditions
- retrieval timeouts
- unexpected URI scheme handling

Implementations SHOULD restrict permitted URI schemes, network locations, retrieval methods, object sizes, and timeout behavior according to the applicable security policy.

Implementations SHOULD NOT automatically trust externally retrieved data solely because the corresponding hash value matches.

The security, trustworthiness, persistence, availability, and retrieval policy of externally referenced data are outside the scope of this specification.

## Signature Timestamp and Archive Timestamp Validation

LTV-JWS uses {{!RFC3161}} timestamps to prove that signatures, validation information, and archive information existed prior to a specific point in time.

A signature timestamp proves that the signed state consisting of the JWS protected header, payload, and signature existed prior to a specific point in time.

An archive timestamp protects signatures, timestamps, validation information, and archive information existing prior to it, and enables continued long-term validation.

Implementations MUST reconstruct the hash input of signature timestamps and archive timestamps according to the methods defined in this specification and verify that it matches the messageImprint contained in the TimeStampToken.

Implementations MUST perform signature validation of all TimeStampTokens and PKIX validation of TSA certificates.

If signature timestamp validation fails, the proof of existence (signature time) of the corresponding SIG-B based on that signature timestamp is not established.

If archive timestamp validation fails, the proof of existence of the protected contents covered by the corresponding archive timestamp is not established.

For long-term validation, implementations MUST perform timestamp and certificate validation using an appropriate validation reference time according to the long-term signature approach.

The time of the signature timestamp MUST be used as the validation reference time for SIG-B signature validation.

For validation of each validations object, the time of the nearest protecting archive timestamp for the corresponding validation information MUST be used as the validation reference time. If no corresponding archive timestamp exists, the current time MUST be used as the validation reference time.

Improper timestamp validation or incorrect use of validation reference time may result in incorrect long-term validation results.

## Validation Policy and Trust Anchors

Validation results of LTV-JWS depend on the applicable validation policy and trust anchors.

Even for the same LTV-JWS object, validation results may differ depending on the trust anchors used, revocation information, permitted cryptographic algorithms, timestamp validation conditions, determination of validation reference time, or policies regarding the use of external validation information.

Implementations MUST appropriately manage trust anchors, certificate validation conditions, revocation checking conditions, timestamp validation conditions, and long-term validation requirements according to the applicable validation policy.

During long-term validation, the retention period and availability of certificates, CRLs, OCSP responses, TimeStampTokens, and other validation information may affect validation results.

Changes, removal, or revocation of trust anchors, or changes in validation policy, may cause an LTV-JWS object previously determined as Valid to later be determined as Indeterminate or Invalid.

## Encoding Confusion Between BASE64URL and BASE64

LTV-JWS uses both BASE64URL encoding and BASE64 encoding depending on the purpose of the represented data.

Values used for JWS signing input construction, timestamp hash input construction, external reference hash input construction, and hash-based identifiers use BASE64URL encoding.

In contrast, DER-encoded ASN.1 binary objects such as certificates, CRLs, OCSP responses, and {{!RFC3161}} TimeStampTokens use BASE64 encoding consistent with the "x5c" parameter defined in {{!RFC7515}}.

BASE64URL encoding and BASE64 encoding are not interchangeable and differ in character sets and padding rules.

Implementations MUST use the correct encoding for each parameter and hash input construction defined in this specification.

Incorrect encoding processing, incorrect decoding processing, or confusion between BASE64URL encoding and BASE64 encoding may result in signature validation failure, timestamp validation failure, external reference hash mismatch, or long-term validation failure.

## Recursive Archive Validation

LTV-JWS preserves long-term signature validity by continuously adding archive timestamps using a recursive archive structure.

Each archive timestamp protects signatures, timestamps, validation information, and archive information existing prior to it.

Implementations MUST reconstruct the hash input corresponding to each archive timestamp according to the method defined in this specification and verify that it matches the messageImprint contained in the TimeStampToken.

For validation of each archive timestamp, the time of the corresponding nearest protecting archive timestamp MUST be used as the validation reference time. If no nearest protecting archive timestamp exists, the current time MUST be used as the validation reference time.

Archive timestamp validation may be performed from outermost to innermost, or from innermost to outermost, depending on implementation or validation policy. However, implementations SHOULD apply validation processing consistently.

If recursive archive validation fails, the long-term validation state protected by the corresponding archive timestamp MUST be treated as untrusted.

Implementations and operational environments SHOULD avoid excessive nesting of recursive archive structures.

## Cryptographic Algorithm Agility

LTV-JWS adopts a long-term signature approach that assumes cryptographic algorithm migration in order to address cryptographic algorithm obsolescence, weakening, or compromise.

Cryptographic algorithms used for signatures, timestamps, certificate validation, and external reference hashes may lose security over time.

Implementations and operational environments MUST migrate to cryptographic algorithms providing sufficient security according to the applicable security policy.

By continuously adding archive timestamps, past signatures, timestamps, validation information, and archive information can continue to be protected using newer cryptographic algorithms.

For external references, before the currently used hash algorithm becomes weak or compromised, overlapping assurance using both old and new hash algorithms can be maintained by adding "rehashes".

In the long-term signature approach, even cryptographic algorithms that are currently considered weak or deprecated may still be considered valid if the corresponding signatures, timestamps, or hash values were generated at a time when those algorithms were considered sufficiently secure.

Therefore, implementations and operational environments MUST appropriately manage the validity period, security evaluation, and applicable usage period of each cryptographic algorithm.

Implementations MUST appropriately identify weak or deprecated cryptographic algorithms and determine validation results according to the applicable validation policy.

Failure of cryptographic algorithm migration, inappropriate algorithm selection, or insufficient archive timestamp renewal may result in loss of long-term signature validity.

## Canonicalization and Deterministic Inputs

LTV-JWS achieves deterministic signature inputs and hash inputs without using JSON canonicalization by reusing the JWS signing input model defined in {{!RFC7515}}.

JWS signature inputs, signature timestamp hash inputs, and archive timestamp hash inputs are constructed by concatenating BASE64URL-encoded values using the "." character.

For external references, the external reference hash input uses the externally referenced binary data itself as the hash input by default. If `type=jws` is used, the same BASE64URL concatenation model as the JWS signing input model is used.

This approach avoids differences caused by JSON serialization order, whitespace, indentation, or other variations in JSON textual representation.

Even in recursive archive structures, hash inputs are deterministically reconstructed without requiring JSON canonicalization because the hash inputs are constructed as ordered concatenations of BASE64URL-encoded values.

Implementations MUST reconstruct signature inputs and hash inputs according to the exact input construction methods defined in this specification.

Incorrect JSON re-serialization, misuse of JSON canonicalization, incorrect BASE64URL processing, or incorrect input construction ordering may result in signature validation failure, timestamp validation failure, or long-term validation failure.

In particular, archive timestamp validation requires exact reconstruction of hash inputs including prior archive structures, and incorrect input reconstruction may cause failure of the entire long-term signature validation.

## Local Signing Time Considerations

The "signingTime" parameter in LTV-JWS represents the local system time used at the time of signing.

The "signingTime" value is informational and does not by itself provide cryptographically protected proof of signing time.

Trusted signing time is generally established using trusted timestamp mechanisms such as {{!RFC3161}} timestamps.

Implementations MUST NOT treat "signingTime" as trusted signing time. However, this restriction does not apply if the applicable validation policy permits the use of trusted local system time.

The "signingTime" value may contain incorrect values due to system clock manipulation, timezone misconfiguration, incorrect local time configuration, or malicious signers.

If the "signingTime" value significantly differs from the trusted timestamp time, the validation policy should treat the difference as a suspicious time discrepancy warning.

For long-term validation, validation reference times should generally be based on trusted timestamps.

## Validation Information Applicability

For long-term validation in LTV-JWS, appropriate validation information MUST be used for the applicable validation reference time.

Certificates, CRLs, OCSP responses, TimeStampTokens, and other validation information should be valid and applicable at the corresponding validation reference time.

Implementations MUST appropriately validate certificate validity periods, CRL issuance times, nextUpdate values, OCSP response validity, timestamp times, and other validation-related time information.

Use of inappropriate validation information for a given validation reference time may result in incorrect revocation status determination or incorrect validation results.

In the long-term signature approach, validation information itself also needs to be continuously protected by archive timestamps.

Missing, corrupted, or inapplicable validation information for a given validation reference time may cause long-term validation results to become Indeterminate.

Implementations and operational environments should appropriately preserve validation information required for long-term validation and continue adding archive timestamps appropriately.

# IANA Considerations

This document requests the registration of the "ltv" JOSE Header Parameter in the JSON Web Signature and Encryption Header Parameters Registry defined by {{!RFC7515}}.

This specification uses the "ltv" object within the JWS Protected Header, JWS Unprotected Header, and the payload.

The "ltv" object used in the Protected Header and Unprotected Header is treated as a JOSE Header Parameter and is subject to IANA registration.

The "ltv" object used within the payload is a payload structure used to contain external references ("refs") and is not subject to the JOSE Header Parameter Registry.

This specification defines additional Long-Term Validation related parameters and extension attributes managed within the "ltv" object.

Additional LTV-related parameters may be defined by this specification or future extension specifications.

## Registration of the "ltv" JOSE Header Parameter

This specification defines the "ltv" JOSE Header Parameter for use in JWS Protected Headers and Unprotected Headers.

This specification also uses an "ltv" object within the payload for external reference ("refs") structures.

The "ltv" Header Parameter contains long-term validation related information including signature extension information, timestamps, validation information, and archive information.

This specification requests registration of the "ltv" parameter in the "JSON Web Signature and Encryption Header Parameters" registry.

The registration is as follows:

- Header Parameter Name: "ltv"
- Header Parameter Description: Long-Term Validation information container for JWS
- Header Parameter Usage Location(s):
  JWS Protected Header,
  JWS Unprotected Header
- Change Controller: IETF
- Specification Document(s): This specification

The "ltv" Header Parameter MAY be included in the "crit" Header Parameter defined in {{!RFC7515}}.

Implementations that do not understand the "ltv" Header Parameter and encounter "ltv" in "crit" MUST reject the JWS.

## LTV-JWS External Reference Type Identifiers

This specification defines external reference "type" identifiers used by LTV-JWS.

These identifiers are used to identify external reference hash input construction methods and external reference signing models.

This specification defines the following initial values:

- "raw": uses externally referenced binary data directly as the hash input
- "jws": uses the same BASE64URL concatenation model as the JWS signing input model for externally referenced JWS objects

Additional external reference type identifiers may be defined by future specifications.

## LTV-JWS Timestamp Type Identifiers

This specification defines timestamp "type" identifier used by LTV-JWS.

These identifiers are used to identify the timestamp format contained in timestamp objects.

This specification defines the following initial value:

- "rfc3161": {{!RFC3161}} TimeStampToken

Additional timestamp type identifiers may be defined by future specifications.

## Hash Algorithm Identifiers

This specification defines "S256", "S384", and "S512" as hash algorithm identifiers.

These are shorthand identifiers representing SHA-256, SHA-384, and SHA-512 respectively.

Additional hash algorithm identifiers may be defined by future specifications.

## No New Cryptographic Algorithms

This specification does not define new signature algorithms, hash algorithms, timestamp algorithms, or cryptographic primitives.

LTV-JWS reuses existing algorithms and processing models defined in {{!RFC7515}}, {{!RFC7518}}, {{!RFC3161}}, and existing PKIX/CMS related specifications.

--- back

# References

## Normative References


## Informative References

[ISO14533-2] ISO/IEC 14533-2:2012, "Processes, data elements and documents in support of long-term signature profiles - Part 2: Profiles for XML advanced electronic signatures (XAdES)"

# Appendix: LTV-JWS Example

## Example SIG-B (examples/sig-b.json)

This example shows a SIG-B LTV-JWS object.

The example demonstrates:

* JWS JSON Serialization
* `crit` usage for `"ltv"`
* `signingTime`
* `signingCertHash`
* indirect signing using `ltv.refs`
* external references (`test1.txt` and `test2.json`)
* optional signer validation information using `header.ltv.signing`

For minimal SIG-B objects, the `header.ltv.signing` parameter MAY be omitted if signer validation information is not embedded.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "eyJsdHYiOnsicmVmcyI6W3sidXJpIjoidGVzdDEudHh0IiwiaGFzaEFsZyI6IlMyNTYiLCJoYXNoVmFsdWUiOiJxVFZ2S3pVLTFHeFRpcmN4dkdTbTQ1bGUyb2Ixdks3cHVBemJYdVhWYm9vIn0seyJ1cmkiOiJ0ZXN0Mi5qc29uIiwiaGFzaEFsZyI6IlMyNTYiLCJoYXNoVmFsdWUiOiIzWURVZ2xGNDF4dmdmSUF2OFRxYS03OTVxaG8zaUlKaXB2LXYzS1JtMldBIn1dfX0",
  "protected": "eyJhbGciOiJSUzI1NiIsImNyaXQiOlsibHR2Il0sImx0diI6eyJzaWduaW5nVGltZSI6IjIwMjYtMDUtMThUMDE6NDg6MDVaIiwic2lnbmluZ0NlcnRIYXNoIjp7Imhhc2hBbGciOiJTMjU2IiwiaGFzaFZhbHVlIjoibnhsWDFtOXRycEFkN1dTMjN5OEV2TG5PTlppVEZoLWtKQlpTRkdyNXZ4NCJ9fX0",
  "signature": "SD1LmkNPQFotnNvmkdymyrMTWVb_NTYjnY_VZ8Od5GGroA0RH3acuHhqjH9Yw46BnDPfr5Fjb996QBU8NhMzvrrBT38BBCFzyTMlChq_EkaRXeZyb3Ag1AClAe0Wz1g41tpQxnzWGRR8v6ozymb6g-T17Z_3I0SZlMKbmPdHVY0uFwYsgRpntqxM9_mG1kSZjyvYfrk-ZufyPuwgj8_l08-mN1Us6Gp6csqJJ8-QTd5-JPdKAFnj14QfYuEeyQcx5ZZaizDlEQnXghstkEJJ-dUIhlx5Pl0waAQvSp5H65bx0PDmVASfHQUIoy_Y3z5zqrGHRpJqPgAxOh770CzaQtF_yjujHWeVNqpgdX36zPl4f18jKbGRgEggl5iszoNWu-IVAFMVBQdR7vwXfdbuetjttfYuqo5Yk3Gw2bv7T3YeV2_7_1yGwByc8SXcP3PkStpDqIz_SRJ6sQ9K0cvwR6araBXpbkRqCECfG8DFfwlZqyPwgXG10B8N-2ABbPro",
  "header": {
    "ltv": {
      "signing": "eyJ2YWxpZGF0aW9ucyI6eyJjZXJ0cyI6WyJNSUlGaXpDQ0Ez..."
    }
  }
}
~~~
Figure 11: Example of SIG-B

This example shows a SIG-B LTV-JWS object including optional signer validation information.

Decoded payload:

~~~ json
{
  "ltv": {
    "refs": [
      {
        "uri": "test1.txt",
        "hashAlg": "S256",
        "hashValue": "qTVvKzU-1GxTircxvGSm45le2ob1vK7puAzbXuXVboo"
      },
      {
        "uri": "test2.json",
        "hashAlg": "S256",
        "hashValue": "3YDUglF41xvgfIAv8Tqa-795qho3iIJipv-v3KRm2WA"
      }
    ]
  }
}
~~~
Figure 12: Example of SIG-B Decoded payload

Decoded protected header:

~~~ json
{
  "alg": "RS256",
  "crit": ["ltv"],
  "ltv": {
    "signingTime": "2026-05-18T01:48:05Z",
    "signingCertHash": {
      "hashAlg": "S256",
      "hashValue": "nxlX1m9trpAd7WS23y8EvLnONZiTFh-kJBZSFGr5vx4"
    }
  }
}
~~~
Figure 13: Example of SIG-B Decoded protected header

Decoded BASE64URL value of `header.ltv.signing`:

~~~ json
{
  "validations": {
    "certs": [
      "MIIFizCCA3OgAwIBAgIDEAAXMA0GCSqGSIb3DQEBCwUAME4x..."
    ]
  }
}
~~~
Figure 14: Example of SIG-B value of `header.ltv.signing`

The `header.ltv.signing` value itself is a BASE64URL-encoded JSON object.

The embedded certificate values use BASE64 encoding because they are DER-encoded ASN.1 objects.

The JWS Signing Input for this example is constructed as:

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload)
~~~
Figure 15: JWS Signature (SIG-B) Input Construction

This example uses indirect signing through the `ltv.refs` array.

The externally referenced files identified by `ltv.refs` are:

* `test1.txt`
* `test2.json`

The `hashValue` of each reference is calculated over the externally referenced binary data itself because the default external reference type is `raw`.

### Example SIG-B with x5c

This example shows a SIG-B LTV-JWS object using the existing JWS "x5c" header parameter instead of the optional `header.ltv.signing` parameter.

The example demonstrates:

* use of the existing JWS `x5c` header parameter
* preservation of the signing certificate using `x5c`
* compatibility with existing JWS processing models
* indirect signing using `ltv.refs`

The `x5c` parameter is stored directly in the JWS header according to {{!RFC7515}}.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-B example)",
  "protected": "(same as SIG-B example)",
  "signature": "(same as SIG-B example)",
  "header": {
    "x5c":["MIIFizCCA3OgAwIBAgIDEAAXMA0GCSqGSIb3DQEBCwUAME4x..."]
  }
}
~~~
Figure 16: Example of SIG-B using 'x5c'

## Example SIG-T (examples/sig-t.json)

This example shows a SIG-T LTV-JWS object generated by adding a signature timestamp to the previous SIG-B example.

The example demonstrates:

* {{!RFC3161}} signature timestamp
* `header.ltv.timestamp`
* signature timestamp hash input construction

The `header.ltv.timestamp` value itself is a BASE64URL-encoded JSON object.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-B example)",
  "protected": "(same as SIG-B example)",
  "signature": "(same as SIG-B example)",
  "header": {
    "ltv": {
      "signing": "(same as SIG-B example)",
      "timestamp": "eyJ0c3QiOiJNSUlQRVFZSktvWklodmNOQVFjQ29JSVBBakND..."
    }
  }
}
~~~
Figure 17: Example of SIG-T

Decoded BASE64URL value of `header.ltv.timestamp`:

~~~ json
{
  "tst": "MIIPEQYJKoZIhvcNAQcCoIIPAjCCDv4CAQMxCzAJBgUrDgMCGgUAMIHU..."
}
~~~
Figure 18: Example of SIG-T Decoded value of `header.ltv.timestamp`

The `tst` value contains a BASE64-encoded {{!RFC3161}} TimeStampToken in DER format.

The Signature Timestamp Input for this example is constructed as:

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature)
~~~
Figure 19: Signature Timestamp Input Construction

The {{!RFC3161}} `messageImprint` value is calculated from the hash value of the Signature Timestamp Input.

If the `type` parameter is omitted, the timestamp type defaults to `"rfc3161"`.

The decoded BASE64URL value of `header.ltv.timestamp` MAY additionally contain a `validations` object preserving validation information for the TSA certificate.

## Example SIG-LTV (examples/sig-ltv.json)

This example shows a SIG-LTV LTV-JWS object generated by adding validation information to the previous SIG-T example.

The example demonstrates:

* validation information embedding
* `header.ltv.signing`
* validations within the decoded BASE64URL value of `header.ltv.timestamp`
* preservation of certificates and CRLs for long-term validation

The `header.ltv.signing` and `header.ltv.timestamp` values are BASE64URL-encoded JSON objects.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-T example)",
  "protected": "(same as SIG-T example)",
  "signature": "(same as SIG-T example)",
  "header": {
    "ltv": {
      "signing": "eyJ2YWxpZGF0aW9ucyI6eyJjZXJ0cyI6WyJNSUlGMlRDQ0E4...Il0sImNybHMiOlsiTUlJQ3JEQ0JsVEFO...",
      "timestamp": "eyJ0c3QiOiJNSUlQRVFZSktvWklodmNOQVFjQ29JSVBBakND...LCJ2YWxpZGF0aW9ucyI6eyJjZXJ0cyI6WyJNSUlGMlRDQ0E4..."
    }
  }
}
~~~
Figure 20: Example of SIG-LTV

Decoded BASE64URL value of `header.ltv.signing`:

~~~ json
{
  "validations": {
    "certs": [
      "MIIF2TCCA8GgAwIBAgIHA41+pMaABTANBgkqhkiG9w0BAQsF...",
      "MIIFizCCA3OgAwIBAgIDEAAXMA0GCSqGSIb3DQEBCwUAME4x..."
    ],
    "crls": [
      "MIICrDCBlTANBgkqhkiG9w0BAQsFADBOMQswCQYDVQQGEwJK..."
    ]
  }
}
~~~
Figure 21: Example of SIG-LTV Decoded value of `header.ltv.signing`

Decoded BASE64URL value of `header.ltv.timestamp`:

~~~ json
{
  "tst": "MIIPEQYJKoZIhvcNAQcCoIIPAjCCDv4CAQMxCzAJBgUrDgMC...",
  "validations": {
    "certs": [
      "MIIF2TCCA8GgAwIBAgIHA41+pMaABjANBgkqhkiG9w0BAQsF...",
      "MIIFozCCA4ugAwIBAgIDIAAwMA0GCSqGSIb3DQEBCwUAME4x..."
    ],
    "crls": [
      "MIICrDCBlTANBgkqhkiG9w0BAQsFADBOMQswCQYDVQQGEwJK..."
    ]
  }
}
~~~
Figure 22: Example of SIG-LTV Decoded value of `header.ltv.timestamp`

The `validations.certs` array within the decoded BASE64URL value of `header.ltv.signing` contains the signing certificate and certificates required for PKIX validation of the signing certificate.

The `validations.certs` array within the decoded BASE64URL value of `header.ltv.timestamp` contains the TSA certificate and certificates required for PKIX validation of the TSA certificate associated with the {{!RFC3161}} timestamp.

The embedded certificate values use BASE64 encoding because they are DER-encoded ASN.1 certificate objects.

The embedded CRL values use BASE64 encoding because they are DER-encoded ASN.1 CRL objects.

The `tst` value contains a BASE64-encoded {{!RFC3161}} TimeStampToken in DER format.

Validation information preserved in SIG-LTV MAY include certificates, CRLs, OCSP responses and other validation-related information required for Long-Term Validation.

SIG-LTV validation can be performed using only the preserved validation information without requiring external validation services or network access.

## Example SIG-LTA (examples/sig-lta.json)

This example shows a SIG-LTA LTV-JWS object generated by adding an archive timestamp to the previous SIG-LTV example.

The example demonstrates:

* archive timestamp protection
* `header.ltv.archive.timestamp`
* long-term preservation of validation information
* renewable archive protection

The `header.ltv.archive.timestamp` value is a BASE64URL-encoded JSON object.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-LTV example)",
  "protected": "(same as SIG-LTV example)",
  "signature": "(same as SIG-LTV example)",
  "header": {
    "ltv": {
      "signing": "(same as SIG-LTV example)",
      "timestamp": "(same as SIG-LTV example)",
      "archive": {
        "timestamp": "eyJ0c3QiOiJNSUlQRUFZSktvWklodmNOQVFjQ29JSVBBVEND..."
      }
    }
  }
}
~~~
Figure 23: Example of SIG-LTA

Decoded BASE64URL value of `header.ltv.archive.timestamp`:

~~~ json
{
  "tst": "MIIPEAYJKoZIhvcNAQcCoIIPATCCDv0CAQMxCzAJBgUrDgMC..."
}
~~~
Figure 24: Example of SIG-LTA Decoded value of `header.ltv.archive.timestamp`

The `tst` value contains a BASE64-encoded {{!RFC3161}} TimeStampToken in DER format.

The Archive Timestamp Input for this example is constructed as:

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing)
~~~
Figure 25: Archive Timestamp Input

The {{!RFC3161}} `messageImprint` value of the archive timestamp is calculated from the hash value of the Archive Timestamp Input.

The archive timestamp protects the accumulated long-term validation state existing at the time of archive timestamp generation.

* the JWS signature
* the external reference hashes
* the signature timestamp
* the preserved validation information

Additional archive timestamps MAY be added recursively for long-term renewal protection.

## Example SIG-LTV 2nd (examples/sig-ltv-2nd.json)

This example shows a second-generation SIG-LTV object generated by adding validation information for the archive timestamp TSA certificate to the previous SIG-LTA example.

The example demonstrates:

* preservation of archive timestamp TSA validation information
* recursive long-term validation state accumulation
* second-generation long-term validation information

The `header.ltv.archive.timestamp` value is a BASE64URL-encoded JSON object.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-LTA example)",
  "protected": "(same as SIG-LTA example)",
  "signature": "(same as SIG-LTA example)",
  "header": {
    "ltv": {
      "signing": "(same as SIG-LTV example)",
      "timestamp": "(same as SIG-LTV example)",
      "archive": {
        "timestamp": "eyJ0c3QiOiJNSUlQRUFZSktvWklodmNOQVFjQ29JSVBBVEND..."
      }
    }
  }
}
~~~
Figure 26: Example of SIG-LTV 2nd

Decoded BASE64URL value of `header.ltv.archive.timestamp`:

~~~ json
{
  "tst": "MIIPEAYJKoZIhvcNAQcCoIIPATCCDv0CAQMxCzAJBgUrDgMC...",
  "validations": {
    "certs": [
      "MIIF2TCCA8GgAwIBAgIHA41+pMaABjANBgkqhkiG9w0BAQsF...",
      "MIIE..."
    ],
    "crls": [
      "MIICrDCBlTANBgkqhkiG9w0BAQsFADBOMQswCQYDVQQGEwJK..."
    ]
  }
}
~~~
Figure 27: Example of SIG-LTV 2nd Decoded value of `header.ltv.archive.timestamp`

The `header.ltv.archive.timestamp.validations.certs` array contains the archive TSA certificate and certificates required for PKIX validation of the archive TSA certificate associated with the {{!RFC3161}} archive timestamp.

The embedded certificate values use BASE64 encoding because they are DER-encoded ASN.1 certificate objects.

The embedded CRL values use BASE64 encoding because they are DER-encoded ASN.1 CRL objects.

The `tst` value contains a BASE64-encoded {{!RFC3161}} TimeStampToken in DER format.

The archive timestamp validation information becomes part of the accumulated long-term validation state protected by subsequent archive timestamps.

Additional archive timestamps MAY be added recursively for long-term renewal protection.

## Example SIG-LTA 2nd (examples/sig-lta-2nd.json)

This example shows a second-generation SIG-LTA object generated by adding a new archive timestamp to the previous SIG-LTV 2nd example.

The example demonstrates:

* recursive archive timestamp protection
* second-generation archive timestamps
* preservation of accumulated long-term validation state
* renewable archive timestamp protection

The `header.ltv.archive.archive.timestamp` value is a BASE64URL-encoded JSON object.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-LTV 2nd example)",
  "protected": "(same as SIG-LTV 2nd example)",
  "signature": "(same as SIG-LTV 2nd example)",
  "header": {
    "ltv": {
      "signing": "(same as SIG-LTV example)",
      "timestamp": "(same as SIG-LTV example)",
      "archive": {
        "timestamp": "(same as SIG-LTV 2nd example)",
        "archive": {
          "timestamp": "eyJ0c3QiOiJNSUlQRUFZSktvWklodmNOQVFjQ29JSVBBVEND..."
        }
      }
    }
  }
}
~~~
Figure 28: Example of SIG-LTA 2nd

Decoded BASE64URL value of `header.ltv.archive.archive.timestamp`:

~~~ json
{
  "tst": "MIIPEAYJKoZIhvcNAQcCoIIPATCCDv0CAQMxCzAJBgUrDgMC..."
}
~~~
Figure 29: Example of SIG-LTA 2nd Decoded value of `header.ltv.archive.archive.timestamp`

The `tst` value contains a BASE64-encoded {{!RFC3161}} TimeStampToken in DER format.

The second-generation archive timestamp protects the accumulated long-term validation state existing at the time of archive timestamp generation, including:

* the previous archive timestamp
* the JWS signature
* the external reference hashes
* the signature timestamp
* the embedded validation information

The Archive Timestamp Input for this example is constructed as:

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing) || "." ||
BASE64URL(header.ltv.archive.timestamp)
~~~
Figure 30: 2nd Archive Timestamp Input

Additional archive timestamps MAY be added recursively for long-term renewal protection.

## Example SIG-LTA refs renew hashes (examples/sig-rehash.json)

This example shows a SIG-LTA archive object containing renewed hash information for external references (`refs`).

The example demonstrates:

* renewal of external reference hash algorithms
* preservation of original external reference ordering
* archive protection of renewed hash values
* continued verification after cryptographic algorithm updates

The `header.ltv.archive.archive.rehashes` value is a BASE64URL-encoded JSON object.

The decoded JSON object preserves the same external references and ordering as the payload `ltv.refs` array while updating only the `hashAlg` and `hashValue` parameters.

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "(same as SIG-LTV example)",
  "protected": "(same as SIG-LTV example)",
  "signature": "(same as SIG-LTV example)",
  "header": {
    "ltv": {
      "signing": "(same as SIG-LTV example)",
      "timestamp": "(same as SIG-LTV example)",
      "archive": {
        "timestamp": "(same as SIG-LTV example)",
        "archive": {
          "rehashes": "eyJsdHYiOnsicmVmcyI6W3sidXJpIjoidGVzdDEudHh0...",
          "timestamp": "eyJ0c3QiOiJNSUlQRVFZSktvWklodmNOQVFjQ29JSVBB..."
        }
      }
    }
  }
}
~~~
Figure 31: Example of SIG-LTA refs renew hashes

Decoded BASE64URL value of `header.ltv.archive.archive.rehashes`:

~~~ json
{
  "ltv": {
    "refs": [
      {
        "uri": "test1.txt",
        "hashAlg": "S512",
        "hashValue": "eNAei4m6TiNr3Uuf57rMw82b4JQs3GvFGCRGsiFJi6ViuWV0lQWCbIi9oY_WT-SqK7B3SnLcaw7bQY1XJKJBLQ"
      },
      {
        "uri": "test2.json",
        "hashAlg": "S512",
        "hashValue": "lJjoF-6wuQ-OcrC6MGWBQvY1WfAFnX60El-HDejKCYQWt0Pa0qs8zWHJ5XdyuraN0c96IgE3wDIIfAdIIBGcWw"
      }
    ]
  }
}
~~~
Figure 32: Example of SIG-LTA refs renew hashes Decoded value of `header.ltv.archive.archive.rehashes`

The renewed `ltv.refs` array preserves the same external references and the same ordering as the payload `ltv.refs` array.

Only the `hashAlg` and `hashValue` parameters are updated.

The Archive Timestamp Input for this example is constructed as:

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature) || "." ||
BASE64URL(header.ltv.timestamp) || "." ||
BASE64URL(header.ltv.signing) || "." ||
BASE64URL(header.ltv.archive.timestamp) || "." ||
BASE64URL(header.ltv.archive.archive.rehashes)
~~~
Figure 33: 2nd Archive Timestamp Input with rehashed

The resulting archive timestamp is stored in `header.ltv.archive.archive.timestamp`.

The generated archive timestamp protects:

* the original JWS signature
* the signature timestamp
* the signing validation information
* the previous archive timestamp
* the renewed external reference hashes

Additional archive timestamps and renewed hash information MAY be added recursively for continued long-term preservation.


## Example Chained Signing (examples/sig-chained.json)

This example shows a chained signing model using external references.

The example demonstrates:

* signing of external data using `refs`
* indirect signing of another JWS object
* chained verification using referenced JWS signatures
* use of mixed external reference types

The payload contains two external references:

* `test1.txt`
  * referenced as raw external data
* `sig-b.json`
  * referenced as a JWS object using `"type": "jws"`

Some long BASE64URL and BASE64 values are shortened for readability.

~~~ json
{
  "payload": "eyJsdHYiOnsicmVmcyI6W3sidXJpIjoidGVzdDEudHh0IiwiaGFzaEFsZyI6IlMyNTYiLCJoYXNoVmFsdWUiOiJxVFZ2S3pVLTFHeFRpcmN4dkdTbTQ1bGUyb2Ixdks3cHVBemJYdVhWYm9vIn0seyJ1cmkiOiJzaWctYi5qc29uIiwiaGFzaEFsZyI6IlMyNTYiLCJoYXNoVmFsdWUiOiJuVlFTblF1M2YwMlI2cURmdURkaV9LWkY1YjRLeDVHUWtNTjRWdXdiSjNVIiwidHlwZSI6Imp3cyJ9XX19",
  "protected": "eyJhbGciOiJSUzI1NiIsImNyaXQiOlsibHR2Il0sImx0diI6eyJzaWduaW5nVGltZSI6IjIwMjYtMDUtMThUMDE6NDg6MDVaIiwic2lnbmluZ0NlcnRIYXNoIjp7Imhhc2hBbGciOiJTMjU2IiwiaGFzaFZhbHVlIjoibnhsWDFtOXRycEFkN1dTMjN5OEV2TG5PTlppVEZoLWtKQlpTRkdyNXZ4NCJ9fX0",
  "signature": "Fot99xm-cHhZlAE5uR9sTfeOmsTPx-ozWVpAI0PEUHILsbNcKNr4lVfUQtB3vWkQy2iOkgr5Xi_tCXPYgMa2noWXN7BPBfA2ObTGy05tEZvFLc1TkkGgpeaQg0rOZbdkxAYgD8V6HPx7V1FZNz3sL-GQGLEXQ8CTjSmyapjt4Pq03OQhiFHxc3cnuD7euBBXfUwwPIF1RALAaAjqxQfWUQ9yuyLnUZsBoIVv_UqTGQBHS3DoKGD7klwDswGIU-4xVU0i4GUaRyLM4hs4IZkHizowMasS-i4765UIy6jXiEpCJQz0-UBKzCkwyQAS8qagM8hzXPrPC9wt3VJoFDVWCjONItdR1b9l6nudRDjiQ4w6hr3niuDnIGNk46YN5n9IHOyQb-V6ZQ9BP0STyjoAQ-VX379E9J7iOTY_HDr6Sm7rNjPVADA-QVNRHCctk9-Elw3Lk3qEAtHfAs8FSHl3cfMhYjpa9Q2VsNRD1dOTg19sue6YGFnNlDK9b95Gf0MK",
  "header": {
    "ltv": {
      "signing": "eyJ2YWxpZGF0aW9ucyI6eyJjZXJ0cyI6WyJNSUl..."
    }
  }
}
~~~
Figure 34: Example of Chained Signing

Decoded BASE64URL value of `payload`:

~~~ json id="1bzh0x"
{
  "ltv": {
    "refs": [
      {
        "uri": "test1.txt",
        "hashAlg": "S256",
        "hashValue": "qTVvKzU-1GxTircxvGSm45le2ob1vK7puAzbXuXVboo"
      },
      {
        "uri": "sig-b.json",
        "hashAlg": "S256",
        "hashValue": "nVQSnQu3f02R6qDfuDdi_KZF5b4Kx5GQkMN4VuwbJ3U",
        "type": "jws"
      }
    ]
  }
}
~~~
Figure 35: Example of Chained Signing Decoded value of `payload`:

The `test1.txt` reference is verified as raw external data.

The `sig-b.json` reference is verified as a JWS object.

For `"type": "jws"`, the referenced hash input is constructed as:

~~~ text
BASE64URL(protected) || "." ||
BASE64URL(payload) || "." ||
BASE64URL(signature)
~~~
Figure 36: Chained Signing Hash Input Construction

The verifier computes the hash value of the referenced JWS signing input and compares it with the `hashValue` parameter.

This model allows chained signing and indirect protection of previously signed JWS objects while preserving the original JWS signatures.

The `signing` parameter is optional for SIG-B.

