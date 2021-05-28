---
icip: 1
title: Internet Computer Token Standard
status: Draft
type: Financial
author: Norton Wang (@floorlamp)
created: 2020-09-24
---

<em>Since official "Internet Computer Improvement Proposals" (ICIPs) do not
exist yet, this document refers to "ICIP-1" for convenience only.</em>

## Table Of Contents

- [Summary](#summary)
- [Motivation](#motivation)
- [Abstract](#abstract)
- [General](#general)
- [Interface Specification](#interface-specification)
  - [Entrypoint Semantics](#entrypoint-semantics)
    - [`transfer`](#transfer)
      - [Core Transfer Behavior](#core-transfer-behavior)
      - [Default Transfer Permission Policy](#default-transfer-permission-policy)
    - [`getBalance`](#getBalance)
    - [Operators](#operators)
      - [`updateOperator`](#updateOperator)
      - [`isAuthorized`](#isAuthorized)
    - [`getMetadata`](#getMetadata)
- [Implementing Different Token Types with ICIP-1](#implementing-different-token-types-with-ICIP-1)
  - [Single Fungible Token](#single-fungible-token)
  - [Multiple Fungible Tokens](#multiple-fungible-tokens)
  - [Non-fungible Tokens](#non-fungible-tokens)
  - [Mixing Fungible and Non-fungible Tokens](#mixing-fungible-and-non-fungible-tokens)
  - [Non-transferable Tokens](#non-transferable-tokens)
- [Additional Ideas](#additional-ideas)
- [References](#references)
- [Copyright](#copyright)

## Summary

ICIP-1 proposes a standard for a unified token canister interface, supporting a
wide range of token types and implementations. This document provides an
overview and rationale for the interface, token transfer semantics, and support
for various transfer permission policies.

**PLEASE NOTE:** This API specification is a work-in-progress.

## Motivation

There are multiple dimensions and considerations while implementing a particular
token canister. Tokens might be fungible or non-fungible. A variety of transfer
permission policies can be used to define how many tokens can be transferred,
who can perform a transfer, and who can receive tokens. A token canister can be
designed to support a single token type (e.g. ERC-20 or ERC-721) or multiple
token types (e.g. ERC-1155) to optimize batch transfers and atomic swaps of the
tokens.

Such considerations can easily lead to the proliferation of many token
standards, each optimized for a particular token type or use case. This
situation is apparent in the Ethereum ecosystem, where many standards have been
proposed, but ERC-20 (fungible tokens) and ERC-721 (non-fungible tokens) are
dominant.

Token wallets, token exchanges, and other clients then need to support multiple
standards and multiple token APIs. The ICIP-1 standard proposes a unified token
canister interface that accommodates all mentioned concerns. It aims to provide
significant expressivity to canister developers to create new types of tokens
while maintaining a common interface standard for wallet integrators and
external developers.

## Abstract

This standard defines the unified canister interface and its behavior to support
a wide range of token types and implementations. The particular ICIP-1
implementation may support either a single token type per canister or multiple
tokens per canister, including hybrid implementations where multiple token kinds
(fungible, non-fungible, non-transferable etc) are supported.

All of the entrypoints are batch operations that allow querying or transfer of
multiple token types atomically.

Most token standards specify logic that validates a transfer transaction and can
either approve or reject a transfer. Such logic could validate who can perform a
transfer, the transfer amount and who can receive tokens. This standard calls
such logic a _transfer permission policy_. The ICIP-1 standard defines the
[default `TransferRequest` permission policy](#default-transfer-permission-policy)
that specify who can transfer tokens. The default policy allows transfers by
either token owner (an principal that holds token balance) or by an operator (an
principal that is permitted to manage tokens on behalf of the token owner).

Transfer permission policies can be customized.

This specification defines some standard error variants to be used when
implementing ICIP-1. However, some implementations MAY introduce their custom
errors that MUST follow the same pattern as standard ones.

## General

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”,
“SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

- Tokens are uniquely identified by a pair composed of the token canister ID and
  token ID, a natural number (`Nat`). If the underlying canister implementation
  supports only a single token type (e.g. ERC-20-like contract), the token ID
  MUST be `0n`. In the case when multiple token types are supported within the
  same ICIP-1 token canister (e. g. ERC-1155-like contract), the canister is
  fully responsible for assigning and managing token IDs.

- The ICIP-1 batch entrypoints accept a list (batch) of parameters describing a
  single operation or a query. The batch MUST NOT be reordered or deduplicated
  and MUST be processed in the same order it is received.

- Empty batch is a valid input and MUST be processed as a non-empty one. For
  example, an empty transfer batch will not affect token balances, but
  applicable transfer core behavior and permission policy MUST be applied.

- If the underlying token implementation supports only a single token type, the
  batch may contain zero or multiple entries where token ID is a fixed `0n`
  value. Likewise, if multiple token types are supported, the batch may contain
  zero or more entries and there may be duplicate token IDs.

- The choice of `Nat32` for a `tokenId` type implies each canister can store
  `2**32` individual tokens

## Interface Specification

Token canisters implementing the ICIP-1 standard MUST have the following
entrypoints. Notation is given in
[Motoko](https://sdk.dfinity.org/docs/language-guide/motoko.html). Candid
specifications can be generated as needed.

```
type Token = actor {
  getBalance: query (requests: [BalanceRequest]) -> async BalanceResponse;

  getMetadata: query (tokenIds: [TokenId]) -> async MetadataResponse;

  transfer: shared (requests: [TransferRequest]) -> async TransferResponse;

  updateOperator: shared (requests: [OperatorRequest]) -> async OperatorResponse;

  isAuthorized: query (requests: [IsAuthorizedRequest]) -> async IsAuthorizedResponse;
};
```

### Entrypoint Semantics

#### `transfer`

```
type User = Principal;

type TokenId = Nat32;

type Balance = Nat;

type TransferRequest = {
  from: User;
  to: User;
  tokenId: TokenId;
  amount: Balance;
};

type TransferResponse = Result.Result<(), {
  #Unauthorized;
  #InvalidDestination: User;
  #InvalidToken: TokenId;
  #InsufficientBalance;
}>;

type transfer = (requests: [TransferRequest]) -> async TransferResponse;
```

Each transfer in the batch is specified between one source (`from`) and
destination (`to`) pair. Each `TransferRequest` specifies token ID and the
amount to be transferred from the source principal to the destination principal.

ICIP-1 does NOT specify an interface for mint and burn operations; however, if
an ICIP-1 token canister implements mint and burn operations, it SHOULD, when
possible, enforce the same logic (core transfer behavior and transfer permission
logic) applied to the token transfer operation. Mint and burn can be considered
special cases of the transfer. Although, it is possible that mint and burn have
more or less restrictive rules than the regular transfer. For instance, mint and
burn operations may be invoked by a special privileged administrative principal
only. In this case, regular operator restrictions may not be applicable.

##### Core Transfer Behavior

ICIP-1 token canisters MUST always implement this behavior.

- Every batch transfer operation MUST happen atomically and in order. If at
  least one transfer in the batch cannot be completed, the whole transaction
  MUST fail, all token transfers MUST be reverted, and token balances MUST
  remain unchanged.

- Each transfer in the batch MUST decrement token balance of the source (`from`)
  principal by the amount of the transfer and increment token balance of the
  destination (`to`) principal by the amount of the transfer.

- If the transfer amount exceeds current token balance of the source principal,
  the whole transfer operation MUST fail with the error variant
  `InsufficientBalance`.

- If the token owner does not hold any tokens of type `tokenId`, the owner's
  balance is interpreted as zero. No token owner can have a negative balance.

- The transfer MUST update token balances exactly as the operation parameters
  specify it. Transfer operations MUST NOT try to adjust transfer amounts or try
  to add/remove additional transfers like transaction fees.

- Transfers of zero amount MUST be treated as normal transfers.

- Transfers with the same principal (`from` equals `to`) MUST be treated as
  normal transfers.

- If one of the specified `tokenId`s is not defined within the ICIP-1 contract,
  the entrypoint MUST fail with the error variant `InvalidToken`.

- Transfer implementations MUST apply transfer permission logic (either
  [default transfer permission policy](#default-transfer-permission-policy) or a
  custom one). If permission logic rejects a transfer, the whole operation MUST
  fail.

- Core transfer behavior MAY be extended. If additional constraints on tokens
  transfer are required, ICIP-1 token canister implementation MAY invoke
  additional permission policies. If the additional permission fails, the whole
  transfer operation MUST fail with a custom error variant.

##### Default Transfer Permission Policy

- Token owner principal MUST be able to perform a transfer of its own tokens (e.
  g. `caller` equals to `from` parameter in the `TransferRequest`).

- An operator (a principal that performs token transfer operation on behalf of
  the owner) MUST be permitted to manage the specified owner's tokens before it
  invokes a transfer transaction (see [`updateOperator`](#updateOperator)).

- If the principal that invokes a transfer operation is neither a token owner
  nor one of the permitted operators, the transaction MUST fail with the error
  variant `Unauthorized`. If at least one of the `TransferRequest`s in the batch
  is not permitted, the whole transaction MUST fail.

#### `getBalance`

```
type BalanceRequest = {
  user: User;
  tokenId: TokenId
};

type BalanceResponse = Result.Result<[Balance], {
  #InvalidToken: TokenId;
}>;

type getBalance = query (requests: [BalanceRequest]) -> async BalanceResponse;
```

Gets the balance of multiple principal/token pairs. Accepts a list of
`BalanceRequest`s and returns a `BalanceResponse`.

- There may be duplicate `BalanceRequest`'s, in which case they should not be
  deduplicated nor reordered.

- If the principal does not hold any tokens, the principal balance is
  interpreted as zero.

- If one of the specified `tokenId`s is not defined within the ICIP-1 contract,
  the entrypoint MUST fail with the error variant `InvalidToken`.

#### Operators

**Owner** is a principal which can hold tokens.

**Operator** is a principal that originates token transfer operation on behalf
of the owner.

An operator, other than the owner, CAN be approved to manage specific tokens
held by the owner to transfer them from the owner principal.

ICIP-1 interface specifies an entrypoint to update operators. Operators are
permitted per specific token owner and token ID (token type). Once permitted, an
operator can transfer tokens of that type belonging to the owner.

##### `updateOperator`

```
type TokenIds = {
  #All;
  #Some: (TokenId, ?Balance);
};

type OperatorAction = {
  #SetOperator: TokenIds;
  #RemoveOperator: ?[TokenId];
};

type OperatorRequest = {
  owner: User;
  operators: [(User, OperatorAction)]
};

type OperatorResponse = Result.Result<(), {
  #Unauthorized;
  #InvalidOwner: User;
}>;

type updateOperator = (requests: [OperatorRequest]) -> async OperatorResponse;
```

Update or Remove token operators for the specified token owners, token IDs and balances.

- The entrypoint accepts a list of `OperatorRequest`s. If two different requests
  in the list add and remove an operator for the same token owner and token ID,
  the last command in the list MUST take effect.

- Adding an operator for `#All` token IDs MUST grant permissions to all current
  and future tokens owned by the owner for ALL balances. Similarly, removing an operator for
  `#All` token IDs MUST remove permissions for all current and future tokens.

- It is possible to update operators for a token owner that does not hold any
  token balances yet.

- Operator relation is not transitive. If C is an operator of B and if B is an
  operator of A, C cannot transfer tokens that are owned by A, on behalf of B.

The standard does not specify who is permitted to update operators on behalf of
the token owner. Depending on the business use case, the particular
implementation of the ICIP-1 contract MAY limit operator updates to a token
owner (`owner == caller`) or be limited to an administrator. If so, the
`Unauthorized` error variant MUST be used.

##### `isAuthorized`

```
type IsAuthorizedRequest = {
  owner: User;
  operator: User;
  tokenId: TokenId;
  amount: Balance;
};

type IsAuthorizedResponse = [Bool];

type isAuthorized = query (requests: [IsAuthorizedRequest]) -> async IsAuthorizedResponse;
```

Checks whether the specified `operator` principal is authorized to transfer on
behalf of the `owner` principal the token `tokenId` of `amount`.

- Results MUST be consistent with authorization checks in `transfer` operations.
  If `isAuthorized` returns `True` for a given operator and owner pair, then
  that operator must be able perform `transfer`s without receiving an
  `Unauthorized` error. Likewise, if `isAuthorized` returns `False`, then that
  operator MUST NOT be able to `transfer` and should receive an `Unauthorized`
  error.

##### `getMetadata`

```
type Metadata = Blob;

type MetadataResponse = Result.Result<[Metadata], {
  #InvalidToken: TokenId;
}>;

type getMetadata = query (tokenIds: [TokenId]) -> async MetadataResponse;
```

Returns token metadata for the given `tokenId`s.

Token metadata is primarily useful in user-facing contexts (e.g. wallets,
explorers, marketplaces). Some attributes that are commonly defined are symbol,
name, description, and decimals. The specification for the metadata format is a
work-in-progress.

## Implementing Different Token Types With ICIP-1

The ICIP-1 interface is designed to support a wide range of token types and
implementations. This section gives examples of how different types of the
ICIP-1 contracts MAY be implemented and what are the expected properties of such
an implementation.

### Single Fungible Token

An ICIP-1 contract represents a single token similar to the ERC-20 standard.

| Property          | Constraints |
| ----------------- | ----------- |
| `tokenId`         | Always `0n` |
| transfer amount   | Nat         |
| principal balance | Nat         |

### Multiple Fungible Tokens

An ICIP-1 contract may represent multiple tokens similar to ERC-1155 standard.
The implementation can have a fixed predefined set of supported tokens or tokens
can be created dynamically.

| Property          | Constraints |
| ----------------- | ----------- |
| `tokenId`         | Nat         |
| transfer amount   | Nat         |
| principal balance | Nat         |

### Non-fungible Tokens

An ICIP-1 contract may represent non-fungible tokens (NFT) similar to ERC-721
standard. For each individual non-fungible token the implementation assigns a
unique `tokenId`. The implementation MAY support either a single kind of NFTs or
multiple kinds. If multiple kinds of NFT is supported, each kind MAY be assigned
a continuous range of natural number (that does not overlap with other ranges)
and have its own associated metadata.

| Property          | Constraints  |
| ----------------- | ------------ |
| `tokenId`         | Nat          |
| transfer amount   | `0n` or `1n` |
| principal balance | `0n` or `1n` |

For any valid `tokenId` only one principal CAN hold the balance of one token
(`1n`). The rest of the principals MUST hold zero balance (`0n`) for that
`tokenId`.

### Mixing Fungible and Non-fungible Tokens

An ICIP-1 contract MAY mix multiple fungible and non-fungible tokens within the
same contract similar to ERC-1155. The implementation MAY chose to select
individual natural numbers to represent `tokenId` for fungible tokens and
continuous natural number ranges to represent `tokenId`s for NFTs.

| Property          | Constraints                                      |
| ----------------- | ------------------------------------------------ |
| `tokenId`         | Nat                                              |
| transfer amount   | `0n` or `1n` for NFT and Nat for fungible tokens |
| principal balance | `0n` or `1n` for NFT and Nat for fungible tokens |

### Non-transferable Tokens

Either fungible and non-fungible tokens can be non-transferable.
Non-transferable tokens can be represented by the ICIP-1 token that has custom
permission logic. Tokens cannot be transferred either by the token owner or by
any operator. Only privileged operations like mint and burn can assign tokens to
owner principals.

## Additional Ideas

The following are some other features for consideration that could offer more
flexibility and UX improvements.

- Receiver hooks, for token receivers to perform operations immediately after
  receiving tokens
- More detailed transfer permission policies, such as setting a token to only be
  transferrable by an operator
- A `Decimal` type instead of `Nat` for balances, removing the complexity of
  emulating precision with large exponents

## Motoko Types

```motoko
import Hash "mo:base/Hash";
import Nat32 "mo:base/Nat32";
import Principal "mo:base/Principal";
import Result "mo:base/Result";
import Word32 "mo:base/Word32";

// A user can be any principal or canister
type User = Principal;

// A Nat32 implies each canister can store 2**32 individual tokens
type TokenId = Nat32;

// Token amounts are unbounded
type Balance = Nat;

// Details for a token, eg. name, symbol, description, decimals.
// Metadata format TBD, possible option is JSON blob
type Metadata = Blob;
type MetadataResponse = Result.Result<[Metadata], {
  #InvalidToken: TokenId;
}>;

// Request and responses for getBalance
type BalanceRequest = {
  user: User;
  tokenId: TokenId;
};
type BalanceResponse = Result.Result<[Balance], {
  #InvalidToken: TokenId;
}>;

// Request and responses for transfer
type TransferRequest = {
  from: User;
  to: User;
  tokenId: TokenId;
  amount: Balance;
};
type TransferResponse = Result.Result<(), {
  #Unauthorized;
  #InvalidDestination: User;
  #InvalidToken: TokenId;
  #InsufficientBalance;
}>;

// Request and responses for updateOperator
type TokenIds = {
  #All;
  #Some: (TokenId, ?Balance);
};
type OperatorAction = {
  #SetOperator: TokenIds;
  #RemoveOperator: ?[TokenId];
};
type OperatorRequest = {
  owner: User;
  operators: [(User, OperatorAction)];
};
type OperatorResponse = Result.Result<(), {
  #Unauthorized;
  #InvalidOwner: User;
}>;

// Request and responses for isAuthorized
type IsAuthorizedRequest = {
  owner: User;
  operator: User;
  tokenId: TokenId;
  amount: Balance;
};
type IsAuthorizedResponse = [Bool];

// Utility functions for User and TokenId, useful when implementing containers
module User = {
  public let equal = Principal.equal;
  public let hash = Principal.hash;
};

module TokenId = {
  public func equal(id1 : TokenId, id2 : TokenId) : Bool { id1 == id2 };
  public func hash(id : TokenId) : Hash.Hash { Word32.fromNat(Nat32.toNat(id)) };
};

// Uniquely identifies a token
type TokenIdentifier = {
  canister: Token;
  tokenId: TokenId;
};

// Utility functions for TokenIdentifier
module TokenIdentifier = {
  // Tokens are equal if the canister and tokenId are equal
  public func equal(id1 : TokenIdentifier, id2 : TokenIdentifier) : Bool {
    Principal.fromActor(id1.canister) == Principal.fromActor(id2.canister)
    and id1.tokenId == id2.tokenId
  };
  // Hash the canister and xor with tokenId
  public func hash(id : TokenIdentifier) : Hash.Hash {
    Principal.hash(Principal.fromActor(id.canister)) ^ Word32.fromNat(Nat32.toNat(id.tokenId))
  };
  // Join the principal and id with a '_'
  public func toText(id : TokenIdentifier) : Text {
    Principal.toText(Principal.fromActor(id.canister)) # "_" # Nat32.toText(id.tokenId)
  };
};

/**
  A token canister that can hold many tokens.
*/
type Token = actor {
  /**
    Batch get balances.
    Any request with an invalid tokenId should cause the entire batch to fail.
    A user that has no token should default to 0.
  */
  getBalance: query (requests: [BalanceRequest]) -> async BalanceResponse;

  /**
    Batch get metadata.
    Any request with an invalid tokenId should cause the entire batch to fail.
  */
  getMetadata: query (tokenIds: [TokenId]) -> async MetadataResponse;

  /**
    Batch transfer.
    A request should fail if:
      - the caller is not authorized to transfer for the sender
      - the sender has insufficient balance
    Any request that fails should cause the entire batch to fail, and to
    rollback to the initial state.
  */
  transfer: shared (requests: [TransferRequest]) -> async TransferResponse;

  /**
    Batch update operator.
    A request should fail if the caller is not authorized to update operators
    for the owner.
    Any request that fails should cause the entire batch to fail, and to
    rollback to the initial state.
  */
  updateOperator: shared (requests: [OperatorRequest]) -> async OperatorResponse;

  /**
    Batch function to check if a user is authorized to transfer for an owner.
  */
  isAuthorized: query (requests: [IsAuthorizedRequest]) -> async IsAuthorizedResponse;
};
```

## References

**Standards**

- [ERC-20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [ERC-721 Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721)
- [ERC-1155 Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155)

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
