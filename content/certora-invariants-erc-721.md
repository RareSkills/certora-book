# Invariants for ERC-721


## Introduction


In previous chapters, we explored how CVL invariants work: properties that must hold throughout all contract execution. Invariants can function as preconditions in rules and `preserved` conditions in other invariants, which allows verification to build on already-proven guarantees and removes the risk of flawed assumptions. Invariants are also automatically checked across all state-changing functions.



This chapter continues as part (2/5) of the code walkthrough of [OpenZeppelin's ERC-721 CVL specification](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/36bf1e46fa811f0f07d38eb9cfbc69a955f300ce/certora/specs/ERC721.spec) and focuses on invariant properties. It verifies the following five ERC-721 invariants:

- `balanceOfConsistency`

    A user's balance and ownership count are always equal.

- `ownerHasBalance`

    Token owners always have positive balances.

- `ownedTotalIsSumOfBalances`

    The total count of owned tokens equals the sum of all user balances.

- `notMintedUnset`

    Unminted tokens have no approved address set.

- `zeroAddressHasNoApprovedOperator`

    Unminted tokens have no approved operators.


These invariants guarantee that balances accurately reflect ownership and approvals remain consistent with ownership state.


## The `balanceOfConsistency` invariant


In ERC-721, there are three ways to determine an account’s token balance:

- by calling the public `balanceOf(address)` function
- by reading from the `_balances` mapping
- by reading from the storage `_owners` mapping and counting how many tokens are owned by the address

These three quantities must be equal because they each represent the same underlying fact: how many tokens the user owns. Any discrepancy would indicate inconsistent accounting between balances and ownership tracking. 


In the invariant `balanceOfConsistency`, these quantities correspond to `balanceOf(user)`, `_balances[user]`, and `_ownedByUser[user]`, each representing the user’s token balance:



```solidity
invariant balanceOfConsistency(address user)
    to_mathint(balanceOf(user)) == _ownedByUser[user] &&
    to_mathint(balanceOf(user)) == _balances[user]
    ...
```


### **How each balance representation is derived**


To verify this invariant, the specification captures each of these representations. The following explains how each one is derived.



**`balanceOf(user)`** **— public view function of a user’s balance**


The `balanceOf(user)` is a public view function that we can directly invoke in CVL to retrieve the user's balance. This function returns the value stored in the storage mapping variable `_balances`.


**`_balances[user]`** **—  token balances mirrored from storage**


The `_balances[user]` is a ghost variable that reads and mirrors the private storage mapping variable `_balances` through an `Sload` hook. 


It is initialized to zero for all addresses using an `init_state` axiom with a `forall` quantifier, which states that every address starts with a zero balance in the initial state, matching the contract’s default storage state: 


```solidity
ghost mapping(address => mathint) _balances {
    init_state axiom forall address a. _balances[a] == 0;
}
...
```


Without this initialization, the Prover assumes arbitrary starting values (base case) for balances, which can lead to incorrect accounting from the initial state. This invalidates the tracking logic and results in false positive violations during verification.


After initialization, the specification must reflect the storage reads in the ghost state using an `Sload` hook. The `Sload` hook reads values from the actual `_balances` storage slot and syncs those values with the ghost variable (the ghost and the storage variable share the same name but do not conflict):



```solidity
ghost mapping(address => mathint) _balances {
    init_state axiom forall address a. _balances[a] == 0;
}

hook Sload uint256 value _balances[KEY address user] {
    require _balances[user] == to_mathint(value);
}
```


The sync happens when the storage value returned by the `Sload` hook is constrained through `require` to equal the ghost variable. This sync is necessary because ghost variables do not update automatically on storage reads.



**`_ownedByUser[user]`** **— token ownership count** **derived from storage**


The `_ownedByUser[user]` is a ghost variable that represents the count of all `tokenId`s for which `ownerOf(tokenId) == user`:


```solidity
ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}
...
```


In other words, if we list all tokens and count how many are owned by `user`, we obtain `_ownedByUser[user]`. We do not want a scenario where `balanceOf(user)` reports 2, but the actual ownership count is 3 (for example, if `ownerOf(token1) == user`, `ownerOf(token2) == user`, and `ownerOf(token3) == user`).


The ghost variable `_ownedByUser[user]` stores how many tokens a user owns through the `Sstore` hook. The hook runs whenever the contract writes a new value to the contract’s private storage mapping `_owners[tokenId]`, and it observes both `oldOwner` and `newOwner` to determine how ownership has changed. 


By comparing these two values, we can classify the update as a mint, burn, or transfer:

- Mint: `oldOwner == address(0)` and `newOwner != address(0)`
- Burn: `oldOwner != address(0)` and `newOwner == address(0)`
- Transfer: both `oldOwner` and `newOwner` are nonzero addresses

The hook updates the ghost mapping `_ownedByUser` to reflect these ownership changes:


```solidity
ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + to_mathint(newOwner != 0 ? 1 : 0);
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - to_mathint(oldOwner != 0 ? 1 : 0);
    // _ownedTotal = _ownedTotal + to_mathint(newOwner != 0 ? 1 : 0) - to_mathint(oldOwner != 0 ? 1 : 0);
}
```


_Note: We intentionally commented out the line_ _`_ownedTotal = ...`_  _because it is not necessary at this point. We are not tracking the total number of owned tokens; we are only tracking the tokens owned by each individual user._


The first line in the `Sstore` hook `_owners` handles the token increment, which means mint and transfer happens:


```solidity
_ownedByUser[newOwner] = _ownedByUser[newOwner] + to_mathint(newOwner != 0 ? 1 : 0);
```

- If `newOwner != 0`, 1 is added to the `newOwner` token count. This applies to:
    - Mint: when a token is created and assigned to a user (`address(0)` to `newOwner`)
    - Transfer: when a token moves from one user to another (`oldOwner` to `newOwner`)
- If `newOwner == 0`, this indicates a burn, so no `newOwner` receives the token. The ternary expression evaluates to 0, so no addition occurs. However, that token loss (burn) still needs to be accounted for, and that is handled by the second line.

The second line in the `Sstore` hook `_owners` handles the token decrement, which means burn and transfer (also) happens:


```solidity
_ownedByUser[oldOwner] = _ownedByUser[oldOwner] - to_mathint(oldOwner != 0 ? 1 : 0);
```

- If `oldOwner != 0`, the previous (nonzero) owner held the token and now loses it, so we subtract 1. This applies to both transfer and burn operations.
- If `oldOwner == 0`, this indicates a mint, where the token is newly created with no previous owner. The ternary evaluates to 0, so no subtraction occurs. The ownership count is instead accounted for in the first line — the one that handles token increments.

Now that we have discussed the invariant expression and how the three representations of a user's token balance are derived, one final piece remains to complete the specification: handling external callbacks from safe mint and safe transfer operations.


Invariants automatically invoke all state-changing functions during verification, which means the Prover calls `safeMint()` and `safeTransferFrom()` — both of which trigger external callbacks to `onERC721Received()` on the recipient contract. We need to configure how the Prover handles this external call. Otherwise, these external calls are treated as unresolved by the Prover, which causes havoc on ghost variables and storage, and leads to false-positive violations of the invariant property.


### DISPATCHER — resolving `onERC721Received` callback


To resolve this callback, the specification dispatches the `onERC721Received()` call to a mock receiver contract that returns the `bytes4` selector `0x150b7a02` to signal ERC-721 support. To implement the dispatch, we need to:

- create a simple ERC721 receiver contract (mock),
- include it in the verification [scene](https://docs.certora.com/en/latest/docs/user-guide/glossary.html#term-scene),
- add a dispatch instruction in the `methods` block.

**The ERC721 mock receiver contract:**
Here is the receiver contract, implemented as a harness that returns  the `bytes4` selector `0x150b7a02`:


```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import "contracts/interfaces/IERC721Receiver.sol";

contract ERC721ReceiverHarness is IERC721Receiver {
    function onERC721Received(address, address, uint256, bytes calldata) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }
}
```


_**Note**__: Importing IERC721Receiver serves as an interface check, confirming that the contract conforms to the expected ERC721 receiver specification._


**Add mock receiver contract to the scene:**


Next, we include `ERC721ReceiverHarness.sol` in the scene by adding it to the `.conf` configuration file:


 


```json
// .conf

{
  "files": [
    "certora/harness/ERC721Harness.sol", 
    "certora/harness/ERC721ReceiverHarness.sol", 
  ],
  ...
}
```


At this point, the mock receiver is part of the verification scene and is available as a dispatch target.



**Add the DISPATCH instruction:** 


Then, in the `methods` block of the CVL specification, we add the dispatch instruction:



```javascript
// .spec

methods {
    function balanceOf(address) external returns (uint256) envfree;
    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}
```


The dispatch line: `function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true)` instructs the Prover to route `onERC721Received()` calls to all contracts in the verification scene (`_` is a wildcard) that implement `onERC721Received()` and test its implementation against the `balanceOfConsistency` invariant. Since we only have one contract that implements `onERC721Received`, which is the mock receiver, then that is the only contract that the Prover will test.



With `DISPATCHER(true)`, only the contracts in the scene are selected for dispatch, and the Prover tries each matching implementation to determine if any can cause a violation.


The mock receiver contract used here is intentionally minimal. It implements only the `onERC721Received()` function and returns the required selector `0x150b7a02`. It contains no logic that changes the state or affects the storage of the calling contract. As a result, the callback cannot introduce any invariant violations.



Even if the mock receiver causes `safeMint()` or `safeTransferFrom()` (including their `bytes` variants) to revert — for example, by returning an incorrect callback value — the invariants remain unaffected because reverts do not cause any state changes. However, this is not the case when we formally verify the safe mint and safe transfer functions in a later chapter, where reverting callbacks cause assertion failures.


In this CVL invariant specification, the role of `DISPATCHER` is simply to resolve the external call. Without it, `safeMint()` and `safeTransferFrom()` — including their variants — invoke unresolved external calls, which causes havoc and leads to false-positive invariant failures.


### The full specification and Prover run for the `balanceOfConsistency` invariant


Here's the complete CVL specification for the `balanceOfConsistency` invariant, which includes the `methods` block, ghost variable declarations and hook implementations: 


```solidity
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: ownership count                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + to_mathint(newOwner != 0 ? 1 : 0);
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - to_mathint(oldOwner != 0 ? 1 : 0);
    // _ownedTotal = _ownedTotal + to_mathint(newOwner != 0 ? 1 : 0) - to_mathint(oldOwner != 0 ? 1 : 0);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: balances                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mapping(address => mathint) _balances {
    init_state axiom forall address a. _balances[a] == 0;
}

hook Sload uint256 value _balances[KEY address user] {
    require _balances[user] == to_mathint(value);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: balanceOf is the number of tokens owned                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant balanceOfConsistency(address user)
    to_mathint(balanceOf(user)) == _ownedByUser[user] &&
    to_mathint(balanceOf(user)) == _balances[user];
    // {
    //     preserved {
    //         require balanceLimited(user);
    //     }
    // }
```


_Note: The preserved block is intentionally commented out because the invariant successfully verifies withou__t it, which_ _means it is not strictly necessary._ 


Here's the verified Prover [run](https://prover.certora.com/output/541734/0e14c478096945f0b4ed79ff5df37c65?anonymousKey=5d00536de74e5aa6b4529ebf48ddf6308dfc95a1).


## The `ownerHasBalance` invariant — t**oken owners have positive balances**


This invariant states that if a `tokenId` exists (owner is nonzero), then that owner's balance must be greater than zero: 


 


```solidity
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0
    ...
```


This invariant is used as a precondition in later chapters when verifying transfers, where the sender should have a positive balance. We could hardcode `unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0` directly into the rule, but proving it as an invariant provides stronger safety guarantees.


A preserved block is needed here to require `balanceOfConsistency(ownerOf(tokenId))` before `ownerHasBalance` runs. This tells the Prover: “Before checking whether the owner has balance, first confirm that `balanceOf` matches the ownership count tracked by the ghosts.” It prevents the Prover from entering inconsistent states where an owner exists but appears to have a zero balance due to havoc.


Here's the CVL invariant: 


```solidity
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0
    {
        preserved {
            requireInvariant balanceOfConsistency(ownerOf(tokenId));
            // require balanceLimited(ownerOf(tokenId));
        }
    }
```


### The full specification and Prover run for the `ownerHasBalance` invariant


Here’s the complete CVL specification for the `ownerHasBalance` invariant, including the `methods` block, ghost variables, hooks, and the `preserved` `balanceOfConsistency` invariant: 




```javascript
methods {
    function ownerOf(uint256) external returns (address) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: ownership count                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + to_mathint(newOwner != 0 ? 1 : 0);
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - to_mathint(oldOwner != 0 ? 1 : 0);
    // _ownedTotal = _ownedTotal + to_mathint(newOwner != 0 ? 1 : 0) - to_mathint(oldOwner != 0 ? 1 : 0);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: balances                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mapping(address => mathint) _balances {
    init_state axiom forall address a. _balances[a] == 0;
}

hook Sload uint256 value _balances[KEY address user] {
    require _balances[user] == to_mathint(value);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: balanceOf is the number of tokens owned                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant balanceOfConsistency(address user)
    to_mathint(balanceOf(user)) == _ownedByUser[user] &&
    to_mathint(balanceOf(user)) == _balances[user];
    // {
    //     preserved {
    //         require balanceLimited(user);
    //     }
    // }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: owner of a token must have some balance                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0 // fixed for Prover v8.3.1
    {
        preserved {
            requireInvariant balanceOfConsistency(ownerOf(tokenId));
            // require balanceLimited(ownerOf(tokenId));
        }
    }
```


_Note: The original invariant expression was_ _`balanceOf(ownerOf(tokenId)) > 0`__, but it was updated to_ _`unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0`_ _to verify with Prover version 8.3.1._


Here's the Prover [run](https://prover.certora.com/output/541734/6c199d1ba9454f5b920cddf5d00c1820?anonymousKey=94c99f4243131bfdbbc85dccdd50faa3eb6a6651).


## **The** **`ownedTota`****`lI`****`sSumOfBalances`** **invariant — total owned tokens equal sum of all balances**


In this invariant specification, the total number of owned tokens and the total supply are derived because the core ERC-721 standard does not expose any notion of total supply — only ERC-721 Enumerable defines `totalSupply()`. Since this invariant cannot be directly verified against a standard function, these values are reconstructed using ghost variables (`_ownedTotal` and `_supply`) and hooks.



`_ownedTotal` counts how many tokens currently exist by observing ownership changes through the `Sstore` hook on the storage variable `_owners`. `_supply` is the sum of all user balances, also tracked as a ghost.


With these two values reconstructed and tracked as ghosts, the invariant verifies that `_ownedTotal` equals `_supply`. This ensures consistent balance accounting. If owned tokens exceed supply, balances are undercounted (tokens exist but are not reflected in balances). If supply exceeds owned tokens, balances are overcounted (balances claim more tokens than actually exist). 


This equality therefore guarantees that every owned token is counted exactly once in the balance totals: 



```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: number of owned tokens is the sum of all balances                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant ownedTotalIsSumOfBalances()
	_ownedTotal == _supply
{
    // preserved blocks will be shown later
}
```


Both `_supply` and `_ownedTotal` are ghost mapping variables used to track the total token supply via `Sstore` hooks, but from different perspectives:

- `_supply` tracks the total supply by subtracting the old balance and adding the new balance each time there is a storage write to `_balances`:

    ```solidity
    hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
        _supply = _supply - oldValue + newValue;
    }
    ```


    By observing balances at the storage level, `_supply` is derived, and the effect on `_supply` depends on the type of operation: 

    - When a token is minted to a user and the balance changes from `oldValue = 2` to `newValue = 3`, `_supply` is computed as `_supply = _supply - 2 + 3`, which results in a net increase of 1.
    - When a token is burned from the owner and the balance changes from `oldValue = 3` to `newValue = 2`, `_supply` is computed as `_supply = _supply - 3 + 2`, which results in a net decrease of 1.
    - When a user transfers a token to another user, the sender’s balance decreases by 1 while the recipient’s balance increases by 1. The decrease from the sender offsets the increase to the recipient, so `_supply` remains unchanged.

     

- `_ownedTotal` tracks the number of owned tokens via an `Sstore` hook by subtracting 1 when ownership is cleared and adding 1 when a token is assigned to a nonzero address:

    ```solidity
    hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
        ...
        _ownedTotal = _ownedTotal + to_mathint(newOwner != 0 ? 1 : 0) - to_mathint(oldOwner != 0 ? 1 : 0);
    }
    ```


    Let’s simulate this based on ownership changes, but first, let’s rearrange the code for clarity:


    ```solidity
    _ownedTotal = _ownedTotal 
    			+ to_mathint(newOwner != 0 ? 1 : 0) 
    			- to_mathint(oldOwner != 0 ? 1 : 0);
    ```

    - If both `oldOwner` and `newOwner` are nonzero addresses, then a transfer occurs:
        - `newOwner` is nonzero, so the ternary expression evaluates to `1`, and `1` is added.
        - `oldOwner` is nonzero, so the ternary expression evaluates to `1`, and `1` is subtracted.

        The net effect is no change to `_ownedTotal`, since `1` is added and `1` is subtracted. By principle, a transfer operation does not affect the total ownership count.

    - If `oldOwner` is zero and `newOwner` is nonzero, then a mint occurs:
        - `newOwner` is nonzero, so the ternary expression evaluates to `1`, and `1` is added.
        - `oldOwner` is zero, so the ternary expression evaluates to `0`, and `0` is subtracted.

        The net effect is `+1`, because `1` is added and nothing is subtracted. By principle, a mint operation adds to the total ownership count.

    - If `oldOwner` is nonzero and `newOwner` is zero, then a burn occurs:
        - `newOwner` is zero, so the ternary expression evaluates to `0`, and `0` is added.
        - `oldOwner` is nonzero, so the ternary expression evaluates to `1`, and `1` is subtracted.

        The net effect is `–1`, because nothing is added and `1` is subtracted. By principle, a burn operation reduces the total ownership count.


**Preserved Clauses:** 


The invariant includes preserved clauses. The following summarizes each of them:

- For mint: `mint()`, `safeMint()`, `safeMint(``bytes``)`
    - `require balanceLimited(to)`

        The recipient's balance must remain below `max_uint256` to prevent the Prover from exploring unrealistic overflow paths.

- For burn:`burn()`
    - `requireInvariant ownerHasBalance(tokenId)`

        The token must have an owner with a positive balance before being burned.

- For transfer:`transferFrom()`, `safeTransferFrom()`, `safeTransferFrom(``bytes``)`
    - `require balanceLimited(to)`

        The recipient’s balance must stay below `max_uint256`.

    - `requireInvariant ownerHasBalance(tokenId)`

        The token must come from a valid owner with a nonzero balance.


Here is the invariant that includes the preserved clauses:


```javascript
invariant ownedTotalIsSumOfBalances()
    _ownedTotal == _supply
    {
        preserved mint(address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
        }
        preserved safeMint(address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
        }
        preserved safeMint(address to, uint256 tokenId, bytes data) with (env e) {
            require balanceLimited(to);
        }
        preserved burn(uint256 tokenId) with (env e) {
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(ownerOf(tokenId));
        }
        preserved transferFrom(address from, address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
        preserved safeTransferFrom(address from, address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
        preserved safeTransferFrom(address from, address to, uint256 tokenId, bytes data) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
    }
```


_Note: The invariant_ _`ownedTotalIsSumOfBalances`_ _successfully verifies even without_ _`requireInvariant balanceOfConsistency`__. For discussion purposes, we commented it out so that the reader can see which preserved conditions are strictly required to verify the invariant._ _This doesn't mean it's useless — since it is already proven, it can be assumed to always hold. Hence, it can safely be reinstated. Adding it as preserved condition generally shortens Prover run times._


### The full specification and Prover run for the `ownedTotalIsSumOfBalances` invariant


Here’s the complete CVL specification for the `ownedTotalIsSumOfBalances` invariant, which includes the `methods` block, ghost variables, hooks, and the `preserved` `ownerHasBalance` and `balanceOfConsistency` invariants: 


```solidity
methods {
    function ownerOf(uint256) external returns (address) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: ownership count                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

ghost mathint _ownedTotal {
    init_state axiom _ownedTotal == 0;
}

hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + to_mathint(newOwner != 0 ? 1 : 0);
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - to_mathint(oldOwner != 0 ? 1 : 0);
    _ownedTotal = _ownedTotal + to_mathint(newOwner != 0 ? 1 : 0) - to_mathint(oldOwner != 0 ? 1 : 0);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: sum of all balances                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint _supply {
    init_state axiom _supply == 0;
}

ghost mapping(address => mathint) _balances {
    init_state axiom forall address a. _balances[a] == 0;
}

hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
    _supply = _supply - oldValue + newValue;
}

hook Sload uint256 value _balances[KEY address user] {
    require _balances[user] == to_mathint(value);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: balanceOf is the number of tokens owned                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant balanceOfConsistency(address user)
    to_mathint(balanceOf(user)) == _ownedByUser[user] &&
    to_mathint(balanceOf(user)) == _balances[user];
    // {
    //     preserved {
    //         require balanceLimited(user);
    //     }
    // }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: owner of a token must have some balance                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0
    {
        preserved {
            requireInvariant balanceOfConsistency(ownerOf(tokenId));
            // require balanceLimited(ownerOf(tokenId));
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: number of owned tokens is the sum of all balances                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant ownedTotalIsSumOfBalances()
    _ownedTotal == _supply
    {
        preserved mint(address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
        }
        preserved safeMint(address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
        }
        preserved safeMint(address to, uint256 tokenId, bytes data) with (env e) {
            require balanceLimited(to);
        }
        preserved burn(uint256 tokenId) with (env e) {
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(ownerOf(tokenId));
        }
        preserved transferFrom(address from, address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
        preserved safeTransferFrom(address from, address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
        preserved safeTransferFrom(address from, address to, uint256 tokenId, bytes data) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
    }
```


Prover run: [link](https://prover.certora.com/output/541734/362a9792e1dd4eb0857c3150736d50c0?anonymousKey=1a0edaa499a7e67df5ae9453b745bd580ec7c112)


## The `notMintedUnset` invariant — u**nminted tokens have no approvals**


If a token is not minted (i.e. it has no owner), then it must also have no approved address:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: tokens that do not exist are not owned and not approved                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant notMintedUnset(uint256 tokenId)
    unsafeOwnerOf(tokenId) == 0 => unsafeGetApproved(tokenId) == 0;
```


The `unsafeOwnerOf()` and `unsafeGetApproved()` are harness view functions that return the zero address instead of reverting when a `tokenId` is unminted. In contrast, the standard `ownerOf()` and `getApproved()` functions revert for unminted tokens and therefore never return these zero address values.


Using `ownerOf()` and `getApproved()` would create a problem for invariant evaluation. Invariants must be boolean expressions that evaluate to either `true` or `false`, but when the Prover evaluates an invariant that calls `ownerOf(tokenId)` or `getApproved(tokenId)` on an unminted `tokenId`, the function call reverts. A revert is neither `true` nor `false`, which prevents the invariant expression from evaluating to a boolean value. While rules can use `@withrevert` and inspect the `lastReverted` flag to handle reverting calls, invariants are pure boolean expressions and do not support such control-flow logic.


By using harness functions `unsafeOwnerOf()` and `unsafeGetApproved()` that return the zero address instead of reverting, the Prover can evaluate the invariant expression to a boolean value for all `tokenId` values, including unminted tokens.   



This invariant is important because if an approval is somehow set for a `tokenId` that does not exist, and that `tokenId` is later minted to a different address, the previously approved address would still retain transfer rights. This could allow unauthorized transfers immediately after minting. The invariant guarantees that all tokens start with the default state — no owner, no approval.


### The full specification and Prover run for the `notMintedUnset` invariant



```javascript
methods {
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;

    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: tokens that do not exist are not owned and not approved                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant notMintedUnset(uint256 tokenId)
    unsafeOwnerOf(tokenId) == 0 => unsafeGetApproved(tokenId) == 0;
```


Prover run: [link](https://prover.certora.com/output/541734/999aeca09efa465f994611e1785c5680?anonymousKey=0d03a13d7dae940d43045e00a5743dcf6f6e936a)


## **The** **`zeroAddressHasNoApprovedOperator`**  **invariant —** **unminted tokens have no approved** **operator**


`isApprovedForAll` is a public function that checks whether an operator (an address) is authorized to manage all tokens owned by a particular account. This invariant states that no address should ever be approved as an operator for `address(0)`. Since `address(0)` is the implicit owner of unminted tokens, this prevents any operator from being pre-authorized to manage tokens before they're minted.



Therefore, the view function `isApprovedForAll(0, a)` must always return `false`:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: address(0) has no authorized operator                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant zeroAddressHasNoApprovedOperator(address a)
    !isApprovedForAll(0, a)
    {
        preserved with (env e) {
            require nonzerosender(e);
        }
    }
```


However, this holds only under the preserved condition that the caller (`msg.sender`) is not the zero address (`require nonzerosender(e)`) as per the definition: 


 


```solidity
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
```


This assumption makes sense because `address(0)` has no private key, so no one can initiate transactions from this address, which makes it impossible to call any function in reality. However, since the `setApprovalForAll()` function does not explicitly prevent the zero address from calling it, the Prover will explore this possibility, which can lead to false positive violations, where the Prover reports a violation that cannot occur in practice.


With the preserved condition ensuring `msg.sender` is nonzero, this invariant guarantees that `address(0)` can never have approved operators. Hence, unminted tokens cannot have an operator.


### The full specification and Prover run for the `zeroAddressHasNoApprovedOperator` invariant


Now here's the full specification of the invariant `zeroAddressHasNoApprovedOperato``r`, which includes the `methods` block and the `definition`: 



```javascript
methods {
    function isApprovedForAll(address,address) external returns (bool) envfree;
    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonzerosender(env e) returns bool = e.msg.sender != 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: address(0) has no authorized operator                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant zeroAddressHasNoApprovedOperator(address a)
    !isApprovedForAll(0, a)
    {
        preserved with (env e) {
            require nonzerosender(e);
        }
    }
```


Prover run: [link](https://prover.certora.com/output/541734/2f5386ccd3664da398d2a334507b78e0?anonymousKey=a9c6fba6f8b9cdd4ad246388674e122190797d9b)


## Consolidated CVL Specification For All Invariants


Now that we have discussed each invariant independently, we can consolidate them into a single specification: 


```javascript
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function ownerOf(uint256) external returns (address) envfree;
    function isApprovedForAll(address,address) external returns (bool) envfree;
    
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;

    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: ownership count                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint _ownedTotal {
    init_state axiom _ownedTotal == 0;
}

ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + to_mathint(newOwner != 0 ? 1 : 0);
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - to_mathint(oldOwner != 0 ? 1 : 0);
    _ownedTotal = _ownedTotal + to_mathint(newOwner != 0 ? 1 : 0) - to_mathint(oldOwner != 0 ? 1 : 0);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: balances                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint _supply {
    init_state axiom _supply == 0;
}

ghost mapping(address => mathint) _balances {
    init_state axiom forall address a. _balances[a] == 0;
}

hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
    _supply = _supply - oldValue + newValue;
}

hook Sload uint256 value _balances[KEY address user] {
    require _balances[user] == to_mathint(value);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: balanceOf is the number of tokens owned                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant balanceOfConsistency(address user)
    to_mathint(balanceOf(user)) == _ownedByUser[user] &&
    to_mathint(balanceOf(user)) == _balances[user];
    // {
    //     preserved {
    //         require balanceLimited(user);
    //     }
    // }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: owner of a token must have some balance                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0 // fixed for Prover V8
    {
        preserved {
            requireInvariant balanceOfConsistency(ownerOf(tokenId));
            // require balanceLimited(ownerOf(tokenId));
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: number of owned tokens is the sum of all balances                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant ownedTotalIsSumOfBalances()
    _ownedTotal == _supply
    {
        preserved mint(address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
        }
        preserved safeMint(address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
        }
        preserved safeMint(address to, uint256 tokenId, bytes data) with (env e) {
            require balanceLimited(to);
        }
        preserved burn(uint256 tokenId) with (env e) {
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(ownerOf(tokenId));
        }
        preserved transferFrom(address from, address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
        preserved safeTransferFrom(address from, address to, uint256 tokenId) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
        preserved safeTransferFrom(address from, address to, uint256 tokenId, bytes data) with (env e) {
            require balanceLimited(to);
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant balanceOfConsistency(from);
            // requireInvariant balanceOfConsistency(to);
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: tokens that do not exist are not owned and not approved                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant notMintedUnset(uint256 tokenId)
    unsafeOwnerOf(tokenId) == 0 => unsafeGetApproved(tokenId) == 0;


/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: address(0) has no authorized operator                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant zeroAddressHasNoApprovedOperator(address a)
    !isApprovedForAll(0, a)
    {
        preserved with (env e) {
            require nonzerosender(e);
        }
    }
```


  Here's the Prover [run](https://prover.certora.com/output/541734/f7aa089c0cb5442c9d225d91c5279080?anonymousKey=7b79ecebe4d14c20a9b0a143a7735a9b2a857742).
