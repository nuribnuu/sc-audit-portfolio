# title: PasswordStore Audit Report
# author: Muhammad Nur Ibnu Hubab

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
