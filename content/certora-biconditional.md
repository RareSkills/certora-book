# Biconditional Operator


## Introduction


The biconditional operator enables us to assert if-and-only-if relationships between boolean values. Implication (`P => Q`) states that if condition P is satisfied, then Q is observed. The biconditional adds the reverse requirement: if Q is observed, then condition P must also be satisfied. In other words, `P <=> Q` is equivalent to asserting both `P => Q` and `Q => P`.


_NOTE: The reader is assumed to have already read the chapter "Implication Operator", as it lays the foundation for this chapter._


## `setX()` Function


Let’s start with a basic setter function. It takes a `uint256` input and updates the storage variable `x`, but only if the new value is strictly greater than the current one. This restriction is enforced using a `require` statement, which reverts the transaction if the condition is not met:


```solidity
/// Solidity

uint256 public x;

function setX(uint256 _x) external {
    require(_x > x, "x must be greater than the previous value");
    x = _x;
}
```


To formally verify this function using the implication operator, we can express the relationship between the transaction’s success and the resulting state change as two assert implication statements:


```solidity
rule setX_usingImplication() {
    uint256 _x;
    
    mathint xBefore = x();
    
    setX@withrevert(_x);
    bool success = !lastReverted;
    
    mathint xAfter = x();

    assert success => xAfter > xBefore; // If the call succeeds, then x increased (P => Q)
    assert xAfter > xBefore => success; // If x increased, then the call must have succeeded (Q => P)
}
```


![image](media/certora-biconditional/image1.png)


