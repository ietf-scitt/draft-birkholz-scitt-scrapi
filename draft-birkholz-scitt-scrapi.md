---
title: SCITT Reference APIs
abbrev: SCRAPI
docname: draft-birkholz-scitt-scrapi-latest
stand_alone: true
ipr: trust200902
area: Security
submissiontype: IETF
wg: TBD
kw: Internet-Draft
cat: std
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

author:
- name: Henk Birkholz
  org: Fraunhofer SIT
  abbrev: Fraunhofer SIT
  email: henk.birkholz@sit.fraunhofer.de
  street: Rheinstrasse 75
  code: '64295'
  city: Darmstadt
  country: Germany
- ins: O. Steele
  name: Orie Steele
  organization: Transmute
  email: orie@transmute.industries
- ins: J. Geater
  name: Jon Geater
  organization: RKVST Inc.
  email: jon.geater@rkvst.com
  country: UK
  country: United States

normative:

  RFC7807:
  RFC7231:

informative:

  RFC2046:
  RFC6838:

--- abstract

Abstract Text

--- middle

# Introduction

Introduction Text

## Requirements Notation

{::boilerplate bcp14-tagged}

# Relation to Identity

The SCITT REST API is designed to support identifier systems that are currently relevant to supply chains, including DID, x509 and PGP.

In order to support these systems, the API must be aware of specific header parameters, in particular, `kid`, `x5u` and `x5c`.

The API enables implementers to deploy interoperable URIs for disclosing information feeds related to supply chain actors, and artifacts accessible via transparency services.

## Authenticating Clients

TBD (comments on OAuth / Client Attestation).

## Discovering Federation

TBD (comments on GAIN / OIDC).

## Discovering Feeds

TBD (comments on URLs / QR Codes).


# SCITT Reference REST API

## Messages

All messages are sent as HTTP GET or POST requests.

If the Transparency Service cannot process a client's request, it MUST return an HTTP 4xx or 5xx status code, and the body SHOULD be a JSON problem details object ({{RFC7807}}) containing:

- type: A URI reference identifying the problem.
To facilitate automated response to errors, this document defines a set of standard tokens for use in the type field within the URN namespace of: "urn:ietf:params:scitt:error:".

- detail: A human-readable string describing the error that prevented the Transparency Service from processing the request, ideally with sufficient detail to enable the error to be rectified.

Error responses SHOULD be sent with the `Content-Type: application/problem+json` HTTP header.

As an example, submitting a Signed Statement with an unsupported signature algorithm would return a `400 Bad Request` status code and the following body:

~~~json
{
  "type": "urn:ietf:params:scitt:error:badSignatureAlgorithm",
  "detail": "The Statement was signed with an algorithm the server does not support"
}
~~~

Most error types are specific to the type of request and are defined in the respective subsections below.
The one exception is the "malformed" error type, which indicates that the
Transparency Service could not parse the client's request because it did not comply with this document:

- Error code: `malformed` (The request could not be parsed).

Clients SHOULD treat 500 and 503 HTTP status code responses as transient failures and MAY retry the same request without modification at a later date.
Note that in the case of a 503 response, the Transparency Service MAY include a `Retry-After` header field per {{RFC7231}} in order to request a minimum time for the client to wait before retrying the request.
In the absence of this header field, this document does not specify a minimum.

### Register Signed Statement

#### Request

~~~
POST <Base URL>/entries
~~~

Headers:

- `Content-Type: application/cose`

Body: SCITT COSE_Sign1 message

#### Response

One of the following:

- Status 201 - Registration is successful.
  - Header `Location: <Base URL>/entries/<Entry ID>`
  - Header `Content-Type: application/json`
  - Body `{ "entryId": "<Entry ID"> }`

- Status 202 - Registration is running.
  - Header `Location: <Base URL>/operations/<Operation ID>`
  - Header `Content-Type: application/json`
  - (Optional) Header: `Retry-After: <seconds>`
  - Body `{ "operationId": "<Operation ID>", "status": "running" }`

