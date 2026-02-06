# Transfer and Approval Rules for ERC-721


## Introduction


This chapter continues as the third part (3/5) of our code walkthrough of [OpenZeppelin’s ERC-721 CVL specification](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/36bf1e46fa811f0f07d38eb9cfbc69a955f300ce/certora/specs/ERC721.spec) and focuses on token transfer and approval rules, specifically:

- `transferFrom()`
- `approve()`
- `setApprovalForAll()`
- `zeroAddressBalanceRevert()`

These rules verify ownership transfers, per-token approvals, operator approvals for all tokens, and that querying `balanceOf(0)` reverts.


## Formally verify `transferFrom()`


The `transferFrom()` function changes the ownership of an NFT to another address. It assumes the NFT has already been minted, and it can only be called by the owner, an address approved for that token ID, or an operator approved for all tokens of the owner.


Here’s the `transferFrom` rule which we will explain section by section after the code block:


```solidity
rule transferFrom(env e, address from, address to, uint256 tokenId) {
	// preconditions
    require nonpayable(e);
    require authSanity(e);

    address operator = e.msg.sender;
    uint256 otherTokenId;
    address otherAccount;

    requireInvariant ownerHasBalance(tokenId);
    require balanceLimited(to);
    
    // pre-call state
    uint256 balanceOfFromBefore  = balanceOf(from);
    uint256 balanceOfToBefore    = balanceOf(to);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
    address approvalBefore       = unsafeGetApproved(tokenId);
    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);
		
	// method call
    transferFrom@withrevert(e, from, to, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        from == ownerBefore &&
        from != 0 &&
        to   != 0 &&
        (operator == from || operator == approvalBefore || isApprovedForAll(ownerBefore, operator))
    );

    // effect
    assert success => (
        to_mathint(balanceOf(from))            == balanceOfFromBefore - assert_uint256(from != to ? 1 : 0) &&
        to_mathint(balanceOf(to))              == balanceOfToBefore   + assert_uint256(from != to ? 1 : 0) &&
        unsafeOwnerOf(tokenId)                 == to &&
        unsafeGetApproved(tokenId)             == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


### **Preconditions**


These preconditions specify what must be true before the Prover executes `transferFrom()`. We explain each line in the code block afterward: 


```solidity
require nonpayable(e);
require authSanity(e);
...
requireInvariant ownerHasBalance(tokenId);
require balanceLimited(to);
```


`require nonpayable(e)`


This requires that `transferFrom()` is called without sending ETH, since the function is non-payable. It is expressed as a definition:


```javascript
definition nonpayable(env e) returns bool = e.msg.value == 0;
```



It defines the condition `e.msg.value == 0` as a reusable expression named `nonpayable(e)`, which checks that the call carries no ETH.


Although the [ERC-721 standard ](https://eips.ethereum.org/EIPS/eip-721)specifies `transferFrom()` as payable, OpenZeppelin implements it as non-payable. In practice, ETH is not expected to be sent alongside NFT transfers, so this has become a practical convention. The CVL rule enforces`e.msg.value == 0`.



`require authSanity(e)`


This requires that the caller is not the zero address. It is also expressed as a `definition`:


```solidity
definition authSanity(env e) returns bool = e.msg.sender != 0
```


Without this requirement, the Prover treats `address(0)` as a valid caller, even though no real call can originate from `address(0)`. The contract's authorization logic never permits `address(0)` as an owner, approved address, or operator, so any violations from `address(0)` acting as a caller are false positives.


`requireInvariant ownerHasBalance(tokenId)` 


This precondition requires the `ownerHasBalance(tokenId)` invariant (which guarantees that if a token exists, its owner has a positive balance) to hold at the start of the rule. It forces the Prover to begin in a state where any existing token has a valid (nonzero) owner:


```solidity
rule transferFrom(env e, address from, address to, uint256 tokenId) {
	// preconditions
	require nonpayable(e);
	require authSanity(e);
	...
	requireInvariant ownerHasBalance(tokenId); // invariant as a precondition
	require balanceLimited(to);
	...
}
```


```javascript
// invariant as a precondition

invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0
    ...