Prover run: [link](https://prover.certora.com/output/541734/885900ff6e4b4d50995371de86dd206d?anonymousKey=79765145fd63da9363a34e78a345f6c66d04918c)


While logically correct, writing two one-way implications (`P => Q` and `Q => P`) is unnecessarily verbose. 


## Biconditional (`<=>`) in CVL


The intent here is to express a precise rule: “the transaction succeeds if and only if the input value is greater than the current value of `x`.”



This can be written more cleanly as:


```solidity
rule setX_TxSucceeds_iff_XIncreases() {
    uint256 _x;
    
    mathint xBefore = x();
    
    setX@withrevert(_x);
    bool success = !lastReverted;
    
    mathint xAfter = x();
    assert success <=> xAfter > xBefore;
}
```


This single assert biconditional statement is more concise and directly expresses the two-way relationship between the call’s success and the change in state x (increase). In other words, the function only succeeds if `x` increases, and `x` only increases if the call succeeds.


As expected, this rule verifies:


![image](media/certora-biconditional/image2.png)


Prover run: [link](https://prover.certora.com/output/541734/0cb9f701278841e1884d502397f09fd9?anonymousKey=be23c334e574e147a5a514577ee9805f49aae739)


Implicitly, we can also reason that: if the call failed, then `x` did not increase:


```solidity
assert !success <=> !(xAfter > xBefore);
```


or 


```solidity
assert !success <=> xAfter <= xBefore;
```


This is already implied by the biconditional and does not need to be written separately.


## Formally Verifying Solmate’s `mulDivUp()`


In the previous chapter (Implication Operator), we discussed the behavior of Solmate’s `mulDivUp()` function, its overflow checks, and rounding logic. Here's a quick recap:

- It reverts if `denominator == 0` (division by zero) or if `x * y > MAX_UINT256` (overflow).
- `mulDivUp(x, y, denominator)` multiplies `x` and `y`, divides the result by `denominator`, and rounds up if there’s a remainder; otherwise, it returns the exact quotient.

Now let’s formally verify these behaviors using the biconditional operator.


### Revert Behavior


Let’s start with the revert behavior. We begin by writing the simplest revert condition, `denominator == 0`, then work our way up to a complete rule.


This will fail, as we know, because it doesn’t capture all revert scenarios, but it serves as a good starting point:


```solidity
rule mulDivUp_denominatorIsZero_revert() {
    uint256 x;
    uint256 y;
    uint256 denominator; 

    mulDivUp@withrevert(x, y, denominator);
    assert denominator == 0 <=> lastReverted;
}
```



The rule fails due to a counterexample where the product of `x` and `y` exceeds `max_uint256`. We need to include that as an additional revert condition:


![image](media/certora-biconditional/image3.png)


Here’s the corrected CVL rule:


```solidity
rule mulDivUp_allConditions_revert() {
    uint256 x;
    uint256 y;
    uint256 denominator; 

    mulDivUp@withrevert(x, y, denominator);
    assert denominator == 0 || x * y > max_uint256 <=> lastReverted;
}
```


![image](media/certora-biconditional/image4.png)


Prover run: [link](https://prover.certora.com/output/541734/bbef2b36f358411c92b8eda3b7b364ed?anonymousKey=5fb7a527daee3b46cbbdc2dfb6f888fda2e86c29)


Note that the biconditional is reversible (unlike implication), so we can rewrite the assertion like this:


```solidity
rule mulDivUp_allConditions_revert() {
    uint256 x;
    uint256 y;
    uint256 denominator; 

    mulDivUp@withrevert(x, y, denominator);
    assert lastReverted <=> denominator == 0 || x * y > max_uint256;
}
```


![image](media/certora-biconditional/image5.png)


Prover run: [link](https://prover.certora.com/output/541734/695777bc9bf44d438d421b0fd1f7a55b?anonymousKey=5a19eaa0fabd2cbd0be353fc1272e40b258a538b)


Also, since a biconditional is a combined two-way implication (`P => Q` and `Q => P`), the two are logically bound together; therefore, expressions reasoned through `<=>` are inseparable.


### Rounding Behavior


The rule below specifies the rounding behavior of the `mulDivUp()` function. It asserts that the function returns a rounded-up value if and only if a remainder is present. The biconditional operator guarantees that a remainder is the sole condition that triggers rounding up and that no other condition can affect the return value:


```solidity
rule mulDivUp_roundUp() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mathint result = mulDivUp(x, y, denominator);
    assert ((x * y) % denominator > 0) <=> (result == (x * y / denominator) + 1);
}
```


![image](media/certora-biconditional/image6.png)


Prover run: [link](https://prover.certora.com/output/541734/faf390806e124743a098eaaeb2512339?anonymousKey=5e3a9aa509f01cdb5a8e9351d0165296e170521d)


The rule below covers the “other” case. It asserts that the function returns the exact quotient if and only if there is no remainder. The biconditional operator guarantees that the only reason for returning an unrounded result is the absence of a remainder:



```solidity
rule mulDivUp_noRoundUp() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mathint result = mulDivUp(x, y, denominator);
    assert ((x * y) % denominator == 0) <=> (result == x * y / denominator);
}
```


![image](media/certora-biconditional/image7.png)


Prover run: [link](https://prover.certora.com/output/541734/7cb0979fee584275a4a22d8c28610d3d?anonymousKey=8594f4c1dd1caddf1589e9426af45b391e89a608)


## When To Use Biconditional Over Implication


### Exclude Uncertainty


The rule above, as we saw in the previous chapter, can be written using `=>` because the property being verified happens to have only one condition that leads to the outcome. In such cases, `=>` can be replaced with `<=>`. This replacement is preferable because `<=>`  states that nothing other than the specified condition can cause the outcome. The implication operator does not provide this guarantee, so in cases like these, it is better to use the biconditional to make that guarantee explicit. 


In the previous example (max function), we accounted for all the revert conditions because we were verifying the function's correctness as a whole. However, when dealing with a function that calls several internal functions, each potentially causing different revert conditions, using an implication can be more practical. This approach focuses only on the necessary condition(s) for reverting, without exhaustively dealing with every possible failure scenario.


### Verify Mutual Dependency


Mutual dependency occurs when two variables are interdependent, meaning a change in one triggers a corresponding change in the other, and vice versa.


Take an AMM WETH/USDC pool as an example: During swaps, the WETH balance increases if and only if the USDC balance decreases, and vice versa. This creates a two-way dependency where the balance of one token must adjust in response to changes in the other, indicating that neither balance can change independently.


## Summary

- The biconditional (`<=>`) explicitly guarantees the implication in both directions. If condition P is satisfied, Q holds, and if Q holds, condition P must also be satisfied.
- The implication operator (`=>`) works when condition(s) lead to an outcome, but it doesn’t guarantee that the outcome can only be caused by that condition.
- Using biconditional can be impractical when there are a lot of potential conditions that result in the outcome.

## Exercises for the reader

1. Formally verify [Solady min](https://github.com/Vectorized/solady/blob/dcdfab80f4e6cb9ac35c91610b2a2ec42689ec79/src/utils/FixedPointMathLib.sol#L1100) always returns the min.
2. Formally verify [Solady zeroFloorSub](https://github.com/Vectorized/solady/blob/dcdfab80f4e6cb9ac35c91610b2a2ec42689ec79/src/utils/FixedPointMathLib.sol#L657) returns `max(0, x - y)`. In other words, if `x` is less than `y`, it returns `0`. Otherwise, it returns `x - y`.
