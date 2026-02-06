# Formally Verify Solady WETH


## Introduction


ETH, widely used in DeFi for swaps, liquidity provision, staking, and collateral, needs an ERC-20-compatible version so protocols can interact with it through the same standardized interface used for other tokens.


This gave rise to Wrapped ETH (WETH), whose principle is straightforward: depositing native ETH mints an equivalent amount of WETH to the user’s account, while withdrawing burns the corresponding WETH and returns the native ETH to the user’s account.


In the previous chapter, we formally verified the essential properties of a typical ERC-20 contract. Since WETH is itself an ERC-20, we will not repeat those specifications here and will instead focus solely on the functionalities specific to WETH. We will formally verify the [Solady WETH](https://github.com/vectorized/solady/blob/main/src/tokens/WETH.sol) implementation, which uses inline assembly for gas optimization.


## WETH properties to formally verify


When a user deposits ETH, they must receive an equivalent amount of WETH. When they withdraw, they must be able to redeem that WETH for an equal amount of ETH at any time.


These two behaviors define the core guarantees of the WETH contract beyond the standard ERC-20 functionalities. To uphold them, the contract must satisfy two key invariants:

1. **The ETH held by the contract must always be greater than or equal to the WETH total supply.** This guarantees that all WETH holders can redeem their tokens for ETH at any time, regardless of redemption order or timing.
2. **The WETH total supply must always be greater than or equal to the sum of all WETH balances.** This guarantees that the contract cannot distribute more tokens than it has minted.

However, we will not formally verify the second invariant because the Solady WETH contract inherits an ERC-20 implementation that does not use a mapping for account balances. Instead, it reads from and writes to storage using a calculated slot for each account.


Therefore, verifying this invariant requires CVL “summarization”: we would need to summarize functions that modify account balance storage, mirror them using a ghost mapping, and reason about the invariant through that ghost state.  This topic is beyond the scope of this chapter. For further information, see the Certora documentation on [summarization](https://docs.certora.com/en/latest/docs/prover/approx/summarization.html).


Instead of verifying this invariant, we formally verify another: no account balance exceeds the total supply. It may seem trivial, but it becomes useful later when deposit and withdraw rules rely on balance-related preconditions. This invariant allows us to replace those assumptions using `requireInvariant`, which provides guarantees that those preconditions we assume are indeed valid.


## **ETH deposited equals the WETH received**


When a user deposits ETH into the contract using the function `deposit()`,  their WETH balance increases and their ETH balance decreases by the same amount. To formally verify this behavior in CVL, we proceed as follows:

1. Set preconditions:
    - The sender must not be the contract itself (`e.msg.sender != currentContract`) because self-calls break ETH balance accounting, and no execution path allows such calls.
    - The sender's WETH balance plus the deposited amount must not overflow (`balanceOf(e.msg.sender) + e.msg.value <= max_uint256`).
2. Record the user's ETH and WETH balances before invoking `deposit()`.
3. Invoke `deposit()` with the ETH amount in `e.msg.val``ue`.
4. Record the user's ETH and WETH balances after invoking `deposit()`.
5. Assert the correctness conditions:
    - The final ETH balance equals the initial ETH balance minus the deposited amount.
    - The final WETH balance equals the initial WETH balance plus the deposited amount.

Here is the CVL rule implementation: 


```solidity
rule deposit_ethDepositedEqualsWethReceived(env e) {
    require e.msg.sender != currentContract;
    require balanceOf(e.msg.sender) + e.msg.value <= max_uint256; // will be replaced by an invariant

    mathint ethBalanceBefore = nativeBalances[e.msg.sender];
    mathint wethBalanceBefore = balanceOf(e.msg.sender);
    
    deposit(e);

    mathint ethBalanceAfter = nativeBalances[e.msg.sender];
    mathint wethBalanceAfter = balanceOf(e.msg.sender);

    assert ethBalanceAfter == ethBalanceBefore - e.msg.value;
    assert wethBalanceAfter == wethBalanceBefore + e.msg.value;
}
```


Now let's break this down. 


### **Preconditions**


The first precondition instructs the Prover to exclude cases where the caller (`msg.sender`) is the contract itself:


```solidity
require e.msg.sender != currentContract;
```


Without this precondition, the Prover assumes that the caller can be the `currentContract`, which causes the rule to fail. For example, if the contract were to deposit 1 ETH into itself, there would be no net change in its ETH holdings (as it sends and receives the same amount to itself). However, the contract would still mint 1 WETH, causing the assertion `ethBalanceAfter == ethBalanceBefore - e.msg.value (amount` `deposited``)` to fail.


The second precondition, `require balanceOf(e.msg.sender) + e.msg.value <= max_uint25``6`, rules out the possibility of an overflow when adding the caller’s initial WETH balance and the ETH deposit. 


Since `require` statements are assumptions that the Prover does not verify — it simply takes them as given — we need to formally prove this as an invariant in a later section. Once proven, we can replace this assumed precondition with the verified invariant using `requireInvariant`.


### **Record the user's ETH and WETH** **balances before and after the** **`deposit()`** **call**


After all preconditions are set, we must record the ETH and WETH balances of the caller before and after invoking `deposit(``)`. These recorded values will be used in the assertions to reason about the balance changes:


 


```solidity
mathint ethBalanceBefore = nativeBalances[e.msg.sender];
mathint wethBalanceBefore = balanceOf(e.msg.sender);

deposit(e);

mathint ethBalanceAfter = nativeBalances[e.msg.sender];
mathint wethBalanceAfter = balanceOf(e.msg.sender);
```


### **Assertions**


The first assertion checks that the caller’s ETH balance decreased by exactly the amount of ETH sent with the transaction. The second assertion checks that the caller's WETH balance increased by exactly the deposited ETH amount: 


```solidity
assert ethBalanceAfter == ethBalanceBefore - e.msg.value; // ETH is deposited -- ETH balance decreases
assert wethBalanceAfter == wethBalanceBefore + e.msg.value; // WETH is received -- WETH balance increases
```



Here's the verified Prover [run](https://prover.certora.com/output/541734/81a587bf17514eeca0f3c5200b40676a?anonymousKey=206b62220c55df541dbef27cd11afc5c1123872d) for the `deposit_ethDepositedEqualsWethReceived` rule.


## ETH deposit increases WETH total supply


In addition to changing balances, another expected effect of a `deposit()` call is an increase in the WETH `totalSupply`. The following CVL rule captures this behavior:


```solidity
rule deposit_ethDepositIncreasesWETHTotalSupply(env e) {
    mathint totalSupplyBefore = totalSupply();
    
    deposit(e);
    mathint totalSupplyAfter = totalSupply();

    assert totalSupplyAfter == totalSupplyBefore + e.msg.value;
}
```


The variables `totalSupplyBefore` and `totalSupplyAfter` record the total supply values before and after the `deposit()` call to verify the expected change in the assertion.


Since the `deposit()` function internally calls `_mint()`, the total supply of WETH increases by the deposited amount, as reflected in the assertion:


```solidity
assert totalSupplyAfter == totalSupplyBefore + e.msg.value;
```


Notice there is no precondition `require msg.sender != currentContract` because the assertion concerns only the supply increase. The supply increases by the deposited ETH amount (`msg.value`) regardless of who the sender is.


Here's the verified Prover [run](https://prover.certora.com/output/541734/cbbd5ce0f767409cbba0f2856d6959da?anonymousKey=678bb705820892c463c86f1f510bbc0ae98dbd06) for the `deposit_ethDepositIncreasesWETHTotalSupply` rule.


## **Verifying causes of revert in** **`deposit()`** 


Let’s now verify the revert path of the `deposit()` function. We want to detect cases where it either reverts unexpectedly or fails to revert when it should. 


To formally verify this behavior in CVL, we proceed as follows:

1. Set a precondition that the caller’s WETH balance plus the deposited amount must not overflow (`balanceOf(caller) + ethDeposit <= max_uint256`).
2. Record the WETH `totalSupply` and the caller’s ETH balance before invoking `deposit()`.
3. Invoke `deposit()` with the `@withrevert` tag to allow the rule to observe reverts.
4. Use `lastReverted`, which returns a boolean indicating whether the last call reverted.
5. Assert that the function reverts if and only if (`<=>`) either of the following conditions is true:
    - The caller’s ETH balance before the call is less than the deposit amount (`e.msg.value`).
    - The total WETH supply before the call plus the deposit amount exceeds `max_uint256`.

Here's the CVL rule:


```solidity
rule deposit_revert(env e) {
    address caller = e.msg.sender;
    uint256 ethDeposit = e.msg.value;

    require balanceOf(caller) + ethDeposit <= max_uint256; // will be replaced by an invariant
    
    mathint totalSupplyBefore = totalSupply();
    mathint ethBalanceBefore = nativeBalances[caller];

    deposit@withrevert(e);
    bool isLastReverted = lastReverted;
    
    assert isLastReverted <=> (
        ethBalanceBefore < ethDeposit || 
        totalSupplyBefore + ethDeposit > max_uint256
    );
}
```


Let’s break down the rule further.


For readability, we assign `e.msg.sender` and `e.msg.value` to `caller` and `ethDeposit`, respectively:


 


```solidity
address caller = e.msg.sender;
uint256 ethDeposit = e.msg.value;
```


The precondition, shown below, also appeared in the previous rule `deposit_ethDepositedEqualsWethReceived()`: 


```solidity
require balanceOf(caller) + ethDeposit <= max_uint256; // will be replaced by an invariant
```


Its reappearance here as a precondition, rather than as part of the revert conditions, further reinforces the need to formally verify it.


Next, we record the values of `totalSupplyBefore` (WETH total supply) and the caller’s `nativeBalances` before invoking the `deposit()` function:


```solidity
mathint totalSupplyBefore = totalSupply();
mathint ethBalanceBefore = nativeBalances[caller];
```


Now, we invoke the `deposit()` function with the `@withrevert` tag and record whether the call reverted:


```solidity
deposit@withrevert(e);
bool isLastReverted = lastReverted;
```


Finally, the assertion uses the biconditional operator (`<=>`) and lists all the revert cases disjunctively (using OR):

- The function reverts when the caller’s ETH balance before (`ethBalanceBefore`) is less than the ETH deposit (`e.msg.value`).

    This means the caller doesn’t have enough ETH to make the deposit.

- The function also reverts when the WETH total supply (`totalSupplyBefore`) plus the ETH deposit would exceed `max_uint256`.

    Since each deposit mints an equal amount of WETH, exceeding the `uint256` limit would result in an overflow.


```solidity
assert isLastReverted <=> (
    ethBalanceBefore < ethDeposit || // the caller doesn't have enough eth to deposit
    totalSupplyBefore + ethDeposit > max_uint256 // results in overflow
);
```


The precondition `msg.sender != currentContract` is unnecessary because the revert conditions concern insufficient balance and overflow, which are independent of the caller's identity.


Here's the verified Prover [run](https://prover.certora.com/output/541734/024961a5760d4a6da8e86054cd5321c8?anonymousKey=44cdcb861c77a47d390cc9f44ef4fe49d9d69a9a) for the `deposit_revert` rule.


## ETH withdrawn equals WETH reduced


When a user withdraws ETH from the contract, their ETH balance increases and their WETH balance decreases by the same amount. The following CVL rule captures this behavior:


```solidity
rule withdraw_ethWithdrawnEqualsWETHReduced(env e) {
    uint256 amount;

    require e.msg.sender != currentContract;

    mathint ethBalanceBefore = nativeBalances[e.msg.sender];
    mathint wethBalanceBefore = balanceOf(e.msg.sender); 

    withdraw(e,amount);

    mathint ethBalanceAfter = nativeBalances[e.msg.sender];
    mathint wethBalanceAfter = balanceOf(e.msg.sender);

    assert ethBalanceAfter == ethBalanceBefore + amount;
    assert wethBalanceAfter == wethBalanceBefore - amount;
}
```


This follows the same structure as `rule deposit_ethDepositedEqualsWethReceived()`. By now, the pattern should be familiar:

- Set preconditions
- Record the caller's ETH and WETH balances before invoking `withdraw()`
- Invoke `withdraw()`
- Record the caller’s ETH and WETH balances afterward

```solidity
require e.msg.sender != currentContract;

mathint ethBalanceBefore = nativeBalances[e.msg.sender];
mathint wethBalanceBefore = balanceOf(e.msg.sender); 

withdraw(e,amount);

mathint ethBalanceAfter = nativeBalances[e.msg.sender];
mathint wethBalanceAfter = balanceOf(e.msg.sender);
```


Now, let’s go straight to the assertion.


In the assertions below, we verify that the ETH balance increases and the WETH balance decreases by the same withdrawn `amount`:


```solidity
assert ethBalanceAfter == ethBalanceBefore + amount;
assert wethBalanceAfter == wethBalanceBefore - amount;
```


The precondition `require msg.sender != currentContract` is necessary because a self-call would produce no change in the contract's ETH balance (depositing to itself yields no net change), which causes the assertion `ethBalanceAfter == ethBalanceBefore + amount` to fail (a false positive).



Here's the verified Prover [run](https://prover.certora.com/output/541734/d73b9ea5f2bf45a3a4dc21b59dce1495?anonymousKey=443164a2e2ddee94e79c749bd973dcaa894a9d2b) for the `withdraw_ethWithdrawnEqualsWETHReduced` rule.


## **ETH withdrawn decreases the WETH total supply**


When a user withdraws ETH, the corresponding WETH amount is burned, which reduces the total supply. This is an expected effect of the `withdraw()` function. The following CVL rule captures this behavior:


```solidity
rule withdraw_ethWithdrawDecreasesWETHSupply(env e) {
    uint256 amount;
    
	require balanceOf(e.msg.sender) <= totalSupply(); // will be replaced by an invariant
    mathint totalSupplyBefore = totalSupply();
    
    withdraw(e, amount);
    mathint totalSupplyAfter = totalSupply();

    assert totalSupplyAfter == totalSupplyBefore - amount;
}
```


Since the withdrawal reduces total supply by the withdrawn amount (which equals the burned balance), the following precondition ensures that the `totalSupply` subtraction cannot underflow by requiring that no individual balance exceeds the total supply:


 


```solidity
require balanceOf(e.msg.sender) <= totalSupply(); // will be replaced by an invariant
```


As with the previous rule, this precondition represents an assumption that must be formally verified later.


Next, we follow a familiar pattern, where we record the WETH total supply before and after the `withdraw` call:


 


```solidity
mathint totalSupplyBefore = totalSupply();
    
withdraw(e, amount);
mathint totalSupplyAfter = totalSupply();
```


Then we assert that the WETH `totalSupply` is reduced by the expected amount:


```solidity
assert totalSupplyAfter == totalSupplyBefore - amount;
```


The precondition `require e.msg.sender != currentContract` is unnecessary here because the supply decrease (or supply change) is independent of who the caller is.


This rule confirms that burning WETH during withdrawal correctly decreases the total supply. Here's the verified Prover [run](https://prover.certora.com/output/541734/ecfe34e56f3c4b029c70c6dad7fe0412?anonymousKey=639d411a33d3e139740db44aa1890c404ddbf05e).


## **Verifying causes of revert in** **`withdraw()`**


The `withdraw_revert` rule follows a different pattern from the `deposit_revert` rule. To understand why, let’s first take a look at the `withdraw` function:


```solidity
/// @dev Burns `amount` WETH of the caller and sends `amount` ETH to the caller.
function withdraw(uint256 amount) public virtual {
    _burn(msg.sender, amount);
    /// @solidity memory-safe-assembly
    assembly {
        // Transfer the ETH and check if it succeeded or not.
        if iszero(call(gas(), caller(), amount, codesize(), 0x00, codesize(), 0x00)) {
            mstore(0x00, 0xb12d13eb) // `ETHTransferFailed()`.
            revert(0x1c, 0x04)
        }
    }
}
```


Besides the `_burn()` function, which implements its own revert conditions, the `withdraw()` function includes assembly code that checks whether the ETH transfer succeeded:


```solidity
assembly {
    // Transfer the ETH and check whether it succeeded.
    if iszero(call(gas(), caller(), amount, codesize(), 0x00, codesize(), 0x00)) {
        mstore(0x00, 0xb12d13eb) // `ETHTransferFailed()`.
        revert(0x1c, 0x04)
    }
}
```


The `iszero` expression returns true if the ETH transfer fails, which triggers a revert. Otherwise, it returns `false`, meaning the ETH transfer succeeded. To verify this assembly behavior, we use a CVL hook (`CALL`) to observe whether the low-level ETH transfer call reverts as expected.


Here's the CVL rule that verifies all `withdraw()` revert conditions, including the low-level call failure:  


```javascript
persistent ghost bool g_lowLevelCallFail;


hook CALL(uint gas, address to, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc {    

    if (rc == 0) {
        g_lowLevelCallFail = true; 
    } else {
        g_lowLevelCallFail = false; 
    }
}

rule withdraw_revert(env e) {
    uint256 amount; // the amount of eth to claim    
    
    mathint wethBalanceBefore = balanceOf(e.msg.sender); 

    withdraw@withrevert(e, amount);
    bool isLastReverted = lastReverted;

    assert isLastReverted <=> (
        amount > wethBalanceBefore ||
        e.msg.value != 0 ||
        g_lowLevelCallFail
    );
}
```


Let's explain the rule above. 


The line below is the declaration of the persistent ghost variable: 


```solidity
persistent ghost bool g_lowLevelCallFail;
```


_Note: See the separate chapter "Using Persistent_ _Ghost__s" for discussion of the difference between regular ghost and persistent ghost._


The ghost variable is updated by the `CALL` hook. The hook is triggered when the `withdraw()` method is invoked, specifically by the low-level call that transfers ETH to the caller. If the low-level call fails (`rc == 0`), the ghost variable `g_lowLevelCallFail` is set to `true`; if the call succeeds (`rc != 0`), it is set to `false`, as shown in the `CALL` hook below:  


```solidity
hook CALL(uint gas, address to, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc {    
    if (rc == 0) {
        g_lowLevelCallFail = true; 
    } else {
        g_lowLevelCallFail = false; 
    }
}
```


This gives the Prover a way to track ETH transfer success/failure inside assembly blocks. 


 


Now let's go to the rule. 


As usual, we record the balance before calling the `withdraw()` function:


```solidity
mathint wethBalanceBefore = balanceOf(e.msg.sender);
```


We then invoke the `withdraw()` function with the `@withrevert` tag and record whether the call reverted:


 


```solidity
withdraw@withrevert(e, amount);
bool isLastReverted = lastReverted;
```


In the assertion below, the function will revert if and only if (`<=>`) one of the following conditions is met:

- The withdrawal `amount` exceeds the caller’s WETH balance (`wethBalanceBefore`)

    This means the caller is attempting to withdraw more ETH than their WETH balance allows.

- ETH was sent along with the call

    This is disallowed because `withdraw` is not a payable function.

- The ETH transfer back to the caller failed

    This condition is tracked via the low-level `CALL` result using the `g_lowLevelCallFail` ghost variable.


```javascript
assert isLastReverted <=> (
    amount > wethBalanceBefore ||   // insufficient balance: withdrawal amount exceeds weth balance
    e.msg.value != 0 ||             // non-payable: ETH was sent to a non-payable function
    g_lowLevelCallFail              // transfer failure: low-level call ETH transfer failed
);
```


Here's the verified Prover [run](https://prover.certora.com/output/541734/bccb33b602ad4b7186a20e775d08c852?anonymousKey=3b5bd63a45a67d740c67a4efc3b2126e65265fb8) for the `withdraw_revert` rule.


## Invariant: Total ETH deposits are greater than or equal to WETH total supply (solvency)


For users to successfully redeem ETH using their WETH, the ETH held by the contract must always be greater than or equal to the total WETH supply. Here's the CVL invariant that captures this property:


```solidity
invariant ethDepositsAlwaysGTEWethTotalSupply() 
    nativeBalances[currentContract] >= totalSupply()
{
    preserved with (env e) {
        require e.msg.sender != currentContract;
    }

    preserved withdraw(uint256 amount) with (env e) {
        require balanceOf(e.msg.sender) <= totalSupply();
    }
}
```


The line below expresses the invariant: 


```solidity
nativeBalances[currentContract] >= totalSupply()
```


However, this invariant does not hold universally. It holds only under specific conditions defined in the `preserved` blocks. The preserved block below requires that the caller (`e.msg.sender`) is not equal to the `currentContract`. 


```solidity
preserved with (env e) {
    require e.msg.sender != currentContract;
}
```


It is not a coincidence that this `require` condition appears in the following rules:

- `deposit_ethDepositedEqualsWethReceived()`
- `withdraw_ethWithdrawnEqualsWETHReduced()`

It's clear that this relates to the accounting between ETH deposits and WETH issuance. If the contract calls itself and deposits ETH, there's no net ETH gain, but WETH may still be minted, which causes `totalSupply` to exceed actual ETH deposits.


The next `preserved` block requires that `msg.sender`'s balance is less than or equal to `totalSupply()`, which prevents the withdraw operation from causing underflow:


```solidity
preserved withdraw(uint256 amount) with (env e) {
    require balanceOf(e.msg.sender) <= totalSupply();
}
```


It’s no surprise this condition reappears in a related rule. While it shows up here as a preserved condition for `withdraw()`, it previously appeared as a precondition in the rule `withdraw_ethWithdrawDecreasesWETHSupply`.


In both cases, the aim is to prevent underflow-related counterexamples. This is a strong reason to prove the condition as an invariant, which we will do in the next section.


Here's the verified Prover [run](https://prover.certora.com/output/541734/f411b3e70a07420db3ee704d37e5f6c6?anonymousKey=1ce8727e9955df02bee5bce40a70005d1d13ec69) for the `ethDepositsAlwaysGTEWethTotalSupply` invariant.


## Invariant: No account balance exceeds WETH total supply


This invariant, expressed as `balanceOf(address) <= totalSupply()`, guarantees the following:

- No account balance ever exceeds the total supply.
- As a consequence, since `totalSupply()` itself cannot exceed `max_uint256`, individual account balances also cannot overflow.

In the previous rules we discussed, the following preconditions were introduced to guard against overflow:


```solidity
rule deposit_ethDepositedEqualsWethReceived(env e) {
    ...
    require balanceOf(e.msg.sender) + e.msg.value <= max_uint256; // precondition
    
    ...
}
```


```solidity
rule deposit_revert(env e) {
    ...
    require balanceOf(caller) + ethDeposit <= max_uint256; // precondition
    
    ...
}
```


In the `withdraw` rule below, the precondition was added to guard against underflow: 


```solidity
rule withdraw_ethWithdrawDecreasesWETHSupply(env e) {
    ...
    require balanceOf(e.msg.sender) <= totalSupply(); // precondition
    
    ...
}
```


In the `ethDepositsAlwaysGTEWethTotalSupply` invariant below, it is also used as a preserved condition to guard against underflow: 


 


```solidity
invariant ethDepositsAlwaysGTEWethTotalSupply() 
    nativeBalances[currentContract] >= totalSupply()
{
    ...

    preserved withdraw(uint256 amount) with (env e) {
        require balanceOf(e.msg.sender) <= totalSupply(); 
    }
}
```


The invariant below directly addresses these recurring preconditions by formally verifying that no account balance ever exceeds the total supply:


```solidity
invariant noAccountBalanceExceedsTotalSupply(env e1) 
    balanceOf(e1.msg.sender) <= totalSupply()
    
    filtered {
        f -> f.selector != sig:transfer(address,uint256).selector && // excludes standard ERC-20 transfer function from this invariant
			f.selector != sig:transferFrom(address,address,uint256).selector // excludes standard ERC-20 transferFrom function from this invariant
    }
    {
        preserved withdraw(uint256 amount) with (env e2) {
            require e1.msg.sender == e2.msg.sender;
        }
    }
```


Let’s break it down.


The line below is the invariant:


```solidity
balanceOf(e1.msg.sender) <= totalSupply()
```


We filter out `transfer()` and `transferFrom()` using `filtered` because these functions compute and write to storage slots, rather than using a mapping variable for account balances. Hence, tracking account balances would require summarizing these functions using CVL summaries (a topic beyond the scope of this chapter).


```solidity
filtered {
    f -> f.selector != sig:transfer(address,uint256).selector &&
        f.selector != sig:transferFrom(address,address,uint256).selector
}
```


It is reasonable to filter out `transfer()` and `transferFrom()`, as only `deposit()` and `withdraw()` (which invoke `_mint()` and `_burn()` respectively) can affect the invariant. `transfer()` and `transferFrom()` simply move tokens already accounted for in the total supply. If the total supply is incorrect, the issue originates from `deposit()` or `withdraw()`, not from transfers.


Had Solady WETH used mapping variables (which it intentionally doesn’t by design), tracking balances and defining this invariant would have been straightforward, and these filters wouldn’t be necessary.


That said, like preconditions and preserved blocks, filters should be used with caution, as they can potentially hide real bugs.


Lastly, the preserved block requires that `e1.msg.sender`(the address whose balance we're checking in the invariant) equals the caller during the execution of the `withdraw` function: 


 


```solidity
{
    preserved withdraw(uint256 amount) with (env e2) {
        require e1.msg.sender == e2.msg.sender;
    }
}
```


The preserved condition `require e1.msg.sender == e2.msg.sender` prevents the Prover from testing the invariant on an account that does not call `withdraw()`. 


The `withdraw()` function triggers a burn, which reduces total supply while decreasing only the caller’s balance. If the invariant checks a different account whose balance does not change, the Prover may observe `balanceOf(differentAccount) > totalSupply()` after the burn and report a false violation. 


By requiring the invariant to follow the same address that executes `withdraw()`, the preserved block enforces that the check applies to the account whose balance actually decreases.


Here's the verified Prover [run](https://prover.certora.com/output/541734/6960cd158ce3445da05b1306aecaf065?anonymousKey=c20be469d8ee89ae7dff0b8f24f8c505774af69f).


## Replace Preconditions With `requireInvariant`


Since `noAccountBalanceExceedsTotalSupply()` is now formally verified, we can replace the preconditions and preserved conditions with this invariant in the following specifications: 

- `rule deposit_ethDepositedEqualsWethReceived()`
- `rule deposit_revert()`
- `rule withdraw_ethWithdrawDecreasesWETHSupply()`
- `invariant ethDepositsAlwaysGTEWethTotalSupply()`

Here are the refactored rules and invariant using `requireInvariant`:


```solidity
rule deposit_ethDepositedEqualsWethReceived_withInvariant(env e) {
    require e.msg.sender != currentContract;
    requireInvariant noAccountBalanceExceedsTotalSupply(e);

    mathint ethBalanceBefore = nativeBalances[e.msg.sender];
    mathint wethBalanceBefore = balanceOf(e.msg.sender);
    
    deposit(e);

    mathint ethBalanceAfter = nativeBalances[e.msg.sender];
    mathint wethBalanceAfter = balanceOf(e.msg.sender);

    assert ethBalanceAfter == ethBalanceBefore - e.msg.value;
    assert wethBalanceAfter == wethBalanceBefore + e.msg.value;
}

rule deposit_revert_withInvariant(env e) {
    address caller = e.msg.sender;
    uint256 ethDeposit = e.msg.value;

    requireInvariant noAccountBalanceExceedsTotalSupply(e);
    
    mathint totalSupplyBefore = totalSupply();
    mathint ethBalanceBefore = nativeBalances[caller];

    deposit@withrevert(e);
    bool isLastReverted = lastReverted;
    
    assert isLastReverted <=> (
        ethBalanceBefore < ethDeposit || 
        totalSupplyBefore + ethDeposit > max_uint256
    );
}

rule withdraw_ethWithdrawDecreasesWETHSupply_withInvariant(env e) {
    uint256 amount;
    requireInvariant noAccountBalanceExceedsTotalSupply(e);
    
    mathint totalSupplyBefore = totalSupply();
    
    withdraw(e, amount);
    mathint totalSupplyAfter = totalSupply();

    assert totalSupplyAfter == totalSupplyBefore - amount;
}

invariant ethDepositsAlwaysGTEWethTotalSupply_withInvariant() 
    nativeBalances[currentContract] >= totalSupply()
{
    preserved with (env e) {
        require e.msg.sender != currentContract;
    }

    preserved withdraw(uint256 amount) with (env e) {
        requireInvariant noAccountBalanceExceedsTotalSupply(e);
    }
}
```


And here's the verified Prover [run](https://prover.certora.com/output/541734/a3bb2cfafc74468bb5e3899b8db869a6?anonymousKey=5db240f01f268c7376bf7d5bff7ef0d9a63e05b6) for these rules and the invariant.


## Full CVL specification and Prover run


Here's the complete specification written in this chapter:  


 


```javascript
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function totalSupply() external returns (uint256) envfree;
}

persistent ghost bool g_lowLevelCallFail;

hook CALL(uint gas, address to, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc {    
    if (rc == 0) {
        g_lowLevelCallFail = true; 
    } else {
        g_lowLevelCallFail = false; 
    }
}

rule deposit_ethDepositedEqualsWethReceived(env e) {
    require e.msg.sender != currentContract;
    require balanceOf(e.msg.sender) + e.msg.value <= max_uint256; // will be replaced by an invariant

    mathint ethBalanceBefore = nativeBalances[e.msg.sender];
    mathint wethBalanceBefore = balanceOf(e.msg.sender);
    
    deposit(e);

    mathint ethBalanceAfter = nativeBalances[e.msg.sender];
    mathint wethBalanceAfter = balanceOf(e.msg.sender);

    assert ethBalanceAfter == ethBalanceBefore - e.msg.value;
    assert wethBalanceAfter == wethBalanceBefore + e.msg.value;
}

rule deposit_ethDepositedEqualsWethReceived_withInvariant(env e) {
    require e.msg.sender != currentContract;
    requireInvariant noAccountBalanceExceedsTotalSupply(e);

    mathint ethBalanceBefore = nativeBalances[e.msg.sender];
    mathint wethBalanceBefore = balanceOf(e.msg.sender);
    
    deposit(e);

    mathint ethBalanceAfter = nativeBalances[e.msg.sender];
    mathint wethBalanceAfter = balanceOf(e.msg.sender);

    assert ethBalanceAfter == ethBalanceBefore - e.msg.value;
    assert wethBalanceAfter == wethBalanceBefore + e.msg.value;
}

rule deposit_ethDepositIncreasesWETHTotalSupply(env e) {
    mathint totalSupplyBefore = totalSupply();
    
    deposit(e);
    mathint totalSupplyAfter = totalSupply();

    assert totalSupplyAfter == totalSupplyBefore + e.msg.value;
}

rule deposit_revert(env e) {
    address caller = e.msg.sender;
    uint256 ethDeposit = e.msg.value;

    require balanceOf(caller) + ethDeposit <= max_uint256; // will be replaced by an invariant
    
    mathint totalSupplyBefore = totalSupply();
    mathint ethBalanceBefore = nativeBalances[caller];

    deposit@withrevert(e);
    bool isLastReverted = lastReverted;
    
    assert isLastReverted <=> (
        ethBalanceBefore < ethDeposit || 
        totalSupplyBefore + ethDeposit > max_uint256
    );
}

rule deposit_revert_withInvariant(env e) {
    address caller = e.msg.sender;
    uint256 ethDeposit = e.msg.value;

    requireInvariant noAccountBalanceExceedsTotalSupply(e);
    
    mathint totalSupplyBefore = totalSupply();
    mathint ethBalanceBefore = nativeBalances[caller];

    deposit@withrevert(e);
    bool isLastReverted = lastReverted;
    
    assert isLastReverted <=> (
        ethBalanceBefore < ethDeposit || 
        totalSupplyBefore + ethDeposit > max_uint256
    );
}

rule withdraw_ethWithdrawnEqualsWETHReduced(env e) {
    uint256 amount;

    require e.msg.sender != currentContract;

    mathint ethBalanceBefore = nativeBalances[e.msg.sender];
    mathint wethBalanceBefore = balanceOf(e.msg.sender); 

    withdraw(e,amount);

    mathint ethBalanceAfter = nativeBalances[e.msg.sender];
    mathint wethBalanceAfter = balanceOf(e.msg.sender);

    assert ethBalanceAfter == ethBalanceBefore + amount;
    assert wethBalanceAfter == wethBalanceBefore - amount;
}

rule withdraw_ethWithdrawDecreasesWETHSupply(env e) {
    uint256 amount;

    require balanceOf(e.msg.sender) <= totalSupply(); // will be replaced by an invariant
    
    mathint totalSupplyBefore = totalSupply();
    
    withdraw(e, amount);
    mathint totalSupplyAfter = totalSupply();

    assert totalSupplyAfter == totalSupplyBefore - amount;
}

rule withdraw_ethWithdrawDecreasesWETHSupply_withInvariant(env e) {
    uint256 amount;
    requireInvariant noAccountBalanceExceedsTotalSupply(e);
    
    mathint totalSupplyBefore = totalSupply();
    
    withdraw(e, amount);
    mathint totalSupplyAfter = totalSupply();

    assert totalSupplyAfter == totalSupplyBefore - amount;
}

rule withdraw_revert(env e) {
    uint256 amount; // the amount of eth to claim    
    
    mathint wethBalanceBefore = balanceOf(e.msg.sender); 

    withdraw@withrevert(e, amount);
    bool isLastReverted = lastReverted;

    assert isLastReverted <=> (
        amount > wethBalanceBefore ||
        e.msg.value != 0 ||
        g_lowLevelCallFail
    );
}

invariant ethDepositsAlwaysGTEWethTotalSupply() 
    nativeBalances[currentContract] >= totalSupply()
{
    preserved with (env e) {
        require e.msg.sender != currentContract;
    }

    preserved withdraw(uint256 amount) with (env e) {
        require balanceOf(e.msg.sender) <= totalSupply();
    }
}

invariant ethDepositsAlwaysGTEWethTotalSupply_withInvariant() 
    nativeBalances[currentContract] >= totalSupply()
{
    preserved with (env e) {
        require e.msg.sender != currentContract;
    }

    preserved withdraw(uint256 amount) with (env e) {
        requireInvariant noAccountBalanceExceedsTotalSupply(e);
    }
}

invariant noAccountBalanceExceedsTotalSupply(env e1) 
    balanceOf(e1.msg.sender) <= totalSupply()
    
    filtered {
        f -> f.selector != sig:transfer(address,uint256).selector &&
            f.selector != sig:transferFrom(address,address,uint256).selector
    }
    {
        preserved withdraw(uint256 amount) with (env e2) {
            require e1.msg.sender == e2.msg.sender;
        }
    }
```


Here's the verified Prover [run](https://prover.certora.com/output/541734/9f253f7a8e1d4222811f0acedc9f8e13?anonymousKey=06fa7f35bf88e1f52c40de012e405e7d95e3bb78) of the WETH specification discussed in this chapter.