```


The `ownerHasBalance` invariant relies on another invariant `balanceOfConsistency` through preserved block, which verifies that `balanceOf`, `_ownedByUser`, and the ghost `_balances` are equal: 


```solidity
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0
    {
        preserved { // invariant as a preserved condition
            requireInvariant balanceOfConsistency(ownerOf(tokenId));  
        }
    }
```


```solidity
// invariant as a preserved condition 

invariant balanceOfConsistency(address user)
    to_mathint(balanceOf(user)) == _ownedByUser[user] &&
    to_mathint(balanceOf(user)) == _balances[user];
```


By using the invariant `ownerHasBalance(tokenId)` as a precondition, the `transferFrom` rule inherits the `balanceOfConsistency` guarantee: 


```solidity
rule transferFrom(env e, address from, address to, uint256 tokenId) {
	// preconditions
	require nonpayable(e);
	require authSanity(e);
	...
	requireInvariant ownerHasBalance(tokenId); // invariant as a precondition
	require balanceLimited(to);
	...
}
```


If this precondition is removed, the Prover can start the rule in impossible states where ownership and balances do not match — for example, an address could be recorded as the owner of a token while its balance is zero because the ghost values do not align with storage. This leads to false positive violations.


`require balanceLimited(to)`


The `balanceLimited(to)` limits the balance of the recipient (`to`) address to stay below `max_uint256` as expressed in the definition:


```solidity
definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;
```


The contract increments balances inside an `unchecked` block for gas efficiency. Although overflow is practically impossible, the Prover still explores overflow cases because `transferFrom()` calls `_update()`, where the balance increment occurs inside an `unchecked` block:


```solidity
// ERC721.sol

