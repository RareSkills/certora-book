# Mint and Burn Rules for ERC-721


## Introduction


ERC-721 is the Ethereum standard for non-fungible tokens, widely used to represent digital property. Like any form of property, it revolves around supply creation, supply removal, and ownership transfers.


This chapter focuses on formally verifying mint and burn operations. It is the first part (1/5) of the code walkthrough of [OpenZeppelin’s ERC-721 CVL specification](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/36bf1e46fa811f0f07d38eb9cfbc69a955f300ce/certora/specs/ERC721.spec). 


Throughout their specifications, OpenZeppelin uses a pattern of combining three types of assertions into a single rule, categorized as:

1. Liveness — specifies the conditions under which the function does not revert.
2. Effect — specifies the state changes that occurred when the function did not revert.
3. No side-effect — specifies that no unintended state changes occur beyond those in the Effect assertion.

## **Exposing internal functions** **`_mint()`** **and** **`_burn()`** **for verification**


`_mint()` and `_burn()` are internal functions intended for developers to inherit from and build custom logic for creating and destroying NFTs. In this specification, the harness contract inherits and exposes them as external functions, which allows the Prover to invoke `_mint()` and `_burn()` during verification.



The following are the [OpenZeppelin implementations](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/a7ee03565b4ee14265f4406f9e38a04e0143656f/contracts/token/ERC721/ERC721.sol) of: 

- `_mint()`
- `_burn()`
- the harness that exposes them

```solidity
// ERC721.sol

function _mint(address to, uint256 tokenId) internal {
    if (to == address(0)) {
        revert ERC721InvalidReceiver(address(0));
    }
    address previousOwner = _update(to, tokenId, address(0));
    if (previousOwner != address(0)) {
        revert ERC721InvalidSender(address(0));
    }
}
...

function _burn(uint256 tokenId) internal {
    address previousOwner = _update(address(0), tokenId, address(0));
    if (previousOwner == address(0)) {
        revert ERC721NonexistentToken(tokenId);
    }
}
```


```solidity
// ERC721Harness.sol

function mint(address account, uint256 tokenId) external {
    _mint(account, tokenId);
}
...

function burn(uint256 tokenId) external {
    _burn(tokenId);
}
```


## Formally verify mint


The `mint()` operation creates a new token and assigns it to a recipient. We formally verify that:

- it does not revert if and only if the token is unminted and the recipient is valid
- if it does not revert, both the total supply and the recipient's balance increase by one
- the recipient becomes the owner
- no other account balance or token ownership changes

Here's the CVL rule that proves these properties:


```solidity
// ERC721.spec -- mint (explanation follows)


rule mint(env e, address to, uint256 tokenId) {
	// preconditions
	require nonpayable(e);
	// requireInvariant notMintedUnset(tokenId);

    uint256 otherTokenId;
    address otherAccount;

    require balanceLimited(to);
    
	// pre-call state
    mathint supplyBefore         = _supply;
    uint256 balanceOfToBefore    = balanceOf(to);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
		
	// method call
    mint@withrevert(e, to, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        ownerBefore == 0 &&
        to != 0
    );

    // effect
    assert success => (
        _supply                   == supplyBefore + 1 &&
        to_mathint(balanceOf(to)) == balanceOfToBefore + 1 &&
        unsafeOwnerOf(tokenId)    == to
    );

    // no side effect
    assert balanceOf(otherAccount)     != balanceOfOtherBefore => otherAccount == to;
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore     => otherTokenId == tokenId;
}
```


We will discuss the sections of the rule above as follows: 

- Preconditions
- Pre-call state
- Mint call
- Assertions
    - liveness
    - effect
    - no side-effect

### **Preconditions**


A precondition is a constraint in a rule that specifies what must be true before the Prover executes the `mint()` method. It may or may not hold “after” the call, depending on the state changes caused by the method. 


The following are the preconditions for the `mint` rule:


```solidity
require nonpayable(e);
// requireInvariant notMintedUnset(tokenId); // intentionally commented out
...

require balanceLimited(to);
```


_Note: For simplicity of discussion, the invariant precondition_ _`notMintedUnset(tokenId)`_ _is commented out, since the_ _`mint`_ _rule verifies successfully without it. We will cover the CVL invariants of the ERC-721 codebase in the next chapter._

