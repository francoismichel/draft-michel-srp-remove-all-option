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
title: "Bandwidth optimization extensions for SRP"
abbrev: "SRP Bandwidth Optimizations"
category: std

docname: draft-michel-srp-bandwidth-optimizations-latest
number:
date:
v: 3
area: Internet
wg: DNSSD
venue:
  group: DNSSD
  type: Working Group
  mail: dnssd@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dnssd/
keyword:
 - SRP
 - remove
 - all
 - option

author:
 -
    fullname: François Michel
    organization: Apple
    email: f_michel@apple.com
 -
    fullname: Esko Dijk
    organization: IoTconsultancy.nl
    email: esko.dijk@iotconsultancy.nl

normative:
   SRP: RFC9665
   EDNS0: RFC6891

informative:

...

--- abstract

This document describes a new EDNS(0) option for an SRP Update to remove all previously registered services for a hostname before adding new services for that same hostname.
This allows an SRP requester to replace all its previous service registrations with new ones using a single
SRP Update.


--- middle

# Introduction

Some constrained devices may not afford storing the services, that they have currently registered to an SRP {{SRP}} registrar, in persistent memory.
Instead, they only store their hostname and their SRP public/private key pair.
Upon a reboot, they ensure no stale service registrations remain on the SRP registrar by first sending an SRP Update to remove all their previously registered services per {{Section 3.2.5.5.1 of SRP}}.
Once that is done, they register their current services through another SRP Update.
Since removing all services requires the lease time in the Update Lease option to be zero, and adding any service(s) requires the same option to have a nonzero lease value, SRP effectively prevents the removal of all previous services and registering new services for a same hostname in the same Update ({{Section 3.2.5.5.1 of SRP}}).
Therefore, this has to be done using two separate, successive SRP Updates.

This document defines a new EDNS(0) {{EDNS0}} option called SRP Optimizations for SRP Updates to require smaller data exchanges. The option enables omitting the key in an update, making the update significantly smaller without compromising the security guarantees of SRP updates.
It also allows including the previous services removal operation in the same SRP Update that registers the new services.
This also allows an SRP requester to send a single SRP Update that removes all its registered services, while keeping its hostname registered, which is not possible currently with {{SRP}}.
This significantly reduces the amount of data transmitted over the network for doing these operations and reduces the risk of congestion caused by the operation of SRP in constrained networks.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The SRP Optimisations EDNS(0) option

The SRP Optimisations Option contains the following one-byte bitmap:

~~~~
+---------------+
|7|6|5|4|3|2|1|0|
+---------------+
| (Reserved)|K|R|
+---------------+
~~~~

- R: When set in an SRP Update for a particular hostname (as defined in the Host Description Instruction) it signals to the registrar to first remove all published services for that hostname before processing the Service Discovery Instructions and Service Description Instructions contained in the Update.
It is almost equivalent to first sending an SRP Update as defined by {{Section 3.2.5.5.1 of SRP}} before sending this Update.
The only difference is that when the SRP Optimisations Option is used, the "Removing All Published Services" operation and the subsequent SRP Update are considered as a single atomic transaction that either entirely succeeds, or fails.

- K: When set, it signals that the key record for the hostname is omitted, asking the SRP registrar to use the key it has in its local database for verifying the Update signature.

# Server-side processing of the SRP Optimisations Option

This server describes the server-side behaviour when processing the option.
Every flag of the option is processed independently by the server.
In response to an Update containing the SRP Optimisations Option, the SRP registrar MUST include the option to indicate that the option is supported.
The value of the K and R flags in the response depends on the success of the corresponding operations.
This is done regardless of whether any of the additional operations induced by the option, or the instructions contained in the SRP Update, succeed or fail.

## Processing K=1

An SRP Update carrying the SRP Optimisations Option with K=1 does not contain any KEY RR. The SRP registrar retrieves the public key associated with the Host Description Instruction in its local database.
The keytag of the SIG(0) RR and the hostname can be used to identify the signer's public key more precisely.
If the SRP registrar cannot disambiguate the right key among several, it MAY try the different keys until one validates the signature correctly.
If no key successfully validates the signature, the registrar MUST stop processing the SRP update, reject the SRP Update as defined in {{SRP}} and set the K=1 to signal the error.

## Processing R=1

Upon receiving a valid SRP Update containing the SRP Optimisations Option with R set to 1, the SRP registrar first removes all service registrations for the hostname in the Host Description Instruction.
This includes all SRV/TXT records for all service instance names of which the SRV record has this hostname as a target.
It also includes all PTR records that point to these service instance names.
Then, it processes the remaining instructions of the SRP Update as defined by {{SRP}}.

### Error when processing "Delete All RRsets From A Name"

If the "Delete All RRsets From A Name" operations induced by R=1 results in an error on the SRP registrar, it SHOULD immediately stop processing the SRP Update. The registrar MUST return the adequate response code as it would have done in {{SRP}} when processing a regular "Delete All RRsets From A Name" and set R=1 in the response to signal that an error occurred when processing this operation.

### Error when processing the SRP Update

If all the "Delete All RRsets From A Name" operations implied by R=1 succeed, but the subsequent SRP Update processing fails, then all the implied "Delete All RRsets From A Name" operations are undone and the adequate error response code for the SRP Update failure is returned as defined by {{SRP}}. The response is sent with the R bit set to 0.

# Security Considerations

TODO: security considerations for K=1

The "Remove All Services" operation induced by R=1 relies on existing security mechanisms defined in {{SRP}}. The SRP requester MUST be properly authenticated for the hostname contained in the Host Description Instruction before the SRP registrar processes the "Delete All RRsets From A Name" operations induced by the option.

Since an SRP attacker can replay any SRP Update, it can also replay the "Delete All RRsets From A Name" operations induced by the option.

# IANA Considerations

IANA is requested to allocate a new OPT RR option code from the DNS EDNS0 Option Codes (OPT) registry for the 'SRP Optimisations' Option. The Name shall be 'SRP-OPTI'. The value shall be allocated by IANA. The meaning shall be 'SRP Optimisations'. Reference shall refer to this document, once published. IANA shall determine the registration date.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
