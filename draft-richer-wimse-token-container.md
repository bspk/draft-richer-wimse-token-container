---
title: "Multi-token Container Data Structure"
abbrev: "Multi-token Container Data Structure"
category: info

docname: draft-richer-wimse-token-container-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
#area: SEC
#workgroup: WIMSE

keyword:
 - access token
 - http

author:
 -
    fullname: Justin Richer
    organization: Bespoke Engineering
    email: ietf@justin.richer.org
    role: editor

normative:
    RFC8941:

informative:


--- abstract

In many use cases, particularly in workload environments, a single access token or similar security artifact is not sufficient for conveying the type of security and provenance information necessary. This draft describes a data model and format for an additive data container to address these concerns.


--- middle

# Introduction

In most HTTP-based protocols, there's a single item sent with the message to represent authentication and authorization - an OAuth access token, a username/password pair, or a TLS client certificate.

Within many API deployments, multiple independent services are tied together into webs of processes that work together in a concerted fashion to fulfill the incoming requests. These systems need a means for different nodes to:

- convey the current state of the transaction to processing nodes further down the graph
- augment this state with new information, optionally tied to existing elements of this state
- attest to the state at the time it was processed by the node
- work in a non-linear fashion

In order to address this set of concerns, this draft defines a data structure for a single element of transactional state as well as a container for holding multiples of these elements in a cohesive and addressable structure.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Single Element Containers

A single element container represents a token and its set of properties.

The token value is a string whose contents are opaque to the container. Commonly, the token value will contain things like an OAuth access token, a JWT assertion, or another security artifact. The token value is REQUIRED, MUST NOT be the empty string, is covered by the hash, and is immutable.

The single element has a deterministic hash that is calculated from the token value and several core properties.

## Properties

format:
: A string that indicates the format of the token value, to aid downstream nodes in reading and processing the token value. This is covered by the hash.

tag:
: A string that indicates an application-specific tag, to aid downstream nodes in identifying the source and purpose of a given single element within a collection. This is covered by the hash.

parents:
: An ordered set of single element hashes, indicating other single element containers that this element is dependent on or otherwise related to. This is covered by the hash.

signatures:
: An ordered set of signatures for this element, coupled with the identifiers of the key that created the signature. This is not covered by the hash.

## Calculating the Hash

In order to calculate a hash, the element needs to be canonicalized and serialized in a deterministic fashion. For the purposes of this exercise, a serialization algorithm is defined that is inspired by, but not compatible with, HTTP Structured Fields defined in [RFC8941]. An interoperable implementation of this data structure would need to define a more robust hash base generation algorithm. For example, this algorithm defines a fixed order for properties, and does not allow for extension. Additionally, the values are not appropriately mapped to robust serializations (accounting for escaping syntax-relevant characters like double quotes, semicolons, and equal signs). All of these would need to be addressed for a full specification.

1. Start with an empty string
1. Take the token value and represent it as an `sf-string` value (enclosed in double quotes with internal double quotes, control characters, and escape characters escaped).
1. If the `tag` property is set, append ";tag=" and the tag value.
1. If the `format` property is set, append ";format=" and the format value.
1. If the `parents` property is set and is not empty, append ";parents=("
    1. For each parent, append the hash value encoded in base64
    1. If there are more parents, append "," and repeat the previous step on the next parent
    1. When all parents are accounted for, append ")"
1. Take the resulting string and hash it using SHA256 (note: there should be some crypto agility here probably?)
1. Encode the resulting hash value in base64

Note that this algorithm does not include the `signatures` property in the hash calculation.

## Immutability

Once the hash is calculated on a single element, all properties included in the hash calculation MUST be immutable. Additional properties that would be including in the hash calculation MUST NOT be added to the single element.

Effectively this means that an element is considered immutable once it is added as the parent of another element, is added to a multiple element container, or is signed.

## Adding a Signature

