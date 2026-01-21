---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Privacy Framework for Remote ATtestation procedureS"
abbrev: "RATS Privacy Framework"
category: info
# category: experimental

docname: draft-ounsworth-rats-privacy-framework-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: SEC
workgroup: RATS
keyword:
 - remote attestation
 - privacy
# venue:
#   group: RATS
#   type: Working Group
#   mail: WG@example.com
#   arch: https://example.com/WG
#   github: USER/REPO
#   latest: https://example.com/LATEST

author:
 -
    fullname: Mike Ounsworth
    organization: Cryptic Forest Software
    email: mike@ounsworth.ca

normative:

informative:
  CMW: I-D.draft-ietf-rats-msg-wrap
  GDPR:
    target: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:02016R0679-20160504
    title: "General Data Protection Regulation"
    author:
      org:
      - European Parliament
  WebAuthn2:
    target: https://www.w3.org/TR/webauthn-2/
    title: "Web Authentication: An API for accessing Public Key Credentials Level 2"
    author:
      org:
      - Word Wide Web Consortium
  IAB-Statement-2023:
    target: https://datatracker.ietf.org/doc/statement-iab-statement-on-the-risks-of-attestation-of-software-and-hardware-on-the-open-internet/
    title: "IAB Statement on the Risks of Attestation of Software and Hardware on the Open Internet"
    author:
      org:
      - "Internet Architecture Board"


...

--- abstract

Remote Attestation Evidence, ie the measured running state of a device, can sometimes contain sensitive data that requires sensitive handling.
This document defines several categories of sensitive measured claims, including Personally Identifiable Information (PII), Device Identifiers such as serial numbers or fingerprints which can be used for tracking and therefore act as a proxy for PII, and Vendor Info which can be used to enforce vendor lock-in and thus threaten the openness and interoperability of the Internet.

This document provides a framework whereby compliant devices categorize the claims they are capable of producing, maintain trust stores of Trusted Verifiers, and will only release sensitive claims to Verifiers in the correct trust store, and the sensitive claims only leave the device encrypted for a certificate belonging to the Trusted Verifier. This framework provides both access control for sensitive evidence claims, as well as transport confidentiality to protect against incidental logging and monitoring.


--- middle

# Introduction

## The Problem

In its conception, remote attestation focuses on measured boot where measurement claims evidence typically consists of measurements of hardware and firmware components and this data is typically not thought of as being privacy-sensitive. This document aims to challenge that assumption and provide a framework providing technical privacy controls under which remote attestation technology can safely grow to include sensitive claims.

PII, as defined by the European Union's General Data Protection Regulation [GDPR] defines PII as data that can be used, either in isolation or in aggregate with other data, to identify and de-anonymize a human.
This obviously includes names, user names, email addresses, and the like, which are unlikely to appear in measured boot evidence.
However, the GDPR also considers machine identifiers that can be used as a proxy for PII either in isolation or in aggregate, and often considers cookies, IP addresses, and the like to be protected as PII.
This starts to get much closer to the types of data that could be reported by a measured boot system; consider in particular the detailed software manifest of an end-user device such as a mobile phone, and whether the specific combination of apps on the device on which you're reading this document could be used to uniquely identify the device, and by proxy, you.
Similarly, remote attestation evidence container formats such as the RATS Conceptual Message Wrapper [CMW] can be used to carry authentication formats such as WebAuthn [WebAuthn2] which can directly carry PII such as usernames and other PII-proxies.


Device ID (DevID) is defined as any data that can, in isolation or in aggregate, with other data be used to uniquely identify the device. This includes the obvious things such as serial numbers of either the device or of certificates issued to the device, cryptographic public keys, and static addresses.
DevIDs also include data that act as a device fingerprint such as the manifest of connected peripheral devices, the manifest of installed software, the hash of customized config files, etc. On end-user devices, DevIDs quickly become a proxy for PII and must be protected as such under the GDPR and similar regulation.

Vendor Info is defined as any data that can, in isolation or in aggregate, be used to identify the manufacturer and model of the device.
This category of info is not known to have GDPR implications, but it is the subject of the 2023 IETF IAB Statement on the Risks of Attestation of Software and Hardware on the Open Internet [IAB-Statement-2023].
The statement notes that the Internet is built on a philosophy that any client which correctly implements the wire protocol can participate, regardless of its manufacturer.
The statement notes that remote attestation technology provides a mechanism whereby a service provider could restrict services based on the manufacturer of the client device.
While this may be a reasonable thing to do in a private enterprise environment, it represents an existential threat to the openness of the public Internet.
For this reason, Vendor Info is considered to be a sensitive category of remote attestation evidence that carries human rights implications.

A single remote attestation claim could be in more than one category.
For example, a detailed hardware or software manifest could be both a fingerprint-style DevID (and therefore PII in the case of an end-user device), and also Vendor Info.


## The Framework

This document provides a remote attestation privacy framework in two parts.

First is a conceptual framework for classifying evidence claims as PII, DevID, Vendor Info, or unclassified.

Second, is a technical framework whereby an attesting client (RATS Attester or Presenter) maintains one or more lists of Trusted Verifiers, represented by X.509 certificate trust stores, which are authorized to request evidence claims from one or more sensitivity categories.
This provides an enforcement point for access control, and leverages the commonly-understood implications of adding a root CA to your device.
Moreover, in addition to controlling their list of Trusted Verifiers, clients might also wish to protect against logging and monitoring of these claims in transit to the verifier, which is achieved by requiring the Verifier to provide an encryption certificate which is used by the Attester or Presenter to wrap the entire attestation payload containing sensitive claims in a JWE or CWE encrypted envelope. This confidentiality mechanism provides additional benefit to Relying Parties which need to carry Evidence to the Verifier but actually do not want to be in a position of handling GDPR-protected data.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Classification of Claims

