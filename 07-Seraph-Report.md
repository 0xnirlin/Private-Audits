# Seraph ERC1155 Audit Findings - August 2024

## Summary Table

| Severity | Number of Findings |
|----------|---------------------|
| Medium   | 1                   |
| Low      | 2                   |
| Gas Optimizations | 3          |
| Total    | 6                   |

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| M-01 | URI function is non-compliant with ERC-1155 | Medium | Fixed |
| L-01 | Use string.concat() or bytes.concat() instead of abi.encodePacked | Low | Fixed |
| L-02 | Do not use deprecated library functions | Low | N/A |
| G-01 | Don't use _msgSender() if not supporting EIP-2771 | Gas | Fixed |
| G-02 | Use Custom Errors instead of Revert Strings to save Gas | Gas | Fixed |
| G-03 | Using private rather than public for constants, saves gas | Gas | Fixed |

## Medium Risk

### [M-01] URI function is non-compliant with ERC-1155

**Severity:** Medium

**Context:** SeraphERC1155.sol

**Description:**
The ERC1155 standard states in the natspec of URI that the URI must point to a JSON that conforms to the appropriate JSON schema. However, in the current implementation, if there is no set URI, it simply returns the ID that was passed as an argument as a string, which is not a link to JSON.

```solidity
function uri(uint256 id) public view override returns (string memory)
{
    require(exists(id), "URI: nonexistent token");
    return bytes(super.uri(id)).length > 0 ?
        string(abi.encodePacked(abi.encodePacked(super.uri(id), id.toString()),
        ".json")) : id.toString();
}
```

**Recommendation:**
Ensure that the URI function always returns a valid URI pointing to a JSON file that conforms to the "ERC-1155 Metadata URI JSON Schema".

## Low Risk

### [L-01] Use string.concat() or bytes.concat() instead of abi.encodePacked

**Severity:** Low

**Context:** SeraphERC1155.sol

**Description:**
Solidity version 0.8.4 introduces `bytes.concat()` and version 0.8.12 introduces `string.concat()`, which are more efficient and safer alternatives to `abi.encodePacked()`.

```solidity
return bytes(super.uri(id)).length > 0 ?
    string(abi.encodePacked(abi.encodePacked(super.uri(id), id.toString()),
    ".json")) : id.toString();
```

**Recommendation:**
Use `string.concat()` or `bytes.concat()` instead of `abi.encodePacked()` for string concatenation.

### [L-02] Do not use deprecated library functions

**Severity:** Low

**Context:** SeraphERC1155.sol

**Description:**
In OpenZeppelin V4, `_setupRole` has been deprecated in favor of `grantRole`. While not an issue with the current version, it should be considered if upgrading to the latest OZ version before deployment.

```solidity
_setupRole(DEFAULT_ADMIN_ROLE, owner_);
_setupRole(MINTER_ROLE, owner_);
_setupRole(PAUSER_ROLE, owner_);
```

**Recommendation:**
Consider using `grantRole` instead of `_setupRole` to future-proof the contract.

## Gas Optimizations

### [G-01] Don't use _msgSender() if not supporting EIP-2771

**Context:** SeraphERC1155.sol

**Description:**
Using `_msgSender()` instead of `msg.sender` incurs unnecessary gas costs if the contract does not implement EIP-2771 trusted forwarder support.

```solidity
require(hasRole(role, _msgSender()), "Permission denied");
```

**Recommendation:**
Use `msg.sender` if the code does not implement EIP-2771 trusted forwarder support.

### [G-02] Use Custom Errors instead of Revert Strings to save Gas

**Context:** SeraphERC1155.sol

**Description:**
Custom errors, available from Solidity version 0.8.4, save ~50 gas each time they're hit by avoiding having to allocate and store the revert string.

```solidity
require(hasRole(role, _msgSender()), "Permission denied");
require(exists(id), "URI: nonexistent token");
```

**Recommendation:**
Replace revert strings with custom errors to save gas.

### [G-03] Using private rather than public for constants, saves gas

**Context:** SeraphERC1155.sol

**Description:**
Using `private` instead of `public` for constants saves gas in deployment and reduces the contract size.

```solidity
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
```

**Recommendation:**
Change `public` constants to `private` to save gas. If the values need to be accessed, they can be read from the verified contract source code or a getter function can be implemented.