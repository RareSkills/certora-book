# Partially Parametric Rules for ERC-721


## Introduction


This chapter is the final part (5/5) of the code walkthrough of [OpenZeppelin’s ERC-721 CVL specification](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/36bf1e46fa811f0f07d38eb9cfbc69a955f300ce/certora/specs/ERC721.spec) which formally verifies the following rules:

- `supplyChange`

    Total supply changes only through mint or burn operations, and by exactly one.

- `balanceChange`

    Account balance changes only through mint, burn, or transfer operations, and by exactly one.

- `ownershipChange`

    Token ownership changes only through mint, burn, or transfer operations.

- `approvalChange`

    Per-token approval changes only through `approve()` call or as a side effect of burn or transfer operations.

- `approvedForAllChange`

    Operator approval changes only through `setApprovalForAll()` call.


These rules use a partially parametric rule pattern (to be discussed in the next section), which builds on the parametric rules concept covered in the chapter “Introduction to Parametric Rules”.


The properties in this chapter differ from those in previous chapters. Earlier, we verified specific method behaviors: does `mint()` increase total supply? Does `transferFrom()` update balances as expected? Those properties tested individual methods using ordinary (non-parametric) rules.



**The properties here ask a different question: which** **methods** **can change a particular** **state**  **(such as total supply or an account's balance), and can we confirm that no other methods do?** Ordinary rules cannot answer this because they test one method at a time. Even with one rule per method per property, we cannot verify that unlisted methods do not cause changes.


One might ask whether CVL invariants could serve the same purpose. However, CVL invariants are not suitable here because they cannot capture how a value changes across a method call. Invariants cannot access pre-call values or compare them with post-call values, so they cannot express properties such as “this state may change only when specific methods are called” or “this state changes by exactly this amount.” These are state-transition properties, which require reasoning about the pre-call and post-call state.


A parametric pattern solves this by testing all contract methods in a single rule and reasoning about the pre-call and post-call state. This makes it possible to verify both: which methods are allowed to change state and that no other methods can cause the same change. 


## The Partially Parametric Rule


In the earlier chapter, “Introduction To Parametric Rules”, we learned that:

> _“… a parametric rule verifies a property holds true after any function is called with any valid arguments.”_

The key phrase is: ”_after any function is called with any valid arguments,”_ which corresponds to the following code pattern:



```solidity
rule parametricExample(env e, method f, calldataarg args) {
    /// set precondition
		
	/// record pre-call state

    /// parametric call: invokes all functions in the scene with arbitrary arguments 
    f(e, args);

    /// record post-call state

    /// assertions

}
```


   


This pattern (fully parametric) works well when the same preconditions apply to all methods. However, it is not appropriate when different methods need different constraints.


Instead of the fully parametric `f(e, args)`, we can use a "partially parametric" pattern. This is not CVL syntax, but a specification design pattern. As the [Certora documentation](https://docs.certora.com/en/latest/docs/user-guide/patterns/partial-apply.html#partially-parametric-rules) describes:

> _“This partially parametric rule demonstrates conditional logic based on the type of method invoked, allowing for specific actions and assertions tailored to different scenarios within the smart contract.”_

The key phrase is _"specific actions and assertions tailored to different scenarios"_, which corresponds to the following code pattern:


```solidity
rule partiallyParametricExample(env e) {
	method f;

	if (f.selector == sig:exampleMethod1(uint256, address).selector) {
			// method-specific logic
	}
	else if (f.selector == sig:exampleMethod2(address, address).selector) {
			// method-specific logic
	}
	else {
			// method-specific logic
	}
}
```


The method-specific logic can be preconditions, method calls, or even assertions. For cleaner code, these conditional statements can be abstracted into one CVL function, as shown below: 


```solidity
rule partiallyParametricExample(env e) {
	method f;
	helperFunction(e, f); // contains all conditional logic
}
```


OpenZeppelin's specification uses the helper function `helperSoundFnCall()` to implement this pattern. Here's an example from the `supplyChange` rule (which we'll examine in detail later):


```solidity
/// ERC721.spec  

rule supplyChange(env e) {
    ...
    method f; helperSoundFnCall(e, f);
    ...
    
    /// assert
    /// assert
}
```


 


The “Sound” in `helperSoundFnCall()` is deliberate. Soundness means that the verification does not miss any real bugs, and a common source of unsoundness is excluding valid execution paths through overly restrictive preconditions. 


 


The helper routes `f(e, args)` in a sound manner by applying preconditions selectively based on the invoked method rather than globally to all calls. Applying a precondition to methods that do not require it would exclude valid execution paths.



In the next section, we will see how the arbitrary method call `f(e, args)` is routed to the appropriate method-specific preconditions that keep the specification sound.


## `helperSoundFnCall()` routes the `f(e, args)` call  to the correct method branch


The CVL function `helperSoundFnCall()` routes the arbitrary method call `f(e, args)` to the correct method branch by matching the function selector. For each matched selector, the CVL helper function does the following: 

- declares the required arguments,
- applies the method-specific preconditions, and
- calls the corresponding concrete function.

### For mint operations — `mint()`, `safeMint()`, and `safeMint(bytes)`


When the invoked function is a mint variant, the helper function:

- Declares the required arguments: `a``ddress to`, `uint256 tokenId`, and for the bytes variant, `bytes data`.
- Applies method-specific preconditions:
    - `require balanceLimited(to)`  — constrains the recipient's balance to under `max_uint256` to prevent overflow
    - `require data.length < 0xffff` — limits the data length to under 65,535 bytes (`0xffff`) to avoid unrealistically large arrays during Prover execution
- Calls the corresponding functions: `mint(e, to, tokenId)`, `safeMint(e, to, tokenId)` or `safeMint(e, to, tokenId, data)`.

```solidity
function helperSoundFnCall(env e, method f) {
    if (f.selector == sig:mint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        mint(e, to, tokenId);
    }
    else if (f.selector == sig:safeMint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        safeMint(e, to, tokenId);
    }
    else if (f.selector == sig:safeMint(address,uint256,bytes).selector) {
        address to; uint256 tokenId; bytes data;
        require data.length < 0xffff;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        safeMint(e, to, tokenId, data);
    }
		...
}
```


### For burn operation — `burn()`


When the invoked function is `burn()`, the helper function:

- Declares the required argument`uint256 tokenId`.
- Applies method-specific precondition `requireInvariant ownerHasBalance(tokenId)` which requires that the token exists (has a nonzero owner) before the burn.
- Calls the corresponding function`burn(e, tokenId)`.

    ```javascript
    function helperSoundFnCall(env e, method f) {
        ...
        } else if (f.selector == sig:burn(uint256).selector) {
            uint256 tokenId;
            requireInvariant ownerHasBalance(tokenId);
            // requireInvariant notMintedUnset(tokenId);
            burn(e, tokenId);
        } 
    	...
    }
    ```


### For transfer operations — `transferFrom()`, `safeTransferFrom()`, and `safeTransferFrom(bytes)`


When the invoked function is a transfer variant, the helper function: 

- Declares the required arguments: `address` `from`, `address to`, `uint256 tokenId`, and for the bytes variant, `bytes data`
- Applies method-specific preconditions:
    - `require balanceLimited(to)` — constrains the recipient's balance to be strictly less than `max_uint256` to prevent the Prover from exploring unrealistic recipient balances.
    - `requireInvariant ownerHasBalance(tokenId)` — ensures the token exists before it can be transferred. Without this, the Prover would consider cases where a token is transferred from the zero address.
    - `requireInvariant notMintedUnset(tokenId)` — rules out any approval for an unminted token. Without this, the Prover can treat a nonexistent token as if it has an approved operator.
    - `require data.length < 0xffff` — (bytes variant only) limits the data length to under 65,535 bytes (`0xffff`) to avoid unrealistically large arrays during Prover execution.
- Calls the corresponding functions: `transferFrom()`, `safeTransferFrom()`, `safeTransferFrom(bytes)`

```solidity
function helperSoundFnCall(env e, method f) {
    ...
    } else if (f.selector == sig:transferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        transferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        safeTransferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector) {
        address from; address to; uint256 tokenId; bytes data;
        require data.length < 0xffff;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        safeTransferFrom(e, from, to, tokenId, data);
		...
}
```


### For all operations other than mint, burn and transfer


Functions other than mint, burn, or transfer are handled with a fully parametric call `f(e, args)` with arbitrary arguments:


```solidity
function helperSoundFnCall(env e, method f) {
    ...
    } else {
        calldataarg args;
        f(e, args);
    }
}
```


### Complete implementation of CVL function `helperSoundFnCall()` 


Here's the complete helper CVL function:


```solidity
function helperSoundFnCall(env e, method f) {
    if (f.selector == sig:mint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        mint(e, to, tokenId);
    } else if (f.selector == sig:safeMint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        safeMint(e, to, tokenId);
    } else if (f.selector == sig:safeMint(address,uint256,bytes).selector) {
        address to; uint256 tokenId; bytes data;
        require data.length < 0xffff;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        safeMint(e, to, tokenId, data);
    } else if (f.selector == sig:burn(uint256).selector) {
        uint256 tokenId;
        requireInvariant ownerHasBalance(tokenId);
        // requireInvariant notMintedUnset(tokenId);
        burn(e, tokenId);
    } else if (f.selector == sig:transferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        transferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        safeTransferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector) {
        address from; address to; uint256 tokenId; bytes data;
        require data.length < 0xffff;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        safeTransferFrom(e, from, to, tokenId, data);
    } else {
        calldataarg args;
        f(e, args);
    }
}
```


_Note: In the mint and burn branches_ _we commented out_ _`requireInvariant notMintedUnset(tokenId)`_ _to show only preconditions that are strictly necessary for each invoked method._ _This does not mean the invariant is useless — since it is proven as a separate invariant in the specification, it can be assumed to always hold and safely reinstated. Adding it as a precondition generally shortens Prover run times._  


With the method-specific preconditions handled in `helperSoundFnCall()`, we can proceed to the rule discussions in the next section.



## `supplyChange`  — total supply can change only through `mint()`, `safeMint()`, `safeMint(bytes)` and `burn()` 


This rule verifies which contract methods can increase or decrease the total supply by 1. Here's the rule that captures this behavior, which we will discuss in detail next:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: total supply can only change through mint and burn                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule supplyChange(env e) {
    require nonzerosender(e);
    requireInvariant zeroAddressHasNoApprovedOperator(e.msg.sender);

    mathint supplyBefore = _supply;
    method f; helperSoundFnCall(e, f);
    mathint supplyAfter = _supply;

    assert supplyAfter > supplyBefore => (
        supplyAfter == supplyBefore + 1 &&
        (
            f.selector == sig:mint(address,uint256).selector ||
            f.selector == sig:safeMint(address,uint256).selector ||
            f.selector == sig:safeMint(address,uint256,bytes).selector
        )
    );
    assert supplyAfter < supplyBefore => (
        supplyAfter == supplyBefore - 1 &&
        f.selector == sig:burn(uint256).selector
    );
}
```


Prover run: [link](https://prover.certora.com/output/541734/cf14580f393e40f4ac4d838512c694fc?anonymousKey=405189efeaabcd309a577f256c9011ff757f9e09)


### **Preconditions**

- `require nonzerosender(e)`

    This precondition constrains the caller to be nonzero. Without this constraint, the Prover can explore executions in which a transfer originates from `address(0)` and moves a token to a nonzero address, which is effectively a mint executed through a transfer operation.

- `requireInvariant zeroAddressHasNoApprovedOperator(e.msg.sender)`

    This invariant precondition prevents `address(0)` from approving operators. Without this constraint, the Prover can explore states in which `address(0)` has granted operator permissions to nonzero addresses.


### **Pre-call and post-call states**

- `supplyBefore = _supply`

    Records the total supply before executing the partially parametric call ( `method f; helperSoundFnCall(e, f)` ).

- `method f; helperSoundFnCall(e, f)`

    Executes the partially parametric call discussed earlier.

- `mathint supplyAfter = _supply`

    Records the total supply after executing the partially parametric call.


```solidity
mathint supplyBefore = _supply;
method f; helperSoundFnCall(e, f);
mathint supplyAfter = _supply;
```


ERC-721 does not track total supply, so we use a ghost variable `_supply` that tracks the sum of all `_balances` via an `Sstore` hook: 


```solidity
ghost mathint _supply {
    init_state axiom _supply == 0;
}

hook Sstore _balances[KEY address addr] uint256 newValue (uint256 oldValue) {
    _supply = _supply - oldValue + newValue;
}
```


### **Assertions** 

- The first assertion states that if the total supply increases (`supplyAfter > supplyBefore`), the increase must be exactly 1 and can only be caused by `mint()`, `safeMint()`, or `safeMint(bytes)`:

    ```solidity
    rule supplyChange(env e) {
        ...
        assert supplyAfter > supplyBefore => (
            supplyAfter == supplyBefore + 1 &&
            (
                f.selector == sig:mint(address,uint256).selector ||
                f.selector == sig:safeMint(address,uint256).selector ||
                f.selector == sig:safeMint(address,uint256,bytes).selector
            )
        );
        ...
    }
    ```

- The second assertion covers the burn case. If the total supply decreases (`supplyAfter < supplyBefore`), the decrease must be exactly 1 and can only be caused by `burn()`:

    ```solidity
    rule supplyChange(env e) {
        ...
        assert supplyAfter < supplyBefore => (
            supplyAfter == supplyBefore - 1 &&
            f.selector == sig:burn(uint256).selector
        );
    }
    ```


## `balanceChange` — account balances can only change through mint, burn and transfer operations


The following rule verifies two properties about balance changes:

- Balances change by exactly one token per operation
- Only mint, burn, or transfer operations can change balances

```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: balanceOf can only change through mint, burn or transfers. balanceOf cannot change by more than 1.           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule balanceChange(env e, address account) {
   
 // requireInvariant balanceOfConsistency(account);
    // require balanceLimited(account);


    mathint balanceBefore = balanceOf(account);
    method f; helperSoundFnCall(e, f);
    mathint balanceAfter  = balanceOf(account);

    // balance can change by at most 1
    assert balanceBefore != balanceAfter => (
        balanceAfter == balanceBefore - 1 ||
        balanceAfter == balanceBefore + 1
    );

    // only selected function can change balances
    assert balanceBefore != balanceAfter => (
        f.selector == sig:transferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector ||
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256,bytes).selector ||
        f.selector == sig:burn(uint256).selector
    );
}
```


Prover run: [link](https://prover.certora.com/output/541734/9dafe73bd99b4fd5a4aae5d797fe2db2?anonymousKey=92578204ddacb27b80cb4762ddcb75fa49762601)


### Preconditions


The `requireInvariant balanceOfConsistency(account)` and `require balanceLimited(account)` are commented out because `helperSoundFnCall()` already enforces these preconditions for all balance-changing methods. The `balanceOfConsistency` is enforced through the preserved block of the `ownerHasBalance` invariant in the burn and transfer branches.


### **Pre-call and post-call** **states**


The rule records the account balance before and after the `helperSoundFnCall(e, f)` call to compare the values and determine whether the method call changed the balance, and by how much:


 


```solidity
mathint balanceBefore = balanceOf(account);
method f; helperSoundFnCall(e, f);
mathint balanceAfter  = balanceOf(account);
```


### **Assertions**


The first assertion states that if an account’s balance changes (`balanceBefore != balanceAfter`) after the `helperSoundFnCall(e, f)` call, the change must be plus or minus one — no more, no less.


```solidity
// balance can change by at most 1
assert balanceBefore != balanceAfter => (
    balanceAfter == balanceBefore - 1 ||
    balanceAfter == balanceBefore + 1
);
```


The second assertion states that only the following functions can change balances:

- mint functions (`mint()`, `safeMint()`, `safeMint(bytes)`)
- burn (`burn()`)
- transfer functions (`transferFrom()`, `safeTransferFrom()`, `safeTransferFrom(bytes)`)

```solidity
// only selected function can change balances
assert balanceBefore != balanceAfter => (
    f.selector == sig:transferFrom(address,address,uint256).selector ||
    f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
    f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector ||
    f.selector == sig:mint(address,uint256).selector ||
    f.selector == sig:safeMint(address,uint256).selector ||
    f.selector == sig:safeMint(address,uint256,bytes).selector ||
    f.selector == sig:burn(uint256).selector
);
```


 


If any other function not listed causes a balance change, the assertion will fail and the Prover will report a violation.


## `ownershipChange()` — ownership can change only through mint, burn and transfer operations


The following rule verifies that the ownership of a token (`ownerOf(tokenId)`) can change only through mint, burn, or transfer operations. The type of operation depends on how the owner address changes:

- From the zero address to a nonzero address — indicates a mint, which only `mint()`, `safeMint()`, or `safeMint(bytes)` can cause.
- From a nonzero address to the zero address — indicates a burn, which only the `burn()` can cause.
- From one nonzero address to another nonzero address — indicates a transfer, which only `transferFrom()`, `safeTransferFrom()`, or `safeTransferFrom(bytes)` can cause.

```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: ownership can only change through mint, burn or transfers.                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule ownershipChange(env e, uint256 tokenId) {
    require nonzerosender(e);
    requireInvariant zeroAddressHasNoApprovedOperator(e.msg.sender);

    address ownerBefore = unsafeOwnerOf(tokenId);
    method f; helperSoundFnCall(e, f);
    address ownerAfter  = unsafeOwnerOf(tokenId);

    assert ownerBefore == 0 && ownerAfter != 0 => (
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256,bytes).selector
    );

    assert ownerBefore != 0 && ownerAfter == 0 => (
        f.selector == sig:burn(uint256).selector
    );

    assert (ownerBefore != ownerAfter && ownerBefore != 0 && ownerAfter != 0) => (
        f.selector == sig:transferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector
    );
}
```


Prover run: [link](https://prover.certora.com/output/541734/217a0c2147c34536b99495351133cf56?anonymousKey=4f634a57f4728e3c8a7874f3581400ab56734de2)


### Preconditions

- `require nonzerosender(e)`

    This precondition requires the caller (`e.msg.sender`) to be nonzero. Without it, the Prover can model executions where `address(0)` calls `transferFrom` directly, which creates ownership from `address(0)`, violating the rule that only mint functions can do this.

- `requireInvariant zeroAddressHasNoApprovedOperator(e.msg.sender)`
This invariant precondition prevents the zero address from approving any operator. Without it, the Prover can assume that `address(0)` approved a nonzero operator. This allows `transferFrom(from = address(0), ...)` to succeed even when the caller is nonzero, which creates an ownership transition from the zero address and violates the rule that only mint functions may create ownership.

### **P****re-call and post-call states**


These lines record the token’s owner before and after `helperSoundFnCall(e, f)`: 


```solidity
address ownerBefore = unsafeOwnerOf(tokenId);
method f; helperSoundFnCall(e, f);
address ownerAfter  = unsafeOwnerOf(tokenId);
```


Using `unsafeOwnerOf(tokenId)` allows the rule to reason about ownership changes even when the token does not exist, in which case the owner is `address(0)`.


### **Assertions**

- `assert ownerBefore == 0 && ownerAfter != 0 => (...)`

    If ownership changes from the zero address to a nonzero address, meaning a mint occurred, then only the following functions could have caused it: `mint()`, `safeMint()`, or `safeMint(bytes)`:


    ```solidity
    assert ownerBefore == 0 && ownerAfter != 0 => (
		f.selector == sig:mint(address,uint256).selector ||
		f.selector == sig:safeMint(address,uint256).selector ||
		f.selector == sig:safeMint(address,uint256,bytes).selector
    );
    ```

- `assert ownerBefore != 0 && ownerAfter == 0 => (...)`

    If the ownership changes from nonzero to zero address, meaning burn occurred, then only the `burn()` function is responsible for this:


     


    ```solidity
    assert ownerBefore != 0 && ownerAfter == 0 => (
        f.selector == sig:burn(uint256).selector
    );
    ```

- `assert (ownerBefore != ownerAfter && ownerBefore != 0 && ownerAfter != 0) => (...)`

    Lastly, if the change is from a nonzero address to another nonzero address, then a transfer occurred. Only `transferFrom()`, `safeTransferFrom()`, and `safeTransferFrom(bytes)` are responsible for these changes:


    ```solidity
    assert (ownerBefore != ownerAfter && ownerBefore != 0 && ownerAfter != 0) => (
        f.selector == sig:transferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector
    );
    ```


## `approvalChange()` — approval changes only through approve, transferFrom and burn functions


The following rule verifies that the approval of a specific token (`getApproved(tokenId)`) can only change in three ways:

- directly set by a call to the `approve()` function
- reset to `address(0)` as a side-effect of a transfer
- reset to `address(0)` as a side-effect of burn

```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: token approval can only change through approve or transfers (implicitly).                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule approvalChange(env e, uint256 tokenId) {
    address approvalBefore = unsafeGetApproved(tokenId);
    method f; helperSoundFnCall(e, f);
    address approvalAfter  = unsafeGetApproved(tokenId);

    // approve can set any value, other functions reset
    assert approvalBefore != approvalAfter => (
        f.selector == sig:approve(address,uint256).selector ||
        (
            (
                f.selector == sig:transferFrom(address,address,uint256).selector ||
                f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
                f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector ||
                f.selector == sig:burn(uint256).selector
            ) && approvalAfter == 0
        )
    );
}
```


Prover run: [link](https://prover.certora.com/output/541734/6b33267777564799a3fe1ebb28ed0626?anonymousKey=ff12144bbac0ee70a140619379a8f7990c092ca7)


### **Pre-call and post-call states**


The line below records the approved address before and after the partially parametric call `method f; helperSoundFnCall(e, f)` to determine whether the approval changed as a result of the listed invoked method, and whether it was cleared to `address(0)`:


```solidity
address approvalBefore = unsafeGetApproved(tokenId);
method f; helperSoundFnCall(e, f);
address approvalAfter  = unsafeGetApproved(tokenId);
```


### **Assertion**


This assertion states that a change in the approved address can only occur if:

- The function call is `approve()`, which explicitly sets the approved address, or
- The function call is `transferFrom()`, `safeTransferFrom()` (with or without bytes data), or `burn()`, and the approved address is cleared to `address(0)` (`approvalAfter == 0`) as a side effect.

```solidity
// approve can set any value, other functions reset
assert approvalBefore != approvalAfter => (
    f.selector == sig:approve(address,uint256).selector ||
    (
        (
            f.selector == sig:transferFrom(address,address,uint256).selector ||
            f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
            f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector ||
            f.selector == sig:burn(uint256).selector
        ) && approvalAfter == 0
    )
);
```


The assertion will fail if:

- a function not listed in the rule clears or modifies the approved address.
- `transferFrom`, `safeTransferFrom`, or `burn` does not clear the approved address to `address(0)`.

## `approvedForAllChange` — approval for all only changes through `setApprovalForAll()` function


This rule verifies that the approved-for-all status of a spender (`isApprovedForAll(owner, spender)`) can only change through an explicit call to the `setApprovalForAll()` function:


```solidity
/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: approval for all tokens can only change through isApprovedForAll.                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule approvedForAllChange(env e, address owner, address spender) {
    bool approvedForAllBefore = isApprovedForAll(owner, spender);
    method f; helperSoundFnCall(e, f);
    bool approvedForAllAfter  = isApprovedForAll(owner, spender);

    assert approvedForAllBefore != approvedForAllAfter => f.selector == sig:setApprovalForAll(address,bool).selector;
}
```


Prover run: [link](https://prover.certora.com/output/541734/e4dfd40ef9cf4724b189a084488ee359?anonymousKey=5487cf57e602b3474f064527bceb612654b8e01d)


### **Pre-call and post-call states**

- `bool approvedForAllBefore = isApprovedForAll(owner, spender);`

    This records the approved-for-all status before the `helperSoundFnCall(e, f)` call.

- `bool approvedForAllAfter = isApprovedForAll(owner, spender);`

    This records the approved-for-all status after the `helperSoundFnCall(e, f)` call.


```solidity
bool approvedForAllBefore = isApprovedForAll(owner, spender);
method f; helperSoundFnCall(e, f);
bool approvedForAllAfter  = isApprovedForAll(owner, spender);
```


### **Assertion**


If the approved-for-all status for the spender changes, then the only cause is a call to `setApprovalForAll()`:



```solidity
assert approvedForAllBefore != approvedForAllAfter => f.selector == sig:setApprovalForAll(address,bool).selector;
```


Unlike `approve()`, which resets on transfer or burn, `isApprovedForAll` does not change during mint, transfer, or burn. Only an explicit call to `setApprovalForAll` can modify it. If any other function changes this state, the assertion fails and the Prover reports a violation.


## Full specifications


Here's the complete specification for the partially parametric rules:


```solidity
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
│ Helper                                                                                                              │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/

