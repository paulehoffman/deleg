---
title: Extensible Delegation for DNS
abbrev: DELEG
docname: draft-dnsop-deleg-latest
date: {DATE}
category: std
updates: 1035

ipr: trust200902
submissiontype: IETF
workgroup: dnsop
area: Internet
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
-
    ins: T. April
    name: Tim April
    organization:
    email: ietf@tapril.net
-
    ins: P.Špaček
    name: Petr Špaček
    organization: ISC
    email: pspacek@isc.org
-
    ins: R.Weber
    name: Ralf Weber
    organization: Akamai Technologies
    email: rweber@akamai.com
-
    ins: D.Lawrence
    name: David C Lawrence
    organization: Salesforce
    email: tale@dd.org

contributor:
-
    name: Christian Elmerot
    organization: Cloudflare
    email: christian@elmerot.se
-
    name: Edward Lewis
    organization: ICANN
    email: edward.lewis@icann.org
-
    name: Roy Arends
    organization: ICANN
    email: roy.arends@icann.org
-
    name: Shumon Huque
    organization: Salesforce
    email: shuque@gmail.com
-
    name: Klaus Darilion
    organization: nic.at
    email: klaus.darilion@nic.at
-
    name: Libor Peltan
    organization: CZ.nic
    email: libor.peltan@nic.cz
-
    name: Vladimír Čunát
    organization: CZ.nic
    email: vladimir.cunat@nic.cz
-
    name: Shane Kerr
    organization: NS1
    email: shane@time-travellers.org
-
    name: David Blacka
    organization: Verisign
    email: davidb@verisign.com
-
    name: George Michaelson
    organization: APNIC
    email: ggm@algebras.org
-
    name: Ben Schwartz
    organization: Meta
    email: bemasc@meta.com
-
    name: Jan Včelák
    organization: NS1
    email: jvcelak@ns1.com
-
    name: Peter van Dijk
    organization: PowerDNS
    email: peter.van.dijk@powerdns.com
-
    name: Philip Homburg
    organization: NLnet Labs
    email: philip@nlnetlabs.nl
-
    name: Erik Nygren
    organization: Akamai Technologies
    email: erik+ietf@nygren.org
-
    name: Vandan Adhvaryu
    organization: Team Internet
    email: vandan@adhvaryu.uk
-
    name: Manu Bretelle
    organization: Meta
    email: chantr4@gmail.com

--- abstract
A delegation in the Domain Name System (DNS) is a mechanism that enables efficient and distributed management of the DNS namespace. It involves delegating authority over subdomains to specific DNS servers via NS records, allowing for a hierarchical structure and distributing the responsibility for maintaining DNS records.

An NS record contains the hostname of the nameserver for the delegated namespace. Any facilities of that nameserver must be discovered through other mechanisms. This document proposes a new extensible DNS record type, DELEG, which contains additional information about the delegated namespace and the capabilities of authoritative nameservers for the delegated namespace.
--- middle

# Introduction

In the Domain Name System {{!STD13}}, subdomains within the domain name hierarchy are indicated by delegations to servers which are authoritative for their portion of the namespace.  The DNS records that do this, called NS records, contain hostnames of nameservers, which resolve to addresses.  No other information is available to the resolver. It is limited to connect to the authoritative servers over UDP and TCP port 53.

This limitation is a barrier for efficient introduction of new DNS technology. New features come with additional overhead as they are constrained by the intersection of resolver and nameserver functionality. New functionality could be discovered insecurely by trial and error, or negotiated after first connection, which is costly and unsafe.

The proposed DELEG record type remedies this problem by providing extensible parameters to indicate capabilities that a resolver may use for the delegated authority, for example that it should be contacted using a transport mechanism other than DNS over UDP or TCP on port 53.

DELEG records are served with NS and DS records in the Authority section of DNS delegation type responses.  Standard behavior of legacy DNS resolvers is to ignore the DELEG type and continue to rely on NS and DS records (see compliance testing described in Appendix A).  Resolvers that do understand DELEG and its associated parameters can efficiently switch to the new mechanism.

