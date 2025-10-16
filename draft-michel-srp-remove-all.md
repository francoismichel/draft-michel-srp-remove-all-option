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

This document describes a new EDNS(0) option for an SRP Update to remove all previously registered services for a name before adding new services for that same name.
This allows a host to replace all its previous registrations using a single
SRP Update.


--- middle

# Introduction

Some constrained devices may not afford storing their currently registered services to persistent memory and only store the key.
Upon a reboot, they ensure no stale services remain by first sending an SRP Update removing all their previously registered services.
Once that is done, they register their new services through a new SRP Update.
Since removing services sets the Update Lease option to zero and adding service sets the same option to a nonzero value, SRP {{SRP}} prevents from removing all services and registering new services for a same name in the same Update ({{Section 3.2.5.5.1 of SRP}}).
This has to be done using two separate, successive updates.
This document defines a new EDNS(0) {{EDNS0}} option called SRP Remove All Services allowing to include the previous services removal operation in the Update that registers the new services.
This significantly reduces the amount of data needed for doing these operations and reduces congestion with SRP in constrained networks.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The SRP Remove All Services EDNS(0) option

The SRP Remove All Services Option has a length of zero and therefore has no payload.
Its presence in an SRP Update for a particular hostname asks the registrar to first remove all published services for that hostname before processing the Update contained in the packet.
It is almost equivalent to first sending an SRP Update containing one "Delete All RRsets From A Name" instruction before sending this update. The only difference is that when the SRP Remove All Services Option is used, the "Delete All RRsets From A Name" operation and the subsequent update are considered as a single transaction that entirely succeeds or fails.

# Server-side processing of the SRP Remove All Services Option

An SRP registrar receiving a valid SRP Update containing the SRP Remove All Services Option first removes all service registrations for the hostname in the Host Description Instruction.
It then processes the payload of the SRP Update.
In response to an update containing the SRP Remove All Services Option, the SRP registrar MUST include the option in its response to indicate that it is supported. This is done regardless of whether any of the "Delete All RRsets From A Name" operation induced by the option or the update contained in the packet succeed or fail.

## Error cases

If the "Delete All RRsets From A Name" operation induced by the SRP Remove All Services Option results in an error on the SRP registrar, it SHOULD immediately stop processing the SRP Update and MUST return the adequate response code as it would have done it when processing a regular "Delete All RRsets From A Name".

If the "Delete All RRsets From A Name" operation succeeds but the SRP Update contained in the packet fails, the "Delete All RRsets From A Name" operation is undone and the adequate error response code for the Update failure is returned {{SRP}}.

# Security Considerations

The SRP Remove All Services Option relies on existing security mechanisms defined in {{SRP}}. The client MUST be properly authenticated for the hostname before the registrar processes the "Delete All RRsets From A Name" operation induced by the option. Since an SRP attacker can replay any SRP Update, it can also replay the "Delete All RRsets From A Name" operation induced by the option.

# IANA Considerations

IANA is requested to allocate a new OPT RR option code from the DNS EDNS0 Option Codes (OPT) registry for the 'SRP Remove All Services' Option. The Name shall be 'SRP-RAS'. The value shall be allocated by IANA. The meaning shall be 'SRP Remove All Services'. Reference shall refer to this document, once published. IANA shall determine the registration date.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
