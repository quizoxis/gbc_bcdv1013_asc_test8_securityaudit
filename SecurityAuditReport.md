# BadKickback Security Audit

March 26, 2020
Authored by Mohammad Jamshed Qureshi (GBC Student: 101260567 / T175 Blockchain Development Fall 2019)

# Introduction

As part of the Blockchain course our instructor requested to perform a security audit of the BadKickback contract.

## About BadKickback
BadKickback contract is used to encourage higher event turnout rate. The rule is to charge everyone a small fee when they sign up for the event. The fee will be refunded after the event check-in. No-shows will lose their fee, which will be split amongst the attendees.

The BadKickback contract is hosted at:

https://github.com/ethereumgb/security-audit

"BadKickback contract - BadKickback.sol" resides in the "contracts" folder.

The git commit hash I evaluated is:
81133e3a008a55bf87972739e7be69bde226c47c

# Disclaimer

The audit makes no statements or warrantees about utility of the code, safety of the code, suitability of the business model, regulatory regime for the business model, or any other statements about fitness of the contracts to purpose, or their bugfree status. The audit documentation is for learning and discussion purposes only.

# Executive Summary

BadKickback contract has some critical security and design issues and should addressed before putting the contract into a production environment. Failing to do so would have severe business impact and loss of users and trust.

# Findings

During the audit in total, 5 issues were found.

| Severity Level  | Count        |
| --------------- | ------------ |
| Critical | 2 |
| High | 2 |
| Low | 1 |

## Vulnerabilities

| No.  | Issue        | Severity       |
| ---- | ------------ |:-------------:|
| 1    | Unprotected selfdestruct      | Critical |
| 2    | Race Conditions, Reentrancy attack       | Critical      |
| 3    | Unexpected Contract Balance | High |
| 4    | Denial of service (DoS) with block gas limit | High   |
| 5    | CALL Return Values Unchecked | Low      |


## Finding 1: Unprotected selfdestruct (Critical)

***Severity*** : Critical

*destroy()* function has a visibility of *public* and can be invoked by any user resulting in the full destruction of the contract. Self destruction of the contract results in removal of all bytecode from the contract address and any leftover ether balance is sent to a specified address.

```solidity
function destroy() public {
  selfdestruct(owner);
}
```

***Recommendation***

A role must be defined, who is allowed to invoke the *destroy()* function. Use of *require()* is highly recommended as well considering what context can invoke the function. Based on this information appropriate modifier should be used. As an example:

```solidity
function destroy() public onlyOwner {
  _destroy();
}
```

- Above function has public visibility which makes it be callable from anywhere.
- The *onlyOwner* modifier provides the restriction so that only owner of the contract is allowed to invoke it.
- Lastly the function delegates the actual destruction of the contract to an internal function *_destroy*.

```solidity
function _destroy() internal {
  selfdestruct(owner);
}
```

Here;
- visibility of the *_destroy* is marked as internal i.e. so it can only be called from within the contract.


## Finding 2: Reentrancy Attack (Critical)

***Severity*** : Critical

There's a vulnerability at line 50, it is here when contract sends the amount using an external call. A malicious attacker can hijack this call and force the contract to execute further code via fallback function. Ref: the infamous DAO Attack [1][2].

```solidity
(bool success, ) = payee.call.value(amount)("");
```

***Recommendation***

There are number of ways to avoid reentrancy Vulnerabilities:

- Use of buit-in transfer function, it only attaches 2300 gas with external call, 2300 gas is not enough for the destination contract to call another contract.

## Finding 3: Unexpected Contract Balance (High)

***Severity*** : High

There's a poor use of this.balance at line 43. A malicious attacker could forcibly send an amount of ether via the selfdestruct function.

***Recommendation***

In order to avoid this vulnerability, balance of the contract values must not be used because it can be manipulated externally. Balance of values can be tracked using a self-defined variable, which is incremented in *participate()* function. During a selfdestruct call this variable will not be affected.

## finding 4: Denial of service (DoS) with block gas limit (High)

***Severity*** : High

Payout functions accepts an array of address and then loops through it. This array is external to the contract and can be manipulated or inflated. This vulnerability has the potential to make the contract inoperable for a period of time or event infinite.

***Recommendation***

A withdrawal pattern can be applied where each participant calls the payout to claim their ethers independently.

## Finding 5: CALL Return Values Unchecked (Low)

***Severity*** : Low

At line 50, *call* function is used without checking the response. In a situation where amount transfer (external call) fails the payee would not receive the amount. Any left-over ether will be locked in the contract.

***Recommendation***

 *call* function returns a boolean indicating its success or failure state. But if the external call fails it will not revert and would simply return a false. So to avoid this, either use transfer which will revert if the external transaction fails or put a check on the return value of call and take appropriate actions. And mark the participant's payout state.

# Conclusion
In the light of above findings, it is recommended that any critical and high issues must be resolved before moving into production setting. Failing to resolve these important issue will have severe impacts on the  business, loss of user trust and possibly legal implications.

# Appendix

## Severity Levels

- **Critical** : can cost unlimited amount of users' funds

- **High** : can cost limited amount of users' funds

- **Low** : minor problems or better solutions

# References

- [1] Ethereum Smart Contract Best Practices - Reentrancy https://consensys.github.io/smart-contract-best-practices/known_attacks/#reentrancy

- [2] https://en.wikipedia.org/wiki/The_DAO_(organization)

- [3] Ethereum Smart Contract Best Practices - Forcibly Sending Ether to a Contract https://consensys.github.io/smart-contract-best-practices/known_attacks/#forcibly-sending-ether-to-a-contract
