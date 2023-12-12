---
title: Protocol Audit Report
author: Kazuki Hashimoto
date: March 7, 2023
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
\centering
\begin{figure}[h]
\centering
\includegraphics[width=0.5\textwidth]{logo.pdf}
\end{figure}
\vspace{2cm}
{\Huge\bfseries Protocol Audit Report\par}
\vspace{1cm}
{\Large Version 1.0\par}
\vspace{2cm}
{\Large\itshape Kazuki Hashimoto\par}
\vfill
{\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Kazuki Hashimoto](https://github.com/setouchi-h) <br>
Lead Security Researcher:

- Kazuki Hashimoto

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Passwords stored on-chain are visable to anyone, not matter solidity variable visibility](#h-1-passwords-stored-on-chain-are-visable-to-anyone-not-matter-solidity-variable-visibility)
    - [\[H-2\] `PasswordStore::setPassword` is callable by anyone](#h-2-passwordstoresetpassword-is-callable-by-anyone)
  - [Informational](#informational)
    - [\[I-1\] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect](#i-1-the-passwordstoregetpassword-natspec-indicates-a-parameter-that-doesnt-exist-causing-the-natspec-to-be-incorrect)

# Protocol Summary

PasswordStore is a protocol dedicated to storage and retrieval of a user's passwords. The protocol is designed to be used by a single user, and is not designed to be used by multiple users. Only the owner should be able to set and access this password.

# Disclaimer

The Kazuki team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details

**The findings described in this document correspond the following commit hash:**

```
7d55682ddc4301a7b13ae9413095feffd9924566
```

## Scope

```
src/
--- PasswordStore.sol
```

## Roles

- Owner: Is the only one who should be able to set and access the password.

For this contract, only the owner should be able to interact with the contract.

# Executive Summary

We spent 1 day with patrik using foundry.

## Issues found

| Severity          | Number of issues found |
| ----------------- | ---------------------- |
| High              | 2                      |
| Medium            | 0                      |
| Low               | 0                      |
| Info              | 1                      |
| Gas Optimizations | 0                      |
| Total             | 3                      |

# Findings

## High

### [H-1] Passwords stored on-chain are visable to anyone, not matter solidity variable visibility

**Description:** All data stored on-chain is visible to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable, and only accessed through the `PasswordStore::getPassword` function, which is intended to be only called by the owner of the contract.

However, anyone can direclty read this using any number of off chain methodologies

**Impact:** The password is not private.

**Proof of Concept:** The below test case shows how anyone could read the password directly from the blockchain. We use [foundry's cast](https://github.com/foundry-rs/foundry) tool to read directly from the storage of the contract, without being the owner.

1. Create a locally running chain

```bash
make anvil
```

2. Deploy the contract to the chain

```
make deploy
```

3. Run the storage tool

We use `1` because that's the storage slot of `s_password` in the contract.

```
cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```

You'll get an output that looks like this:

`0x6d7950617373776f726400000000000000000000000000000000000000000014`

You can then parse that hex to a string with:

```
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

And get an output of:

```
myPassword
```

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.

### [H-2] `PasswordStore::setPassword` is callable by anyone

**Description:** The `PasswordStore::setPassword` function is set to be an `external` function, however the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

```javascript
    function setPassword(string memory newPassword) external {
@>      // @audit - There are no access controls here
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set/change the password of the contract.

**Proof of Concept:**

Add the following to the `PasswordStore.t.sol` test suite.

```javascript
function test_anyone_can_set_password(address randomAddress) public {
    vm.prank(randomAddress);
    string memory expectedPassword = "myNewPassword";
    passwordStore.setPassword(expectedPassword);
    vm.prank(owner);
    string memory actualPassword = passwordStore.getPassword();
    assertEq(actualPassword, expectedPassword);
}
```

**Recommended Mitigation:** Add an access control modifier to the `setPassword` function.

```javascript
if (msg.sender != s_owner) {
    revert PasswordStore__NotOwner();
}
```

## Informational

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect

**Description:**

```javascript
/**
 * @notice This allows only the owner to retrieve the password.
 * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory) {
```

The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec says it should be `getPassword(string)`.

**Impact:** The natspec is incorrect.

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
-    * @param newPassword The new password to set.
```
