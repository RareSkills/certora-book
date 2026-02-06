# Formally Verifying Address Balance


In the previous chapter, we covered how to reason about environment-dependent functions in CVL by focusing on `msg.sender` in non-payable contexts. In those examples, access control depended on the caller’s address. We also explained how to handle reverts caused by unexpected non-zero `msg.value`.


In this chapter we will cover how to:

- write rules for functions that expect a non-zero payment (i.e., functions marked as `payable`)
- how to make assertions about account balances
- how to get the balance of the current contract, akin to `address(this).balance` in Solidity

## `e.msg.sender` and `e.msg.value` (payable)


Consider a whitelisting contract where users register by calling the `register()` function, which is marked as `payable`, and send at least 0.05 ETH. Once registered, the caller’s address (`msg.sender`) is marked as `whitelisted` by setting its value to true in the `whitelisted` mapping:



```solidity
/// Solidity

contract PayableWhitelist {
    mapping(address => bool) public whitelisted;

    function register() external payable {
        require(msg.value >= 0.05 ether, "whitelist fee is 0.05 eth");
        require(!whitelisted[msg.sender], "already whitelisted");

        whitelisted[msg.sender] = true;
    }

    function isWhitelisted(address _account) external view returns (bool) {
        return whitelisted[_account];
    }
}
```


We will formally verify the property: “the caller can successfully call `register()` if and only if they pay at least 0.05 ETH”. To verify this, we write the following CVL rule:


 


```solidity
rule register_payEthToWhitelist(env e) {
    require !isWhitelisted(e.msg.sender);
    register@withrevert(e);
    assert !lastReverted <=> e.msg.value >= 5 * 10^16;
}
```


_Note: Exponentiation in CVL uses_ _`^`__, while Solidity uses_ _`**`__._


As we know, since we are using the biconditional operator, we have to rule out other revert cases via the precondition. We add `require(!isWhitelisted(e.msg.sender))` as a precondition to rule out the case where `msg.sender` is not initially whitelisted.


Now, for the biconditional assertion:


`assert !lastReverted <=> e.msg.value >= 5 * 10^16`.


Since we ruled out the case where the sender is whitelisted (`require !isWhitelisted(e.msg.sender)`), we expect this to be VERIFIED, meaning the function will not revert only if `e.msg.value >= 5 * 10^16`.


However, as is often the case in formal verification, a sneaky counterexample emerges, causing a violation. It’s when the sender's ETH balance is less than the intended amount to be sent or insufficient balance:


![image](media/cerotra-account-balances/image1.png)


This can be fixed by adding `require(nativeBalances[e.msg.sender] >= e.msg.value)` to the precondition, ruling out insufficient balance as a cause of revert.


### nativeBalances in CVL


`nativeBalances[address]` is a built-in function in CVL that retrieves the current ETH balance of a given address. Below, we require that `nativeBalances` for the sender is at least as large as the `msg.value`:


```solidity
rule register_payEthToWhitelist(env e) {
    require !isWhitelisted(e.msg.sender);
    require nativeBalances[e.msg.sender] >= e.msg.value;

    register@withrevert(e);

    assert !lastReverted <=> e.msg.value >= 5 * 10^16;
}
```


Now, the rule is VERIFIED:


![image](media/cerotra-account-balances/image2.png)


