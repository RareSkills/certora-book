# Formally verifying Initializable.sol


This article describes how Certora formally verified the Initializable.sol OpenZeppelin contract. We assume the reader is already familiar with how this contract is used. If not, please see our article on [Initializable Contracts](https://www.rareskills.io/post/initializable-solidity).


In upgradeable contracts, the constructor is not used. Instead, after deployment, the deployer calls `initialize(...)` and this sets the state variables to their initial conditions. The `initialize(...)` function must be callable only once.


To allow for the storage variables to be set to another value at a later time (such as if an ERC-20 token changes its name or totalSupply) then these changes must be made through the `reinitialize` function. The external functions do not need to be named "initializer" and "reinitializer" but they do need the `initializer()` and `reinitializer()` modifier from the OpenZeppelin library applied to those functions.


For safety purposes, upgradeable contracts must have the following properties with respect to the initializers:

1. Once the initializer is called, it should never be callable again (all calls should revert).
2. It should not be possible to call the initializer in the implementation contract. Only the initializer in the proxy should be callable, since all the state is kept in the proxy contract. This is why the logic contracts of proxy patterns call `_disableInitializers()` in the constructor.
3. Once the initializer and reinitializer are disabled, they should stay disabled permanently.
4. If we call the reinitializer function, we must increment the version from a previous one. The version is stored as a `uint64` variable.

Note that access control to the initializer and reinitializer is out of scope of this contract. The contract doesn't even enforce what those functions are named, it only ensures the functionality of the modifiers is correct.


We will cover most, but not all of the [Initializable.spec](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/certora/specs/Initializable.spec) file for OpenZeppelin, as some of the contents rely on techniques we haven't covered yet. However, the majority of the contents should be understandable based on what we have covered so far in the course.


Here is the first rule we examine:


```solidity
rule cannotInitializeTwice() {
    require isInitialized();

    initialize@withrevert();

    assert lastReverted, "contract must only be initialized once";
}
```


This says that if the contract is initialized, then calling `initialize()` will revert. This rule _does not_ say that if the contract is not initialized, then calling `initialize()` does not revert, or in other words, an uninitialized contract _can_ be initialized. The `require` statement sets the scope of the rule to only cover cases where the initializer has already been called.


The specification that an uninitialized contract can be initialized is covered by the following rule:


```solidity
rule initializeEffects() {
    requireInvariant notInitializing();

    bool isUninitializedBefore = isUninitialized();

    initialize@withrevert();
    bool success = !lastReverted;

    assert success <=> isUninitializedBefore, "can only initialize uninitialized contracts";
    assert success => version() == 1,         "initialize must set version() to 1";
    
}
```


The rule says that `initialize` succeeds if and only if the contract hasn't been initialized before. If the initialization succeeds, then the version must be set to 1. The `requireInvariant` syntax is not something we’ve covered yet, we’ll revisit that in a later tutorial.


The rule that the reinitializer cannot override the disable status is largely self explanatory:


```solidity
rule cannotReinitializeOnceDisabled() {
    require isDisabled();

    uint64 n;
    reinitialize@withrevert(n);

    assert lastReverted, "contract is disabled";
}
```


In other words, if the contract is disabled, then any call to `reinitialize` leads to a revert.


When the reinitializer is called, a new version must be set, and that version must be larger than the old version. The following rule says that if the new version is the version passed in the argument, and if the transaction didn’t revert, that new version is larger than the previous one.


```solidity
rule reinitializeEffects() {
    requireInvariant notInitializing();

    uint64 versionBefore = version();

    uint64 n;
    reinitialize@withrevert(n);
    bool success = !lastReverted;

    assert success <=> versionBefore < n, "can only reinitialize to a later versions";
    assert success => version() == n,     "reinitialize must set version() to n";
}
```


The final rule we examine states that if the contract is not in the “initializing state” (the contract is currently setting the storage variables or reinitializing) then calling disable always succeeds.


```solidity
rule disableEffect() {
    requireInvariant notInitializing();

    disable@withrevert();
    bool success = !lastReverted;

    assert success,      "call to _disableInitializers failed";
    assert isDisabled(), "disable state not set";
}
```


## Summary


The rules we showed here are extremely simple to grasp, but we review them to demonstrate that formal verification doesn’t always have to be complex. The rules we show here could be accomplished with a unit test, but formal verification is more thorough and high assurance.