- Status 400 - Registration was unsuccessful due to invalid input.
  - Error code `badSignatureAlgorithm`
  - TBD: more error codes to be defined

If 202 is returned, then clients should wait until Registration succeeded or failed
by polling the Registration status using the Operation ID returned in the response.
Clients should always obtain a Receipt as a proof that Registration has succeeded.

### Retrieve Operation Status

#### Request

~~~
GET <Base URL>/operations/<Operation ID>
~~~

#### Response

One of the following:

- Status 200 - Registration is running
    - Header: `Content-Type: application/json`
    - (Optional) Header: `Retry-After: <seconds>`
    - Body: `{ "operationId": "<Operation ID>", "status": "running" }`

- Status 200 - Registration was successful
    - Header: `Location: <Base URL>/entries/<Entry ID>`
    - Header: `Content-Type: application/json`
    - Body: `{ "operationId": "<Operation ID>", "status": "succeeded", "entryId": "<Entry ID>" }`

- Status 200 - Registration failed
    - Header `Content-Type: application/json`
    - Body: `{ "operationId": "<Operation ID>", "status": "failed", "error": { "type": "<type>", "detail": "<detail>" } }`
    - Error code: `badSignatureAlgorithm`
    - [TODO]: more error codes to be defined, see [#17](https://github.com/ietf-wg-scitt/draft-ietf-scitt-architecture/issues/17)

- Status 404 - Unknown Operation ID
    - Error code: `operationNotFound`
    - This can happen if the operation ID has expired and been deleted.

If an operation failed, then error details SHOULD be embedded as a JSON problem details object in the `"error"` field.

If an operation ID is invalid (i.e., it does not correspond to any submit operation), a service may return either a 404 or a `running` status.
This is because differentiating between the two may not be possible in an eventually consistent system.

### Retrieve Signed Statement

#### Request

~~~
GET <Base URL>/entries/<Entry ID>
~~~

Query parameters:

- (Optional) `embedReceipt=true`

If the query parameter `embedReceipt=true` is provided, then the Signed Statement is returned with the corresponding Registration Receipt embedded in the COSE unprotected header.

#### Response

One of the following:

- Status 200.
  - Header: `Content-Type: application/cose`
  - Body: COSE_Sign1

- Status 404 - Entry not found.
  - Error code: `entryNotFound`

### Retrieve Registration Receipt

#### Request

~~~
GET <Base URL>/entries/<Entry ID>/receipt
~~~

#### Response

One of the following:

- Status 200.
  - Header: `Content-Type: application/cbor`
  - Body: SCITT_Receipt
- Status 404 - Entry not found.
  - Error code: `entryNotFound`

The retrieved Receipt may be embedded in the corresponding COSE_Sign1 document in the unprotected header.

# Privacy Considerations

Privacy Considerations

# Security Considerations

Security Considerations

# IANA Considerations

Maybe

## Media Type Registration

This section requests registration of the "application/receipt+cose" media type {{RFC2046}} in
the "Media Types" registry in the manner described in {{RFC6838}}.

TODO: Consider negotiation for receipt as "JSON" or "YAML".
TODO: Consider impact of media type on "Data URIs" and QR Codes.

To indicate that the content is a SCITT Receipt:

* Type name: application
* Subtype name: receipt+cose
* Required parameters: n/a
* Optional parameters: n/a
* Encoding considerations: TODO
* Security considerations: TODO
* Interoperability considerations: n/a
* Published specification: this specification
* Applications that use this media type: TBD
* Fragment identifier considerations: n/a
* Additional information:
   Magic number(s): n/a
   File extension(s): n/a
   Macintosh file type code(s): n/a
* Person & email address to contact for further information: TODO
* Intended usage: COMMON
* Restrictions on usage: none
* Author: TODO
* Change Controller: IESG
* Provisional registration?  No


--- back

# Attic

Not ready to throw these texts into the trash bin yet.