function transferFrom(address from, address to, uint256 tokenId) public virtual {
    ...
    address previousOwner = _update(to, tokenId, _msgSender());
    ..
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


Using `require balanceLimited(to)` rules out practically unreachable overflow states that would cause the rule to fail in unrealistic scenarios.


### **Pre-call state**


Before invoking the `tra``nsferFrom()` method , we record state variables for comparison against post-call values:

- `balanceOfFromBefore` is the balance of the sender (`from`) before the transfer.
- `balanceOfToBefore` is the balance of the recipient (`to`) before the transfer.
- `balanceOfOtherBefore` is the balance of an uninvolved account (`otherAccount`) before the transfer.
- `ownerBefore` is the owner of the token to be transferred.
- `otherOwnerBefore` is the owner of an uninvolved token.
- `approvalBefore` is the approval for the token to be transferred.
- `otherApprovalBefore` is the approval of an uninvolved token.

```solidity
uint256 balanceOfFromBefore  = balanceOf(from);
uint256 balanceOfToBefore    = balanceOf(to);
uint256 balanceOfOtherBefore = balanceOf(otherAccount);
address ownerBefore          = unsafeOwnerOf(tokenId);
address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
address approvalBefore       = unsafeGetApproved(tokenId);
address otherApprovalBefore  = unsafeGetApproved(otherTokenId);
```


Notice that we read approvals using `unsafeGetApproved()`, a harness function, rather than the contract's `getApproved()`. The harness version exposes the approval value even when the token is unminted or burned (`owner == address(0)`) because it returns `_getApproved(tokenId)` without checking whether the token exists or has an owner. In contrast, `getApproved()` in the actual contract performs an ownership check and reverts if the token is unminted or burned:


```solidity
// ERC721Harness.sol -- returns approval without checking token existence or ownership

function unsafeGetApproved(uint256 tokenId) external view returns (address) {
    return _getApproved(tokenId);
}
```


```solidity
// ERC721.sol -- performs an ownership check and reverts if the token is unminted or burned

function getApproved(uint256 tokenId) public view virtual returns (address) {
    _requireOwned(tokenId); // reverts if `owner == address(0)`

    return _getApproved(tokenId);
}
```


This prevents querying the approval state of unminted or burned tokens because `getApproved()` reverts in those cases. However, the rule needs to compare approval values across state transitions that pass through the zero-address state:

- token creation (unminted to minted) — owner transitions from `address(0)` to a nonzero address
- token destruction (minted to unminted/burned) — owner transitions from a nonzero address back to `address(0)`

Hence, the need for the `unsafeGetApproved()`, so the rule can read approval values even when the token is in the zero-address owner state.


Now that the pre-call state is recorded, we execute `transferFrom()` and reason about the post-call values in the assertions that follow.


### Transfer call 


The `transferFrom()` call with `@withrevert` instructs the Prover to explore both reverting and non-reverting paths. The `!lastReverted` condition captures whether the call did not revert and stores this as `success`:



```solidity
transferFrom@withrevert(e, from, to, tokenId);
bool success = !lastReverted;
```


### **Assertions — liveness, effect, and no side effect**


**Liveness:**  


A transfer does not revert if and only if (`<=>`) all of the following hold:

- `from == ownerBefore`

    The `from` address (which the token is being transferred from) must be the previous owner of the token.

- `from != 0`

    The `from` address must not be the zero address.

- `to != 0`

    The `to` address (receiver) must not be the zero address.

- `(operator == from || operator == approvalBefore || isApprovedForAll(ownerBefore, operator))`

    The operator (caller) must have authorization through at least one of these:

    - `operator == from` — the owner transfers their own token
    - `operator == approvalBefore` — the operator has specific approval for this token (via `approve()`)
    - `isApprovedForAll(ownerBefore, operator)` — the operator has blanket approval for all tokens from the owner (via `setApprovalForAll()`)

```solidity
// liveness
assert success <=> (
    from == ownerBefore &&
    from != 0 &&
    to   != 0 &&
    (operator == from || operator == approvalBefore || isApprovedForAll(ownerBefore, operator))
);
```


**Effect:**


If the transfer does not revert, the following state changes apply:

- `to_mathint(balanceOf(from)) == balanceOfFromBefore - assert_uint256(from != to ? 1 : 0)`

    The sender’s balance decreases by 1, except in a self-transfer (`from == to`) where it remains unchanged.

- `to_mathint(balanceOf(to)) == balanceOfToBefore + assert_uint256(from != to ? 1 : 0)`

    The receiver’s balance increases by 1, except in a self-transfer (`from == to`) where it remains unchanged.

- `unsafeOwnerOf(tokenId) == to`

    The receiver (`to`) address becomes the new owner of `tokenId`.

- `unsafeGetApproved(tokenId) == 0`

    The approval for `tokenId` is cleared.


```solidity
// effect
assert success => (
    to_mathint(balanceOf(from))            == balanceOfFromBefore - assert_uint256(from != to ? 1 : 0) &&
    to_mathint(balanceOf(to))              == balanceOfToBefore   + assert_uint256(from != to ? 1 : 0) &&
    unsafeOwnerOf(tokenId)                 == to &&
    unsafeGetApproved(tokenId)             == 0
);
```


**No side effect:** 


After asserting the intended state changes (effect), we also check that no unintended effects occur on balances, ownership, or approvals:


- **On balance**

    `balanceOf(otherAccount) != balanceOfOtherBefore => (otherAccount == from || otherAccount == to)`


    If the balance of any account changes, it implies (`=>`) that the account is either the sender or the receiver. No other uninvolved account had its balance modified.

- **On ownership**

    `unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId`


    If the ownership of any token changes, it implies (`=>`) that the token was the one transferred. No other uninvolved token had its ownership changed.

- **On approval**

    `unsafeGetApproved(otherTokenId) != otherApprovalBefore => otherTokenId == tokenId`


    If the approval of any token changes, it implies (`=>`) that the token was the one transferred. No other uninvolved token had its approval modified.


```solidity
// no side effect
assert balanceOf(otherAccount)         != balanceOfOtherBefore => (otherAccount == from || otherAccount == to);
assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
```


Here's the complete specifications for the rule `transferFrom`: 



```javascript
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function ownerOf(uint256) external returns (address) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function isApprovedForAll(address,address) external returns (bool)    envfree;

    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonpayable(env e) returns bool = e.msg.value == 0;
definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;
definition authSanity(env e) returns bool = e.msg.sender != 0;

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
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: transferFrom behavior and side effects                                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule transferFrom(env e, address from, address to, uint256 tokenId) {
    require nonpayable(e);
    require authSanity(e);

    address operator = e.msg.sender;
    uint256 otherTokenId;
    address otherAccount;

    requireInvariant ownerHasBalance(tokenId);
    require balanceLimited(to);

    uint256 balanceOfFromBefore  = balanceOf(from);
    uint256 balanceOfToBefore    = balanceOf(to);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
    address approvalBefore       = unsafeGetApproved(tokenId);
    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);

    transferFrom@withrevert(e, from, to, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        from == ownerBefore &&
        from != 0 &&
        to   != 0 &&
        (operator == from || operator == approvalBefore || isApprovedForAll(ownerBefore, operator))
    );

    // effect
    assert success => (
        to_mathint(balanceOf(from))            == balanceOfFromBefore - assert_uint256(from != to ? 1 : 0) &&
        to_mathint(balanceOf(to))              == balanceOfToBefore   + assert_uint256(from != to ? 1 : 0) &&
        unsafeOwnerOf(tokenId)                 == to &&
        unsafeGetApproved(tokenId)             == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```



Here's the Prover [run](https://prover.certora.com/output/541734/3d9b87eac8b74e67947e89426ee587ab?anonymousKey=414250e5d1cbe19db72aaf84fdb53ed81007e8ed).


Now that we have verified `transferFrom``()`, we can move to the approval mechanisms that authorize these transfers.


Recall that `transferFrom()` succeeds only when the caller has proper authorization: as the token owner, as an address approved for that token, or as an operator approved for all tokens of the owner. The next two rules verify the functions that grant these permissions: `approve()` for per-token authorization and `setApprovalForAll()` for operator-level authorization.


## Formally verify `approve()`


The `approve()` contract function grants an address permission to transfer a specific token. It succeeds only if the token exists and the caller is the owner or an operator (has blanket approval). 


Here’s the rule that verifies this behavior:


```solidity
rule approve(env e, address spender, uint256 tokenId) {
	// preconditions
    require nonpayable(e);
    require authSanity(e);
		
	// pre-call state
    address caller = e.msg.sender;
    address owner = unsafeOwnerOf(tokenId);
    uint256 otherTokenId;

    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);
		
	// method call
    approve@withrevert(e, spender, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        owner != 0 &&
        (owner == caller || isApprovedForAll(owner, caller))
    );

    // effect
    assert success => unsafeGetApproved(tokenId) == spender;

    // no side effect
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


### **Preconditions**


By now, we already know that `nonpayable(e)` requires the call to be sent without Ether, and `authSanity(e)` requires that `msg.sender` is not the zero address:


```solidity
require nonpayable(e);
require authSanity(e);
```


### **Approve call**


As a common pattern in OpenZeppelin's ERC-721 rule specifications, the `approve` method is called with `@withrevert`, and the result of `!lastReverted` is stored in the `success` variable to reason about the case where the call succeeds:


```solidity
approve@withrevert(e, spender, tokenId);
bool success = !lastReverted;
```


### **Assertions — liveness, effect, and no side effect**


**Liveness:**


This means that the function call does not revert “if and only if” the following conditions are met:

- `owner != 0`

    The token owner must not be the zero address. This prevents approving a token that doesn’t exist or was burned.

- `(owner == caller || isApprovedForAll(owner, caller))`

    This is an authorization check where the `caller` must either:

    - Be the token `owner`, or
    - Be an operator approved via `isApprovedForAll(owner, caller)` (returns true)

```solidity
// liveness
assert success <=> (
    owner != 0 &&
    (owner == caller || isApprovedForAll(owner, caller))
);
```


**Effect:**

- `assert success => unsafeGetApproved(tokenId) == spender`

    If the call succeeded (success == true), then the approved address for `tokenId` must now be equal to `spender`.


```solidity
// effect
assert success => unsafeGetApproved(tokenId) == spender;
```


**No side effect:**

- `assert unsafeGetApproved(otherTokenId) != otherApprovalBefore => otherTokenId == tokenId`

    If the approved address for `otherTokenId` changes, then `otherTokenId` must be the same as `tokenId`. It means only `tokenId` (the intended target) has its approval updated and no other token (`otherTokenId`) is accidentally affected.


```solidity
// no side effect
assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
```


Here's the complete specification for the `approve` rule:  


```solidity
methods {
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function isApprovedForAll(address,address) external returns (bool) envfree;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonpayable(env e) returns bool = e.msg.value == 0;
definition authSanity(env e) returns bool = e.msg.sender != 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: approve behavior and side effects                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule approve(env e, address spender, uint256 tokenId) {
    require nonpayable(e);
    require authSanity(e);

    address caller = e.msg.sender;
    address owner = unsafeOwnerOf(tokenId);
    uint256 otherTokenId;

    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);

    approve@withrevert(e, spender, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        owner != 0 &&
        (owner == caller || isApprovedForAll(owner, caller))
    );

    // effect
    assert success => unsafeGetApproved(tokenId) == spender;

    // no side effect
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


Here's the Prover [run](https://prover.certora.com/output/541734/e9d124c9ba9b4e40a5686ed7e03733f9?anonymousKey=ad8ca670261c8589f6032b0faf2d11e9d54aaa70).


## **Formally verify** `setApprovalForAll()`


The `setApprovalForAll()` function authorizes an address to manage all tokens owned by the caller.  Here is the rule: 


```solidity
rule setApprovalForAll(env e, address operator, bool approved) {
	// preconditions
    require nonpayable(e);
		
	// pre-call state
    address owner = e.msg.sender;
    address otherOwner;
    address otherOperator;

    bool otherIsApprovedForAllBefore = isApprovedForAll(otherOwner, otherOperator);

    setApprovalForAll@withrevert(e, operator, approved);
    bool success = !lastReverted;

    // liveness
    assert success <=> operator != 0;

    // effect
    assert success => isApprovedForAll(owner, operator) == approved;

    // no side effect
    assert isApprovedForAll(otherOwner, otherOperator) != otherIsApprovedForAllBefore => (
        otherOwner    == owner &&
        otherOperator == operator
    );
}
```


At this point, we’re already familiar with the pattern for preconditions, pre-call and post-call states, and the method call, so we’ll skip those and go straight to the assertions.


### **Assertions — liveness, effect, and no side effect**


**Liveness:**

- `assert success <=> operator != 0`

    The call succeeds if and only if the operator is not the zero address. ERC-721 prohibits approving the zero address as an operator, so this assertion requires the call to fail when `operator == 0`:


```solidity
// liveness
assert success <=> operator != 0;
```


**Effect:**

- `assert success => isApprovedForAll(owner, operator) == approved`

    If the call succeeded, the approval status of the `(owner, operator)` pair must match the `approved` boolean value. It means that the approval value (true or false) was correctly set.


    ```solidity
    // effect
    assert success => isApprovedForAll(owner, operator) == approved;
    ```


**No side effect:**

- If the approval status of any `(otherOwner, otherOperator)` changes, those addresses must match the caller (`owner`) and the designated operator (`operator`). Any change to a different approval entry causes this assertion to fail.

```solidity
// no side effect
assert isApprovedForAll(otherOwner, otherOperator) != otherIsApprovedForAllBefore => (
    otherOwner    == owner &&
    otherOperator == operator
);
```


Here's the full CVL specification for the `setApprovalForAll` rule: 



```javascript
methods {
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function isApprovedForAll(address,address) external returns (bool) envfree;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonpayable(env e) returns bool = e.msg.value == 0;
definition authSanity(env e) returns bool = e.msg.sender != 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: setApprovalForAll behavior and side effects                                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule setApprovalForAll(env e, address operator, bool approved) {
    require nonpayable(e);

    address owner = e.msg.sender;
    address otherOwner;
    address otherOperator;

    bool otherIsApprovedForAllBefore = isApprovedForAll(otherOwner, otherOperator);

    setApprovalForAll@withrevert(e, operator, approved);
    bool success = !lastReverted;

    // liveness
    assert success <=> operator != 0;

    // effect
    assert success => isApprovedForAll(owner, operator) == approved;

    // no side effect
    assert isApprovedForAll(otherOwner, otherOperator) != otherIsApprovedForAllBefore => (
        otherOwner    == owner &&
        otherOperator == operator
    );
}
```


Here's the Prover [run](https://prover.certora.com/output/541734/e8b8e935833b4f4ba463452898b1444e?anonymousKey=9d9e411d8a2610adfd37092c66e63c2b126395b7).


## **Formally verify that** **`balanceOf(0)`** **reverts**


The following rule verifies that the `balanceOf` function always reverts when the zero address is queried. 


Below is the contract implementation of `balanceOf`. It reverts if `owner == address(0)`:


```solidity
function balanceOf(address owner) public view virtual returns (uint256) {
    if (owner == address(0)) {
        revert ERC721InvalidOwner(address(0));

    }
    return _balances[owner];
}
```


Here is the rule that verifies this behavior: 


```solidity
methods {
    function balanceOf(address) external returns (uint256) envfree;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: balance of address(0) is 0                                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule zeroAddressBalanceRevert() {
    balanceOf@withrevert(0);
    assert lastReverted;
}
```


Here's the Prover [run](https://prover.certora.com/output/541734/b7d49a18a6e7428d9911431668b8dcb1?anonymousKey=0cd432f0ef8543656d9d1f455181aaf67891768c):


## Full specification for transfer and approval


Here's the complete specification and the Prover [run](https://prover.certora.com/output/541734/332f053187cc4a8d9f312cb9005ded60?anonymousKey=f06e67cb584f478c06f5204dfd605080f172a6bc).


```solidity
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function ownerOf(uint256) external returns (address) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function isApprovedForAll(address,address) external returns (bool)    envfree;

    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Definitions                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

definition nonpayable(env e) returns bool = e.msg.value == 0;
definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;
definition authSanity(env e) returns bool = e.msg.sender != 0;

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
        }
    }

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: transferFrom behavior and side effects                                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule transferFrom(env e, address from, address to, uint256 tokenId) {
    require nonpayable(e);
    require authSanity(e);

    address operator = e.msg.sender;
    uint256 otherTokenId;
    address otherAccount;

    requireInvariant ownerHasBalance(tokenId);
    require balanceLimited(to);

    uint256 balanceOfFromBefore  = balanceOf(from);
    uint256 balanceOfToBefore    = balanceOf(to);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
    address approvalBefore       = unsafeGetApproved(tokenId);
    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);

    transferFrom@withrevert(e, from, to, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        from == ownerBefore &&
        from != 0 &&
        to   != 0 &&
        (operator == from || operator == approvalBefore || isApprovedForAll(ownerBefore, operator))
    );

    // effect
    assert success => (
        to_mathint(balanceOf(from))            == balanceOfFromBefore - assert_uint256(from != to ? 1 : 0) &&
        to_mathint(balanceOf(to))              == balanceOfToBefore   + assert_uint256(from != to ? 1 : 0) &&
        unsafeOwnerOf(tokenId)                 == to &&
        unsafeGetApproved(tokenId)             == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: approve behavior and side effects                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule approve(env e, address spender, uint256 tokenId) {
    require nonpayable(e);
    require authSanity(e);

    address caller = e.msg.sender;
    address owner = unsafeOwnerOf(tokenId);
    uint256 otherTokenId;

    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);

    approve@withrevert(e, spender, tokenId);
    bool success = !lastReverted;

    // liveness
    assert success <=> (
        owner != 0 &&
        (owner == caller || isApprovedForAll(owner, caller))
    );

    // effect
    assert success => unsafeGetApproved(tokenId) == spender;

    // no side effect
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: setApprovalForAll behavior and side effects                                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule setApprovalForAll(env e, address operator, bool approved) {
    require nonpayable(e);

    address owner = e.msg.sender;
    address otherOwner;
    address otherOperator;

    bool otherIsApprovedForAllBefore = isApprovedForAll(otherOwner, otherOperator);

    setApprovalForAll@withrevert(e, operator, approved);
    bool success = !lastReverted;

    // liveness
    assert success <=> operator != 0;

    // effect
    assert success => isApprovedForAll(owner, operator) == approved;

    // no side effect
    assert isApprovedForAll(otherOwner, otherOperator) != otherIsApprovedForAllBefore => (
        otherOwner    == owner &&
        otherOperator == operator
    );
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rule: balance of address(0) is 0                                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule zeroAddressBalanceRevert() {
    balanceOf@withrevert(0);
    assert lastReverted;
}
```