The DELEG record leverages the Service Binding (SVCB) record format defined in {{?RFC9460}}, using a subset of the already defined service parameters, however as DELEG creates a zone cut and requires special processing from authoritative name servers as well as resolvers it has to be a new record type similar in handling to the DS record type.

DELEG can use AliasMode, inherited from SVCB, to insert a level of indirection to ease the operational maintenance of multiple zones by the same servers.  For example, an operator can have numerous customer domains all aliased to nameserver sets whose operational characteristics can be easily updated without intervention from the customers.  Most notably, this provides a method for addressing the long-standing problem operators have with maintaining DS records on behalf of their customers. This type of facility will be handled in separate documents.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

Terminology regarding the Domain Name System comes from {{?BCP219}}, with addition terms defined here:

* legacy name servers: An authoritative server that does not support the DELEG record.
* legacy resolvers: A resolver that does not support the DELEG record.

## Motivation for DELEG

* There is no secure way to signal capabilities or new features of an authoritative server, such as authenticated DNS-over-TLS. A resolver must resort to trial-and-error methods that can potentially fall victim to downgrade attacks.
* Delegation point NS records and glue address records are, by design, not DNSSEC signed. This presents a leap of faith. Spoofed delegation point NS records can be detected eventually if the delegated domain was signed, but only after traffic was sent to the (potentially) spoofed endpoint.
* The Registry, Registrar, Registrant (RRR) model has no formally defined role for  DNS operators. Consequently, registrants are the channel between DNS operators and registries/registrars on purely operational elements, such as adding NS records, modify DS records when rolling keys, etc. Deleg's AliasMode allows the registrants to delegate these facilities to a DNS Operator.

## Introductory Examples

To introduce DELEG record, this example shows the authority section of a DNS response that delegates a subdomain to another nameserver.


    example.com.  86400  IN DELEG  1 ns1.example.com. (
                    ipv4hint=192.0.2.1 ipv6hint=2001:DB8::1 )
    example.com.  86400  IN NS     ns1.example.com.
    ns1.example.com.    86400   IN  A  192.0.2.1
    ns1.example.com     86400   IN  AAAA    2001:DB8::1

In this example, the authoritative nameserver is delegating using the same parameters as regular DNS, but the delegation as well as the glue can be signed.

Like in SVCB, DELEG also offer the ability to use the Alias form of delegation. The example below shows an example where example.com is being delegated with a DELEG AliasMode record which can then be further resolved using standard SVCB to locate the actual parameters.

    example.com.  86400  IN DELEG 0   config2.example.net.
    example.com.  86400  IN NS     ns2.example.net.

The example.net authoritative server may return the following SVCB records in response to a query as directed by the above records.

    config2.example.net 3600    IN SVCB . (
                    ipv4hint=192.0.2.54,192.0.2.56
                    ipv6hint=2001:db8:2423::3,2001:db8:2423::4 )

The above records indicate to the client that the actual configuration for the example.com zone can be found at config2.example.net

Later sections of this document will go into more detail on the resolution process using these records.

## Goal of the DELEG record

The primary goal of the DELEG records is to provide zone owners a method to signal capabilities to clients how to connect and validate a subdomain. This method coexists with NS records in the same zone. 

The DELEG record is authoritative in the parent zone and, if signed, has to be signed with the key of the parent zone. The target of an alias record is an SVCB record that exists and can be signed in the zone it is pointed at, including the child zone.

## DNSSEC is RECOMMENDED

While DNSSEC is RECOMMENDED, unsigned DELEG records may be retrieved in a secure way from trusted, Privacy-enabling DNS servers using encrypted transports.

FOR DISCUSSION: This will lead to cyclical dependencies. A DELEG record can introduce a secure way to communicate with trusted, Privacy-enabling DNS servers. For that, it needs to be DNSSEC signed. 

### Preventing downgrade attacks

A flag in the DNSKEY record is used as a backwards compatible, secure signal to indicate to a resolver that DELEG records are present or that there is an authenticated denial of a DELEG record. Legacy resolvers will ignore this flag and use the DNSKEY as is.

