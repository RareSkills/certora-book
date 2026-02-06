# Formally Verifying Ownable.sol


Ownable is an abstract contract that provides owner-based access control. When inherited, it restricts specific functions to the owner using the `onlyOwner` modifier. It has three core mechanisms, each implemented through the following functions:

- **`onlyOwner()`**

    This modifier restricts access to specific functions allowing only the owner to call them.

- **`renounceOwnership()`**

    This public function is protected by the `onlyOwner` modifier, allowing only the owner to call it. When executed, it sets the owner to the zero address (`address(0)`). After renouncement, all functions restricted by `onlyOwner` become permanently inaccessible.

- **`transferOwnership(address)`**

    This is also a public function and, like `renounceOwnership()`, is restricted by the `onlyOwner` modifier. When called with a valid argument address (not `address(0)`), it updates the contract’s ownership to that address, even if it’s the same as the current owner.


Below is the [OpenZeppelin Ownable contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol), provided for reference while explaining their approach to formal verification:


```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v5.0.0) (access/Ownable.sol)

pragma solidity ^0.8.20;

import {Context} from "../utils/Context.sol";

/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * The initial owner is set to the address provided by the deployer. This can
 * later be changed with {transferOwnership}.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
abstract contract Ownable is Context {
    address private _owner;

    /**
     * @dev The caller account is not authorized to perform an operation.
     */
    error OwnableUnauthorizedAccount(address account);

    /**
     * @dev The owner is not a valid owner account. (eg. `address(0)`)
     */
    error OwnableInvalidOwner(address owner);

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev Initializes the contract setting the address provided by the deployer as the initial owner.
     */
    constructor(address initialOwner) {
        if (initialOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(initialOwner);
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view virtual returns (address) {
        return _owner;
    }

    /**
     * @dev Throws if the sender is not the owner.
     */
    function _checkOwner() internal view virtual {
        if (owner() != _msgSender()) {
            revert OwnableUnauthorizedAccount(_msgSender());
        }
    }

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions. Can only be called by the current owner.
     *
     * NOTE: Renouncing ownership will leave the contract without an owner,
     * thereby disabling any functionality that is only available to the owner.
     */
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Can only be called by the current owner.
     */
    function transferOwnership(address newOwner) public virtual onlyOwner {
        if (newOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}
```


# OZ’s Ownable CVL Specifications


Here’s the CVL specification link for the Ownable contract: [Ownable CVL Specification](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/certora/specs/Ownable.spec). 


To simplify, we made minor modifications by placing the following definitions directly into the specification file, rather than importing them from `IOwnable.spec` and `helpers.spec`:

- `function owner() external returns (address) envfree` (placed within the `methods` block)
- `definition nonpayable(env e) returns bool = e.msg.value == 0`

This modified version below is functionally equivalent to the original and will be used for the discussion:


```javascript
methods {
    function restricted() external;
    function owner() external returns (address) envfree; // taken from the import IOwnable.spec
}


definition
 nonpayable(env e) returns bool = e.msg.value == 0; // taken from the helpers.spec

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Function correctness: transferOwnership changes ownership                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule transferOwnership(env e) {
    require nonpayable(e);

    address newOwner;
    address current = owner();

    transferOwnership@withrevert(e, newOwner);
    bool success = !lastReverted;

    assert success <=> (e.msg.sender == current && newOwner != 0), "unauthorized caller or invalid arg";
    assert success => owner() == newOwner, "current owner changed";
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Function correctness: renounceOwnership removes the owner                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule renounceOwnership(env e) {
    require nonpayable(e);

    address current = owner();

    renounceOwnership@withrevert(e);
    bool success = !lastReverted;

    assert success <=> e.msg.sender == current, "unauthorized caller";
    assert success => owner() == 0, "owner not cleared";
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Access control: only current owner can call restricted functions                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule onlyCurrentOwnerCanCallOnlyOwner(env e) {
    require nonpayable(e);

    address current = owner();

    calldataarg args;
    restricted@withrevert(e, args);

    assert !lastReverted <=> e.msg.sender == current, "access control failed";
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: ownership can only change in specific ways                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule onlyOwnerOrPendingOwnerCanChangeOwnership(env e) {
    address oldCurrent = owner();

    method f; calldataarg args;
    f(e, args);

    address newCurrent = owner();

    // If owner changes, must be either transferOwnership or renounceOwnership
    assert oldCurrent != newCurrent => (
        (e.msg.sender == oldCurrent && newCurrent != 0 && f.selector == sig:transferOwnership(address).selector) ||
        (e.msg.sender == oldCurrent && newCurrent == 0 && f.selector == sig:renounceOwnership().selector)
    );
}
```



