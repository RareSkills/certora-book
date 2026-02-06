# SafeMint and SafeTransfer Rules in ERC-721


## Introduction


This chapter is the fourth part (4/5) of our code walkthrough of [OpenZeppelin’s ERC-721 CVL specification](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/36bf1e46fa811f0f07d38eb9cfbc69a955f300ce/certora/specs/ERC721.spec) and focuses on formally verifying safe mint and safe transfer operations, specifically:

- `_safeMint()` and `_safeMint(bytes)`
- `safeTransferFrom()` and `safeTransferFrom(bytes)`

     


The `_safeMint()` function extends `_mint()` with an additional check: if the recipient is a contract, it checks that the contract can handle ERC-721 tokens. This prevents NFTs from getting stuck in contracts that lack the transfer logic.


The ERC-721 in OpenZeppelin has two versions of `safeMint`: one that takes a bytes data parameter, and one that does not. The simpler version (without `bytes`) calls the version with `bytes`, passing an empty string as the `data` parameter: 



```solidity
// ERC721.sol

function _safeMint(address to, uint256 tokenId) internal {
    _safeMint(to, tokenId, "");
}

function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
    _mint(to, tokenId);
    ERC721Utils.checkOnERC721Received(_msgSender(), address(0), to, tokenId, data);
}
```


_Note: The library_ _`ERC721Utils`_ _can be found_ [_here_](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/cc94ea49301460aa6bd39edf29b2ab953d1d80b5/contracts/token/ERC721/utils/ERC721Utils.sol)_._     


Both versions mint an NFT to the address `to`, then call the hook function `onERC721Received()` on the recipient if it is a contract. The version with the `bytes` parameter additionally forwards `data` to the receiving contract’s `onERC721Received` hook, which allows the recipient to customize its behavior based on the provided data. For example, a game contract may mint an NFT directly to a player’s inventory contract and include `data` that specifies the item type, rarity, or initial attributes to trigger game events.


Both versions revert and undo the mint if:

- the recipient does not implement `onERC721Received()`,
- the callback reverts, or
- the callback returns a value other than the expected selector `0x150b7a02`.

It is not possible to formally verify `_safeMint()` without resolving its external call to the recipient contract. 


   


When `_safeMint()` is invoked in a rule, it calls `checkOnERC721Received()`, which in turn makes an external call to `onERC721Received()` on the recipient contract. The Prover cannot determine what `onERC721Received()` will do, so it treats the call as unresolved. Unresolved external calls cause the Prover to assume arbitrary changes to storage, which means that storage variables and ghost variables in post-call assertions take non-deterministic values, making the assertions meaningless.


To resolve the external call, a mock receiver contract is added whose `onERC721Received` function returns `0x150b7a02` (the value of `this.onERC721Received.selector`).


```javascript
// ERC721ReceiverHarness.sol -- this is a mock contract

import "contracts/interfaces/IERC721Receiver.sol";

contract ERC721ReceiverHarness is IERC721Receiver {
    function onERC721Received(address, address, uint256, bytes calldata) external pure returns (bytes4) {
        return this.onERC721Received.selector; // always returns 0x150b7a02
    }
}
```


   


We do not verify the external receiver contract because its implementation is outside our control. Instead, the `onERC721Received` call is dispatched to a mock receiver contract that always returns the expected selector `0x150b7a02`. This means the specification does not test cases where the callback reverts or returns an incorrect value. 


However, this does not mean that `_safeMint()` would not revert if a callback failure occurs in a real execution. The liveness assertion ensures the function behaves correctly in such cases by reverting.


The liveness assertion does not inspect the callback return value itself; it checks whether the call as a whole succeeded or reverted. As discussed in the “Mint and Burn” chapter, mint does not revert (`!lastReverted`) if and only if `ownerBefore == address(0)` and `to != address(0)`:


```solidity
bool success = !lastReverted;
assert success <=> (ownerBefore == 0 && to != 0);
```


  


