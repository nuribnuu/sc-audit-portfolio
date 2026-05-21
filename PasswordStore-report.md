---
title: Protocol Audit Report
author: Muhammad Nur Ibnu Hubab
date: May 21, 2025
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
\vspace*{2cm}
{\Huge\bfseries Protocol Audit Report\par}
\vspace{1cm}
{\Large Version 1.0\par}
\vspace{2cm}
{\Large\itshape nuribnuu.vercel.app\par}
\vfill
{\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Muhammad Nur Ibnu Hubab](https://nuribnuu.vercel.app)
Lead Auditors:

- Muhammad Nur Ibnu Hubab

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
    - [\[H-1\] `PasswordStore::setPassword` has no access controls, meaning a non-owner could change the password](#h-1-passwordstoresetpassword-has-no-access-controls-meaning-a-non-owner-could-change-the-password)
    - [\[H-2\] Storing the password on-chain makes it visible to anyone, no longer private](#h-2-storing-the-password-on-chain-makes-it-visible-to-anyone-no-longer-private)
- [Informational](#informational)
    - [\[I-1\] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist](#i-1-the-passwordstoregetpassword-natspec-indicates-a-parameter-that-doesnt-exist)

# Protocol Summary

A smart contract application for storing a password. Users should be able to store a password and then retrieve it later. Others should not be able to access the password.

# Disclaimer

The Muhammad Nur Ibnu Hubab team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details

Commit Hash: `7d55682ddc4301a7b13ae9413095feffd9924566`

## Scope

```
./src/
#- PasswordStore.sol
```

## Roles

- Owner: The user who can set the password and read the password.
- Outsides: No one else should be able to set or read the password.

## Issues found

| Severiti | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 1                      |
| Total    | 3                      |

# Findings

# High

### [H-1] `PasswordStore::setPassword` has no access controls, meaning a non-owner could change the password

**Description:** All data stored on-chain is visible to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private intended to be only called by the owner of the contract.

We show one such method of reading any data off-chain below.

**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:** (Proof of Code)

The below test case shows how anyone can read the password directly from the blockchain.

1. Create a locally running chain

```bash
make anvil
```

2. Deploy the contract to the chain

```bash
make deploy
```

3. Run the storage tool
   We use `1` because that's the storage slot of `s_password` in the contract.

```bash
cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```

You'll get an output that looks like this:
`0x6d7950617373776f726400000000000000000000000000000000000000000014`

```
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
myPassword
```

And get an output of:

```
myPassword
```

**Recommended Mitigation:** The overall architecture of the contract should be rethought. One possible approach is to
encrypt the password off-chain and only store the encrypted version on-chain. This would
require the user to remember a separate decryption key off-chain. Additionally, the
`PasswordStore::getPassword` view function should be removed to prevent users from accidentally sending
their decryption key as a transaction, which would expose it on-chain.

### [H-2] Storing the password on-chain makes it visible to anyone, no longer private

**Description:** The `PasswordStore:setPassword` function is set to be an `external` function, however, the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

```solidity
    function setPassword(string memory newPassword) external {
@>      // @audit - there are no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set the password of the contract, severly breaking the contract intended functionality.

**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file.

<details>
<summary>Code</summary>

```solidity
    function test_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```

</details>

**Recommended Mitigation:** Add an access control conditional to the `PasswordStore::setPassword` function.

```solidity
if(msg.sender != owner) {
  revert PasswordStore__NotOwner;
}
```

# Informational

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist

**Description:**

```solidity
    /*
     * @notice This allows only the owner to retrieve the password.
@>   * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
```

The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec says it should be `getPassword(string)`.

**Impact:** The natspec is incorrect.

**Recommended Mitigation:** Remove the incorrect natspect line.

```diff
- * @param newPassword The new password to set.
```
