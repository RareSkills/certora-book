# Introduction to Formal Verification

Formal verification is the process of mathematically proving a program adheres to a specification. 

This article introduces conceptually how formal verification works, how it compares to fuzzing, and formal verification’s limitations and advantages.

## A “specification” definition

A specification is a well-defined behavior under specific circumstances.

Here are some examples of specifications that apply to DeFi smart contracts:

(1) Lenders should earn no interest if no time has passed.

(2) If time has passed, and there are active borrows, the lender should earn more than zero interest.

(3) If the protocol is not paused, borrowers can always repay their loans.

(4) If the supply cap is not exceeded and the protocol is not paused, lenders can always deposit.

Formal verification also allows us to reason about the internal state of a contract. We can write specifications such as the following which solely inspect the storage variables:

(5) For any sequence of transactions, the sum of balances of an ERC-20 token should equal the total supply.

(6) If `deposit()` is not called, then a `balance` cannot increase.

Let’s suppose we had a way of knowing for sure that “(1) Lenders should earn no interest if no time has passed.” In that case, we know that the *particular* attack vector of stealing money from the protocol via interest that shouldn’t be earned is *impossible.* While we don’t guarantee that the protocol has no bugs, we know it does not have a *particular* bug.

Consider the implications if the specification “(2) If time has passed, and there are active borrows, the lender should earn more than zero interest” is violated. It means that someone is borrowing money, but the lender is not earning interest. That means there is either

- a bug where the protocol is earning the interest, but not the lender
- the borrower is paying interest which is going to some account it shouldn’t — or the interest is getting burned
- the borrower is not paying any interest

The nice thing about knowing with certainty that specification 2 holds is we know that none of the above bugs apply to the protocol.

If 3 and 4 hold, then that eliminates entire categories of denial-of-service attacks. If an attacker had a way to prevent borrowers from repaying, then they could force the borrowers to rack up interest and then liquidate the borrowers for an unfair profit. On the flip side, if attackers can block lenders from depositing, then a malicious competitor could try to direct liquidity away from that protocol by preventing deposits in the first place.

If 5 and 6 hold, we can be sure there aren’t bugs related to mismatched accounting of the ERC-20 storage variables.

Formal verification can only catch bugs we create a specification for. Using the ERC-20 example, if we leave the `burn(address from, uint256 amount)` function unprotected (i.e. we let *anyone* call it — there is no access control), none of the specifications above will catch the missing access control.

The formal verification tool we will study in this series, the Certora Prover, reasons about smart contracts at the bytecode level. This means that *if a bug is introduced by the compiler, then formal verification can catch it, as long as the bug results in a violation of one of our specifications*.

### A specification as a statement about state transitions

Using the specification “the sum of all balances of an ERC-20 token should equal the total supply” the formal verification tool would define two things:

1. A state transition function $\phi$
2. A representation of the state (a collection of values at a certain point in time, such as state variables of a contract and Ethereum account balances), which we simply call $\texttt{state}$. We may refer to a *particular* state as $\texttt{state}_i$ and a different state as $\texttt{state}_j$. We may also refer to $\texttt{state}_{i+1}$ which is the state after a transaction that happened during $\texttt{state}_i$ and caused some storage variables or account balances to change.

The first is a state transition function $\phi$ that, given a state (such as storage variables and account balances), the environment (such as the block timestamp), and calldata of a transaction, produces a new state:

$$
\text{state}_{i+1}=\phi(\text{state}_i,\texttt{calldata},\texttt{environment})
$$

This function $\phi$ is a mathematical representation of the bytecode of the smart contract. Then, our specification “the sum of all balances equals the total supply” would be compiled to the following mathematical formula:

$$
\forall_i\sum_a\texttt{state}_i\texttt{.balances}_a=\texttt{state}_i\texttt{.total\_supply}
$$

In other words, for any state $i$, the sum of the balances $\left(\sum_a\texttt{state}_i\texttt{.balances}_a\right)$ in that state for each account $a$ equals the `total_supply` of that state.

Now suppose we are given a state transition function $\phi:\texttt{state}\rightarrow\texttt{state}'.$ We want to know that if, for any $\texttt{state}$ plugged into $\phi$ that obeys the formula above, the resulting state $\texttt{state}'$ also obeys the formula. (Note that our specification is independent of $\phi$).

The prover would then attempt to prove by induction (don’t worry if this term is unfamiliar — we will talk about it in a later chapter) that if $\texttt{state}_i$ follows the specification above, then any state that comes after must also follow the specification.

$\phi$ is the smart contract itself, since it takes a state, calldata, and an environment and produces a new state. The formula we wrote above is a property we want to ensure the smart contract has.

As a side benefit, if there is a $\text{state}$ and $\texttt{calldata}$ that lead to a violation, the Certora prover will provide an example of it.

