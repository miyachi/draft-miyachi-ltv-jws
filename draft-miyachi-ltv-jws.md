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
（本仕様は、JSON Web Signature（JWS, RFC 7515）を拡張し、長期検証（Long-Term Validation: LTV）可能な署名を実現する軽量な長期署名フォーマット LTV-JWS を定義する。）

LTV-JWS adds signature extension elements, timestamps, validation information (certificates, CRLs, and OCSP responses), and archive structures as minimal extensions, thereby enabling the validity of signatures to be verified over extended periods of time. In addition, archive timestamps enable continued validation even after the obsolescence or compromise of cryptographic algorithms.
（LTV-JWS は、署名拡張要素、タイムスタンプ、検証情報（証明書、CRL、OCSP）、およびアーカイブ構造を最小限の拡張として導入することで、長期間にわたり署名の有効性を検証可能とする。またアーカイブタイムスタンプにより、暗号アルゴリズムの危殆化対応においても有効性を継続的に検証可能とする。）

LTV-JWS preserves the simple structure and concept of JWS while progressively adding timestamps and validation information. It also enables more general-purpose signing use cases through indirect signatures using external references (refs).
（LTV-JWS は、JWS のシンプルな構造とコンセプトを維持しつつ、段階的な検証情報およびタイムスタンプを追加する。また外部参照（refs）による間接署名により汎用的な利用を可能とする。）

# Status of This Memo

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79.

# Introduction

JSON Web Signatures (JWS) [RFC7515], a JSON-based signature format, are widely used to ensure the authenticity of target data. When data and JWS objects are stored over long periods of time, issues arise such as the expiration of signing certificates and the obsolescence or compromise of cryptographic algorithms.
（JSONベースの署名形式である JSON Web Signatures (JWS) [RFC7515] は、対象データの真正性を保証するために広く利用されている。長期間にわたりデータとJWSを保管する場合には、署名証明書の有効期限切れや暗号アルゴリズムの危殆化が問題となる。）

As an approach to Long-Term Validation (LTV), this specification defines four signature levels: the base signature level (SIG-B), the signature timestamp level (SIG-T), the long-term validation signature level containing validation information (SIG-LTV), and the long-term archive timestamp signature level (SIG-LTA). Furthermore, the continued addition of validation information and archive timestamps enables continued verification of signature validity, even after the obsolescence or compromise of cryptographic algorithms.
（長期検証（Long-Term Validation: LTV）のアプローチとして、ベース署名（SIG-B）、署名タイムスタンプ付き署名（SIG-T）、検証情報を含む長期検証署名（SIG-LTV）、およびアーカイブタイムスタンプによる長期保管署名（SIG-LTA）の4ステップを定義する。更に検証情報およびアーカイブタイムスタンプを継続的に追加することで、署名の有効性の継続的な検証を可能とし、暗号アルゴリズムの危殆化後においても有効性の検証を可能とする。）

Long-Term Validation for JSON Web Signature (LTV-JWS) is a JWS extension specification that defines a long-term signature format based on JWS JSON Serialization and a long-term validation approach similar to XAdES (XML Advanced Electronic Signature) for XML signatures.
（Long-Term Validation for JSON Web Signature (LTV-JWS) は、XML署名の長期検証を実現するXAdES（XML Advanced Electronic Signature）と類似した長期検証のアプローチと、JWS JSON Serializationのformatをベースとすることで、長期検証可能な長期署名フォーマットを実現する、JWS拡張仕様である。）

In addition, this specification supports an indirect signing model using external references (refs), allowing multiple arbitrary files, including non-JSON data, to be used as indirect signature targets. This indirect signing mechanism enables JWS to be used as a more general-purpose signature.
（また署名対象としてJSONデータ以外の任意の複数ファイルを利用可能とするため、外部参照（refs）による間接署名（detached reference）の仕組みをサポートする。外部参照の仕組みによりJWSをより汎用的な署名として利用できる。）