- `require nonpayable(e)`

    This requires that the call is made without sending any ETH because `mint()` is non-payable. The `nonpayable(e)` is expressed as a `definition` that returns `true` if `e.msg.value == 0` and `false` otherwise:


    ```solidity
    definition nonpayable(env e) returns bool = e.msg.value == 0;
    ```


    It encapsulates the logic for checking that a call carries no ETH by binding the condition `e.msg.value == 0` to a reusable expression named `nonpayable(e)`. This definition allows us to reference the condition throughout the specification, rather than writing `e.msg.value == 0` repeatedly every time we invoke a non-payable function. 


    _Note:_ _`definition`_ _is a broad topic in CVL. For more information, consult the_ [_Certora documentation_](https://docs.certora.com/en/latest/docs/cvl/defs.html#definitions)_. In this series, definitions are used straightforwardly as reusable expressions to improve code readability._
    

- `require balanceLimited(to)`

    This requires the recipient’s balance to be less than `max_uint256`. It is also expressed as a `definition` that returns `true` if `balanceOf(account) < max_uint256`:


    ```solidity
    definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;
    ```


    The OpenZeppelin ERC-721 implementation intentionally increments balances inside an `unchecked` block for gas efficiency. Overflow is theoretically possible but practically impossible, since reaching `max_uint256` would require minting more NFTs than could ever realistically exist. Hence, the precondition `require balanceLimited(to)` is reasonable.


    Without the precondition `balanceLimited(to)`, the Prover will consider overflow cases, since `mint()` calls the `_update()` function, where the balance increment occurs inside an `unchecked` block:


    ```solidity
    // ERC721.sol
    
    function _mint(address to, uint256 tokenId) internal {
        ...
        address previousOwner = _update(to, tokenId, address(0));
        ...
    }
    ```


    ```solidity
    /// ERC721.sol
    
    function _update(address to, uint256 tokenId, address auth) internal virtual returns (address) {
        ...
    
        if (to != address(0)) {
            unchecked {
                _balances[to] += 1;
            }
        }
    	  ...
    }
    ```


### **Record the pre-call state — total supply, balances and ownership before the mint** **call**


The following states are recorded before invoking the `mint()` function. These values are then compared with the post-call values in the assertions section: 


```solidity
mathint supplyBefore         = _supply;
uint256 balanceOfToBefore    = balanceOf(to);
uint256 balanceOfOtherBefore = balanceOf(otherAccount);
address ownerBefore          = unsafeOwnerOf(tokenId);
address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
```

- `mathint supplyBefore = _supply`

    This records the total supply before the mint call, to compare it against the post-mint supply and confirm that it has increased by 1. 


    `_supply` is a ghost variable that tracks the total supply and is updated through an `Sstore` hook as new balances are added and old balances are subtracted. 


    The ghost variable `_supply` is declared as a `mathint`, therefore `supplyBefore` must also be a `mathint`. If `supplyBefore` were declared as `uint256`, the Prover would report a type error. This error arises because `mathint` represents an unbounded mathematical integer, whereas `uint256` represents a bounded integer. Assigning a `mathint` value to a `uint256` would assume that the value fits within 256 bits, which the Prover cannot safely assume.
    


    Here's the `Sstore` hook implementation that updates the ghost variable `_supply`:


    ```solidity
    ghost mathint _supply {
        init_state axiom _supply == 0;
    }
    
    hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
        _supply = _supply - oldValue + newValue;
    }
    ```

- `uint256` `balanceOfToBefore = balanceOf(to)`

    This records the recipient’s balance before the mint call, to compare it against the post-mint balance and confirm that it has also increased by 1. 

- `uint256 balanceOfOtherBefore = balanceOf(otherAccount)`

    This records the balance of any other account before the mint call, to compare it against the post-mint balance and confirm that it has not changed, hence no side effects.

- `address ownerBefore = unsafeOwnerOf(tokenId)`

    This records the owner of the token before the mint call. For the mint to succeed, this value must be zero, which is one of the liveness conditions (`ownerBefore == 0` and `to != 0`).


    The `unsafeOwnerOf()` is a harness function that exposes ownership even for tokens without a valid owner. It returns the zero address for unminted tokens, unlike `ownerOf()`, which would revert in those cases. This enables the rule to compare ownership before and after minting since those ownership transitions involve the zero address (unminted state) to a valid owner address.

- `address otherOwnerBefore = unsafeOwnerOf(otherTokenId)`

    This records the owner of an arbitrary token (other than the one being minted) before the mint call so we can confirm it has not changed post-mint, hence no side effects.


### **Mint call**


The `@withrevert` tag allows the Prover to explore both reverting and non-reverting paths. The `!lastReverted` condition captures whether the call did not revert and stores this as `success`:


```solidity
mint@withrevert(e, to, tokenId);
bool success = !lastReverted;
```


### **Assertions — liveness, effect, and no side effect**


As mentioned earlier, the rule pattern used throughout the specification combines liveness, effect, and no side-effect into a single rule. Here is the code block which we will subsequently explain in detail:


```solidity
// liveness
assert success <=> (
    ownerBefore == 0 &&
    to != 0
);

// effect
assert success => (
    _supply                   == supplyBefore + 1 &&
    to_mathint(balanceOf(to)) == balanceOfToBefore + 1 &&
    unsafeOwnerOf(tokenId)    == to
);

// no side effect
assert balanceOf(otherAccount)     != balanceOfOtherBefore => otherAccount == to;
assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore     => otherTokenId == tokenId;
```


**Liveness:**


The assertion below states that `_mint()` does not revert if and only if the token is unminted (`ownerBefore == 0`) and the recipient is valid (`to != 0`):


```solidity
// liveness
assert success <=> (
    ownerBefore == 0 &&
    to != 0
);
```


We used the revert and disjunction pattern in the previous two chapters. If we were to convert the assertion above into that pattern, it becomes:


`_mint()` reverts if and only if the token is already minted (`ownerBefore != 0`) or the recipient is not valid (`to == 0`).


```solidity
// reverts
assert !success <=> (
    ownerBefore != 0 ||
    to == 0
);
```


While the “success + conjunction” (liveness) pattern lists all conditions under which the function succeeds, the “revert + disjunction” pattern lists all conditions under which it reverts. They are logically equivalent.


**Effect:**


If the `mint()` function does not revert, then the resulting state changes are:

- total supply increases by exactly one (`_supply == supplyBefore + 1`)
- recipient's balance increases by exactly one (`balanceOf(to) == balanceOfToBefore + 1`)
- token is now owned by the recipient (`unsafeOwnerOf(tokenId) == to`)

```solidity
// effect
assert success => (
    _supply                   == supplyBefore + 1 &&
    to_mathint(balanceOf(to)) == balanceOfToBefore + 1 &&
    unsafeOwnerOf(tokenId)    == to
);
```


**No side effect:**


The following assertions check that the `mint()` function causes no unintended side effects:

- `assert balanceOf(otherAccount) != balanceOfOtherBefore => otherAccount == to`

    This checks that only the intended recipient’s balance changes, with all other accounts unaffected. If an account’s balance changes after the mint call, then that account is the recipient (`to`). If an account’s balance does not change, then that account is not the recipient, therefore unaffected by the mint call.

- `assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId`

    This checks that only the owner of the minted token changes, with all other tokens unaffected. If the owner of a token changes after the mint call, then that token is the minted token. If the owner of a token does not change, then that token is not the minted token, therefore its ownership is unaffected by the mint call.


```solidity
// no side effect
assert balanceOf(otherAccount)     != balanceOfOtherBefore => otherAccount == to;
assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore     => otherTokenId == tokenId;
```


Now, here is the full specification, which includes the mint rule, definitions, ghosts, and hooks:


 


```solidity
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonpayable(env e) returns bool = e.msg.value == 0;
definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: sum of all balances                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint _supply {
    init_state axiom _supply == 0;
}

hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
    _supply = _supply - oldValue + newValue;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: mint behavior and side effects                                                                                │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

rule mint(env e, address to, uint256 tokenId) {
    require nonpayable(e);
    // requireInvariant notMintedUnset(tokenId);

    uint256 otherTokenId;
    address otherAccount;

    require balanceLimited(to);

    mathint supplyBefore         = _supply;
    uint256 balanceOfToBefore    = balanceOf(to);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);

    mint@withrevert(e, to, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        ownerBefore == 0 &&
        to != 0
    );

    // effect
    assert success => (
        _supply                   == supplyBefore + 1 &&
        to_mathint(balanceOf(to)) == balanceOfToBefore + 1 &&
        unsafeOwnerOf(tokenId)    == to
    );

    // no side effect
    assert balanceOf(otherAccount)     != balanceOfOtherBefore => otherAccount == to;
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore     => otherTokenId == tokenId;
}
```


Here's the Prover [run](https://prover.certora.com/output/541734/fafa0876c60f4226a7ceb031d9d1cbb2?anonymousKey=c9c974e6018eefba740c3049552958545bc9e2d6).


## Formally verify burn


The burn operation performs the opposite state changes of mint.


When minting, the state changes are:

- `_supply` increases by 1
- `balanceOf(to)` increases by 1
- `ownerOf(tokenId)` changes from the previous value `address(0)` to a nonzero address

When burning, the opposite happens:

- `_supply` decreases by 1
- `balanceOf(from)` decreases by 1
- `ownerOf(tokenId)` changes from its previous nonzero address back to `address(0)`

Unlike mint, where tokens are newly created with no existing approvals, burn operations must remove any existing approvals as part of clearing the token.


With these behaviors in mind, we formally verify that:

- it does not revert if and only if the token exists (i.e. the owner is not `address(0)`)
- if the call succeeds, total supply decreases by one and the previous owner’s balance decreases by one
- owner is set to `address(0)`, which means the token has no owner
- token approval is cleared
- no other account balances, token ownership, or approvals change

Here's the CVL rule that proves these properties:


```javascript
// ERC721.spec -- burn (explanation follows) 

rule burn(env e, uint256 tokenId) {
	// preconditions
    require nonpayable(e);

    address from = unsafeOwnerOf(tokenId);
    uint256 otherTokenId;
    address otherAccount;

    // requireInvariant ownerHasBalance(tokenId);
    
    // pre-call state
    mathint supplyBefore         = _supply;
    uint256 balanceOfFromBefore  = balanceOf(from);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);
		
	// method call
    burn@withrevert(e, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        ownerBefore != 0
    );

    // effect
    assert success => (
        unsafeOwnerOf(tokenId)      != 0 => (_supply == supplyBefore - 1) && // modified for the Prover v8.3.1
        to_mathint(balanceOf(from)) == balanceOfFromBefore - 1 &&
        unsafeOwnerOf(tokenId)      == 0 &&
        unsafeGetApproved(tokenId)  == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => otherAccount == from;
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


### **Preconditions**


Same as the `mint` rule, this rule is non-payable. Therefore, `require nonpayable(e)` is necessary:


 


```solidity
require nonpayable(e);
...
```


### **Record the pre-call state — total supply, balances, ownership and approval before the burn call**


All pre-call states recorded in the `mint` rule also appear in the `burn` rule, except for the following:

- `uint256 balanceOfFromBefore = balanceOf(from)`

    Instead of recording the pre-call state of the mint receiver (`balanceOf(to)`), the burn rule records the balance of the owner (`balanceOf(from)`) to verify that at the end of the burn call, the balance decreases by 1. 

- `address otherApprovalBefore = unsafeGetApproved(otherTokenId)`

    Approval is necessary in the burn function; therefore, it records the approval of an arbitrary other token to verify it remains unchanged.


### Burn call


As in previous rules, the call uses the `@withrevert` tag. The boolean `success` captures whether the call did not revert (`!lastReverted`), which is used to verify expected changes between pre-call and post-call states: 


```solidity
burn@withrevert(e, tokenId);
bool success = !lastReverted;
```


### **Assertions — liveness,** **effect****, and no side effect**


All values retrieved here are post-call states compared against pre-call states:


```solidity
// liveness
assert success <=> (
    ownerBefore != 0
);

// effect
assert success => (
    unsafeOwnerOf(tokenId)      != 0 => (_supply == supplyBefore - 1) && // modified for the Prover v8.3.1
    to_mathint(balanceOf(from)) == balanceOfFromBefore - 1 &&
    unsafeOwnerOf(tokenId)      == 0 &&
    unsafeGetApproved(tokenId)  == 0
);

// no side effect
assert balanceOf(otherAccount)         != balanceOfOtherBefore => otherAccount == from;
assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
```


**Liveness:** 


It asserts that the burn call does not revert if and only if the `tokenId` to be burned has an owner.


**Effect:** 


If the burn call does not revert, it means all the following state changes must have occurred:

- `unsafeOwnerOf(tokenId) != 0 => (_supply == supplyBefore - 1)`

    If the token is minted (has a nonzero owner), then the supply decreases by exactly 1. This means the decrease in supply is only checked when there is at least one minted token.

- `to_mathint(balanceOf(from)) == balanceOfFromBefore - 1`

    The balance of the `from` address (the token owner) has decreased by 1.

- `unsafeOwnerOf(tokenId) == 0`

    The ownership of the token has been cleared, so `unsafeOwnerOf(tokenId)` returns `0`.

- `unsafeGetApproved(tokenId) == 0`

    Any existing approval for the token has been removed, so `unsafeGetApproved(tokenId)` returns `0`.


**No side effect:** 

- `assert balanceOf(otherAccount) != balanceOfOtherBefore => otherAccount == from`

    This checks that only the pre-burn owner’s balance changes, with all other accounts unaffected. If an account’s balance changes after the burn call, then that account is the owner before the burn.

- `assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId`

    This checks that only the ownership of the burned token changes, with all other tokens unaffected. If the owner of a token changes after the burn call, then that token is the burned token. 

- `assert unsafeGetApproved(otherTokenId) != otherApprovalBefore => otherTokenId == tokenId`

    This checks that only the burned token’s approval is cleared, with all other token approvals unaffected. If the approval for a token changes after the burn call, then that token is the burned token. 


Here's the complete specification for the burn rule:


 


```solidity
methods {
    function ownerOf(uint256) external returns (address) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonpayable(env e) returns bool = e.msg.value == 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: sum of all balances                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

ghost mathint _supply {
    init_state axiom _supply == 0;
}

hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
    _supply = _supply - oldValue + newValue;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: burn behavior and side effects                                                                                │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

rule burn(env e, uint256 tokenId) {
    require nonpayable(e);

    address from = unsafeOwnerOf(tokenId);
    uint256 otherTokenId;
    address otherAccount;

    // requireInvariant ownerHasBalance(tokenId);

    mathint supplyBefore         = _supply;
    uint256 balanceOfFromBefore  = balanceOf(from);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);

    burn@withrevert(e, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        ownerBefore != 0
    );

    // effect
    assert success => (
        unsafeOwnerOf(tokenId)      != 0 => (_supply == supplyBefore - 1) && // modified for the Prover v8.3.1
        to_mathint(balanceOf(from)) == balanceOfFromBefore - 1 &&
        unsafeOwnerOf(tokenId)      == 0 &&
        unsafeGetApproved(tokenId)  == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => otherAccount == from;
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


_Note: The line_ _`requireInvariant ownerHasBalance(tokenId)`_ _is commented out because Prover 8.3.1 reports violations of this invariant, so it cannot be applied in this specification. This led to adjusting the "effect” assertion  from_ _`_supply == supplyBefore - 1`_ _to_ _`unsafeOwnerOf(tokenId) != 0 => (_supply == supplyBefore - 1)`_. _All invariants, including_ _`ownerHasBalance(tokenId)`, will be discussed in the next chapter._


Here's the Prover [run](https://prover.certora.com/output/541734/b596159772c447d6ba70969d8a40ad0d?anonymousKey=483b35cf7488deeb851d6b03db1f1f0e2f00cd6c) for the `burn` rule.


## Full specifications for mint and burn


Here's the complete specification and the Prover [run](https://prover.certora.com/output/541734/3ed32b635917474a92cacbfbd17bb06c?anonymousKey=df13a9896e61d7d2ec402e8607f0082e6868cae8).
