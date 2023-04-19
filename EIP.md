# Multi-Token Standard

**Requires** [EIP-165](https://eips.ethereum.org/EIPS/eip-165)

## Table of Contents

- [Multi-Token Standard](#multi-token-standard)
  - [Table of Contents](#table-of-contents)
  - [Abstract](#abstract)
  - [Motivation](#motivation)
  - [Rationale](#rationale)
    - [Granular Approvals](#granular-approvals)
    - [Removal of Batching](#removal-of-batching)
    - [Removal of Required Callbacks](#removal-of-required-callbacks)
    - [Removal of "Safe" Naming](#removal-of-safe-naming)
  - [Specification](#specification)
    - [Definitions](#definitions)
    - [Methods](#methods)
      - [totalSupply](#totalsupply)
      - [decimals](#decimals)
      - [balanceOf](#balanceof)
      - [allowance](#allowance)
      - [transfer](#transfer)
      - [transferFrom](#transferfrom)
      - [approve](#approve)
      - [setOperator](#setoperator)
    - [Events](#events)
      - [Transfer](#transfer-1)
      - [OperatorSet](#operatorset)
      - [Approval](#approval)
  - [Interface ID](#interface-id)
  - [Extensions](#extensions)
    - [ERCNMetadata](#ercnmetadata)
      - [Methods](#methods-1)
        - [name](#name)
        - [symbol](#symbol)
        - [tokenURI](#tokenuri)
      - [Metadata Structure](#metadata-structure)
  - [Backwards Compatibility](#backwards-compatibility)
  - [Reference Implementation](#reference-implementation)

## Abstract

The following standard specifies a multi-token contract as a simplified alternative to the
[ERC-1155 Multi-Token Standard](https://eips.ethereum.org/EIPS/eip-1155).

## Motivation

The ERC-1155 includes unnecessary features such as requiring recipient accounts with code to
implement callbacks returning specific values and batch-calls in the specification. In addition, the
single operator permission scheme grants unlimited allowance on every token ID in the contract.

## Rationale

### Granular Approvals

While the "operator model" from ERC-1155 allows an account to set another account as an operator,
giving full permissions to transfer any amount of any token id on behalf of the owner, this may not
always be the desired permission scheme. The "allowance model" from ERC-20 allows an account to set
an explicit amount of the token that another account can spend on the owner's behalf. This standard
requires both be implemented, with the only modification being to the "allowance model" where the
token id must be specified as well. This allows an account to grant specific approvals to specific
token ids, infinite approvals to specific token ids, or infinite approvals to all token ids. If an
account is set as an operator, the allowance SHOULD NOT be decreased when tokens are transferred on
behalf of the owner.

### Removal of Batching

While batching operations is useful, its place should not be in the standard itself, but rather on
a case-by-case basis. This allows for different tradeoffs to be made in terms of calldata layout,
which may be especially useful for specific applications such as roll-ups that commit calldata to
global storage.

### Removal of Required Callbacks

Callbacks MAY be used within a multi-token compliant contract, but it is not required. This allows
for more gas efficient methods by reducing external calls and additional checks.

### Removal of "Safe" Naming

The `safeTransfer` and `safeTransferFrom` naming conventions are misleading, especially in the
context of ERC-1155 and ERC-721, as they require external calls to receiver accounts with code,
passing the execution flow to an arbitrary contract, provided the receiver contract returns a
specific value. The combination of removing mandatory callbacks and removing the word "safe" from
all method names improves the safety of the control flow by default.

## Specification

### Definitions

- infinite: The maximum value for a uint256 (`2 ** 256 - 1`).
- caller: The caller of the current context (`msg.sender`).
- spender: An account that transfers tokens on behalf of another account.
- operator: An account that has unlimited transfer permissions on all token ids for another account.
- mint: The creation of an amount of tokens. This MAY happen in a mint method or as a transfer from the zero address.
- burn: The removal an amount of tokens. This MAY happen in a burn method or as a transfer from the zero address.

### Methods

#### totalSupply

The total supply for a token id.

MUST be equal to the sum of the `balanceOf` of all accounts of the token id.

```yaml
- name: totalSupply
  type: function
  stateMutability: view

  inputs:
    - name: id
      type: uint256

  outputs:
    - name: amount
      type: uint256
```

#### decimals

The number of decimals for a token id.

MAY be ignored if not explicitly set to a non-zero value.

```yaml
- name: decimals
  type: function
  stateMutability: view

  inputs:
    - name: id
      type: uint256

  outputs:
  - name: amount
    type: uint8
```

#### balanceOf

The total number of units of a token id that an account owns.

```yaml
- name: balanceOf
  type: function
  stateMutability: view

  inputs:
    - name: owner
      type: address
    - name: id
      type: uint256

  outputs:
    - name: amount
      type: uint256
```

#### allowance

The total number of units of a token id that a spender is permitted to transfer on behalf of an
owner.

```yaml
- name: allowance
  type: function
  stateMutability: view

  inputs:
    - name: owner
      type: address
    - name: spender
      type: address
    - name: id
      type: uint256

  outputs:
    - name: amount
      type: uint256
```

#### transfer

Transfers an amount of units of a token id from the caller to the receiver.

MUST revert when the caller's balance for the token id is insufficient.

MUST log the Transfer event.

```yaml
- name: transfer
  type: function
  stateMutability: nonpayable

  inputs:
    - name: receiver
      type: address
    - name: id
      type: uint256
    - name: amount
      type: uint256

  outputs: []
```

#### transferFrom

Transfers an amount of units of a token id from a sender to a receiver by the caller.

MUST revert when the caller is not an operator for the spender and the caller's allowance for the
token id for the sender is insufficient.

MUST revert when the caller's balance for the token id is insufficient.

MUST log the Transfer event.

MUST decrease the caller's allowance by the same amount of the sender's balance decrease if the
caller's allowance is not infinite.

SHOULD NOT decrease the caller's allowance for the token id for the sender if the allowance is
infinite.

SHOULD NOT decrease the caller's allowance for the token id for the sender if the caller is an
operator.

```yaml
- name: transferFrom
  type: function
  stateMutability: nonpayable

  inputs:
    - name: sender
      type: address
    - name: receiver
      type: address
    - name: id
      type: uint256
    - name: amount
      type: uint256

  outputs: []
```

#### approve

Approves an amount of units of a token id that a spender is permitted to transfer on behalf of the
caller.

MUST set the allowance of the spender of the token id for the caller to the amount.

MUST log the Approval event.

```yaml
- name: approve
  type: function
  stateMutability: nonpayable

  inputs:
    - name: spender
      type: address
    - name: id
      type: uint256
    - name: amount
      type: uint256

  outputs: []
```

#### setOperator

Sets a spender as an operator, granting unlimited transfer permissions for any token id for the
caller.

MUST set the operator status to the approved value.

MUST log the OperatorSet event.

```yaml
- name: setOperator
  type: function
  stateMutability: nonpayable

  inputs:
    - name: spender
      type: address
    - name: approved
      type: bool
```

### Events

#### Transfer

The sender has transferred an amount of a token id to a receiver.

MUST be logged when an amount of a token id is transferred from one account to another.

SHOULD be logged with the sender address as the zero address when an amount of a token id is minted.

SHOULD be logged with the receiver address as the zero address when an amount of a token id is
burned.

```yaml
- name: Transfer
  type: event

  inputs:
    - name: sender
      indexed: true
      type: address
    - name: receiver
      indexed: true
      type: address
    - name: id
      indexed: true
      type: address
    - name: amount
      indexed: false
      type: address
```

#### OperatorSet

The owner has set the approved status to the spender.

MUST be logged when the operator status is set.

MAY be logged when the operator status is set to the same status it was before the current call.

```yaml
- name: OperatorSet
  type: event

  inputs:
    - name: owner
      indexed: true
      type: address
    - name: spender
      indexed: true
      type: address
    - name: approved
      indexed: false
      type: bool
```

#### Approval

The owner has approved a spender to transfer an amount of a token id to be transferred on the
owner's behalf.

MUST be logged when the approval is set by the owner.

MAY be logged when the approval is decreased by the transferFrom method.

```yaml
- name: Approval
  type: event

  inputs:
    - name: owner
      indexed: true
      type: address
    - name: spender
      indexed: true
      type: address
    - name: id
      indexed: true
      type: uint256
    - name: amount
      indexed: false
      type: uint256
```

## Interface ID

The interface ID is `0x8da179e8`.

## Extensions

### ERCNMetadata

#### Methods

##### name

The name of the contract.

```yaml
- name: name
  type: function
  stateMutability: view

  inputs: []

  outputs:
    - name: name
      type: string
```

##### symbol

The ticker symbol of the contract.

```yaml
- name: symbol
  type: function
  stateMutability: view

  inputs: []

  outputs:
    - name: symbol
      type: string
```

##### tokenURI

The URI for a token id.

MAY revert if the token id does not exist.

```yaml
- name: tokenURI
  type: function
  stateMutability: view

  inputs:
    - name: id
      type: uint256

  outputs:
    - name: uri
      type: string
```

#### Metadata Structure

The metadata specification closely follows that of the ERC-721 JSON schema.

```json
{
  "title": "Asset Metadata",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Identifies the token"
    },
    "description": {
      "type": "string",
      "description": "Describes the token"
    },
    "image": {
      "type": "string",
      "description": "A URI pointing to an image resource."
    }
  }
}
```

## Backwards Compatibility

This is not backwards compatible with ERC-1155 as some methods are removed. however, wrappers can
be implemented for ERC-20, ERC-721, and ERC-1155 tokens.

## Reference Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

/// @title ERCN Multi-Token Reference Implementation
/// @author jtriley.eth
contract ERCN {
    /// @dev Thrown when owner balance for id is insufficient.
    /// @param owner The address of the owner.
    /// @param id The id of the token.
    error InsufficientBalance(address owner, uint256 id);

    /// @dev Thrown when spender allowance for id is insufficient.
    /// @param spender The address of the spender.
    /// @param id The id of the token.
    error InsufficientPermission(address spender, uint256 id);

    /// @notice The event emitted when a transfer occurs.
    /// @param sender The address of the sender.
    /// @param receiver The address of the receiver.
    /// @param id The id of the token.
    /// @param amount The amount of the token.
    event Transfer(address indexed sender, address indexed receiver, uint256 indexed id, uint256 amount);

    /// @notice The event emitted when an operator is set.
    /// @param owner The address of the owner.
    /// @param spender The address of the spender.
    /// @param approved The approval status.
    event OperatorSet(address indexed owner, address indexed spender, bool approved);

    /// @notice The event emitted when an approval occurs.
    /// @param owner The address of the owner.
    /// @param spender The address of the spender.
    /// @param id The id of the token.
    /// @param amount The amount of the token.
    event Approval(address indexed owner, address indexed spender, uint256 indexed id, uint256 amount);

    /// @notice The total supply of each id.
    mapping(uint256 id => uint256 amount) public totalSupply;

    /// @notice The number of decimals for each id.
    mapping(uint256 id => uint8 amount) public decimals;

    /// @notice Owner balance of an id.
    mapping(address owner => mapping(uint256 id => uint256 amount)) public balanceOf;

    /// @notice Spender allowance of an id.
    mapping(address owner => mapping(address spender => mapping(uint256 id => uint256 amount))) public allowance;

    /// @notice Checks if a spender is approved by an owner as an operator.
    mapping(address owner => mapping(address spender => bool)) public isOperator;

    /// @notice Transfers an amount of an id from the caller to a receiver.
    /// @param receiver The address of the receiver.
    /// @param id The id of the token.
    /// @param amount The amount of the token.
    function transfer(address receiver, uint256 id, uint256 amount) public {
        if (balanceOf[msg.sender][id] < amount) revert InsufficientBalance(msg.sender, id);
        balanceOf[msg.sender][id] -= amount;
        balanceOf[receiver][id] += amount;
        emit Transfer(msg.sender, receiver, id, amount);
    }

    /// @notice Transfers an amount of an id from a sender to a receiver.
    /// @param sender The address of the sender.
    /// @param receiver The address of the receiver.
    /// @param id The id of the token.
    /// @param amount The amount of the token.
    function transferFrom(address sender, address receiver, uint256 id, uint256 amount) public {
        if (sender != msg.sender && !isOperator[sender][msg.sender]) {
            if (allowance[sender][msg.sender][id] < amount) {
                revert InsufficientPermission(msg.sender, id);
            }
            allowance[sender][msg.sender][id] -= amount;
        }
        if (balanceOf[sender][id] < amount) revert InsufficientBalance(sender, id);
        balanceOf[sender][id] -= amount;
        balanceOf[receiver][id] += amount;
        emit Transfer(sender, receiver, id, amount);
    }

    /// @notice Approves an amount of an id to a spender.
    /// @param spender The address of the spender.
    /// @param id The id of the token.
    /// @param amount The amount of the token.
    function approve(address spender, uint256 id, uint256 amount) public {
        allowance[msg.sender][spender][id] = amount;
        emit Approval(msg.sender, spender, id, amount);
    }

    /// @notice Sets or removes a spender as an operator for the caller.
    /// @param spender The address of the spender.
    /// @param approved The approval status.
    function setOperator(address spender, bool approved) public {
        isOperator[msg.sender][spender] = approved;
        emit OperatorSet(msg.sender, spender, approved);
    }

    /// @notice Checks if a contract implements an interface.
    /// @param interfaceId The interface identifier, as specified in ERC-165.
    /// @return supported True if the contract implements `interfaceId` and
    function supportsInterface(bytes4 interfaceId) public pure returns (bool supported) {
        return interfaceId == 0x8da179e8 || interfaceId == 0x01ffc9a7;
    }
}
```