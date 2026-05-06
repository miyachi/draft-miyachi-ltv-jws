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
長期署名検証（Long-Term Validation: LTV）を実現する
軽量な署名フォーマット LTV-JWS を定義する。

LTV-JWS は、署名属性、タイムスタンプ、検証情報（証明書、CRL、OCSP）、
およびアーカイブ構造を最小限の拡張として導入することで、
証明書の有効期限後および暗号アルゴリズムの危殆化後においても
署名の有効性を継続的に検証可能とする。

本仕様は、JWS のシンプルな構造を維持しつつ、
外部参照（refs）による間接署名および段階的な検証情報の追加を
可能とする。

# Status of This Memo

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79.

# Introduction

TBD

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