Prover run: [link](https://prover.certora.com/output/541734/12df2da7107e4af68548359d5b63b47e?anonymousKey=a62d2c52a31b37be6b2279e3bd8edbd807e4bcb5)


### Verifying Whitelist Status  


Now that we have verified that “the caller can successfully call `register()` if and only if they pay at least 0.05 ETH”, let’s improve the rule scope by also asserting that the caller is `isWhitelisted` after a successful call.


Here’s the updated property to formally verify: “the transaction succeeds if and only if the caller sends at least 0.05 ETH and becomes whitelisted.” To achieve this, we can reuse the previous rule and add an additional assertion: `isWhitelisted`.


Here’s the updated CVL rule:


```solidity
rule register_payEthToWhitelist_modified(env e) {
    require !isWhitelisted(e.msg.sender);
    require nativeBalances[e.msg.sender] >= e.msg.value;

    register@withrevert(e);

    assert !lastReverted <=> e.msg.value >= 5 * 10^16 && isWhitelisted(e.msg.sender);
}
```


The rule above means that the transaction does not revert (`!lastReverted`) if and only if the sender sends at least 0.05 ETH (`e.msg.value >= 5 * 10^16`) and becomes whitelisted (`isWhitelisted(e.msg.sender)`).


In other words, if `!lastReverted` is true (the transaction succeeded), both conditions — sufficient ETH and whitelisting — must hold.


Here’s the Prover report and it is VERIFIED:


![image](media/cerotra-account-balances/image3.png)


Prover run: [link](https://prover.certora.com/output/541734/03bf5b4af2b048cfb7163b631c4c3220?anonymousKey=dc3b03095a1815375c0e14b8f177e912deac2f52)


### Relaxing Preconditions With `=>` Instead Of `<=>`


Another approach is to use the implication operator. If your goal is simply to check that the function reverts when an incorrect `msg.value` is sent, the following rule is sufficient. This rule doesn’t preclude other revert causes, such as the sender not having enough balance. It only says if less than 0.05 ether is sent, then it _definitely_ reverts.



```solidity
rule register_payEthToWhitelist_implication(env e) {
    register@withrevert(e);
    assert e.msg.value < 5 * 10^16 => lastReverted;
}
```


If we use this rule, we see that it is VERIFIED:


![image](media/cerotra-account-balances/image4.png)


Prover run: [link](https://prover.certora.com/output/541734/121a88b21fcd4a5e9b8bf32c1cd17c4c?anonymousKey=5a254c0c12d71d7100bf133a823b4e2d389ddc1f)


### **Verifying ETH Balance Transfers** **And The Self-Transfer Corner Case**


Suppose we want to formally verify balance updates where `msg.sender`'s balance decreases by `msg.value` and the contract’s balance increases by the same amount.


Below is the CVL rule, which will be explained in the next section:


```solidity
rule register_nativeBalances(env e) {
    mathint balanceBefore = nativeBalances[currentContract];
    register(e);

    mathint balanceAfter = nativeBalances[currentContract];
    assert balanceAfter == balanceBefore + e.msg.value;
}
```


### The currentContract


In the rule above, we retrieve the prior ETH balance (`balanceBefore`) and the final balance (`balanceAfter`) of the `currentContract`. `currentContract` is a built-in variable that refers to the contract being verified. Passing this variable to `nativeBalances[currentContract]` retrieves the balance of the said contract. In between these retrieval calls, the `register()` function is invoked, so we expect the balance to increase by an amount equal to `msg.value`.


An unexpected counterexample will cause this rule to fail. The Prover will generate a counterexample where `msg.sender == currentContract`. The Prover explores all possible scenarios, including cases where the contract calls itself.

This indicates that the contract sent ETH to itself rather than receiving new ETH from an external sender. As a result, the balance did not actually increase as expected (since sending ETH to itself does not change the total balance), causing the assertion `balanceAfter == balanceBefore + e.msg.value` to fail.


![image](media/cerotra-account-balances/image5.png)


To resolve the issue, since `msg.sender` should not be the contract itself (`currentContract`), we need to filter out self-transfers when verifying ETH balance changes by adding a precondition: `require(e.msg.sender != currentContract)`.  


However, there is a caveat: in real projects, make sure the contract cannot call its own functions, as certain cases may still lead to self-calls. If not properly checked, this could hide a bug.


Here’s the corrected rule and the Prover report. As expected, this is VERIFIED:


```javascript
rule register_nativeBalances(env e) {
    mathint balanceBefore = nativeBalances[currentContract];

    require e.msg.sender != currentContract;
    register(e);

    mathint balanceAfter = nativeBalances[currentContract];
    assert balanceAfter == balanceBefore + e.msg.value;
}
```


![image](media/cerotra-account-balances/image6.png)


Prover run: [link](https://prover.certora.com/output/541734/d5e02935beb74700a9ad52445efab7f3?anonymousKey=926ed9ab42e12e755bd511d74ae285b36de5ae57)


### Verifying All Revert Cases Using `<=>` and `||`


Suppose we want to verify that every possible reason for reverting is explicitly accounted for. We can express the assertion as a biconditional (`<=>`) and disjunctively (using ORs `||`) list all revert cases. 


The rule below states that the only possible reasons for reverting are the following:

- the sender is already whitelisted
- the sender’s balance is insufficient
- the sender didn’t transfer enough funds

If `register()` reverts for any other reason, we would get an assert violation. 


Here’s the rule:


 


```solidity
rule register_allRevertCases(env e) {
    bool wasWhitelisted = isWhitelisted(e.msg.sender);
    mathint balanceBefore = nativeBalances[e.msg.sender];

    register@withrevert(e);

    assert lastReverted <=> (
        wasWhitelisted ||              // sender already whitelisted
        balanceBefore < e.msg.value || // sender's balance insufficient
        e.msg.value < 5 * 10^16        // sender's transfer amount insufficient 
    );
}
```



As expected, the rule is VERIFIED:


![image](media/cerotra-account-balances/image7.png)


Prover run: [link](https://prover.certora.com/output/541734/9126e4a5e6a9475989fe71a6cdff0e3a?anonymousKey=193309b12710b0aaef6003063aa9ae48a101a64d)


## Summary

- In CVL, the environment variable `e.msg.value` is used to express and verify whether a payable function should succeed or revert, typically by asserting a minimum or exact required amount.
- `nativeBalances[address]` is a built-in function that retrieves the current ETH balance of a given address and can be used to verify transfers by checking for balance changes.
- `currentContract` is a built-in variable that refers to the contract being verified.
- Self-calls are included in test cases by the Prover, so in our example, we rule them out by adding `require(e.msg.sender != currentContract)` to avoid false counterexamples; be cautious, as self-calls can be plausible and may reveal real bugs in real-world contracts.
