# Long-Term Validation for JSON Web Signature (LTV-JWS)

This repository contains the working files for the Internet-Draft:

**Long-Term Validation for JSON Web Signature (LTV-JWS)**

* Draft: `draft-miyachi-ltv-jws`
* Author: Naoto Miyachi
* Status: Individual Internet-Draft
* Intended Status: Informational

## Abstract

LTV-JWS extends JSON Web Signature (JWS) with lightweight long-term validation capabilities.

The specification defines:

* Signature levels (SIG-B, SIG-T, SIG-LTV, SIG-LTA)
* RFC 3161 timestamps
* Embedded validation information (certificates, CRLs, OCSP responses)
* Archive timestamps for long-term preservation
* External references (`refs`) for indirect signing of arbitrary data

The design preserves the existing JWS processing model while enabling long-term validation similar to XAdES for XML signatures.

## Current Draft

* `draft-miyachi-ltv-jws`

The latest Internet-Draft is available from the IETF Datatracker:

<https://datatracker.ietf.org/doc/draft-miyachi-ltv-jws/>

## Repository Contents

```
README.md                    README
draft-miyachi-ltv-jws.md     Markdown source
draft-miyachi-ltv-jws.xml    RFCXML source
draft-miyachi-ltv-jws.txt    Internet-Draft text
draft-miyachi-ltv-jws.html   HTML rendering
examples/                    Example JSON files
```

## Building

This draft is written in Markdown and converted to RFCXML using kramdown-rfc.

Typical workflow:

```
Markdown
   ↓
RFCXML
   ↓
xml2rfc
   ├── TXT
   └── HTML
```

## Feedback

Comments and suggestions are welcome.

Please use the IETF JOSE Working Group mailing list for technical discussion.

Email: miyachi@langedge.jp

## License

This work is intended for publication as an IETF Internet-Draft and is subject to the IETF Trust Legal Provisions.