The *huge* benefit of an automatic formal verification tool like Certora Prover is that it figures out the function $\phi$ for us by translating the bytecode of the smart contract to a mathematical function of the state variables, calldata, and environment variables. This is an immensely complex process which requires substantial research and development, but thankfully we can mostly treat it as a black box (there are some important exceptions that we will explore later).

### Example specification

Using our example of “lenders should earn no interest if no time has passed” can be expressed mathematically as:

$$
\forall i,j:\texttt{state}_i\texttt{.timestamp}\texttt{ == } \texttt{state}_j\texttt{.timestamp}\rightarrow \texttt{state}_i\texttt{.interestEarned == }\texttt{state}_j\texttt{.interestEarned}
$$

What this is saying is “if the block timestamp of states $i$ and $j$  are equal, then the `interestEarned` must be the same.”

Don’t worry if the math feels a little strange to read. We can translate it as:

- $\forall_{i,j}$ means “for all $i$ and $j$” or “for any two values for $i$ and $j$”
- $\texttt{state}_i\texttt{.timestamp}\texttt{ == } \texttt{state}_j\texttt{.timestamp}\rightarrow$  if those two states have the same timestamp
- $\texttt{state}_i\texttt{.interestEarned == }\texttt{state}_j\texttt{.interestEarned}$ then the `interestEarned` is the same

Note that the $\rightarrow$ behaves like “if then” in normal programming. The actual syntax Certora uses looks more like Solidity than the math we have above, but the underlying meaning would be the same.

## Formal Verification vs Fuzzing

Formal verification and fuzzing accomplish the same purpose, but use different strategies for accomplishing the same goal. Fuzzing uses a random, but often guided, sequence of transactions to try to find violations to invariants. They do this by running thousands, possibly millions, of tests.

Formal verification on the other hand proves the invariants hold mathematically.

By contrast, fuzzing cannot prove the absence of bugs. It can only show that the bug was absent in the cases that it tested.

Fuzzing tests the smart contract for thousands or millions of random cases. Formal verification tests for every possible combination of inputs and storage variable values.

### Limitations of Formal Verification

Formal verification is essentially writing math proofs automatically — and it can be quite powerful.

For example, computers have proven or played a significant role in proving the following (examples taken from the Wikipedia article on [Computer Assisted Proofs](https://en.wikipedia.org/wiki/Computer-assisted_proof)):

- Any 2D map can be four-colored
- Whoever goes first in the game Connect-4 always wins, provided they play the optimal move
- Kepler Conjecture is true — if spheres are placed in a box, the highest density of volume taken up by packed spheres over the total volume of a box cannot exceed $\frac{\pi}{3\sqrt{2}}$.
- A Rubik’s Cube can be solved in at most 20 moves

As can be seen above, computer-assisted mathematical proofs can be extremely powerful, but they cannot solve any problem thrown at them (if they could, there would be no more open problems in mathematics).

## You don’t need to know a lot of math to use formal verification

Even if you haven’t written a mathematical proof in your life, you can still use the Certora Prover. Almost everything is abstracted away from the dev, and the developers can define the specifications using a language similar to Solidity.

However, it is still helpful to have *some* idea of what the Prover is doing. For example, one can write and audit smart contracts without knowing how the EVM works, but this could lead to inefficient code. Similarly, writing specifications for Certora Prover without some idea of what is going on under the hood could lead to proofs that take too long to generate.

Just as how you wouldn’t try to store a jpeg in EVM storage variables due to the high cost, there are pitfalls you must consciously avoid when using formal verification.

Here are some fundamental limitations to formal verification:

**1. The Halting Problem** — given an arbitrary program, detecting infinite loops reliably is mathematically impossible.

If we could detect infinite loops, then we could easily solve an open question in mathematics, the *twin prime conjecture.* Examples of twin primes are (3,5), (11,13), (17,19) — they have a difference of two.

The open question is whether an infinite number of these primes exist. If we could detect infinite loops reliably, then we could write a program that searches for the largest possible twin prime and then halts. If the program never halts, then we know there are an infinite number of twin primes.

**2. Cryptography** — given a 32-byte sequence which is supposedly the output of a hash, does a pre-image to that hash exist? Hashes are extremely hard (for all practical purposes, impossible) to find preimages for. Computer assisted proofs don’t let us break their hardness.

**3. Large non-linear systems of equations** — non-linear systems of equations with a lot of variables are hard to solve in a short period of time. Here, “linear” means that variables we are solving for are only multiplied by constants or added to other unknown variables.

If two unknown variables are multiplied together, or they are raised to some power, then we have a non-linear system of equations. If a formal verification tool has to indirectly solve a non-linear system of equations, then the tool could spend an excessive amount of time looking for a solution.

## How this tutorial series is structured

This tutorial series focuses on

1. The syntax of how to write specifications for Certora Prover
2. Plenty of examples to pattern-match for your own use
3. How to avoid bugs in the specification
4. How to formally verify real DeFi protocols

Each part in the series builds on concepts from previous chapters. For this reason, we advise readers to follow the series in order instead of skipping around.
