# First Flight #1: PasswordStore

PasswordStore is a smart contract application for storing a password. Users should be able to store a password and then retrieve it later. Others should not be able to access the password.


## Roles

Owner - Only the owner may set and retrieve their password

## Quickstart

```
git clone https://github.com/Cyfrin/2023-10-PasswordStore
cd 2023-10-PasswordStore
​forge install foundry-rs/forge-std --no-commit
forge build
```

### [H-1] Storing the password on-chain makes it visible to anyone, and no longer private

**Description:** All data stored on blockchain is visible to anyone, can be read by directly from the blockchain. The `PasswordStore.sol::s_password` variable is intended to be a private variable and only accessed through `PasswordStore.sol::getPassword` function, which is intended to be only called by the owner of the contract.

**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:**

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

we use `1` because that's the storage slot of `s_password` in the contract.

```bash
cast storage <ADDRESS> 1 --rpc-url http://127.0.0.1:8545
```

you'll get an output that looks like this.

`0x6d7950617373776f726400000000000000000000000000000000000000000014`

you can then parse that hex to string with

```bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

and get an output of:

```
myPassword
```

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.

### [H-2] `PasswordStore.sol::setPassword` has no access control, meaning a non-owner could change the password.

**Description:** The `PasswordStore.sol::setPassword` function is set to be an `external` function, however the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`
 
 ```js
    function setPassword(string memory newPassword) external {
@>      // @audit - There are no access controls.
        s_password = newPassword;
        emit SetNetPassword();
    }
 ```
    
**Impact:** Anyone can set/change the password of the contract, severly breaking the contract intended functionality.

**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file.

<details>

<summary>Code</summary>

```js
    function test_anyone_can_set_password(address randomAddress, string memory newPassword) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        passwordStore.setPassword(newPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, newPassword);
    }
```

</details>

**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

```js
if(msg.sender != s_owner) {
    revert PasswordStore__NotOwner();
}
```
