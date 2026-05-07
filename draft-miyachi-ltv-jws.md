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

本仕様は、JSON Web Signature（JWS, RFC 7515）を拡張し、
長期検証（Long-Term Validation: LTV）可能な署名を実現する
軽量な長期署名フォーマット LTV-JWS を定義する。

LTV-JWS は、署名属性、タイムスタンプ、検証情報（証明書、CRL、OCSP）、
およびアーカイブ構造を最小限の拡張として導入することで、
電子証明書の有効期限後および暗号アルゴリズムの危殆化後においても
署名の有効性を継続的に検証可能とする。

LTV-JWS は、JWS のシンプルな構造とコンセプトを維持しつつ、
外部参照（refs）による間接署名および段階的な検証情報の追加を
可能とする。

# Status of This Memo

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79.

# Introduction

JSONデータの真正性を保証するためには、多くの場合、JSON Web Signatures
(JWS) [RFC7515] を使用します。JSONデータを長期保管する場合には、
署名証明書の有効期限切れや暗号アルゴリズムの危殆化が問題となる。
長期検証（Long-Term Validation: LTV）のアプローチは、検証情報（証明書、CRL、OCSP）
およびタイムスタンプ [RFC 3161] を追加して行くことで、段階的にJWSを保護する。

Long-Term Validation for JSON Web Signature (LTV-JWS) は、XML署名の
長期検証を実現するXAdES（XML Advanced Electronic Signature）と類似した
長期検証のアプローチと、JWS JSON Serializationをベースとすることで、
長期検証可能な長期署名フォーマットを実現する、JWS拡張仕様である。

また署名対象としてJSONデータ以外の任意の複数ファイルを利用可能とする
ため、外部参照（refs）による間接署名（Indirect Signing）の仕組みを
サポートする。外部参照の仕組みによりJWSをより汎用的な署名として利用できる。

LTV-JWSはインターネット上で手軽に利用することを目的として、JWSに対して
最小限の構造と属性を追加することで、軽量な実装と利用を可能とする。
特に署名対象やハッシュ対象はJWSの署名入力形式に従い、BASE64URLエンコード
された要素をピリオド"."で結合する仕様とすることで、実装を容易とし、
相互運用性を高める。

LTV-JWSを用いることで、インターネットで利用される各種のJSONデータおよび
任意形式のデータを、長期間にわたり真正性を検証可能とすることができる。

# Terminology

The key words "MUST", "SHOULD", and "MAY" in this document are to be
interpreted as described in RFC 2119.
- LTV: Long-Term Validation
- ES-BES, ES-T, ES-XL, ES-A
- Indirect Signing
- External Reference (refs)

# Overview

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

# LTV Elements

## ltv (protected)
### signingTime
### signingCertHash

## ltv (header)
### validations
### timestamp
### archive

## ltv (payload)
### refs


# Processing

## ES-BES
### External Reference Hash Construction
### Signature Input Construction
### Signature Generation
### Signature Validation

## ES-T
### Signature Timestamp Input Construction
### Signature Timestamp Generation
### Signature Timestamp Validation

## ES-XL
### Validation Information Construction
### Validation Information Embedding
### Validation Information Validation

## ES-A
### Archive Timestamp Input Construction
### Archive Timestamp Generation
### Archive Timestamp Validation
### Archive Timestamp Addition

# Security Considerations

TBD

# IANA Considerations

TBD

# References

## Normative References

- RFC 7515
- RFC 7518
- RFC 2119
- RFC 3161
- RFC 5280

## Informative References

TBD