To add a signature of a single element, the signer takes the hash value of the element, decoded from base64, as the input to the signature algorithm.

The signer calculates the signed value (note: there needs to be some form of crypto agility here) and adds it to the `signatures` dictionary with the key identifier as the key and the signed output (encoded in base64) as the value.

Note that since signatures are not covered by the element's hash, signatures can be added or removed from the set without violating the element's immutability requirement.

# Multiple Element Collection

The container for multiple single element containers is a dictionary, where the key is the hash of the single element and its set of properties (including signatures). The dictionary structure SHOULD preserve insertion order.

There are no properties tied to the collection itself, nor is there an identifier for the collection. There is not a provision to sign the entire collection, but it is possible to add an element that lists every other node as its parent and apply a signature to that node, effectively giving a signature over a specific state of the collection.

The elements are tied together using the `parent` property. If an element has a `parent` property, all hashes listed in that property MUST exist as keys in the collection before the single element can be added to the collection.


~~~ aasvg

,--------,              ,--------,
| token1 |              | token3 |
| tag=1  |<---------+---+ tag=b  |
`--------`  parents |   `--------`
    ^               v
    |          ,--------, parents ,--------,
    |          | token2 |<--------+ token4 |
    |  parents |        |         `--------`
     `---------+ format |
               `--------`

~~~



Adding an element with an unlisted parent MUST produce an error.

Adding an element with a conflicting hash MUST produce an error.

Single elements in the collection **MAY** be removed if they are not the target of any `parent` properties. Removing an element that is the `parent` of another element MUST produce an error.

# Allowable Actions

Due to the nature of the data structure, the following actions can be taken by processing nodes without affecting existing items in the data structure.

**Adding a new single element**:
: A new node can be added to the multiple element collection without affecting any of the existing elements in the collection.

**Adding a signature to an existing single element**:
: A new signature can be calculated on any single element without affecting its hash or relationship to any other elements in the collection.

**Removing an unattached single element**:
: An element that is not listed as the `parent` of any other node can be safely removed without affecting the collection.

**Re-ordering the collection**
: The collection can be re-ordered, though insertion order should be kept to facilitate processing of element relationships.

Consequently, if a node wants to attest to the existing state of a multiple-element collection, the node can add a new element to the collection that lists a sufficient set of existing nodes as `parents` and signs the new element.

# Example Implementation

An example implementation of this data structure is available at https://github.com/bspk/token-bucket

In this implementation, the single element structure is called a `Bucket`, and the multiple element container is called a `Crate`. Immutability is enforced from the time the hash is calculated.

An example of the program's output follows, where a single crate is filled with several interrelated buckets. The serialization here is inspired by, but not compatible with, HTTP Structured Fields defined in [RFC8941].

~~~
CNgfrWGz8iIRndkFis9RjozfdNUKnx5drNDl_qtmVNE="876ytghj4nb2ghj23rjnfdu o2i3rj asdflk 23r"
   ;tag=weird
   ;format=illgal-probably
   ;parents=(6IdWzEZCGGRbA_mFtcO2Msm2_J2n5lFgARFxcpuJen8)
   ;sig=(k2=MEUCIQDWKyxZdprVENPWrd12MCwXefLBYYa0_7S9wflpNWkTjAIgaCmRB4A0GHf62vV34I2An4UFJacoT3pL6xyNL5J12Wg),

6IdWzEZCGGRbA_mFtcO2Msm2_J2n5lFgARFxcpuJen8="2wsdfghgfr45tyhjkiuytg"
   ;tag=gateway
   ;format=jwt
   ;parents=(NJSQl7yrzij1338sfK4XDE9aP0tZzw7R71cySlQwQ7I)
   ;sig=(k1=KNLuDTHrf9f_rBDwFwsnE-MdGYgGEeqPhlKmEFRoFSoiZ65txKizNTsfW2TXvUJ3zq7BuQa4xn0riBYlfCUiyHUgz6F38LmZkEN9qhFdii5cBY5ite-hQV--2nsKdeyvGrh21wtLhQu4NRhu91lYppn_j8L30nRQ1RlbNlrgpAQ,k2=MEYCIQDJQ-lm_sFnVynXP1BvdwUpH5hwFIGpEPY2DBx9wY523QIhAIsQIlzwGtN_8sAor8eLp-O43N66XHT9wdGsm4RbXFz2),

fV_0qN9qExdZKa46O0i6UE_UfNkxltH_QZjMlZMrCJA="a"
   ;format=secure
   ;parents=(NJSQl7yrzij1338sfK4XDE9aP0tZzw7R71cySlQwQ7I,ISdNpXb-StTmZvWZDTZLHlzhmpyivMorHOSL3UKTP7k)
   ;sig=(k1=O1NlrhtrtQ9ykDTR2uBOwk6ofFcL1_WvF0tj3CkrRbbLaBwVxUVslswCa4iHCbza8GS1G4mhNhJXUJ6O-Y6xG9xupoql1QVl_04a4oJOOStfNnptZ8embVMksXfAJQDhog7XAtsZ0vWmG_VkekIG7ZmvbhfAsS9QalC6oMiiVQs),

NJSQl7yrzij1338sfK4XDE9aP0tZzw7R71cySlQwQ7I="8765trfghjuyt5rtghjki987y6tfghj"
   ;tag=api
   ;format=opaque
   ;sig=(k1=Os27rze-ZOKNAX0BuQUr91ztA59vJIUXznzoG87Ka0IkuwTcWjz6DDsDLhh42Qh9UGH2C6oO9n9TSc41Y0EhUWi8oGt3LKwjHzNm7ziwolosl_JmpaNUDWVn8VabxYGDFUGnieUGMfb7DqRi59qAs6RpH-qTEZMF3Ihw2mPZinc),

ISdNpXb-StTmZvWZDTZLHlzhmpyivMorHOSL3UKTP7k="vcxsawertghju7654rtyuikjhgfr54refghjukjhgtr54redfghj"
   ;tag=magic
   ;parents=(NJSQl7yrzij1338sfK4XDE9aP0tZzw7R71cySlQwQ7I,6IdWzEZCGGRbA_mFtcO2Msm2_J2n5lFgARFxcpuJen8)
   ;sig=(k2=MEUCIHS9M_TkQouH5Icb731gBmKhB5FCKNfiUu6bJc7x0H-tAiEAmVe0mlQcfqASWIGaJYkOc3f6e3iDopJS58YKYBTKSJA)

~~~

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Why Not HTTP-SF?

HTTP Structured Fields [RFC8941] defines a set of data structures and strict serializations for those structures that would be a perfect fit for this multi-token format. The serialization is HTTP-native, not requiring any additional encoding. Furthermore, SF seems to support all of the core elements we need: dictionaries, lists, properties, strings, etc. And finally, unlike general-purpose formats like JSON or CBOR, SF does not provide infinite recursion. Lists can only be two levels deep, dictionaries can't contain other dictionaries, etc.

Unfortunately, SF falls short of being able to naturally represent the data structure here in a couple key ways:

- the multi element collection uses hashes as keys, but binary objects (the natural representation for hashes) cannot be keys
- the value of the parents parameter is a list and the value of the signature parameter is a dictionary, but the values of parameters have to be bare items, not lists or dictionaries

For the purpose of this draft, a new syntax was invented to be able to show the core functions required. Ultimately, a fully robust serialization will need to be specified. Ideally that would be based on SF in some way, but initial experiments in defining that were met with having to follow undesirable workarounds such as using multiple headers to convey a single data structure.

# Acknowledgments
{:numbered="false"}

The author would like to thank UberEther for supporting the initial conception of this work. The author would also like to thank Pieter Kasselman, George Fletcher, and Rifaat Shekh-Yusef for feedback.