Without this secure signal an on-path adversary can remove DELEG records and its RRsig from a response and effectively downgrade this to a legacy DNSSEC signed response.

## Facilities 

The DELEG record is extensible in such a way that future innovations in the domain name system, such as new methods of secure transport, message encoding, error reporting, etc, does not depend on a re-design of the DNS. 

# DELEG Record Type

The SVCB record allows for two types of records, the AliasMode and the ServiceMode. The DELEG record takes advantage of both and each will be described below in depth. The wire format of and the registry for the DELEG record is the same as SVCB record defined in  {{?RFC9460}}

## Difference between the records

This document uses two different resource record types. Both records have the same functionality, with the difference between them being that the DELEG record MUST only be used at a delegation point, while the SVCB is used as a normal resource record and does not indicate that the label is being delegated. For example, take the following DELEG record:

    Zone com.:
    example.com.  86400  IN DELEG 1   config2.example.net.

When a client receives the above record, the resolver should send queries for any name under example.com to the nameserver at config2.example.net unless further delegated. By contrast, when presented with the records below:

    Zone com.:
    example.com.  86400  IN DELEG 0   config3.example.org.

    Zone example.org.:
    config3.example.org.  86400  IN SVCB 1 . ( ipv4hint=192.0.2.54,192.0.2.56
                    ipv6hint=2001:db8:2423::3,2001:db8:2423::4 )

A resolver trying to resolve a name under example.com would get the first record above from the parent authoritative server, .COM, indicating that the SVCB records found at config3.example.org should be used to locate the authoritative nameservers of example.com, and other parameters.

The primary difference between the two records is that the DELEG record means that anything under the record label should be queried at the delegated server while the SVCB record is just for redirection purposes, and any names under the record's label should still be resolved using the same server unless otherwise delegated.

It should be noted that both DELEG and SVCB records may exist for the same label, but they will be in different zones. Below is an example of this:

    Zone com.:
    example.com.  86400  IN DELEG 0   c1.example.org.

    Zone example.org.:
    c1.example.org.  86400  IN DELEG  1   config3.example.net. (
                                ipv6hint=2001:db8:2423::3 )
    c1.example.org.  86400  IN NS test.c1.example.org.
    test.c1.example.org. 600 IN A 192.0.2.1

    Zone c1.example.org:
    c1.example.org.  86400  IN SVCB 1   config2.example.net. (
                        ipv6hint=2001:db8:4567::4  )
    c1.example.org.  86400  IN NS test.c1.example.org.
    test.c1.example.org. 600 IN A 192.0.2.1

In the above case, the DELEG record for c1.example.org would only be used when trying to resolve names at or below c1.example.org. This is why when an AliasMode DELEG or SVCB record is encountered, the resolver MUST query for the SVCB record associated with the given name.


## AliasMode Record Type

In order to take full advantage of the AliasMode of DELEG and SVCB, the parent, child, and resolver must support these records. When supported, the use of the AliasMode will allow zone owners to delegate their zones to another operator with a single record in the parent. If a resolver were to encounter an AliasMode DELEG or SVCB record, it would then resolve the name in the TargetName of the original record using SVCB RR type to receive either another AliasMode record or a ServiceMode SVCB record.

For example, if the name www.example.com was being resolved, the .com zone may issue a referral by returning the following record:

    example.com.    86400    IN  DELEG     0   config1.example.net.

The above record would indicate to the resolver that in order to obtain the authoritative nameserver records for example.com, the resolver should resolve the RR type SVCB for the name config1.example.net.

### Multiple Service Providers

Some zone owners may wish to use multiple providers to serve their zone, in which case multiple DELEG AliasMode records can be used. In the event that multiple DELEG AliasMode records are encountered, the resolver SHOULD treat those as a union the same way this is done with NS records, picking one at random for the first lookup and eventually discovering the others. How exactly DNS questions are directed and split between configuration sets is implementation specific:

    example.com.    86400    IN  DELEG     0   config1.example.net.
    example.com.    86400    IN  DELEG     0   config1.example.org.