Notice the use of the `definition` keyword. It declares reusable logic for common expressions used throughout the specification and is invoked across most rules:



```solidity
definition nonpayable(env e) returns bool = e.msg.value == 0;
```



The `nonpayable(env e)` returns `true` when the call is made without sending Ether, that is, when `msg.value == 0`. 



It’s also worth mentioning that the specification invokes a function named `restricted()`, which is not part of the `Ownable` contract. Instead, it is implemented in the [harness contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/e4f70216d759d8e6a64144a9e1f7bbeed78e7079/certora/harnesses/OwnableHarness.sol) that inherits from the abstract contract `Ownable`. 



When we run verification, we're verifying the harness contract. The harness often does more than that, but that’s a topic for another chapter. For now, what matters is that we added a `restricted()` function and applied the `onlyOwner` modifier.



Below is the implementation of the harness contract:


```solidity
/// OwnableHarness.sol

import {Ownable} from "src/Ownable/Ownable.sol"; // import path: modify as necessary

contract OwnableHarness is Ownable {
    constructor(address initialOwner) Ownable(initialOwner) {}

    function restricted() external onlyOwner {}
}
```


## Access Control


We start with the most fundamental rule: `onlyCurrentOwnerCanCallOnlyOwner().` This rule verifies that the `onlyOwner` modifier blocks non-owners from calling restricted functions:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Access control: only current owner can call restricted functions                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule onlyCurrentOwnerCanCallOnlyOwner(env e) {
    require nonpayable(e);

    address current = owner();

    calldataarg args;
    restricted@withrevert(e, args);

    assert !lastReverted <=> e.msg.sender == current, "access control failed";
}
```



The line `require nonpayable(e)` means the rest of the rule will only be evaluated for transactions that do not send Ether (`e.msg.value == 0`); all other inputs are excluded from analysis.

The line `address current = owner()` retrieves the current owner address, which is then used in the assertion.



Now consider the following lines, which need a bit more explanation:


```solidity
calldataarg args;
restricted@withrevert(e, args);
```



The line `calldataarg args` passes arbitrary arguments to the function call `restricted@withrevert(e, args)`. However, this can be simplified to:


```solidity
restricted@withrevert(e);
```



because the Prover automatically assigns arbitrary values to inputs by default, making `calldataarg` unnecessary in this case (also, the `restricted()` does not have a parameter). Both forms are valid and reflect the spec writer’s preference.


Now the next question is whether it should be included in the `methods` block:


```solidity
methods {
    function restricted() external;
    ...
}
```



The OpenZeppelin specification includes it. However, doing so produces a warning (even if this rule is verified):

> _“Method declaration for_ _`OwnableHarness.restricted()`_ _is neither_ _`envfree`_, _`optional`_, nor summarized, so it has no effect.”


As discussed in our previous chapter, if a function is environment-dependent and treated as such, it does not need to be included in the `methods` block. In this case, we can simplify by removing the line `function restricted() e``xternal` from the methods block.



Now finally, the assertion,


```solidity
assert !lastReverted <=> e.msg.sender == current, "access control failed";
```



This means _“the call succeeds if and only if the caller is the_ _`current`_ _address”_. In all other cases, the function must revert. Of course, the function can revert if it is sent with Ether, but we explicitly excluded that case in this rule using the `require nonpayable(e)` statement.


## **Renounce Ownership**


This rule verifies that only the owner can renounce ownership, and if successful, the owner address is set to `address(0)`. Here's the rule:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Function correctness: renounceOwnership removes the owner                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule renounceOwnership(env e) {
    require nonpayable(e);

    address current = owner();

    renounceOwnership@withrevert(e);
    bool success = !lastReverted;

    assert success <=> e.msg.sender == current, "unauthorized caller";
    assert success => owner() == 0, "owner not cleared";
}
```



