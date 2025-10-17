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
title: "SRP Remove All Services EDNS(0) option"
abbrev: "SRP Remove All Services Option"
category: std

docname: draft-michel-srp-remove-all-latest
number:
date:
v: 3
area: INT
workgroup: dnssd
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

This document defines a new EDNS(0) {{EDNS0}} option called SRP Remove All Services allowing to include the previous services removal operation in the same SRP Update that registers the new services.
The option also allows an SRP requester to send a single SRP Update that removes all its registered services, while keeping its hostname registered, which is not possible currently with {{SRP}}.
This significantly reduces the amount of data transmitted over the network for doing these operations and reduces the risk of congestion caused by the operation of SRP in constrained networks.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The SRP Remove All Services EDNS(0) option

The SRP Remove All Services Option has a length of zero and therefore has no payload.
Its presence in an SRP Update for a particular hostname (as defined in the Host Description Instruction) signals to the registrar to first remove all published services for that hostname before processing the Service Discovery Instructions and Service Description Instructions contained in the Update.
It is almost equivalent to first sending an SRP Update as defined by {{Section 3.2.5.5.1 of SRP}} before sending this Update. The only difference is that when the SRP Remove All Services Option is used, the "Removing All Published Services" operation and the subsequent SRP Update are considered as a single atomic transaction that either entirely succeeds, or fails.

# Server-side processing of the SRP Remove All Services Option

An SRP registrar receiving a valid SRP Update containing the SRP Remove All Services Option first removes all service registrations for the hostname in the Host Description Instruction.
This includes all SRV/TXT records for all service instance names of which the SRV record has this hostname as a target.
It also includes all PTR records that point to these service instance names.
Then, it processes the remaining instructions of the SRP Update as defined by {{SRP}}.
In response to an Update containing the SRP Remove All Services Option, the SRP registrar MUST include the option in its SRP Update response to indicate that it is supported. This is done regardless of whether any of the additional operations induced by the option, or the instructions contained in the SRP Update, succeed or fail.

## Error cases

If the "Delete All RRsets From A Name" operations induced by the SRP Remove All Services Option results in an error on the SRP registrar, it SHOULD immediately stop processing the SRP Update and MUST return the adequate response code as it would have done when processing a regular "Delete All RRsets From A Name".

If all the "Delete All RRsets From A Name" operations implied by the option succeed, but the subsequent SRP Update processing fails, then all the implied "Delete All RRsets From A Name" operations are undone and the adequate error response code for the SRP Update failure is returned as defined by {{SRP}}.

# Security Considerations

The SRP Remove All Services Option relies on existing security mechanisms defined in {{SRP}}. The SRP requester MUST be properly authenticated for the hostname contained in the Host Description Instruction before the SRP registrar processes the "Delete All RRsets From A Name" operations induced by the option.

Since an SRP attacker can replay any SRP Update, it can also replay the "Delete All RRsets From A Name" operations induced by the option.

# IANA Considerations

IANA is requested to allocate a new OPT RR option code from the DNS EDNS0 Option Codes (OPT) registry for the 'SRP Remove All Services' Option. The Name shall be 'SRP-RAS'. The value shall be allocated by IANA. The meaning shall be 'SRP Remove All Services'. Reference shall refer to this document, once published. IANA shall determine the registration date.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
