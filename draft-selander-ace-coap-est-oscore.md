---
title: Protecting EST Payloads with OSCORE
abbrev: EST-oscore
docname: draft-selander-ace-coap-est-oscore-latest

# date: 2017-03-14

# stand_alone: true

ipr: trust200902
area: Security
wg: ACE Working Group
kw: Internet-Draft
cat: std

coding: utf-8
pi:    # can use array (if all yes) or hash here
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      -
        ins: G. Selander
        name: Göran Selander
        org: Ericsson AB
        email: goran.selander@ericsson.com
      -
        ins: S. Raza
        name: Shahid Raza
        org: RISE
        email: shahid.raza@ri.se
      -
        ins: M. Furuhed
        name: Martin Furuhed
        org: Nexus
        email: martin.furuhed@nexusgroup.com
      -
        ins: M. Vucinic
        name: Malisa Vucinic
        org: INRIA
        email: malisa.vucinic@inria.fr
      -
        ins: T. Claeys
        name: Timothy Claeys
        org: INRIA
        email: timothy.claeys@inria.fr


normative:

  RFC2119:
  RFC7049:
  RFC7252:
  RFC7959:
  RFC8152:
  RFC8613:
  I-D.ietf-ace-coap-est:


informative:

  RFC2985:
  RFC2986:
  RFC5272:
  RFC6347:
  RFC7228:
  RFC7030:
  RFC8392:
  I-D.ietf-6tisch-minimal-security:
  I-D.ietf-ace-oscore-profile:
  I-D.ietf-ace-oauth-authz:
  I-D.selander-lake-edhoc:
  I-D.ietf-core-oscore-groupcomm:
  I-D.raza-ace-cbor-certificates:


--- abstract


This document specifies public-key certificate enrollment procedures protected with application-layer security protocols suitable for Internet of Things (IoT) deployments. The protocols leverage payload formats defined in Enrollment over Secure Transport (EST) and existing IoT standards including the Constrained Application Protocol (CoAP), Concise Binary Object Representation (CBOR) and the CBOR Object Signing and Encryption (COSE) format. 

--- middle

# Introduction  {#intro}

One of the challenges with deploying a Public Key Infrastructure (PKI) for the Internet of Things (IoT) is certificate enrollment, because existing enrollment protocols are not optimized for constrained environments {{RFC7228}}.

One optimization of certificate enrollment targeting IoT deployments is specified in EST-coaps ({{I-D.ietf-ace-coap-est}}), which defines a version of Enrollment over Secure Transport {{RFC7030}} for transporting EST payloads over CoAP {{RFC7252}} and DTLS {{RFC6347}}, instead of secured HTTP.

This document describes a method for protecting EST payloads over CoAP or HTTP with OSCORE {{RFC8613}}. OSCORE specifies an extension to CoAP which protects the application layer message and can be applied independently of how CoAP messages are transported. OSCORE can also be applied to CoAP-mappable HTTP which enables end-to-end security for mixed CoAP and HTTP transfer of application layer data. Hence EST payloads can be protected end-to-end independent of underlying transport and through proxies translating between between CoAP and HTTP.

OSCORE is designed for constrained environments, building on IoT standards such as CoAP, CBOR {{RFC7049}} and COSE {{RFC8152}}, and has in particular gained traction in settings where message sizes and the number of exchanged messages needs to be kept at a minimum, see e.g. {{I-D.ietf-6tisch-minimal-security}}, or for securing multicast CoAP messages {{I-D.ietf-core-oscore-groupcomm}}. Where OSCORE is implemented and used for communication security, the reuse of OSCORE for other purposes, such as enrollment, reduces the implementation footprint.