\[ DRAFT NOTE: SVCB says that there "SHOULD only have a single RR". This ignores that but keeps the randomization part. Section 2.4.2 of SVCB \]

### Loop Prevention

The TargetName of an SVCB or DELEG record MAY be the owner of a CNAME record. Resolvers MUST follow CNAMEs as well as further alias SVCB records as normal, but MUST not allow more then 4 total lookups per delegation, with the first one being the DELEG referral and then 3 SVCB/CNAME lookups maximal.

Special care should be taken by both the zone owner and the delegated zone operator to ensure that a lookup loop is not created by having two AliasMode records rely on each other to serve the zone. Doing so may result in a resolution loop, and likely a denial of service. The mechanism on following CNAME and SVCB alias above should prevent exhaustion of server resources. If a resolution can not be found after 4 lookups the server should reply with a SERVFAIL error code.

## Deployment Considerations

The DELEG and SVCB records are intended to replace the NS record while also adding additional functionality in order to support additional transports for the DNS. Below are discussions of considerations for deployment.

### AliasMode and ServiceMode in the Parent

Both the AliasMode and ServiceMode records can be returned for the DELEG record from the parent. This is different from the SCVB  {{?RFC9460}} specification and only applies for the DELEG RRSet in the parent.


### Rollout

When introduced, the DELEG and SVCB records might not initially be supported by the DNS root or TLD operators. Zone owners may place these records into their zones before the zones above them have done so. However, doing so is only useful for further delegations down the tree as an SVCB record at the zone apex alone does not indicate a new delegation type. The only way to discover new delegations is with the DELEG record at the parent.


### Availability

If a zone operator removes all NS records before DELEG and SVCB records are implemented by all clients, the availability of their zones will be impacted for the clients that are using non-supporting resolvers. In some cases, this may be a desired quality, but should be noted by zone owners and operators.

## Response Size Considerations

For latency-conscious zones, the overall packet size of the delegation records from a parent zone to child zone should be taken into account when configuring the NS, DELEG and SVCB records. Resolvers that wish to receive DELEG and SVCB records in response SHOULD advertise and support a buffer size that is as large as possible, to allow the authoritative server to respond without truncating whenever possible.


# Implementation

This document introduces the concept of signaling capabilities to clients on how to connect and validate a subdomain. This section details the implementation specifics of the DELEG record for various DNS components.

## Including DELEG RRs in a Zone

A DELEG RRset MAY be present at a delegation point.  The DELEG RRset MAY contain multiple records. DELEG RRsets MUST NOT appear at a zone's apex.

A DELEG RRset MAY be present with or without NS or DS RRsets at the delegation point. 

Construction of a DELEG RR requires knowledge which implies communication between the
operators of the child and parent zones. This communication is an operational matter not covered by this document.

### Signing DELEG RRs

A DELEG RRset MUST be DNSSEC signed if the zone is signed.

If a signed zone contains DELEG records, the zone MUST be signed with a DNSKEY that has the DELEG flag set.

## Authoritative Name Servers

### Including DELEG RRs in a Response

If a DELEG RRset is present at the delegation point, the name server MUST return both the DELEG RRset and its associated RRSIG RR in the Authority section along with the DS RRset and its associated RRSIG RR and the NS RRset.

If no DELEG RRset is present at the delegation point, and the zone was signed with a DNSKEY that has the DELEG flag set, the name server MUST return the NSEC or NSEC3 RR that proves that the DELEG RRset is not present including its associated RRSIG RR along with the DS RRset and its associated RRSIG RR if present and the NS RRset, if present. 

Including these DELEG, DS, NSEC or NSEC3, and RRSIG RRs increases the size of referral messages. If space does not permit inclusion of these records, including glue address records, the name server MUST set the TC bit on the response.

### Responding to Queries for Type DELEG

DELEG records, when present, are included in referrals. When a parent and child are served from the same authoritative server, this referral will not be sent because the authoritative server will respond with information from the child zone. In that case, the resolver may query for type DELEG.