For the purposes of this document, an attestation client is either an Attesting Environment directly, or a Presenter acting as a proxy between the Attester and the Verifier.
A conforming client is responsible for classifying every Evidence claim that it is capable of producing according to the following categories.

## PII

The definition of Personally Identifiably Information (PII) used here is intended to align precisely with the definition in GDPR or other relevant user privacy regulations. Implementers are free to identify PII as appropriate to their jurisdiction.
This document provides the following generalized definition:

> PII is any data that can be used, either in isolation or in aggregate with other data, to identify and de-anonymize a human.

## DevID

A Device Identifier is any data that can be used to uniquely identify a device.
DevIDs typically fall into one of two categories:

* **Stable Identifiers** such as serial numbers or unique addresses. Sometimes DevIDs are hiding in places that even the implementers might not be aware of, such as DNs, serial numbers, or public keys of certificates issued to the hardware at manufacture-time.
* **Fingerprints** is any collection of data that, while not necessarily unique to the device, often ends up being so in practice.
This includes hardware manifests such as the model numbers of all connected peripheral devices, or manifests of all installed user software.
Care should also be taken when returning a hash of firmware, software, or filesystem contents, especially where this is user-customizable, as this can quickly become a unique fingerprint as well.

In the case of end-user hardware such as laptops and mobile phones, DevIDs can often be used as a proxy for PII and are required to be handled as PII under the GDPR.
In the case of back-end datacenter and networking equipment, or IoT devices not directly related to a single human, DevIDs can be considered to be a distinct category from PII, and while not legally protected in the same way, is still considered sensitive as it can be used to monitor the movement of a physical or logical device across a network.

## Vendor Info

Vendor Info is any Evidence claim that can be used, in isolation or in aggregate, to identify the model and manufacturer of a device.
Obvious examples include version information about hardware, firmware, and software, or manufacture certificates.
But implementers SHOULD also be careful to consider fingerprints that are unique to a given device model or manufacturer, which could include hashes of hardware, firmware, or software, or could even include idiosyncracies in how the device handles certain requests or error conditions.

As noted in the introduction, the main privacy concern here relates to enabling proprietary vendor lock-in, forced obsolescence, or any other circumstance in which a device that is compliant to the established wire protocols is blocked from full participation in the service.
Such behavior is completely reasonable in a closed enterprise network. For example, a corporation, datacenter, or network provider is well within their rights to know exactly what is on their network, but on the open Internet this quickly develops into a human rights problem.
As such, it is in the best interest of end users that devices not disclose Vendor Info unless the device owner has explicitly consented to onboard the device into a network that requires this visibility.

## Unclassified

This document defines an unclassified claim as any claim that does not meet the definition of any of the categories above, and thus MAY be freely released from the device without any access control or confidentiality protection.
This document does not preclude the development of further categories of sensitive claims.

# Technical Framework

The technical framework presented here requires conformant implementations to do two things.
First, Clients MUST maintain lists of Trusted Verifiers and the categories of sensitive claims that they are authorized to request.
Second, Verifiers requesting sensitive claims are required to provide an encryption credential compatible with JWE or CWE in order for evidence to be encrypted for them.

## Trusted Verifiers

Conformant Clients, which could be either an Attesting Environment directly, or a Presenter acting as a proxy for the Attester, MUST have a mechanism to maintain trust stores of X.509 certificates representing Trusted Verifiers. The trusted certificates could be root CAs, self-signed certificates, end-entity certificates, or bare trust anchor public keys as per [?RFC5280] and [?RFC5914], or a JWK key as per [?RFC7517]. The details of maintaining a trust store and verifying a certificate or key against the trust store are left out of this document.

Conformant Clients SHOULD have a mechanism to identify which Trusted Verifiers are authorized to request which categories of sensitive claims, though clients MAY opt for a simpler all-or-nothing approach where a Trusted Verifier can view all categories.


## Evidence Encryption

Conformant Clients MUST NOT release attestation evidence containing sensitive claims in plaintext.
Instead, Verifies MUST provide an encryption credential chaining to a trust anchor in the relevant trust store, that can be used to encrypt the entire attestation payload.

Where the Verifier's encryption credential is an X.509 certificate, it MUST contain an encryption-type keyUsage of keyEncipherment, dataEncipherment, or keyAgreement and MUST contain the Extended Key Usage `id-kp-tbd-evidence-encryption`.

The encryption of the attestation payload is straightforward: if the attestation payload contains at least one claim of a sensitive category, then the entire payload MUST be wrapped inside a JWE [!RFC7516] or CWE [!RFC9052]. Where the attestation payload is a CMW [CMW], the choice of JWE or CWE encryption envelope SHOULD match the underlying type in order to reduce parser burden. The encrypted payload MAY be placed back into a container format such as CMW, but implementers need to be careful that the container metadata does not leak the sensitive information that we are trying to protect!

EDNOTE: does CMW not contain a mechanism for encrypted payloads? Why not?


# Security Considerations

TODO Security -- yeah, I'm sure we can think of some.


# IANA Considerations

This document has no IANA actions, unless we want to register the EKU OID `id-kp-tbd-evidence-encryption` here instead of needing a sister LAMPS doc.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
