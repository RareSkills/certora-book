# Persistent Ghosts


## Introduction


In previous chapters, we used ghost variables (via `Sstore` hooks) to record storage values and quantities that were not explicitly tracked in smart contracts — for example, the sum of all balances. They allow changes in state or relationships between state variables over time to be observed and reasoned about during verification.


However, during rule execution after a contract method is invoked, ghost variables can be:

- havoc’ed when the Prover encounters an unresolved external call, or
- rolled back to their previous state (reset) when the call reverts.

Ghosts are havoc’ed during an unresolved external call because the Prover cannot see the called contract’s implementation. Since the external contract could potentially call back into the caller and modify storage, the Prover cannot rule out that ghost variables might be updated through their hooks. To handle this uncertainty, the Prover assigns nondeterministic values to ghost variables.


When a call reverts, ghosts do not become nondeterministic. Instead, they roll back (reset) to the the value they held before the call began. According to the Certora [docs](https://docs.certora.com/en/latest/docs/cvl/ghosts.html), "Ghosts can be seen as an extension to the state of the contracts under verification. This means that in case a call reverts, the ghost values will revert to their pre-state."


These are the default behaviors of ghosts: they are havoc’ed in unresolved calls and reset when a method call reverts. However, there is a special-purpose type of ghost called a **persistent ghost**, which maintains its value (does not havoc) across both unresolved external calls and reverts.


In this chapter, we will demonstrate how persistent ghosts work, how they differ from regular ghosts, when to use them, and when not to.


## **Declaring a persistent ghost**


A persistent ghost is declared with the `persistent` keyword, as shown below:


```solidity
persistent ghost bool g_flag;
persistent ghost uint256 g_count;
```


For readability, we use the prefix `g_` for all ghost variables in this article, though they may be named freely.


To initialize persistent ghosts with specific values, add `init_state axiom` within the ghost declaration:


```solidity
persistent ghost bool g_flag {
    init_state axiom g_flag == false;
}

persistent ghost uint256 g_count {
    init_state axiom g_count == 0;
}
```


_Note:_ _While persistent ghosts are not havoc'ed during unresolved method calls, they are havoc'ed in the pre-call state for rules and in the base case for invariants; the same goes for regular ghosts. Therefore, they must still be properly constrained as a precondition (via require statements) in rules and_ _initialized using the_ _`init_state axiom`_ _for_ _invariants’_ _base cases_. 


## **The behavior of persistent vs. regular ghosts**


The diagram below shows how a persistent ghost retains its value across reverted calls and unresolved external calls. During rule execution, when a hook assigns a value to `ghostVar`, that value is never reverted (reset) or havoc’ed, so its final state always reflects the last value assigned by the hook:


![image](media/certora-persistent-ghosts/image1.png)


In contrast to persistent ghosts, regular ghosts are havoc'ed on unresolved external calls and reset on reverts:


![image](media/certora-persistent-ghosts/image2.png)


_Note: Both persistent and regular ghosts maintain their values only within a single rule execution and do not carry over between rules. The difference is that persistent ghosts survive unresolved or reverted calls within that execution, while regular ghosts do not._


## **Tracking reverts with persistent ghosts**


This section demonstrates how persistent ghosts enable the tracking of reverts in a way that regular ghosts cannot. It explains why regular ghosts fail in this use case and shows how persistent ghosts solve the problem.



### The regular ghost resets when a method call reverts


When a method reverts, the state of regular ghosts is also rolled back. This behavior aligns with how storage variables revert to their previous values during a failed call.


To illustrate this behavior, consider this contract, `SimpleVault`, where users can deposit and withdraw ETH:


```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

contract SimpleVault {
    mapping(address => uint256) public balanceOf;

    /// deposit ETH into the vault
    function deposit() external payable {
        balanceOf[msg.sender] += msg.value;
    }

    /// withdraw ETH from the vault
    function withdraw(uint256 amount) external {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "ETH transfer failed");
    }
}
```


The goal is to verify that `withdraw()` reverts in exactly these cases:

- the withdrawal amount exceeds the deposited balance (`amount > balanceBefore`)
- a nonzero `msg.value` is sent to this non-payable function (`e.msg.value != 0`)
- the ETH transfer fails, or the low-level call returned false

This section focuses specifically on the third case: tracking ETH transfer failures.


The following rule implements an assertion pattern from the chapter on “Account Balances”. It uses the biconditional operator (`<=>`) and lists all expected revert conditions disjunctively (`||`). This accounts for all possible method reverts if and only if one of these conditions occurs:



```solidity
ghost bool g_lowLevelCallSuccess; // regular ghost

rule withdraw_revert(env e) {
    uint256 amount;
   
 	require g_lowLevelCallSuccess; // by requiring it to be true initially, you can detect when it becomes false during execution

    mathint balanceBefore = balanceOf(e.msg.sender);

    withdraw@withrevert(e, amount);
    bool isLastReverted = lastReverted;

    assert isLastReverted <=> (
        amount > balanceBefore ||   // insufficient balance: withdrawal amount exceeds user balance
        e.msg.value != 0 ||         // non-payable: ETH was sent to this non-payable function
        !g_lowLevelCallSuccess      // transfer failure: low-level call ETH transfer failed
    );
}
```


To track the ETH transfer outcome, we use a `CALL` hook; a CVL feature that monitors the low-level EVM `CALL` instruction and captures its return code. We assign this return code to `g_lowLevelCallSuccess` (`0` for failure, `1` for success):


```solidity
hook CALL(uint gas, address to, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint returnCode {
    if (returnCode == 0) {
        g_lowLevelCallSuccess = false;
    } else {
        g_lowLevelCallSuccess = true;
    }
}
```


Here's the complete specification:



```solidity
methods {
    function balanceOf(address) external returns uint256 envfree;
}

ghost bool g_lowLevelCallSuccess;

hook CALL(uint gas, address to, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint returnCode {
    if (returnCode == 0) {
        g_lowLevelCallSuccess = false;
    } else {
        g_lowLevelCallSuccess = true;
    }
}

rule withdraw_revert(env e) {
    uint256 amount;
    require g_lowLevelCallSuccess;

    mathint balanceBefore = balanceOf(e.msg.sender);

    withdraw@withrevert(e, amount);
    bool isLastReverted = lastReverted;

    assert isLastReverted <=> (
        amount > balanceBefore ||   // insufficient balance: withdrawal amount exceeds user balance
        e.msg.value != 0 ||         // non-payable: ETH was sent to this non-payable function
        !g_lowLevelCallSuccess      // transfer failure: low-level call ETH transfer failed
    );
}
```


Using regular ghosts, the verification fails:


![image](media/certora-persistent-ghosts/image3.png)


See the failed Prover run: [link](https://prover.certora.com/output/541734/825aac56f8da43b89cd7d8369aee1471?anonymousKey=8c9ed01752aba57c3098d832301252e24047963c) 


Let’s examine why the failure happened.


### **Regular ghost value resets** — call trace


The verification fails because `g_lowLevelCallSuccess` is a regular ghost that resets on revert. The `CALL` hook captures the failure (`returnCode == 0`), but when `withdraw` reverts, the ghost resets and loses this value.


As shown in the image below, `g_lowLevelCallSuccess` is `true` before the `withdraw()` method is invoked (pre-call state): 


![image](media/certora-persistent-ghosts/image4.png)


When the low-level ETH transfer call fails, the `CALL` hook captures the failed call (return value of 0), which sets `g_lowLevelCallSuccess` to `false`:



![image](media/certora-persistent-ghosts/image5.png)


When the `CALL` opcode (captured by the CVL `CALL` hook) returns zero, it means the ETH transfer fails. Within the rule, the ghost resets to its pre-call state when the `withdraw@withrevert(e, amount)` invocation reverts. The image below shows the ghost returning to `true` despite the failed transfer:



![image](media/certora-persistent-ghosts/image6.png)


### Fixing the spec with a persistent ghost


To recap what we observed in the call trace with a regular ghost, the execution unfolds as follows:

1. The ghost `g_lowLevelCallSuccess` is initialized to `true` (via `require` statement).
2. The `withdraw` function executes and triggers a low-level `CALL` that fails.
3. The `CALL` hook detects the failure (`returnCode == 0`) and sets `g_lowLevelCallSuccess` to `false`.
4. The `withdraw` function reverts because `require(success)` fails when `success` is `false`.
5. The revert causes the ghost to reset to its pre-call state of `true`.

The root issue is clear: the revert in step 5 resets the ghost back to `true`, erasing the `false` value that recorded the transfer failure in step 3. 


To fix this, declare the ghost as `persistent` so that the boolean value assigned by the `CALL` hook survives the revert and remains accessible in the rule:


```solidity
methods {
    function balanceOf(address) external returns uint256 envfree;
}

persistent ghost bool g_lowLevelCallSuccess;

hook CALL(uint gas, address to, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint returnCode {
    if (returnCode == 0) {
        g_lowLevelCallSuccess = false;
    } else {
        g_lowLevelCallSuccess = true;
    }
}

rule withdraw_revert(env e) {
    uint256 amount;
    require g_lowLevelCallSuccess;

    mathint balanceBefore = balanceOf(e.msg.sender);

    withdraw@withrevert(e, amount);
    bool isLastReverted = lastReverted;

    assert isLastReverted <=> (
        amount > balanceBefore ||   // insufficient balance: withdrawal amount exceeds user balance
        e.msg.value != 0 ||         // non-payable: ETH was sent to this non-payable function
        !g_lowLevelCallSuccess      // transfer failure: low-level call ETH transfer failed
    );
}
```


As expected, the rule verifies:


![image](media/certora-persistent-ghosts/image7.png)


Prover run: [link](https://prover.certora.com/output/541734/502d0c23136841f0baa3ff2e122cb483?anonymousKey=65a6d9019f2936e0e30cf36d60a14e13161262a2)


With the persistent ghost, we successfully tracked the low-level call ETH transfer failure across the revert, something impossible with a regular ghost. This example shows one appropriate use case for persistent ghosts: capturing state information that would otherwise be lost during reverts. 


## When NOT to use persistent ghosts


Having seen a proper use case for persistent ghosts, it's equally important to understand when they should not be used. 


The ease of adding the `persistent` keyword to a regular ghost makes it prone to misuse. Use persistent ghosts intentionally when you need to observe what happens in reverted or unresolved executions — not as a workaround when rules or invariants fail to verify.


To demonstrate the problem with misuse, let's consider a similar vault that accepts ERC-20 tokens instead of ETH. Its balance mapping is private, so a ghost variable is needed to mirror the internal balances because there is no getter function to read from: 


```solidity
/// minimal ERC20 interface
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
	...
}

contract SimpleVault20 {
    IERC20 public immutable token;
    mapping(address => uint256) private balanceOf;
    
    constructor(address _token) {
        token = IERC20(_token);
    }
    
    function deposit(uint256 amount) external {
        require(amount > 0, "Zero deposit");
        
        balanceOf[msg.sender] += amount;
        require(token.transferFrom(msg.sender, address(this), amount), "transferFrom failed");
    }
    
    ...
}
```


Let's write the specifications for the `deposit``()` function. For the ghost and hooks: 

- declare the ghost as a regular ghost
- implement the `Sstore` hook to track the changes in balances per account depositor
- implement the `Sload` hook to sync the reads of the storage and the ghost

```solidity
ghost mapping(address => mathint) g_balanceOf;

hook Sstore balanceOf[KEY address account] uint256 newVal (uint256 oldVal) {
    g_balanceOf[account] = g_balanceOf[account] + newVal - oldVal;
}

hook Sload uint256 balance balanceOf[KEY address account] {
    require g_balanceOf[account] == balance;
}
```


As for the rule, we verify that depositing an amount (`depositAmount`) correctly credits it to the sender's balance in the vault: 


```javascript
rule deposit(env e) {
    uint256 depositAmount;

    require currentContract != e.msg.sender;
    require g_balanceOf[e.msg.sender] == 0;
    
    deposit(e, depositAmount);
    assert g_balanceOf[e.msg.sender] == depositAmount;
}
```


Let's run the verification. Since the deposit function calls an unknown ERC-20 implementation (which will cause havoc), we expect it to fail: 


![image](media/certora-persistent-ghosts/image8.png)


Prover run: [link](https://prover.certora.com/output/541734/a176045767854bb1b2e6cb2f3c4d3a8f?anonymousKey=a9ffaabb713b3ad9bed779846009c2f5f2348556)


### Why the verification failed


The root cause is in the following line of the `SimpleVault20` contract, where it calls an unknown ERC-20 implementation:


```solidity
contract SimpleVault20 {
    ...
    function deposit(uint256 amount) external {
        require(amount > 0, "Zero deposit");
        
        balanceOf[msg.sender] += amount;
        
        // `token` is an unknown ERC-20 implementation
        require(token.transferFrom(msg.sender, address(this), amount), "transferFrom failed");
    }
    ...
}
```


In the image below, we see that the Prover assigned the value `0xa` (or 10) to `depositAmount`, which was then passed via the `Sstore` hook to the ghost mapping variable `g_balanceOf[address]`:


![image](media/certora-persistent-ghosts/image9.png)


Then the ghost was havoc'ed during the call to an external contract because its implementation was unknown:


![image](media/certora-persistent-ghosts/image10.png)


Seeing the havoc error, one might be tempted to add the `persistent` keyword to suppress it. While this makes verification pass, it hides the real issue: we do not know how the external ERC-20 contract is implemented:


![image](media/certora-persistent-ghosts/image11.png)


Prover run: [link](https://prover.certora.com/output/541734/0137cef605f04e39b7409f8928c4d362?anonymousKey=dda1014f4e9469fd09d682fee97569b2fd7dd40f)


### Prover verifies with a wrong assumption


The Prover shows the rule as verified, but this is based on a wrong assumption. By using a `persistent` ghost, the specification assumes the ERC-20 implementation never reverts or modifies the storage of the calling contract. Therefore, it credits the amount `0x``a` (or 10) to `msg.sender` even though the transfer might actually fail: 


![image](media/certora-persistent-ghosts/image12.png)


This demonstrates why using a `persistent` ghost in this case is misleading: it bypasses the uncertainty of external contract behavior, masking potential bugs rather than revealing them.


### **Actual token implementation should be linked to the vault**


The proper solution is NOT to use a persistent ghost. Instead, we must “link” a known ERC-20 implementation into the [scene](https://docs.certora.com/en/latest/docs/user-guide/glossary.html#term-scene) — whether it is the protocol’s own ERC20, WETH, DAI, or any ERC-20 token whose code is available.


Linking instructs the Prover to use the actual contract implementation included in the verification scene for external calls, rather than treating those calls as havoc behavior.


For `SimpleVault20`, the token address is immutable, meaning the ERC-20 to be used is known at the time of deployment and will not change throughout the contract's lifetime. Hence, we can use `link` to “bind” it to an ERC-20 implementation by adding the ERC-20 contract to the scene and configuring it in the spec. This allows the Prover to verify the `SimpleVault20` contract against the actual token implementation. Otherwise, without linking, the Prover cannot accurately verify the vault's interaction with the token, leading to false positives as demonstrated above.


For further details on `link`, see the Certora documentation: [[1](https://docs.certora.com/en/latest/docs/prover/cli/options.html#link), [2](https://docs.certora.com/en/latest/docs/user-guide/multicontract/index.html?utm_source=chatgpt.com#working-with-known-contracts)]


## Summary

- Persistent ghosts survive havoc and reverts, while regular ghosts are havoc'ed during external calls to unknown contracts and reset during reverts.
- External calls to unknown contracts cause havoc because the Prover cannot see the callee's implementation, so it models all possible return values and storage effects.
- A proper use of persistent ghosts is to capture information from CVL hooks — such as `CALL` return codes — that must survive across failing execution paths, like tracking failed low-level ETH transfers.
- Adding the `persistent` keyword as a quick fix to pass failing rules creates a false sense of verification success by ignoring the effects of unknown contract implementations or unresolved calls.