The DELEG resource record type is unusual in that it appears only on the parent zone's side of a zone cut.  For example, the DELEG RRset for the delegation of "foo.example" is part of the "example" zone rather than in the "foo.example" zone.  This requires special processing rules for both name servers and resolvers because the name server for the child zone is authoritative for the name at the zone cut by the normal DNS rules, but the child zone does not contain the DELEG RRset.

A DELEG-aware resolver sends queries to the parent zone when looking for a DELEG RR at a delegation point. However, special rules are necessary to avoid confusing legacy resolvers which might become involved in processing such a query (for example, in a network configuration that forces a DELEG-aware resolver to channel its queries through a legacy recursive name server).  The rest of this section describes how a DELEG-aware name server processes DELEG queries in order to avoid this problem.

The need for special processing by a DELEG-aware name server only arises when all the following conditions are met:

* The name server has received a query for the DELEG RRset at a zone cut.

* The name server is authoritative for the child zone.

* The name server is not authoritative for the parent zone.

* The name server does not offer recursion.

In all other cases, the name server either has some way of obtaining the DELEG RRset or could not have been expected to have the DELEG RRset, so the name server can return either the DELEG RRset or an error response according to the normal processing rules.

If all the above conditions are met, however, the name server is authoritative for the domain name being searching for, but cannot supply the requested RRset. In this case, the name server MUST return an authoritative "no data" response showing that the DELEG RRset does not exist in the child zone's apex.

### Priority of DELEG over NS and Glue Address records

DELEG-aware resolvers SHOULD prioritize the information in DELEG records over NS and glue address records.

# Privacy Considerations

All of the information handled or transmitted by this protocol is public information published in the DNS.

# Security Considerations

TODO: Fill this section out

## Availability of Zones Without NS

## Resolution Procedure

An example of a simplified DNS interaction after priming. This is a query for www.example.com type AAAA with DELEG-aware com and example.com authoritative servers. 

* Ask www.example.com qtype AAAA to a.root-servers.net the answer is:
    Answer section: (empty)
    Authority section:
        com.   172800 IN NS a.gtld-servers.net.
    Additional section:
        a.gtld-servers.net. 172800 IN AAAA 2001:db8:a83e::2:30
* Ask www.example.com qtype AAAA to a.gtld-servers.net the answer is:
        Answer section: (empty)
        Authority section:
            example.com.   172800 IN NS ns1.example.com.
            example.com.   172800 IN DELEG   1 config1.example.com.
                                        ( ipv6hint=2001:db8:440:1:1f::24 )
        Additional section:
            ns1.example.com. 172800 IN AAAA 2001:db8:322c::35:42
* Ask www.example.com qtype AAAA to config1.example.com (2001:db8:1:1f::24) the answer is:
            Answer section:
                www.example.com.  3600 IN AAAA 2001:db8:a0:322c::2
            Authority section: (empty)
            Additional section: (empty)

TODO: more resolution examples (e.g out of bailiwick)

### Failures when DELEG Delegation is Present

When a delegation using DELEG to a child is present, the resolver MUST use it and SERVFAIL if none of the configurations provided work.

# IANA Considerations

DELEG will use the SVCB IANA registry definitions in section 14.3 of {{!RFC9460}}.

The IANA has assigned a bit in the DNSKEY flags field (see Section 7 of {{!RFC4034}} for the DELEG bit (N).
--- back

# Legacy Test Results {#Testing}

In December 2023, Roy Arends and Shumon Huque tested two distinct sets of requirements that would enable the approach taken in this document.

* legacy resolvers ignore unknown record types in the authority section of referrals.
* legacy resolvers ignore an unknown key flag in a DNSKEY.

Various recent implmentations were tested (BIND, Akamai Cacheserve, Unbound, PowerDNS Recursor and Knot) in addition to various public resolver services (Cloudflare, Google, Packet Clearing House). All possible variations of delegations were tested, and there were no issues.
Further details about the specific testing methodology, please see test-plan.

# Acknowledgments {:unnumbered}

This document is heavily based on past work done by Tim April in 
{{?I-D.tapril-ns2}} and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* worked out resolution example including alias form delegation


# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

~~~
01234567890123456789012345678901234567890123456789012345678901234567891
~~~