When the receiver callback causes a revert, the right-hand-side (`ownerBefore == 0 && to != 0`) remains satisfied because these values were captured before the call. However, success is false, which violates the biconditional assertion. The Prover detects this contradiction and reports a violation. Therefore, no additional assertion to explicitly check the callback return value is required.


The same principle discussed above with safe mint also applies to safe token transfers.


The `safeTransferFrom` also has two versions: one that takes a bytes data parameter, and one that does not: 


```solidity
// ERC721.sol

function safeTransferFrom(address from, address to, uint256 tokenId) public {
    safeTransferFrom(from, to, tokenId, "");
}

function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public virtual {
    transferFrom(from, to, tokenId);
    ERC721Utils.checkOnERC721Received(_msgSender(), from, to, tokenId, data);
}
```


In the version with the bytes parameter, `data` is forwarded to the receiving contract’s `onERC721Received` hook so that the recipient can process additional data during the transfer. If the recipient is a contract that doesn't implement `onERC721Received` or returns an invalid response, the entire transaction reverts.


Both `safeMint()` and `safeTransferFrom()` follow the same external call pattern, so the specification uses the same mock receiver contract when verifying these functions.


## Formally verify safe mint


Since `_safeMint()` and `_safeMint(bytes)` are internal functions, OpenZeppelin exposes them through harness functions that the Prover can invoke:


```solidity
// ERC721Harness.sol

function safeMint(address to, uint256 tokenId) external {
    _safeMint(to, tokenId);
}

function safeMint(address to, uint256 tokenId, bytes memory data) external {
    _safeMint(to, tokenId, data);
}
```


  


The `safeMint` rule below uses `method f`, the arbitrary function call, in the same way parametric rules do. In typical parametric rules, both the function and the arguments are arbitrary (`f(e, args)`). Here, the `filtered` block restricts `method f` to only the two `safeMint` variants, and the arguments `to` and `tokenId` are declared explicitly rather than using `args`. This explicit declaration allows the rule to apply preconditions that constrain these arguments, which cannot be done directly within the `calldataarg args` construct. This means the function is arbitrary within an allowed set, while the arguments are controlled, which differs from a full parametric pattern.


Here is the rule, which we will walk through next:


