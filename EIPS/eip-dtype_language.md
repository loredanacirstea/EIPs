---
eip: <to be assigned>
title: dType Language Extension, Data Bridging
author: Loredana Cirstea (@loredanacirstea), Christian Tzurcanu (@ctzurcanu)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2019-06-29
requires: 1900
---

## Simple Summary

This ERC is an extension of ERC-1900, proposing the addition of dType registry fields, in order to support type definitions for other programming languages.

## Abstract

The goal of ERC-1900 is to build a global type registry for Ethereum, to encourage project interoperability and the creation of a global OS. This ERC positions Ethereum as the origin source for a global type system across on-chain and off-chain programming languages.

## Motivation

The current software lacks interoperable standards between projects. This creates a fragmented web, which can have advantages (faster distributed research, diversity of incentives). However, fragmentation can also lead to research stagnation by not combining efforts and to fragile and time-consuming adaptors to bridge systems. Server endpoints can change anytime, liveness and availability is not an invariant.

In order to build a network of interoperable protocols, we need data standardization. Extending dType to other languages encourages interoperability between on-chain and off-chain type systems. There is an entire range of applications for global use that can be modularly build by multiple entities, in parallel (without monopoly), while maintaining consistency of types and endpoints (from education to governance frameworks).

dType will provide type definition and type checking tools, along with references to the generated source code that defines the type, in each language. On top of dType, data bridging and data serialization can be done between dType type instances and type instances from a specific language (using web3 tools) and between type instances from different languages (using tools such as [CBOR](https://cbor.io/)).

There are attempts to create a global type system, such as DefinitelyTyped(https://github.com/DefinitelyTyped/DefinitelyTyped), targeting TypeScript and stored on GitHub, and [Typedefs](https://typedefs.com/), an algebraic data type definition language, with an IPFS-stored content addressable repository of types in plan. However, the medium of storage should be immutable and decentralized. Additionally, the process of proposing, voting and accepting types must be decentralized, to avoid censorship or polluting the registry with duplicate or incomplete type definitions. Therefore, Ethereum is best choice at the moment.

## Specification

### dType Extension

We propose extending the `dType` struct from ERC-1900 to contain `uint16 language` as a field. We will therefore have:

```
struct dType {
    TypeChoices typeChoice;
    address contractAddress;
    bytes32 source;
    uint16 language;
    string name;
    dTypes[] types;
}
```

`language` will uniquely determine a programming language. Solidity will have `language = 0`, with the advantage of having lower storage cost for its type registration.

The type primary `identifier` from ERC-1900 will be defined by both language and name: `keccak256(abi.encodePacked(language, name))`.

The `dType` registry events will also be extended with an `uint16 language` argument for creating caches of all types per language.
```
event LogNew(bytes32 indexed identifier, uint256 indexed index, uint16 language);
event LogUpdate(bytes32 indexed identifier, uint256 indexed index, uint16 language);
event LogRemove(bytes32 indexed identifier, uint256 indexed index, uint16 language);
```

ERC-1900 defines `bytes32 source` as a Swarm (or another decentralized storage system) hash that identifies the type's source code file. For ERC-1900 this meant the Solidity source code containing the type library and other related contracts (e.g. a storage contract, as proposed in [ERC-2158](https://github.com/ethereum/EIPs/pull/2158)). This remains true for this ERC, while `language === 0`.

For other languages, the source file will contain the type definition and rules in that specific language, as a generated cache (dType on-chain data -> language-specific type definition).

#### Multi-language Type Registration

With the the above structure, we see that we must register a specific type for each language. For example:

- Solidity
```
{
  "typeChoice": 0,
  "contractAddress": "0x91E3737f15e9b182EdD44D45d943cF248b3a3BF9",
  "source": "0xef5a57c590b61708514b56c6ab5acd9b3f9ab8c3ff19480cc724ad59fd210268",
  "name": "MarketProduct",
  "language": 0,
  "types": [
    {"name": "bytes32", "label": "id", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "amount", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "price", "relation": 0, "dimensions":[]},
    {"name": "bool", "label": "in_deposit", "relation": 0, "dimensions":[]}
  ],
}
```

- Python
```
{
  "typeChoice": 0,
  "contractAddress": "0x0000000000000000000000000000000000000000",
  "source": "0xe114608ca07ee8b712e1705902035d844304b58eeb4cc2707384445fbf9223fc",
  "name": "MarketProduct",
  "language": 3,
  "types": [
    {"name": "bytes32", "label": "id", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "amount", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "price", "relation": 0, "dimensions":[]},
    {"name": "bool", "label": "in_deposit", "relation": 0, "dimensions":[]}
  ],
}
```

Because the Python `source` file provides the type definition built from the Solidity registration, the `contractAddress` is not needed.

Native types do not need to be registered. All others will need to be registered with dType.
In the above example, subtype `bool` is a native Python type, therefore no registration with dType is needed. However, subtypes `bytes32` and `uint64` are not Python native types and each must be registered in dType and defined in the metadata `source` file:

```
{
  "typeChoice": 0,
  "contractAddress": "0x0000000000000000000000000000000000000000",
  "source": "0xe234608ca07ee8b712e1705902035d844304b58eeb4cc2707384445fbf9223fc",
  "name": "bytes32",
  "language": 3,
  "types": [],
}
```

```
{
  "typeChoice": 0,
  "contractAddress": "0x0000000000000000000000000000000000000000",
  "source": "0xe765608ca07ee8b712e1705902035d844304b58eeb4cc2707384445fbf9223fc",
  "name": "uint64",
  "language": 3,
  "types": [],
}
```

##### Encoded Types for Language-specific Use

There will be cases we want to store tightly packed encoded data on-chain for use in applications in other languages. In this case, we can fully define the language-specific type on-chain, but only store encoded data (e.g. in a format like CBOR).

For example, we can add a `characteristics` subtype to our `MarketProduct` type.

For Solidity, we can use `bytes characteristics`:
```
{
  "typeChoice": 0,
  "contractAddress": "0x91E3737f15e9b182EdD44D45d943cF248b3a3BF9",
  "source": "0xef5a57c590b61708514b56c6ab5acd9b3f9ab8c3ff19480cc724ad59fd210268",
  "name": "MarketProduct",
  "language": 0,
  "types": [
    {"name": "bytes32", "label": "id", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "amount", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "price", "relation": 0, "dimensions":[]},
    {"name": "bool", "label": "in_deposit", "relation": 0, "dimensions":[]},
    {"name": "bytes", "label": "characteristics", "relation": 0, "dimensions":[]}
  ],
}
```

For Python, we will define the `characteristics` type in full:
```
{
  "typeChoice": 0,
  "contractAddress": "0x0000000000000000000000000000000000000000",
  "source": "0xe666608ca07ee8b712e1705902035d844304b58eeb4cc2707384445fbf9223fc",
  "name": "characteristic",
  "language": 3,
  "types": [
    {"name": "uint64", "label": "width", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "height", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "weight", "relation": 0, "dimensions":[]}
  ],
},
{
  "typeChoice": 0,
  "contractAddress": "0x0000000000000000000000000000000000000000",
  "source": "0xe114608ca07ee8b712e1705902035d844304b58eeb4cc2707384445fbf9223fc",
  "name": "MarketProduct",
  "language": 3,
  "types": [
    {"name": "bytes32", "label": "id", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "amount", "relation": 0, "dimensions":[]},
    {"name": "uint64", "label": "price", "relation": 0, "dimensions":[]},
    {"name": "bool", "label": "in_deposit", "relation": 0, "dimensions":[]},
    {"name": "characteristic", "label": "characteristics", "relation": 0, "dimensions":[""]}
  ],
}
```

This way, tools used to decode on-chain data for Python will return the entire `MarketProduct` object, with the `characteristics` object decoded.

TODO: the actual type def source code for Python.

### Data Bridging

TBD data serialization

## Rationale

Extending dType to support multiple programming languages encourages the creation of a truly global type system that will advance interoperability. Ethereum currently has the properties necessary for being the storage system and type checker for a global type system: immutability, availability, non-centralized, supports programatic decentralized governance.

The choice of defining the type's primary identifier by `language` and `name` ensures direct addressability when retrieving types for a specific language. Including a reference to the source code for each type definition enables faster on-demand use for developers.

Registering a type multiple times, for each language in which it is defined enables per-language addressability and immutability, as opposed to registering a type once and then updating the type to support an additional language. Moreover, it enables different type definitions per language, as in the `MarketProduct.characteristics` example.

`language` was chosen to be of type `uint16`. We also considered `bytes32` - an encoded language identifier, that could be `keccak256(abi.encodePacked(human_readable_language_name))` (or another standardized language identifier). The main argument in choosing a `uint` is storage cost related. By assigning Solidity types with language `0`, the storage cost for registering a type is reduced with the cost of one `SSTORE`. Languages themselves can be registered in dType and additional metadata (including a human-readable name) can be added.

## Backwards Compatibility

This proposal does not affect extant Ethereum standards or implementations.

## Test Cases

Will be added.

## Implementation

An in-work implementation can be found at https://github.com/pipeos-one/dType/tree/master/contracts/contracts.
This proposal will be updated with an appropriate implementation when consensus is reached on the specifications.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