LTV-JWS is designed for practical use over the Internet by enabling lightweight implementation and operation through the addition of only minimal structures and attributes to JWS.
In particular, signing inputs and hash inputs follow the JWS signing input model, in which BASE64URL-encoded elements are concatenated using the period "." character, thereby simplifying implementation and improving interoperability.
（LTV-JWSはインターネット上で手軽に利用することを目的として、JWSに対して最小限の構造と属性を追加することで、軽量な実装と利用を可能とする。特に署名対象やハッシュ対象はJWSの署名入力形式に従い、BASE64URLエンコードされた要素をピリオド"."で結合する仕様とすることで、実装を容易とし、相互運用性を高める。）

By using LTV-JWS, various types of JSON data and arbitrary data formats used on the Internet can be verified for authenticity over extended periods of time.
（LTV-JWSを用いることで、インターネットで利用される各種のJSONデータおよび任意形式のデータを、長期間にわたり真正性を検証可能とすることができる。）

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

# Overview

## LTV Diagram
## Goals
## Design Principles
## Relationship to JWS


# Data Model

## JWS Structure

## Protected Header

LTV-JWS において、protected ヘッダには以下を含めなければならない（MUST）：

- "alg"：署名アルゴリズム
- "crit"：配列であり "ltv" を含まなければならない
- "ltv"：LTV 拡張オブジェクト

"crit" に "ltv" を含めることにより、本仕様を理解しない実装は
当該 JWS を拒否しなければならない（MUST）。

protected 内の "ltv" オブジェクトは署名対象に含まれ、
完全性が保護される。

## Unprotected Header

LTV-JWS において、署名後に追加される検証情報およびタイムスタンプは、
unprotected header に格納する。

header 内の "ltv" オブジェクトは署名対象に含まれないため、
単体では信頼してはならない（MUST NOT）。

これらの情報は、タイムスタンプまたはアーカイブ構造により
間接的に保護される。

## Payload

payload は署名対象データを格納する。

LTV-JWS では以下のいずれかの形式を利用する：

1. Attached 形式：
   payload に BASE64URL エンコードされたデータを格納する

2. 間接署名形式：
   payload に "ltv.refs" を含む JSON オブジェクトを格納する

payload を省略する Detached 形式は使用すべきではない（SHOULD NOT）。

## Signature

# LTV Objects

## ltv Protected Header Object

### "signingTime" Parameter

### "signingCertHash" Object

#### "hashAlg" Parameter
The values "S256", "S384", and "S512" are shorthand identifiers for SHA-256, SHA-384, and SHA-512 respectively.

#### "hashValue" Parameter

## ltv Unprotected Header Object

### "validations" Object

#### "certs" Parameter

#### "crls" Parameter

#### "ocsps" Parameter

### "timestamp" Object

#### "tst" Parameter

#### "type" Parameter

#### "validations" Object
Uses the same "validations" object structure described above.

### "archive" Object

#### "payload" Object (refs renew digests)

#### "timestamp" Object
Uses the same "timestamp" object structure described above.

#### "archive" (recursive Object)

## ltv Payload Object

### "refs" Array

#### "uri" Parameter

#### "hashAlg" Parameter
Same as the "hashAlg" parameter described above.

#### "hashValue" Parameter
Same as the "hashValue" parameter described above.

#### "type" Parameter

# Processing

## SIG-B (Signature Base level)
### External Reference Hash Construction
#### type=raw (default)
#### type=jws (Serial Signing)
### Signature Input Construction
### Signature Generation
### Signature Validation

## SIG-T (Signature Timestamp level)
### Signature Timestamp Input Construction
### Signature Timestamp Generation
### Signature Timestamp Validation

## SIG-LTV (Signature Long-Term Validation level)
### Validation Information Construction
#### from SIG-T
#### from SIG-LTA
### Validation Information Embedding
### Validation Information Validation

## SIG-LTA (Signature Long-Term Archive timestamp level)
### Archive Timestamp Input Construction
### Archive Timestamp Generation
### Archive Timestamp Validation
### Addition Validation Information and Next Archive Timestamp

# Security Considerations

## Trust Model of Unprotected Header Information
## Validation Requirements for External References
## Timestamp and TSA Validation
## Recursive Archive Validation
## Cryptographic Algorithm Agility
## Serial Signing Considerations
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


