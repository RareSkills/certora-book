# Testing msg.sender and msg.value in CVL

## **Introduction**

In this chapter, we introduce the `env` variable in CVL, which enables us to make rules for functions that depend on `msg.sender`, `msg.value`, and other global variables in Solidity like `block` and `tx`. 

We will focus on the first two, which can be accessed from the CVL variable `env e` as `e.msg.sender` and `e.msg.value`.

In our previous examples, we ignored them because the functions we verified didn’t rely on environment variables. Hence, we tagged these functions as `envfree` in the methods block. This tells the Prover to treat them as pure logic and disregard environment (global) variables to simplify the analysis.

However, in practice, transactions are heavily dependent on these environment variables (`env`), and here we will explore how to create rules with `msg.sender` and `msg.value`.

## `e.msg.sender` and `e.msg.value` (non-payable)

Consider a simple owner-controlled counter contract, where only the owner is allowed to increment a counter:

```solidity
contract OwnerCounter {
    uint256 public counter;
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function increment() public {
        require(msg.sender == owner, "not owner");
        counter++;
    }
}

```

The function `increment()` has a `require` statement that will cause a revert if the caller is not the owner. Therefore, our rule needs to depend on `msg.sender`. 

To verify the property: *“Only the owner can successfully call `increment()`”*, we write the following CVL rule:

```solidity
methods {
  function owner() external returns (address) envfree;
  function counter() external returns (uint256) envfree;
}

rule increment_onlyOwnerCanCallIncrement(env e) {
    address current = owner();

    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == current, "access control failed";
}
```

In the methods block, functions `owner()` and `counter()` are marked as `envfree` because they are view functions of the storage variables and do not depend on environment variables or `env`.

However, the function `increment()` is environment-dependent (hence the parameter `e`). **Functions do not need to be explicitly declared in the `methods` block if they are assumed to be environment-dependent**. Thus, we don’t include `increment()` in the method’s block.

In the rule block, the `env e` declaration can be placed as a parameter or inside the rule block. Both are valid and are merely code style preferences:

```solidity
rule increment_onlyOwnerCanCallIncrement(env e) {
    ...
}

rule increment_onlyOwnerCanCallIncrement() {
    env e;
    ...
}
```

Now, when we invoke a Solidity function that is environment-dependent, we must include the argument `e`:

```solidity
increment@withrevert(e);
```

Omitting `(e)` would result in a compiler error, forcing you to declare the function as `envfree` in the methods block. However, if you blindly follow this and declare it as `envfree`, violations will occur, and the Prover will indicate that the function is environment-dependent.

Now, finally, let’s move to the assertion:

```solidity
assert !lastReverted <=> e.msg.sender == current, "access control failed";
```

This checks that `increment()` does not revert if and only if `msg.sender == owner`. If the assertion fails, it means the Prover found a counterexample — either the correct owner was blocked from calling `increment()`, or a non-owner was able to call it successfully.

When we run this rule, however, we encounter an unexpected revert case — when `msg.value != 0`. This occurs because `increment()` is a non-payable function, meaning it cannot accept Ether. 

In Solidity, a contract can receive Ether via a function call, but only if the function is explicitly marked as `payable`; otherwise, the transaction will revert. The amount of Ether sent is stored in the global variable `msg.value`, which holds the value in wei (the smallest denomination of Ether).

Since `increment()` is not marked as `payable`, any nonzero `msg.value` causes a revert, as shown in the report below:

![image](media/certora-msgsender-msgvalue/image1.png)

To resolve this, we need to place `e.msg.value == 0` as a precondition.

Then, another unexpected revert occurs when `counter == max_uint256`. Since `max_uint256` is the maximum value the counter can hold, attempting `counter++` will cause an overflow revert (note that we are calling the function with the `withrevert` tag):

![image](media/certora-msgsender-msgvalue/image2.png)

To resolve this, we need to add another precondition `require(counter() < max_uint256)` to prevent the counter from overflowing.

With these two unexpected revert cases identified, both conditions must be included as preconditions. Below is the corrected rule and the Prover report. As expected, the verification passes:

```solidity
rule increment_onlyOwnerCanCallIncrement(env e) {
    address current = owner();

   require e.msg.value == 0;
   require counter() < max_uint256;  

   increment@withrevert(e);

   assert !lastReverted <=> e.msg.sender == current, "access control failed";
}
```

![image](media/certora-msgsender-msgvalue/image3.png)

Prover run: [link](https://prover.certora.com/output/541734/4aac7a40ddba421c89783c6581d11659?anonymousKey=cc8849ab3d456437ba3a64cd5bf315ef3683e260)

### Relaxing Preconditions With `=>` Instead Of `<=>`

The rule we created above says “the function reverts if and only if the caller is not the owner.” Since there are other valid revert conditions, namely `msg.value != 0` and `counter() == max_uint256`, we had to eliminate those possibilities in the precondition for “the function reverts if and only if the caller is not the owner” to be true.

If we wanted to instead state, “if a non-owner calls the function, the transaction reverts” we can use the implication operator without preconditions. Other revert cases (i.e. `msg.value != 0` and the counter being at the maximum) do not cause a violation. We only care that a non-owner calling the function reverts.

Now having said that, here’s the property to formally verify: “if the caller is not the owner, the function must revert.” Below is the rule for that — a simpler one — and as expected, this rule passes:

```solidity
rule increment_notOwnerCannotCallIncrement(env e) {
    address current = owner();

    increment@withrevert(e);
    assert e.msg.sender != current => lastReverted, "access control failed";
}
```

![image](media/certora-msgsender-msgvalue/image4.png)

Prover run: [link](https://prover.certora.com/output/541734/8448bc805955405d81eef50022cb4e01?anonymousKey=c207f4d3ed8ff9c52f6bc6d90eb0d0a9fb0e45a8)

Now, we understand how to write rules for environment-dependent functions. The Prover uses `e.msg.sender` to reason about execution based on the caller and `e.msg.value` to reason about ETH transfers. In the next chapter, we will explore both concepts more extensively.

## Summary

- The `env` variable in CVL allows reasoning about functions that depend on `msg.sender`, `msg.value`, and other global variables.
- Such functions must include `env e`, either as a rule parameter or within the rule block.
- These environment-dependent functions do not need to be declared in the methods block, while `envfree` functions must be explicitly declared and marked as `envfree`.
- If an environment-dependent function is not correctly implemented as such, it will still compile and run on the Prover but will produce violations and warnings.
- If `msg.value` is not properly accounted for, the Prover will generate violations and counterexamples stemming from it.
