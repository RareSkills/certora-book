# Formally Verifying Nonces.Sol in OpenZeppelin


Nonces, which stands for "number used once" are used in digital signature schemes to prevent replay attacks. For the purposes of this article, we assume the reader is already familiar with what nonces are and how they are used.


The [OpenZeppelin nonces.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Nonces.sol) library tracks the signers’ nonces by using a counter that increments for each address. Here is the entire library (with the comments removed for brevity):


```solidity
abstract contract Nonces {

    error InvalidAccountNonce(address account, uint256 currentNonce);

    mapping(address account => uint256) private _nonces;

    function nonces(address owner) public view virtual returns (uint256) {
        return _nonces[owner];
    }

    function _useNonce(address owner) internal virtual returns (uint256) {
        unchecked {
            return _nonces[owner]++;
        }
    }

    function _useCheckedNonce(address owner, uint256 nonce) internal virtual {
        uint256 current = _useNonce(owner);
        if (nonce != current) {
            revert InvalidAccountNonce(owner, current);
        }
    }
}
```


Here is the contract in a nutshell:

- `nonces(address owner)` is a view function that returns the current nonce for `owner`.
- `_useNonce(address owner)` increments the nonce for the owner and return the nonce value before the increment. For example, if Alice’s nonce was 2 and we call `_useNonce` for her account, then `_useNonce` returns 2, but her current nonce becomes 3 after the function executes.
- `_useCheckedNonce(address owner, uint256 nonce)` increments the nonce, but reverts unless `nonce` argument is equal to the current nonce. For example, if Alice’s _current_ nonce is 15, then supplying any other than 15 for the `nonce` argument will cause a revert. After the function call, Alice’s nonce will become 16.

This is not much different from our running Counter example from earlier in our tutorial, so the rules Certora wrote for this library will be easy to explain.


Here are some of the properties that will be proven:

- The nonce only increments
- Incrementing a nonce never fails
- Incrementing one nonce has no effect on other nonces

## Nonces.sol Spec


Here is the [spec Certora wrote for nonces.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/certora/specs/Nonces.spec).


### Helper function


The spec includes a helper CVL function to check that the nonce is not the [maximum value of uint256](https://www.rareskills.io/post/uint-max-value-solidity). In practice, a counter cannot get that high:


```solidity
function nonceSanity(address account) returns bool {
    return nonces(account) < max_uint256;
}
```


### Nonce only increments


We'll start with the easiest rule, since we saw something similar in the chapter on parametric rules:


```solidity
// address account in the argument is equivalent to
// declaring it inside the function body
rule nonceOnlyIncrements(
address account)
 {
    require nonceSanity(account);

    mathint nonceBefore = nonces(account);

    env e; method f; calldataarg args;
    f(e, args);

    mathint nonceAfter = nonces(account);

    assert nonceAfter == nonceBefore || nonceAfter == nonceBefore + 1, "nonce only increments";
}
```


This rule says that "all transactions either result in the nonce not changing or increasing by exactly 1."


### useNonce rule


The `useNonce` rule makes two assertions:

- calling `useNonce()` never reverts
- incrementing the nonce on an `account` A leaves all other accounts unaffected, i.e. their nonce does not change.

```solidity
rule useNonce(address account) {
    require nonceSanity(account);

    address other;

    mathint nonceBefore = nonces(account);
    mathint otherNonceBefore = nonces(other);

    mathint nonceUsed = useNonce@withrevert(account);
    bool success = !lastReverted;

    mathint nonceAfter = nonces(account);
    mathint otherNonceAfter = nonces(other);

    // liveness
    assert success, "doesn't revert";

    // effect
    assert nonceAfter == nonceBefore + 1 && nonceBefore == nonceUsed, "nonce is used";

    // no side effect
    assert otherNonceBefore != otherNonceAfter => other == account, "no other nonce is used";
}
```


Let's explain this piece by piece:


```solidity
require nonceSanity(account);
```


This code ensures the prover doesn't test (impossible) case when the nonce is `type(uint256).max`.


```solidity
address other;
```


This `other` is how we get a reference to "any other account" besides `account` specified in the argument of the rule. Note that the rule does not require `account != other` as a precondition — this is handled later.


```solidity
mathint nonceBefore = nonces(account);
mathint otherNonceBefore = nonces(other);
```


This reads the values of the nonces of `account` and `other` before the state transition.


```solidity
mathint nonceUsed = useNonce@withrevert(account);
bool success = !lastReverted;

mathint nonceAfter = nonces(account);
mathint otherNonceAfter = nonces(other);
```


The code `mathint nonceUsed = useNonce@withrevert(account);` is the key line, as the nonce is incremented there. We record the new nonce state in `nonceAfter` and `otherNonceAfter`.


Now let's look at the assertions:


```solidity
// liveness
    assert success, "doesn't revert";

    // effect
    assert nonceAfter == nonceBefore + 1 && nonceBefore == nonceUsed, "nonce is used";

    // no side effect
    assert otherNonceBefore != otherNonceAfter => other == account, "no other nonce is used";
```


Recall that the function `useNonce` returns the nonce that was just consumed, hence the assertion `nonceBefore == nonceUsed`.


The final assertion states that no other nonce was changed. Its logic can be derived using the contrapositive rule.


By the contrapositive rule of implications, if A → B, then !B → !A. Suppose A means `other ≠ account` and B means `otherNonceBefore = otherNonceAfter`. We can then say that “if the other is not equal to account, then other’s nonce didn’t change. Therefore, we can write that `other ≠ account` implies `otherNonceBefore = otherNonceAfter`, or in other words A → B. Using the contrapositive rule, we can say that !B → !A or equivalently `otherNonceBefore ≠ otherNonceAfter` (change happened) implies `other = account` and this is exactly what the last assertion states.


### useCheckedNonce rule


The final rule re-uses a lot of logic from the previous examples, we won’t explain it here. The key difference is the line `assert success <=> to_mathint(currentNonce) == nonceBefore, "works iff current nonce is correct";`. Recall that `useCheckedNonce` only succeeds (doesn’t revert) if the nonce passed to it equals the current nonce of the user. Hence, the biconditional operator encodes that “if the nonce matches, the transaction is guaranteed to succeed. If not, it is guaranteed to revert.”


```solidity
rule useCheckedNonce(address account, uint256 currentNonce) {
    require nonceSanity(account);

    address other;

    mathint nonceBefore = nonces(account);
    mathint otherNonceBefore = nonces(other);

    useCheckedNonce@withrevert(account, currentNonce);
    bool success = !lastReverted;

    mathint nonceAfter = nonces(account);
    mathint otherNonceAfter = nonces(other);

    //// SEE HERE ////
    // liveness
    // the to_mathint cast makes it explicit to have the same type on both sides
    assert success <=> to_mathint(currentNonce) == nonceBefore, "works iff current nonce is correct";

    // effect
    assert success => nonceAfter == nonceBefore + 1, "nonce is used";

    // no side effect
    assert otherNonceBefore != otherNonceAfter => other == account, "no other nonce is used";
}
```