function helperSoundFnCall(env e, method f) {
    if (f.selector == sig:mint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        mint(e, to, tokenId);
    } else if (f.selector == sig:safeMint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        safeMint(e, to, tokenId);
    } else if (f.selector == sig:safeMint(address,uint256,bytes).selector) {
        address to; uint256 tokenId; bytes data;
        require data.length < 0xffff;
        require balanceLimited(to);
        // requireInvariant notMintedUnset(tokenId);
        safeMint(e, to, tokenId, data);
    } else if (f.selector == sig:burn(uint256).selector) {
        uint256 tokenId;
        requireInvariant ownerHasBalance(tokenId);
        // requireInvariant notMintedUnset(tokenId);
        burn(e, tokenId);
    } else if (f.selector == sig:transferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        transferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        safeTransferFrom(e, from, to, tokenId);
    } else if (f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector) {
        address from; address to; uint256 tokenId; bytes data;
        require data.length < 0xffff;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        requireInvariant notMintedUnset(tokenId);
        safeTransferFrom(e, from, to, tokenId, data);
    } else {
        calldataarg args;
        f(e, args);
    }
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
    unsafeOwnerOf(tokenId) != 0 => balanceOf(ownerOf(tokenId)) > 0 // fixed for Prover v8.3.1
    {
        preserved {
            requireInvariant balanceOfConsistency(ownerOf(tokenId));
            // require balanceLimited(ownerOf(tokenId));
        }
    }

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

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Invariant: tokens that do not exist are not owned and not approved                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
invariant notMintedUnset(uint256 tokenId)
    unsafeOwnerOf(tokenId) == 0 => unsafeGetApproved(tokenId) == 0;

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: total supply can only change through mint and burn                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule supplyChange(env e) {
    require nonzerosender(e);
    requireInvariant zeroAddressHasNoApprovedOperator(e.msg.sender);

    mathint supplyBefore = _supply;
    method f; helperSoundFnCall(e, f);
    mathint supplyAfter = _supply;

    assert supplyAfter > supplyBefore => (
        supplyAfter == supplyBefore + 1 &&
        (
            f.selector == sig:mint(address,uint256).selector ||
            f.selector == sig:safeMint(address,uint256).selector ||
            f.selector == sig:safeMint(address,uint256,bytes).selector
        )
    );
    assert supplyAfter < supplyBefore => (
        supplyAfter == supplyBefore - 1 &&
        f.selector == sig:burn(uint256).selector
    );
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: balanceOf can only change through mint, burn or transfers. balanceOf cannot change by more than 1.           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule balanceChange(env e, address account) {
    // requireInvariant balanceOfConsistency(account);
    // require balanceLimited(account);

    mathint balanceBefore = balanceOf(account);
    method f; helperSoundFnCall(e, f);
    mathint balanceAfter  = balanceOf(account);

    // balance can change by at most 1
    assert balanceBefore != balanceAfter => (
        balanceAfter == balanceBefore - 1 ||
        balanceAfter == balanceBefore + 1
    );

    // only selected function can change balances
    assert balanceBefore != balanceAfter => (
        f.selector == sig:transferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector ||
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256,bytes).selector ||
        f.selector == sig:burn(uint256).selector
    );
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: ownership can only change through mint, burn or transfers.                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule ownershipChange(env e, uint256 tokenId) {
    require nonzerosender(e);
    requireInvariant zeroAddressHasNoApprovedOperator(e.msg.sender);

    address ownerBefore = unsafeOwnerOf(tokenId);
    method f; helperSoundFnCall(e, f);
    address ownerAfter  = unsafeOwnerOf(tokenId);

    assert ownerBefore == 0 && ownerAfter != 0 => (
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256).selector ||
        f.selector == sig:safeMint(address,uint256,bytes).selector
    );

    assert ownerBefore != 0 && ownerAfter == 0 => (
        f.selector == sig:burn(uint256).selector
    );

    assert (ownerBefore != ownerAfter && ownerBefore != 0 && ownerAfter != 0) => (
        f.selector == sig:transferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
        f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector
    );
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: token approval can only change through approve or transfers (implicitly).                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule approvalChange(env e, uint256 tokenId) {
    address approvalBefore = unsafeGetApproved(tokenId);
    method f; helperSoundFnCall(e, f);
    address approvalAfter  = unsafeGetApproved(tokenId);

    // approve can set any value, other functions reset
    assert approvalBefore != approvalAfter => (
        f.selector == sig:approve(address,uint256).selector ||
        (
            (
                f.selector == sig:transferFrom(address,address,uint256).selector ||
                f.selector == sig:safeTransferFrom(address,address,uint256).selector ||
                f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector ||
                f.selector == sig:burn(uint256).selector
            ) && approvalAfter == 0
        )
    );
}

/*
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Rules: approval for all tokens can only change through isApprovedForAll.                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
*/
rule approvedForAllChange(env e, address owner, address spender) {
    bool approvedForAllBefore = isApprovedForAll(owner, spender);
    method f; helperSoundFnCall(e, f);
    bool approvedForAllAfter  = isApprovedForAll(owner, spender);

    assert approvedForAllBefore != approvedForAllAfter => f.selector == sig:setApprovalForAll(address,bool).selector;
}
```


Prover run: [link](https://prover.certora.com/output/541734/bd307ec517f248db809a17a626ab9ff7?anonymousKey=752eb5544863170f99831d7d02fc22f144d9463e)


## A note on the difference between using partially parametric or fully parametric  


These state-change properties can be verified using either a single partially parametric rule or multiple fully parametric rules (one per property). Both approaches use `method f` to reason about state changes across all contract methods, but they differ in how preconditions are scoped.



In a fully parametric rule, all preconditions are applied at the rule level and therefore constrain every possible method invocation. This works well when the same preconditions apply to all methods affecting the state being verified.



A partially parametric rule, on the other hand, uses conditional logic based on the invoked method. Instead of applying preconditions to all method invocations, it routes the arbitrary method call to method-specific branches, each with its own relevant preconditions. Preconditions apply only where they are required, which works well when a rule verifies multiple assertions that require different method-specific constraints. 


  


The choice between them is primarily about code organization: partially parametric rules centralize method-specific logic, while fully parametric rules separate concerns into independent rules. This organizational difference affects maintainability and clarity, but both approaches are equally valid when preconditions are properly scoped.