```solidity
// ERC721.spec -- safeMint

rule safeMint(env e, method f, address to, uint256 tokenId) filtered { f ->
    f.selector == sig:safeMint(address,uint256).selector ||
    f.selector == sig:safeMint(address,uint256,bytes).selector
} {
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

    helperMintWithRevert@withrevert(e, f, to, tokenId);
    bool success = !lastReverted;

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


### Filtering `method f` to `safeMint` variants


The `method f` parameter tells the Prover to test arbitrary functions, but the `filtered` clause limits testing to only the two versions of `safeMint`:  


```solidity
rule safeMint(env e, method f, address to, uint256 tokenId) filtered { f ->
    f.selector == sig:safeMint(address,uint256).selector ||
    f.selector == sig:safeMint(address,uint256,bytes).selector
} {
	...
    helperMintWithRevert@withrevert(e, f, to, tokenId);
	...
}
```


  


To handle both variants within a single rule, we use a helper function `helperMintWithRevert` that routes the call to the appropriate version based on the function selector:


```solidity
function helperMintWithRevert(env e, method f, address to, uint256 tokenId) {
    if (f.selector == sig:safeMint(address,uint256).selector) {
        safeMint(e, to, tokenId);
    } else if (f.selector == sig:safeMint(address,uint256,bytes).selector) {
        bytes params;
        require params.length < 0xffff;
        safeMint(e, to, tokenId, params);
    } else {
        require false;
    }
}
```


The helper function allows the rule to test both `safeMint()` and `safeMint(bytes)` within a single rule using `method f`, so there is no need to write separate rules for each variant. It routes the call based on the function selector and directs execution to the corresponding `if/else` branch.



Let's explain each condition:

- `f.selector == sig:safeMint(address,uint256).selector`

    If the arbitrary method call is the two-parameter `safeMint(address,uint256)`, then the CVL helper function invokes `safeMint(e, to, tokenId)`.

- `f.selector == sig:safeMint(address,uint256,bytes).selector`

    If the arbitrary method call is the three-parameter `safeMint(address, uint256, bytes)`, the CVL helper function invokes `safeMint(e, to, tokenId, params)`, where `params` is constrained to less than `0xffff` ($2^{16} - 1$ or 65,535 bytes) via `require params.length < 0xffff`.
    


    Since `bytes` is a dynamically-sized array, `require params.length <` `0xffff` is a practical verification limit — large enough to cover realistic use cases for the `safeMint` data parameter, but small enough to prevent the Prover from exploring unrealistically large inputs.

- `else { require false; }`

    This branch is included only as a catch-all, but it is unreachable in this rule because the `filtered` clause already restricts `method f` to the two `safeMint` variants. The Prover cannot supply any other function selector, so the `else` block covers only a theoretical case that cannot occur under the filtered rule.


_Note: In the original OpenZeppelin spec, this helper_ _`helperMintWithRevert`_ _also contained a branch for_ _`mint(address,uint256)`__, but that case is already excluded by the_ _`filtered`_ _clause in the rule. Since the rule only permits_ _`safeMint()`_ _and_ _`safeMint(bytes)`__, we removed the unused branch for clarity. For reference, here's the original_ _`helperMintWithRevert`_ [_code_](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/a7ee03565b4ee14265f4406f9e38a04e0143656f/certora/specs/ERC721.spec#L42-L44)_._


### Dispatching `onERC721Received` to mock receiver


The `DISPATCHER` is added in the `methods` block to instruct the Prover to dispatch the call to any contract (denoted by the wildcard `_`) in the verification scene that implements the specified method (`onERC721Received`):


```solidity
function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
```


Since only the mock receiver implements this method, the callback is always dispatched to it. Had multiple contracts implemented `onERC721Received`, the Prover would dispatch to each of them and explore all verification paths.


Without this, the Prover treats the `onERC721Received` call as unresolved. As a result, the Prover produces counterexamples that violate the assertions due to the unresolved external call in the `safeMint` and `safeTransferFrom` rules.



### Dispatchee — mock receiver contract


A dispatchee is a contract in the scene that is the target of the dispatch call. The scene must contain at least one contract that implements `onERC721Received`. The following contract provides a minimal dispatchee for this purpose:


```solidity
contract ERC721ReceiverHarness is IERC721Receiver {
    function onERC721Received(address, address, uint256, bytes calldata) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }
}
```


This is a minimal implementation of the `IERC721Receiver` interface. It accepts any incoming ERC721 token and returns the expected selector (`0x150b7a02`) to signal a successful transfer. While it performs no additional logic, it gives the Prover a valid dispatchee for `onERC721Received`.


### Full specification — safeMint


The assertions are identical to those in the `mint` rule from the “Mint and Burn” chapter, where the liveness, effect, and no side-effect checks are explained in detail. For this reason, we do not repeat this discussion here and refer readers to that chapter for further explanation.


Below is the complete CVL specification for `safeMint`:


```solidity
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function _.onERC721Received(address,address,uint256,bytes) external => DISPATCHER(true);
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
│ Helpers                                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