At a glance, the rule sets `!lastReverted` to `success`, which is used in both assertions.



The first assertion:


```solidity
assert success <=> e.msg.sender == current, "unauthorized caller";
```



confirms the same logic as the last rule: _“the call succeeds if and only if the caller is the current owner”._



The second assertion:


 


```solidity
assert success => owner() == 0, "owner not cleared";
```



is an implication. It means that _“if the call succeeds, then the owner must be cleared (i.e., set to_ _`address(0)`_)”. In other words, ownership must be renounced.


## **Transfer Ownership**


This rule verifies that if the call succeeds, the caller must be the current owner and the new address must be a valid one (i.e., not `address(0)`).


Now we look at the rule for `transferOwnership()`:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Function correctness: transferOwnership changes ownership                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule transferOwnership(env e) {
    require nonpayable(e);

    address newOwner;
    address current = owner();

    transferOwnership@withrevert(e, newOwner);
    bool success = !lastReverted;

    assert success <=> (e.msg.sender == current && newOwner != 0), "unauthorized caller or invalid arg";
    assert success => owner() == newOwner, "current owner changed";
}
```


Like the previous rules, this assertion,  


```solidity
assert success <=> (e.msg.sender == current && newOwner != 0), "unauthorized caller or invalid arg";
```



It is based on the premise that the call succeeds if and only if the caller is the current owner and the `newOwner` is a valid (non-zero) address. The condition `newOwner != 0` is necessary because removing it would violate the Solidity function's internal check and result in a revert (thus a violation).


```solidity
/// Ownable.sol

function transferOwnership(address newOwner) public virtual onlyOwner {
    if (newOwner == address(0)) { // this line will cause a revert if newOwner is zero
        revert OwnableInvalidOwner(address(0));
    }
    _transferOwnership(newOwner);
}
```


Now for the last assertion for this rule,


```solidity
assert success => owner() == newOwner, "current owner changed";
```


this means that if the call succeeds, the owner must be updated to `newOwner`.


## Parametric Rules


Now that the three core features of the `Ownable` contract have been formally verified, one final rule remains: a parametric rule that verifies ownership changes. The rule asserts that if the owner changes, it must occur through:

- `transferOwnership()` which results in a non-zero new owner or
- `renounceOwnership()` which results in a zero address as the new owner

Here’s the rule:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: ownership can only change in specific ways                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule onlyOwnerOrPendingOwnerCanChangeOwnership(env e) {
    address oldCurrent = owner();

    method f; calldataarg args;
    f(e, args);

    address newCurrent = owner();

    // If owner changes, must be either transferOwnership or renounceOwnership
    assert oldCurrent != newCurrent => (
        (e.msg.sender == oldCurrent && newCurrent != 0 && f.selector == sig:transferOwnership(address).selector) ||
        (e.msg.sender == oldCurrent && newCurrent == 0 && f.selector == sig:renounceOwnership().selector)
    );
}
```



This is a parametric rule, as evidenced by the presence of the call `f(e, args)`. In the lines before and after the method `f(e, args)` call, we retrieve `oldCurrent` and `newCurrent` using `owner()`. It’s clear that we are tracking changes to the ownership state. With `f(e, args)` placed between these two calls, we are allowing arbitrary function execution and checking whether it causes a change in ownership.



Now the assertion:


```solidity
assert oldCurrent != newCurrent => (
    (e.msg.sender == oldCurrent && newCurrent != 0 && f.selector == sig:transferOwnership(address).selector) ||
    (e.msg.sender == oldCurrent && newCurrent == 0 && f.selector == sig:renounceOwnership().selector)
);
```



If the owner has changed (`oldCurrent != newCurrent`), then the change must have occurred in one of two valid ways:


    - `e.msg.sender == oldCurrent && newCurrent != 0 && f.selector == sig:transferOwnership(address).selector)`

        
    This means the caller must be the current owner, the new owner is not the zero address, and the function that triggered the change is `transferOwnership()`.
        

    - `e.msg.sender == oldCurrent && newCurrent == 0 && f.selector == sig:renounceOwnership().selector)`

        
    This means the caller is still the current owner, but the new owner is the zero address, and the function used must have been `renounceOwnership()`.
        


These are the only two legitimate paths for changing ownership according to the rule.