Another way to optimize the performance of certificate enrollment and certificate based authentication is the use of more compact representations of EST payloads (see {{cbor-payloads}}) and of X.509 certificates (see {{I-D.raza-ace-cbor-certificates}}. 


## EST-coaps operational differences {#operational}

This specification builds on EST-coaps {{I-D.ietf-ace-coap-est}} but transport layer security provided by DTLS is replaced, or complemented, by protection of the application layer data. This specification deviates from EST-coaps in the following respects:

* The DTLS record layer is replaced, or complemented, with OSCORE.
* The DTLS handshake is replaced, or complemented, with the EDHOC key exchange protocol {{I-D.selander-lake-edhoc}} completing the analogy with EST-coaps. EDHOC can also leverage its support for static Diffie-Hellman keys. The latter enables that certificates containing static DH public keys can be used for authentication of a Diffie-Hellman key exchange.
   * The use of a Diffie-Hellman key exchange authenticated with certificates adds significant overhead in terms of message size and round trips which is not necessary for the enrollment procedure. The main reason for specifying this is to align with EST-coaps, and reuse a security protocol rather than defining a special security protocol for enrollment. {{alternative-auth}} discusses alternative authentication and secure communication methods.
* The EST payloads protected by OSCORE can be proxied between constrained networks supporting CoAP/CoAPs and non-constrained networks supporting HTTP/HTTPs with a CoAP-HTTP proxy protection without any security processing in the proxy.


# Terminology   {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{RFC2119}}. These
words may also appear in this document in lowercase, absent their
normative meanings.

This document uses terminology from {{I-D.ietf-ace-coap-est}} which in turn is based on {{RFC7030}} and, in turn, on {{RFC5272}}. 

# OSCORE and Authenticated Key Establishment
EST-oscore clients and servers MUST perform mutual authentication before EST-oscore functions and services can be accessed. Prior to the initial enrollment the client MUST be configured with an Implicit Trust Anchor (TA) {{RFC7030}} database, enabling the client to authenticate the server. During the initial enrollment the client SHOULD populate its Explicit TA database and use it for subsequent authentications. 

The EST-coaps specification {{I-D.ietf-ace-coap-est}} only supports certificate-based authentication during the DTLS handshake between the EST-coaps server and the client. This specification replaces the DTLS handshake with the certificate-based EDHOC key exchange protocol and provides additional authenticated methods in the {{alternative-auth}} 

The {{RFC5272}} specification describes proof-of-identity as the ability of an endpoint, i.e., client, to prove its possession of a private key which is linked to a certified public key. Additionally, {{RFC5272}} details how to use channel-binding information (extracted from the underlying TLS layer) to link the proof-of-identity to a proof-of-possession. A proof-of-possession is generated by the client when it signs the PKCS#10 Request during the enrollment phase. Connection-based proof-of-possession is OPTIONAL for EST-oscore clients and servers. In case of certificate-based EDHOC key establishment, a set of actions are required which are further described below.

## EDHOC   {#edhoc}
The EST-oscore client and server use the EDHOC key establishment protocol to populate the OSCORE security context. The endpoints MUST use public key certificates to perform mutual authentication. During the initial enrollment the Implicit TA database MUST contain certificates capable of authenticating the EST-oscore server. When the EST-oscore client issues a request to the /crts endpoint of the EST server, it SHALL return a bag of certificates to be installed in the Explicit TA database. 

The cryptographic material used for EST-oscore client authentication can either be

 * previously issued certificates (e.g., an existing certificate issued by the EST server); this could be a common case for simple re-enrollment of clients.
 * previously installed certificates (e.g., installed by the manufacturer). Manufacturer installed certificates are expected to have a very long life, as long as the device, but under some conditions could expire. In that case, the server MAY authenticate a client certificate against its trust store although the certificate is expired ({{sec-cons}}).
 
 When desired the client can use the EDHOC-Exporter API to extract channel-binding information and provide a connection-based proof-of possession. Channel-binding information is obtained as follows 
 
 edhoc-unique = EDHOC-Exporter("EDHOC Unique", length),
 
 where length equals the desired length of the edhoc-unique byte string. The client then adds the edhoc-unique byte string as a ChallengePassword in the attributes section of the PKCS#10 Request to prove that the client is indeed in control of the private key at the time of the EDHOC key exchange.


# Protocol Design and Layering 
EST-oscore uses CoAP {{RFC7252}} and Block-Wise {{RFC7959}} to transfer EST messages in the same way as {{I-D.ietf-ace-coap-est}}. {{fig-stack}} below shows the layered EST-oscore architecture. 

~~~~~~~~~~~

+------------------------------------------------+
|          EST request/response messages         |
+------------------------------------------------+
|   CoAP with OSCORE   |   HTTP with OSCORE      |
+------------------------------------------------+
|   UDP  |  DTLS/UDP   |   TCP   |   TLS/TCP     |
+------------------------------------------------+

~~~~~~~~~~~
{: #fig-stack title="EST protected with OSCORE."}
{: artwork-align="center"}

EST-oscore follows closely the EST-coaps and EST design. 
 

## Discovery and URI     {#discovery}
The discovery of EST resources and the definition of the short EST-coaps URI paths specified in Section 5.1 of {{I-D.ietf-ace-coap-est}}, as well as the new Resource Type defined in Section 9.1 of {{I-D.ietf-ace-coap-est}} apply to EST-oscore. Support for OSCORE is indicated by the "osc" attribute defined in Section 9 of {{RFC8613}}, for example:

~~~~~~~~~~~

     REQ: GET /.well-known/core?rt=ace.est.sen

     RES: 2.05 Content
   </est>; rt="ace.est";osc

~~~~~~~~~~~
 
## Mandatory/optional EST Functions {#est-functions}
The EST-oscore specification has the same set of required-to-implement functions as EST-coaps. The content of {{table_functions}} is adapted from Section 5.2 in {{I-D.ietf-ace-coap-est}} and uses the updated URI paths (see {{discovery}}).

| EST functions  | EST-oscore implementation   |
| /crts          | MUST                        |
| /sen           | MUST                        |
| /sren          | MUST                        |
| /skg           | OPTIONAL                    | 
| /skc           | OPTIONAL                    |
| /att           | OPTIONAL                    |
{: #table_functions cols="l l" title="Mandatory and optional EST-oscore functions"}

## Payload formats
Similar to EST-coaps, EST-oscore allows transport of the ASN.1 structure of a given Media-Type in binary format. In addition, EST-oscore uses the same CoAP Content-Format Options to transport EST requests and responses . {{table_mediatypes}} summarizes the information from Section 5.3 in {{I-D.ietf-ace-coap-est}}.

|  URI  | Content-Format                                       | #IANA |
| /crts | N/A                                            (req) |   -   |
|       | application/pkix-cert                          (res) |  287  |
|       | application/pkcs-7-mime;smime-type=certs-only  (res) |  281  |
| /sen  | application/pkcs10                             (req) |  286  |
|       | application/pkix-cert                          (res) |  287  |
|       | application/pkcs-7-mime;smime-type=certs-only  (res) |  281  |
| /sren | application/pkcs10                             (req) |  286  |
|       | application/pkix-cert                          (res) |  287  |
|       | application/pkcs-7-mime;smime-type=certs-only  (res) |  281  |
| /skg  | application/pkcs10                             (req) |  286  |
|       | application/multipart-core                     (res) |   62  |
| /skc  | application/pkcs10                             (req) |  286  |
|       | application/multipart-core                     (res) |   62  |
| /att  | N/A                                            (req) |   -   |
|       | application/csrattrs                           (res) |  285  |
{: #table_mediatypes cols="l l" title="EST functions and there associated Media-Type and IANA numbers"}

NOTE: CBOR is becoming a de facto encoding scheme in IoT settings. There is already work in progress on CBOR encoding of X.509 certificates {{I-D.raza-ace-cbor-certificates}}, and this can be extended to other EST messages, see {{cbor-payloads}}. 


## Message Bindings
The EST-oscore message characteristics are identical to those specified in Section 5.4 of {{I-D.ietf-ace-coap-est}}. It is RECOMMENDED that

  * The EST-oscore endpoints support delayed responses
  * The endpoints supports the following CoAP options: OSCORE, Uri-Host, Uri-Path, Uri-Port, Content-Format, Block1, Block2, and Accept.
  * The EST URLs based on https:// are translated to coap://, but with mandatory use of the CoAP OSCORE option.


## CoAP response codes
See Section 5.5 in {{I-D.ietf-ace-coap-est}}.

## Message fragmentation
The EDHOC key exchange with asymmetric keys and certificates for authentication can result in large messages. It is RECOMMENDED to prevent IP fragmentation, since it involves an error-prone datagram reconstitution. In addition, this specification targets resource constrained networks such as IEEE 802.15.4 where throughput is limited and fragment loss can trigger costly retransmissions. Even though ECC-based certificates are an improvement over the RSA or DSA counterparts, they can still amount to roughly a 1000 bytes per certificate depending on the used algorithms, curves, OIDs, Subject Alternative Names (SAN) and cert fields. Additionally, in response to a client request to /crts an EST-oscore server might answer with multiple certificates. This is another motivation for the need for more compact EST payloads, see {{cbor-payloads}}. This specification further employs the CoAP Block1 and Block2 fragmentation mechanisms as described in Section 5.6 of {{I-D.ietf-ace-coap-est}} to limit the size of the CoAP payload.


## Delayed Responses
See Section 5.7 in {{I-D.ietf-ace-coap-est}}.

# HTTP-CoAP registrar 
As is noted in Section 6 of {{I-D.ietf-ace-coap-est}}, in real-world deployments, the EST server will not always reside within the CoAP boundary.  The EST-server can exist outside the constrained network in a non-constrained network that does not support CoAP but HTTP, thus requiring an intermediary CoAP-to-HTTP proxy.

Since OSCORE is applicable to CoAP-mappable HTTP the EST payloads can be protected end-to-end between EST client and EST server independent of transport protocol or potential transport layer security which may need to be terminated in the proxy, see {{fig-proxy}}. When using the EDHOC key exchange protocol to establish a shared OSCORE security context, PKCS#10 request MAY be bound to the OSCORE security context using the procedure described in {{edhoc}}. The mappings between CoAP and HTTP referred to in Section 6 of {{I-D.ietf-ace-coap-est}} apply and the additional mappings resulting from the use of OSCORE are specified in Section 11 of {{RFC8613}}. 

OSCORE provides end-to-end security between EST Server and EST Client. The use of TLS and DTLS is optional.

~~~~~~~~~~~
                                       Constrained-Node Network
  .---------.                       .----------------------------.
  |   CA    |                       |.--------------------------.|
  '---------'                       ||                          ||
       |                            ||                          ||
   .------.  HTTP   .-----------------.   CoAP   .-----------.  ||
   | EST  |<------->|  CoAP-to-HTTP   |<-------->| EST Client|  ||
   |Server|  (TLS)  |      Proxy      |  (DTLS)  '-----------'  ||
   '------'         '-----------------'                         ||
                                    ||                          ||
       <------------------------------------------------>       ||
                        OSCORE      ||                          ||
                                    |'--------------------------'|
                                    '----------------------------'
~~~~~~~~~~~
{: #fig-proxy title="CoAP-to-HTTP proxy at the CoAP boundary."}
{: artwork-align="center"}


# Security Considerations  {#sec-cons}

TBD

# Privacy Considerations 

TBD

# IANA Considerations  {#iana}


# Acknowledgments 



--- back

# Other Authentication Methods {#alternative-auth}

In order to protect certificate enrollment with OSCORE, the necessary keying material (notably, the OSCORE Master Secret, see {{RFC8613}}) needs to be established between EST-oscore client and EST-oscore server. In this appendix we analyse alternatives to EDHOC authenticated with certificates, which was assumed in the body of this specification.


## TTP Assisted Authentication
Trusted third party (TTP) based provisioning, such as the OSCORE profile of ACE {{I-D.ietf-ace-oscore-profile}} assumes existing security associations between the client and the TTP, and between the server and the TTP. This setup allows for reduced message overhead and round trips compared to the full-fledged EDHOC key exchange. Following the ACE terminology the TTP plays the role of the Authorization Server (AS), the EST-oscore client corresponds to the ACE client and the EST-oscore server is the ACE Resource Server (RS).

~~~~~~~~~~~

+------------+                               +------------+
|            |                               |            |
|            | ---(A)- Token Request ------> |  Trusted   |
|            |                               |   Third    |
|            | <--(B)- Access Token -------  | Party (AS) |
|            |                               |            |
|            |                               +------------+
| EST-oscore |                                  |     ^
|   Client   |                                 (F)   (E)
|(ACE Client)|                                  V     |
|            |                               +------------+
|            |                               |            | 
|            | -(C)- Token + EST Request --> | EST-oscore | 
|            |                               | server (RS)|
|            | <--(D)--- EST response ------ |            |
|            |                               |            | 
+------------+                               +------------+


~~~~~~~~~~~
{: #fig-ttp title="Accessing EST services using a TTP for authenticated key establishment and authorization."}
{: artwork-align="center"}

During initial enrollment the EST-oscore client uses its existing security association with the TTP, which replaces the Implicit TA database, to establish an authenticated secure channel. The {{I-D.ietf-ace-oscore-profile}} ACE profile RECOMMENDS the use of OSCORE between client and TTP (AS), but TLS or DTLS MAY be used additionally or instead. The client requests an access token at the TTP corresponding the EST service it wants to access. If the client request was invalid, or not authorized according to the local EST policy, the AS returns an error response as described in Section 5.6.3 of {{I-D.ietf-ace-oauth-authz}}. In its responses the TTP (AS) SHOULD signal that the use of OSCORE is REQUIRED for a specific access token as indicated in section 4.3 of {{I-D.ietf-ace-oscore-profile}}. This means that the EST-oscore client MUST use OSCORE towards all EST-oscore servers (RS) for which this access token is valid, and follow Section 4.3 in {{I-D.ietf-ace-oscore-profile}} to derive the security context to run OSCORE. The ACE OSCORE profile RECOMMENDS the use of CBOR web token (CWT) as specified in {{RFC8392}}. The TTP (AS) MUST also provision an OSCORE security context to the EST-oscore client and EST-oscore server (RS), which is then used to secure the subsequent messages between the client and the server.  The details on how to transfer the OSCORE contexts are described in section 3.2 of {{I-D.ietf-ace-oscore-profile}}.

Once the client has retrieved the access token it follows the steps in {{I-D.ietf-ace-oscore-profile}} to install the OSCORE security context and presents the token to the EST-oscore server. The EST-oscore server installs the corresponding OSCORE context and can either verify the validity of the token locally or request a token introspection at the TTP. In either case EST policy decisions, e.g., which client can request enrollment or reenrollment, can be implemented at the TTP. Finally the EST-oscore client receives a response from the EST-oscore server.


## PSK Based Authentication
Another method to bootstrap EST services requires a pre-shared OSCORE security context between the EST-oscore client and EST-oscore server. Authentication using the Implicit TA is no longer required since the shared security context authenticates both parties. The EST-oscore client and EST-oscore server need access to the same OSCORE Master Secret as well as the OSCORE identifiers (Sender ID and Recipient ID) from which an OSCORE security context can be derived, see {{RFC8613}}. Some optional parameters may be provisioned if different from the default value:

  * an ID context distinguishing between different OSCORE security contexts to use,
  * an AEAD algorithm,
  * an HKDF algorithm,
  * a master salt, and 
  * a replay window size.



## EDHOC with alternative authentication methods
{{edhoc}} describes the use of EDHOC in combination with certificates during the initial bootstrap phase. The latter requires that the Implicit TA is equipped with certificates capable of authenticating the EST-oscore server. Since EDHOC also supports authentication with RPKs and PSKs, the Implicit TA can be populated alternatively with RPKs and PSKs. Similarly to {{edhoc}}, client authentication can be performed with long-lived RPKs or long-lived PSKs, installed by the manufacturer. Re-enrollment requests can be authenticated through a valid certificate issued previously by the EST-oscore server or by using the key material available in the Implicit TA.


# CBOR Encoding of EST Payloads {#cbor-payloads}

Current EST based specifications transport messages using the ASN.1 data type declaration. It would be favorable to use a more compact representation better suitable for constrained device implementations. In this appendix we list CBOR encodings of requests and responses of the mandatory EST functions (see {{est-functions}}).


## Distribution of CA Certificates (/crts) ##

The EST client can request a copy of the current CA certificates.
In EST-coaps and EST-oscore this is done using a GET request to /crts (with empty payload). 
The response contains a chain of certificates used to establish an Explicit Trust Anchor database for subsequent authentication of the EST server. 

CBOR encoding of X.509 certificates is specified in {{I-D.raza-ace-cbor-certificates}}. CBOR encoding of certificate chains is specified below. This allows for certificates encoded using the CBOR certificate format, or as binary X.509 data wrapped as a CBOR byte string. 

CDDL:

~~~~~~~~~~~
certificate chain = (
   + certificate : bstr
)
certificate = x509_certificate / cbor_certificate
~~~~~~~~~~~

## Enrollment/Re-enrollment of Clients (/sen, /sren) ##

Existing EST standards specify the enrollment request to be a PKCS#10 formated message {{RFC2986}}. The essential information fields for the CA to verify are the following:

* Information about the subject, here condensed to the subject common name, 
* subject public key, and
* signature made by the subject private key.

CDDL:

~~~~~~~~~~~
certificate request = (
   subject_common_name : bstr,
   public_key : bstr
   signature : bstr,
   ? ( signature_alg : int, public_key_info : int )
)
~~~~~~~~~~~

The response to the enrollment request is the subject certificate, for which CBOR encoding is specified in {{I-D.raza-ace-cbor-certificates}}.

The same message content in request and response applies to re-enrollment.

TBD  PKCS#10 allows inclusion of attributes, which can be used to specify extension requests, see Section 5.4.2 in {{RFC2985}}. Are these attributes important for the scenarios we care about, or can we allow the CA to decide this?

### CBOR Certificate Request Examples ###

Here is an example of CBOR encoding of certificate request as defined in the previous section. 


114 bytes:

(
  h'0123456789ABCDF0',
  h'61eb80d2abf7d7e4139c86b87e42466f1b4220d3f7ff9d6a1ae298fb9adbb464',
  h'30440220064348b9e52ee0da9f9884d8dd41248c49804ab923330e208a168172dcae1
  27a02206a06c05957f1db8c4e207437b9ab7739cb857aa6dd9486627b8961606a2b68ae'
)

In the example above the signature is generated on an ASN.1 data structure. To validate this, the receiver needs to reconstruct the original data structure. Alternatively, in native mode, the signature is generated on the profiled data structure, in which case the overall overhead is further reduced.


### ASN.1 Certificate Request Examples ###

A corresponding certificate request of the previous section using ASN.1 is shown in {{fig-ASN1}}.

~~~~~~~~~~~
SEQUENCE {
   SEQUENCE {
     INTEGER 0
     SEQUENCE {
       SET {
         SEQUENCE {
           OBJECT IDENTIFIER commonName (2 5 4 3)
           UTF8String '01-23-45-67-89-AB-CD-F0'
           }
         }
       }
     SEQUENCE {
       SEQUENCE {
         OBJECT IDENTIFIER ecPublicKey (1 2 840 10045 2 1)
         OBJECT IDENTIFIER prime256v1 (1 2 840 10045 3 1 7)
         }
       BIT STRING
         (65 byte public key)
       }
   SEQUENCE {
     OBJECT IDENTIFIER ecdsaWithSHA256 (1 2 840 10045 4 3 2)
     }
   BIT STRING
     (64 byte signature)
~~~~~~~~~~~
{: #fig-ASN1 title="ASN.1 Structure."}
{: artwork-align="center"}


In Base64, 375 bytes:

~~~~~~~~~~~
-----BEGIN CERTIFICATE REQUEST-----
MIHcMIGEAgEAMCIxIDAeBgNVBAMMFzAxLTIzLTQ1LTY3LTg5LUFCLUNELUYwMFkw
EwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEYeuA0qv31+QTnIa4fkJGbxtCINP3/51q
GuKY+5rbtGSeZn3l8rVbU0jVEBWvKhAd98JeqgsuauGHRNWt2FqJ1aAAMAoGCCqG
SM49BAMCA0cAMEQCIAZDSLnlLuDan5iE2N1BJIxJgEq5IzMOIIoWgXLcrhJ6AiBq
BsBZV/HbjE4gdDe5q3c5y4V6pt2UhmJ7iWFgaitorg==
-----END CERTIFICATE REQUEST-----
~~~~~~~~~~~

In hex, 221 bytes:

~~~~~~~~~~~
3081dc30818402010030223120301e06035504030c1730312d32332d34352d36
372d38392d41422d43442d46303059301306072a8648ce3d020106082a8648ce
3d0301070342000461eb80d2abf7d7e4139c86b87e42466f1b4220d3f7ff9d6a
1ae298fb9adbb4649e667de5f2b55b5348d51015af2a101df7c25eaa0b2e6ae1
8744d5add85a89d5a000300a06082a8648ce3d04030203470030440220064348
b9e52ee0da9f9884d8dd41248c49804ab923330e208a168172dcae127a02206a
06c05957f1db8c4e207437b9ab7739cb857aa6dd9486627b8961606a2b68ae
~~~~~~~~~~~

# Other Explicit TA Material
TBD Currently all EST specifications provide the /crts (or /cacerts) endpoint. A successful request from the client to this endpoint will be answered with a bag of certificates which is subsequently installed in the Explicit TA. Should we optionally support new endpoints, e.g., /rpks and/or /psks, which would return a set or RPKs and PSKs to be installed in the Explicit TA? Support for this type of key material in the Explicit TA result in smaller messages sizes. 

--- fluff

