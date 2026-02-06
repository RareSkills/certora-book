# Implication Operator


## Introduction


The implication operator is frequently used as a substitute for the `if` statement since it is cleaner. 


Consider the following example: a function `mod(x, y)` that takes two unsigned integers as parameters and returns `x` modulo `y`. The modulo operation computes the remainder of dividing `x` by `y`.


The Solidity implementation of this function is as follows:



```solidity
/// Solidity

function mod(uint256 x, uint256 y) external pure returns (uint256) {
    return x % y;
}
```



Suppose we want to formally verify the property: “if `x < y`, then the modulo must equal `x`." Using an `if` statement to express this in CVL causes a compilation error, since the final statement in a rule must be either an `assert` or a `satisfy`:



```solidity
/// this rule will not compile
rule mod_ifXLessThanY_resultIsX_usingIf() {
    uint256 x;
    uint256 y;

    mathint result = mod(x, y);

    if (x < y) {
        assert result == x;
    }
}
```


```plain text
ERROR
: Failed to run Certora Prover locally. Please check the errors below for problems in the specifications (.spec files) or the prover_args defined in the .conf file.

CRITICAL
: [main] ERROR ALWAYS - Found errors in certora/specs/ImplicationOperator.spec:

CRITICAL
: [main] ERROR ALWAYS - Error in spec file (ImplicationOperator.spec:15:5): last statement of the rule mod_ifXLessThanY_resultIsX_usingIf is not an assert or satisfy command (but must be)
```


To resolve this, one workaround is to add a trivial assertion at the end:


```solidity
if (P) { 
	assert Q; 
}
assert true;
```


```solidity
rule mod_ifXLessThanY_resultIsX_usingIf() {
    uint256 x;
    uint256 y;

    mathint result = mod(x, y);

    if (x < y) {
    assert result == x;
    }
    assert true; // trivially TRUE
}
```


However, this solution introduces a pointless assertion that clutters the specification. Another workaround is to use an `if-else` statement, where the else branch executes meaningful assertions:


```solidity
if (P) { 
	assert Q; 
} else {
	assert (something_else);
}
```


```solidity
rule mod_ifXLessThanY_resultIsX_usingIfElse() {
    uint256 x;
    uint256 y;

    mathint result = mod(x, y);

    if (x < y) {
    assert result == x;
    } else {
    assert result != x;
    }
}
```


However, we weren’t originally interested in asserting that: “if `x >= y`, then `result != x`.” In many cases, like the one shown above, we are not interested in creating an assertion for every possible branch.


## Implication (`=>`) In CVL  


To handle this elegantly, we use the implication operator (`=>`). With implication, we can succinctly express conditions while avoiding the control flow complexity of `if-else` statements.

Thus, instead of writing `if (P) assert Q` (which doesn’t compile), we can equivalently use `assert P => Q`:


```solidity
rule mod_ifXLessThanY_thenResultIsX() {
    uint256 x;
    uint256 y;

    mathint result = mod(x, y);
    assert x < y => result == x;
}
```


![image](media/certora-implication/image-1d109cb3.png)