function helperMintWithRevert(env e, method f, address to, uint256 tokenId) {
    if (f.selector == sig:safeMint(address,uint256).selector) {
        // safeMint@withrevert(e, to, tokenId);
        safeMint(e, to, tokenId);
    } else if (f.selector == sig:safeMint(address,uint256,bytes).selector) {
        bytes params;
        require params.length < 0xffff;
        // safeMint@withrevert(e, to, tokenId, params);
        safeMint(e, to, tokenId, params);
    } else {
        require false;
    }
}

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
│ Rule: safeMint behavior and side effects                                                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule safeMint(env e, method f, address to, uint256 tokenId) filtered { f ->
    f.selector == sig:safeMint(address,uint256).selector ||
    f.selector == sig:safeMint(address,uint256,bytes).selector
} {
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

    helperMintWithRevert@withrevert(e, f, to, tokenId);
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


_Note: A new Prover feature introduced in_ [_version 8.1.0_](https://docs.certora.com/en/latest/docs/prover/changelog/prover_changelog.html?utm_source=chatgpt.com#id3) _allows CVL functions to use_ _`@withRevert`__. This changes the syntax inside the function: instead of placing_ _`@withRevert`_ _on each method invocation, it is now applied on the CVL function. We updated the syntax so it works with the latest Prover version._


Here’s the Prover run: [link](https://prover.certora.com/output/541734/328ce562f5bd4ee89094818a22d26a4c?anonymousKey=785d66ee77b78120cd0df9cdb8a7887028cad361)


## Formally verify safe transferFrom


The contract function `safeTransferFrom()` extends `transferFrom()` with the `onERC721Received` safety check. Hence, the preconditions, pre-call state, and assertions of the `safeTransferFrom` rule are identical to those of the `transferFrom` rule from the previous chapter, but it follows the same method execution pattern as `safeMint`:

- it uses `method f` with filtering to test both variants,
- routes execution through a CVL helper function, and
- dispatches `onERC721Received()` to the mock receiver.

  


Here is the rule. As with `safeMint`, we focus on how method filtering, helper function routing, and external callback handling are used to verify both `safeTransferFrom` variants:


```solidity
rule safeTransferFrom(env e, method f, address from, address to, uint256 tokenId) filtered { f ->
    f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
    f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector
} {
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

    helperTransferWithRevert@withrevert(e, f, from, to, tokenId);
    bool success = !lastReverted;

    assert success <=> (
        from == ownerBefore &&
        from != 0 &&
        to   != 0 &&
        (operator == from || operator == approvalBefore || isApprovedForAll(ownerBefore, operator))
    );

    // effect
    assert success => (
        to_mathint(balanceOf(from)) == balanceOfFromBefore - assert_uint256(from != to ? 1: 0) &&
        to_mathint(balanceOf(to))   == balanceOfToBefore   + assert_uint256(from != to ? 1: 0) &&
        unsafeOwnerOf(tokenId)      == to &&
        unsafeGetApproved(tokenId)  == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


### Filtering `method f` to `safeTransferFrom` variants


The `rule safeTransferFrom` constrains execution to two methods (via `method f`), namely:

- `safeTransferFrom(address, address, uint256)`
- `safeTransferFrom(address, address, uint256, bytes)`

It then passes the call to the CVL function `helperTransferWithRevert()` as shown below: 


```solidity
rule safeTransferFrom(env e, method f, address from, address to, uint256 tokenId) filtered { f ->
    f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
    f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector
} {
	...
    
    helperTransferWithRevert@withrevert(e, f, from, to, tokenId);
    bool success = !lastReverted;

    ...
}
```


We have separate conditional logic for `safeTransferFrom()` — one for the variant with the `bytes` parameter and one without. 


For the variant with `bytes`, we constrain the bytes parameter length to be less than `0xffff` to keep the array within a practical bound and limit the exploration space. Apart from this constraint, both `safeTransferFrom()` variants follow the same logic and use the same rule structure: 



```solidity
function helperTransferWithRevert(env e, method f, address from, address to, uint256 tokenId) {
    if (f.selector == sig:safeTransferFrom(address,address,uint256).selector) {
        safeTransferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector) {
        bytes params;
        require params.length < 0xffff; 
        safeTransferFrom(e, from, to, tokenId, params);
    } else {
        calldataarg args;
        f(e, args);
    }
}
```


_Note: Like the_ _`safeMint`_ _rule, the_ _`f.selector == sig:transferFrom(address,address,uint256).selector`_ _conditional branch was removed from this CVL function because it is unnecessary. The rule already filters out all functions except the_ _`safeTransferFrom()`_ _and_  _`safeTransferFrom(bytes)`__. For reference, here is the original_ _`helperTransferWithRevert`_ _code:_ [_link_](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/a7ee03565b4ee14265f4406f9e38a04e0143656f/certora/specs/ERC721.spec#L27-L29)


### Dispatching `onERC721Received` callback


In the `safeMint` section, we configured the mock receiver and the `DISPATCHER(true)` setup so the Prover can model the external callback `onERC721Received`. 


The same setup is required here because both `safeMint` and `safeTransferFrom` trigger an external call to an ERC-721 receiver. Without this dispatch configuration, the Prover would treat the callback as an unresolved external call, which causes havoc and makes the post-call assertions meaningless.



### Full specification — safeTransferFrom


The assertions are identical to those in the `transferFrom` rule from the previous chapter, “Transfer and Approval,” where the liveness, effect, and no side-effect checks are explained in detail. 


Below is the complete CVL specification for `safeTransferFrom`: 


```javascript
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function isApprovedForAll(address,address) external returns (bool) envfree;
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
│ Helpers                                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

function helperTransferWithRevert(env e, method f, address from, address to, uint256 tokenId) {
    if (f.selector == sig:safeTransferFrom(address,address,uint256).selector) {
        // safeTransferFrom@withrevert(e, from, to, tokenId);
        safeTransferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector) {
        bytes params;
        require params.length < 0xffff; 
        // safeTransferFrom@withrevert(e, from, to, tokenId, params);
        safeTransferFrom(e, from, to, tokenId, params);
    } else {
        calldataarg args;
        f@withrevert(e, args);
    }
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Ghost & hooks: sum of all balances                                                                                  │
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
│ Rule: safeTransferFrom behavior and side effects                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule safeTransferFrom(env e, method f, address from, address to, uint256 tokenId) filtered { f ->
    f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
    f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector
} {
    require nonpayable(e);
    require authSanity(e);

    address operator = e.msg.sender;
    uint256 otherTokenId;
    address otherAccount;

    // requireInvariant ownerHasBalance(tokenId);
    require balanceLimited(to);

    uint256 balanceOfFromBefore  = balanceOf(from);
    uint256 balanceOfToBefore    = balanceOf(to);
    uint256 balanceOfOtherBefore = balanceOf(otherAccount);
    address ownerBefore          = unsafeOwnerOf(tokenId);
    address otherOwnerBefore     = unsafeOwnerOf(otherTokenId);
    address approvalBefore       = unsafeGetApproved(tokenId);
    address otherApprovalBefore  = unsafeGetApproved(otherTokenId);

    helperTransferWithRevert@withrevert(e, f, from, to, tokenId);
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
        to_mathint(balanceOf(from)) == balanceOfFromBefore - assert_uint256(from != to ? 1: 0) &&
        to_mathint(balanceOf(to))   == balanceOfToBefore   + assert_uint256(from != to ? 1: 0) &&
        unsafeOwnerOf(tokenId)      == to &&
        unsafeGetApproved(tokenId)  == 0
    );

    // no side effect
    assert balanceOf(otherAccount)         != balanceOfOtherBefore => (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId)     != otherOwnerBefore     => otherTokenId == tokenId;
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore  => otherTokenId == tokenId;
}
```


_Note: A new Prover feature introduced in_ [_version 8.1.0_](https://docs.certora.com/en/latest/docs/prover/changelog/prover_changelog.html?utm_source=chatgpt.com#id3) _allows CVL functions to use_ _`@withRevert`__. This changes the syntax inside the function: instead of placing_ _`@withRevert`_ _on each method invocation, it is now applied on the CVL function. We updated the syntax so it works with the latest Prover version._


Here's the Prover [run](https://prover.certora.com/output/541734/0da14ebdd2b044229ba7bae1e71d9acc?anonymousKey=b7f9046a2be316bcac5fd00b396920254d8b34b2).