Prove run: [link](https://prover.certora.com/output/541734/8902287c8cad40108315113981707b7c?anonymousKey=1da2ab01a326687abc7ecfdb24cb6c717d3d06a9)



The form `P => Q` reads as “If P, then Q,” or simply, “P implies Q.” Note that `Q => P` is not necessarily equivalent, as we'll see in the next section.


### P Implies Q `P => Q`


Let's take another example: [Solady's gas-optimized max function](https://github.com/Vectorized/solady/blob/5dd6f93498b8ebdc9d72194cfa680a90b738e1ad/src/utils/FixedPointMathLib.sol#L1091-L1097). As discussed in the previous chapter, it returns the greater of two inputs, `x` and `y`, defaulting to `x` if they are equal.



Below is the implementation, and as in the previous chapter, we’ve changed the visibility from `internal` to `external` to avoid the need for harnessing (which is beyond the scope of this chapter):


```solidity
/// Solidity

function max(uint256 x, uint256 y) external pure returns (uint256 z) {
    assembly {
        z := xor(x, mul(xor(x, y), gt(y, x)))
    }
}
```



Let’s now formally verify the following property: “if `x` is greater than `y`, the return value of `max(x, y)` must be `x`.” We can express this in CVL as:


```solidity
rule max_ifXGreaterThanY_resultX() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert x > y => result == x;
}
```



In this rule, the condition (`x > y`) and the expected outcome (`result == x`) form the implication `P => Q`. As expected, the property is VERIFIED.


![image](media/certora-implication/image2.png)


Prover run: [link](https://prover.certora.com/output/541734/48ff7cf0e7fe43b8873cdc43aa568399?anonymousKey=004dea34de11b5dbe06cbccc29f344f6da65a720)


_Note: The verification of the_ _`max`_ _function is not yet complete. We will cover that in the next section._


### _`P => Q`_ _Is Not Equivalent To_ _`Q => P`_ 


Now let’s consider the reverse implication: “if the return value of `max(x, y)` is `x`, then `x > y`” or `result == x => x > y`. This may seem valid at first; however, when `x == y`, the function still returns `x`. This contradicts the assertion that `x > y`, making the implication false in that case.


Here's the rule that fails: 


```solidity
rule max_ifResultIsX_thenXGreaterThanY() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert result == x => x > y;
}
```


![image](media/certora-implication/image3.png)


### _Counterexample_


When a rule is violated, the Prover shows a counterexample. In this case, `result == x` not only happens when `x > y` — it also occurs when `x == y`. The counterexample specifically shows x = 3 and y = 3:


![image](media/certora-implication/image4.png)



To fix this, we need to revise the condition to include all cases where `result == x` is valid: both when `x > y` and when `x == y`. 


Here’s the revised rule, where || is the OR operator: 


```solidity
rule max_ifResultIsX_thenXGreaterOrEqualToY() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert result == x => x > y || x == y;
}
```


or, more simply:


```solidity
rule max_ifResultIsX_thenXGreaterOrEqualToY_simplified() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert result == x => x >= y;
}
```


And now it’s VERIFIED:


![image](media/certora-implication/image5.png)


Prover run: [link](https://prover.certora.com/output/541734/6acc70804ec94d61be2b7631194c36ce?anonymousKey=73deae017a3c9e0a34fd2b9de94c5539ee0c446f)



Now, to complete the verification, we will also verify the property that `max(x, y)` returns `y` when `y` is greater than or equal to `x`. Here’s the rule:


```solidity
rule max_ifResultIsY_thenYGreaterOrEqualToX() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert result == y => y >= x;
}
```


As expected, it is VERIFIED:


![image](media/certora-implication/image6.png)


Prover run: [link](https://prover.certora.com/output/541734/bd2c1d2601ee4451a3e638637558699c?anonymousKey=36469d8cb6a396f8a329a914a0bf008b12f79735)


### `P => Q` Is Equivalent to `!Q => !P` 


Rewriting an implication `P => Q` into its contrapositive form `!Q => !P` suggests an alternative way to think about a specification.


Let’s revisit our previous rule with the assertion: `x > y => result == x`:


```solidity
rule max_ifXGreaterThanY_resultX() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert x > y => result == x;
}
```


In its original form, this rule is concerned with what happens when `x` is greater than `y`, which is that the return value of `max(x, y)` is `x`.


In the contrapositive form, `assert !(result == x) => !(x > y)`, we are instead concerned with what happens if the return value of `max(x, y)` is not `x`. In that case, we expect that `x` is not greater than `y` (or simply, `x <= y`):


```solidity
rule max_ifResultNotX_thenXNotGreaterThanY() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert result != x => x <= y;
}
```


![image](media/certora-implication/image7.png)


Prover run: [link](https://prover.certora.com/output/541734/1cdf95d2bd4b42cc8dd11b4d71097cac?anonymousKey=cdfec47ef18bf1c19de3a1aec1d70f091eb39cdf)


Both forms express the same logical truth, but from different perspectives. 


To better understand this, let’s use a common analogy:

- _“If it rained, the ground is wet,”_ or
- `hasRained => isGroundWet`

Its contrapositive is:

- _“If the ground is not wet, then it definitely didn’t rain,”_ or
- `!isGroundWet => !hasRained`

However, the fact that it has not rained does not guarantee that the ground is not wet. Someone could have poured water on it. Therefore, `!hasRained => !isGroundWet` is not logically valid. 


## Vacuous Truth and Tautology In Implication


Now that we understand how implication works, we must also be cautious when writing these statements. We might unknowingly write implications with false assurances, a topic we'll explore next. 


### _When P_ _is always FALSE,_ _`P => Q`_ _is TRUE regardless of Q (__vacuous__)_


An implication becomes vacuously true when the condition (`P`) is unreachable (always false), meaning there’s no possible state where `P` holds. In such cases, the entire statement `P => Q` is automatically considered true, regardless of what the outcome (`Q`) is.


This happens because if `P` can never be true, then there is no instance in which the implication can be violated. Hence, no counterexample exists.


Let’s use this example: “if `x < 0`, then `result == y`.” Since `x` is a `uint256`, it can never be negative. That makes `x < 0` an unreachable state. No matter what we assert on the right-hand side (`Q`), the implication will always verify.


Here’s the CVL vacuous rule:


```solidity
rule max_unreachableCondition_vacuouslyTrue() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert x < 0 => result == y;
}
```


This rule is VERIFIED, but not because `x < 0` leads to `result == y`. It verifies only because `x < 0` is unreachable in any valid execution path, making the implication vacuously true. It confirms that no counterexample exists because the condition can never happen:


![image](media/certora-implication/image8.png)


Prover run: [link](https://prover.certora.com/output/541734/41f50a5158df4a95b80a1fc2e1b931f7?anonymousKey=60b2b67c3037745d6b9867064ff89553716122a0)


Since the condition (`P`) can never occur, the outcome (`Q`) can be anything — even something nonsensical like `1 == 2`, as shown below:


```solidity
rule max_unreachableCondition_vacuouslyTrueObvious() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert x < 0 => 1 == 2;
}
```


![image](media/certora-implication/image9.png)


Prover run: [link](https://prover.certora.com/output/541734/167bc794dda6409bab8b592d23cd7147?anonymousKey=db8f8821571de00cfba190b267a41a1e0d90a802)


### _When_ _Q is always TRUE__,_ _`P => Q`_ _is TRUE regardless of P (tautology)_


An implication becomes tautologically true when the outcome (`Q`) is always true, regardless of the condition (`P`). In such cases, the entire statement `P => Q` is automatically considered true, even if the condition is irrelevant or misleading.


This happens because if `Q` is always true, the outcome is guaranteed; hence, the implication can never be false.


Let’s use this example: “if `x > y`, then `result >= 0`.” 


At first glance, this seems to imply that `x > y` guarantees a non-negative result. But `result` is a `uint256`, which is always greater than or equal to zero by type. So the expression `result >= 0` is always true.


Here’s the CVL tautological rule:


```solidity
rule max_outcomeIsAlwaysTrue_tautology() {
    uint256 x;
    uint256 y;

    mathint result = max(x, y);
    assert x > y => result >= 0;
}
```


This rule is VERIFIED, not because `x > y` leads to `result >= 0`, but because `result >= 0` is always true by type, making the implication tautologically true:


![image](media/certora-implication/image10.png)


Prover run: [link](https://prover.certora.com/output/541734/63d1f7b86c6a4cfe9582db0b8a69bc80?anonymousKey=db8d0fd4e7df6720c25eda6e5c4ce7148adae9de)


## Formally Verifying Solmate’s `mulDivUp()`


Now that we know the fundamentals, we’re formally verifying Solmate’s [`mulDivUp()`](https://github.com/transmissions11/solmate/blob/c93f7716c9909175d45f6ef80a34a650e2d24e56/src/utils/FixedPointMathLib.sol#L53-L69) using the implication operator. As the name suggests, this function multiplies two unsigned integers, divides the result by another unsigned integer, and rounds up if there is a remainder. If there is no remainder, it simply returns the quotient.



The properties we’ll formally verify fall into two categories: rounding behavior and revert behavior. But before we do that, we will make a small adjustment to the original `mulDivUp()` code.


The Solmate’s [`mulDivUp()`](https://github.com/transmissions11/solmate/blob/c93f7716c9909175d45f6ef80a34a650e2d24e56/src/utils/FixedPointMathLib.sol#L53-L69) function has **internal** visibility, and we will change it to **external** for convenience. Otherwise, setting up a harness contract would be necessary, which is beyond the scope of this chapter and may introduce unnecessary mental overhead at this point.


Here’s the `mulDivUp()` function: 


```solidity
/// Solidity

uint256 internal constant MAX_UINT256 = 2**256 - 1;

function mulDivUp(
    uint256 x,
    uint256 y,
    uint256 denominator
) external pure returns (uint256 z) {
    assembly {
        // Equivalent to require(denominator != 0 && (y == 0 || x <= type(uint256).max / y))
        if iszero(mul(denominator, iszero(mul(y, gt(x, div(MAX_UINT256, y)))))) {
            revert(0, 0)
        }

        // If x * y modulo the denominator is strictly greater than 0,
        // 1 is added to round up the division of x * y by the denominator.
        z := add(gt(mod(mul(x, y), denominator), 0), div(mul(x, y), denominator))
    }
}
```


The assembly code explanation is beyond the scope of this chapter. However, the comments help us understand what the code is doing. 


These lines below,


```solidity
// Equivalent to require(denominator != 0 && (y == 0 || x <= type(uint256).max / y))
if iszero(mul(denominator, iszero(mul(y, gt(x, div(MAX_UINT256, y)))))) {
    revert(0, 0)
}
```


as the comment suggests, represent a `require` statement that enforces the following conditions:

- `denominator != 0`, which prevents division by zero, as division by zero is not allowed.
- `y == 0 || x <= type(uint256).max / y`, meaning:
    - If `y == 0`, the condition holds because `x * y` naturally evaluates to `0`. In this case, an overflow check is unnecessary.
    - Otherwise, if `y != 0`, `x` must be less than or equal to `type(uint256).max / y` making sure that `x * y` does not exceed `type(uint256).max`, preventing overflow.

Now we conclude that the properties to be formally verified are the following:

- If `denominator == 0`, then function reverts.
- If  `x * y > MAX_UINT256`, then function reverts.

 


In the next line, 


```solidity
// If x * y modulo the denominator is strictly greater than 0,
// 1 is added to round up the division of x * y by the denominator.
z := add(gt(mod(mul(x, y), denominator), 0), div(mul(x, y), denominator))
```


as the comment suggests, the function handles rounding as follows:

- If `x * y % denominator > 0`, `1` is added to `x * y / denominator` to compensate for Solidity’s truncation in integer division. Since Solidity rounds down, adding `1` effectively rounds up the value.
- If `x * y % denominator == 0`, no rounding adjustment is needed, and the result is returned as is.

Now we conclude that the properties to be formally verified are the following:

- If `x * y % denominator > 0`, then the result is `x * y / denominator + 1`.
- If `x * y % denominator == 0`, then the result is `x * y / denominator`.

Since we now understand how the function works and what properties to formally verify, we can proceed with writing the rules. Let’s start with the rules for rounding behavior.


### _Rounding Behavior_


Below is the CVL rule specification for the rounding behavior property.


The assertion of the rule `mulDivUp_roundUp()` states that when the product of `x` and `y` is divided by the `denominator` and has a remainder, the result is incremented by 1.


Similarly, the assertion in the rule `mulDivUp_noRoundUp()` states that when the product of `x` and `y` is divided by the `denominator` and has no remainder, the result is returned as is.


As expected, these rules are VERIFIED:


```solidity
rule mulDivUp_roundUp() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mathint result = mulDivUp(x, y, denominator);
    assert ((x * y) % denominator > 0) => (result == (x * y / denominator) + 1); 
}

rule mulDivUp_noRoundUp() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mathint result = mulDivUp(x, y, denominator);
    assert ((x * y) % denominator == 0) => (result == x * y / denominator);
}
```


![image](media/certora-implication/image11.png)


![image](media/certora-implication/image12.png)


Prover run: [link](https://prover.certora.com/output/541734/3550e477bbfe4d65805eb047766e1d8d?anonymousKey=a725af8b6f98788cb5c25ae31fef58dea9adb8fb)


Now, let’s move on to the next category of properties for this function: revert behavior.


### _Revert Behavior_


As previously discussed, there are two cases where `mulDivUp()` reverts:

- If `denominator == 0`
- If `x * y > MAX_UINT256`

When formally verifying reverts, we must include the `@withrevert` tag when invoking functions. By default, the Prover executes only non-reverting paths, meaning all arguments passed to `mulDivUp(x, y, denominator)` are non-reverting. However, adding `@withrevert` to the function instructs the Prover to also consider arguments that cause a revert. This allows us to account for reverting cases in our implication statements.


Below is the CVL rule specification:


```solidity
rule mulDivUp_ifDenominatorIsZero_revert() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mulDivUp@withrevert(x, y, denominator);
    bool isReverted = lastReverted;

    assert (denominator == 0) => isReverted;
}

rule mulDivUp_ifXYExceedsMaxUint_revert() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mulDivUp@withrevert(x, y, denominator);
    bool isReverted = lastReverted;

    assert (x * y > max_uint256) => isReverted;
}
```


As expected, these rules are VERIFIED: 



![image](media/certora-implication/image13.png)


![image](media/certora-implication/image14.png)


Prover run: [link](https://prover.certora.com/output/541734/4ee0cdc8d1864c5a8824f71a2705f0fe?anonymousKey=a1005ad87dc3e0329570fd0e5014144297ff8a25)



There’s an even more interesting approach to this. So far, we’ve used the `P => Q` construct, which means (`P`) condition result in (`Q`) outcome. But if we reverse it to `Q => P`, it means (`Q`) outcome results from (`P`) conditions; therefore, we must account for all possible cases of `P` leading to `Q`.



Below is the implementation where, in the first rule, we intentionally omit one revert case, while in the second rule, we account for all cases. As a result, the first rule results in a VIOLATION, while the second rule is VERIFIED:


```solidity
rule mulDivUp_allRevertCases_violated() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mulDivUp@withrevert(x, y, denominator);
    bool isReverted = lastReverted;

    assert isReverted => (denominator == 0);
}

rule mulDivUp_allRevertCases() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mulDivUp@withrevert(x, y, denominator);
    bool isReverted = lastReverted;

    assert isReverted => (denominator == 0 || x * y > max_uint256);
}
```


![image](media/certora-implication/image15.png)


Prover run: [link](https://prover.certora.com/output/541734/4a40d955d3bb42e7b1876ee014517b29?anonymousKey=1cb1075873f3569ccf7881ce001ff45079215e71)


In the next sub-section, we'll formally verify non-reverts, an optional step since we've covered all revert cases. However, we'll still do it as it may have future use cases.


### _Non-Revert Behavior (Optional)_


If we can formally verify reverting paths, we can also verify non-reverting ones. To do so, we need to reframe the statements as:

- `denominator != 0`
- `x * y <= MAX_UINT256`

and combine them as shown in the CVL rule implementation below: 


```solidity
rule mulDivUp_noRevert() {
    uint256 x;
    uint256 y;
    uint256 denominator;

    mulDivUp@withrevert(x, y, denominator);
    bool isReverted = lastReverted;

    assert ((denominator != 0) && (x * y <= max_uint256)) => !isReverted;
}
```


As expected, the rule is VERIFIED:


![image](media/certora-implication/image16.png)


Prover run: [link](https://prover.certora.com/output/541734/794cbdf0f88c49ea904db79d71a7340b?anonymousKey=121409b38f4c0e66c8a05a81d048837ec966d91d)


## Summary

- An implication statement consists of a condition `P`, an outcome `Q`, and the implication operator (`=>`).
- The implication operator defines a one-way conditional relationship, meaning `P => Q` does not imply `Q => P`.
- `P => Q` and its contrapositive `!Q => !P` are logically equivalent but offer different perspectives, shifting focus from when the condition holds to when it does not.
- For an implication to be meaningful (neither vacuous nor tautological), `Q` (the outcome) must depend on `P` (the condition) being true.
- A vacuous truth occurs when `P` is unreachable (always false), making `P => Q` true regardless of `Q` due to the lack of counterexamples.
- A tautology occurs when `Q` is always true, making `P => Q` true regardless of `P`.
- Vacuous truths and tautologies create misleading assurance in formal verification.

## Exercises for the reader

1. Formally verify [Solady min](https://github.com/Vectorized/solady/blob/dcdfab80f4e6cb9ac35c91610b2a2ec42689ec79/src/utils/FixedPointMathLib.sol#L1100) always returns the min.
2. Formally verify [Solady zeroFloorSub](https://github.com/Vectorized/solady/blob/dcdfab80f4e6cb9ac35c91610b2a2ec42689ec79/src/utils/FixedPointMathLib.sol#L657) returns `max(0, x - y)`. In other words, if `x` is less than `y`, it returns `0`. Otherwise, it returns `x - y`.
