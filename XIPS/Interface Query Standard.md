---
xip: <to be assigned>
title: Interface Query Standard
description: Creates a standard method to publish and detect what interfaces a smart contract implements
author: Jon McBee <jon@xdc.community>
discussions-to: <https://www.xdc.dev/walterblueu/proposal-for-interface-query-standard-xip-4id8>
status: Draft
type: Standards
category: Interface
created: 2022-11-17
requires (*optional): <XIP number(s)>
---

<!--This is the suggested template for new XIPs.

Note that an XIP number will be assigned by an editor. When opening a pull request to submit your XIP, please use an abbreviated title in the filename, `xip-draft_title_abbrev.md`.

The title should be 44 characters or less. It should not repeat the XIP number in title, irrespective of the category.--> 
## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the XIP.-->
This XIP defines a method to query whether a contract implements a certain interface or not.
  
## Abstract
This proposal defines:
1. [How interface identifiers are defined](#how-interface-identifiers-are-defined)
2. [How a contract publishes the interfaces it implements](#how-a-contract-publishes-the-interfaces-it-implements)
3. [How to query if a contract implements implements this XIP](#how-to-query-if-a-contract-implements-this-xip)
4. [How to query if a contract implements any given interface](#how-to-query-if-a-contract-implements-any-given-interface)
  
## Motivation
<!--The motivation section should describe the "why" of this XIP. What problem does it solve? Why should someone want to implement this standard? What benefit does it provide to the XDC Network ecosystem? What use cases does this XIP address?-->
Since there is no clear way to find what functions are implemented in a contract, this XIP proposes a standard method to define and query interfaces in a contract.
For example, if we define an interface identifier of [XIP-8](https://xips.xdc.community/XIPS/xip-8), we can easily determine a contract implements [XIP-8](https://xips.xdc.community/XIPS/xip-8) or not.

## Specification
<!--The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current XDC Network platforms.-->
This document derived heavily from Ethereum's [ERC-165](https://eips.ethereum.org/EIPS/eip-165) written by Christian Reitwießner, Nick Johnson, Fabian Vogelsteller, Jordi Baylina, Konrad Feldmeier, and William Entriken.

### How Interface Identifiers are Defined

An *interface identifier* is a combination of function selectors of a contract.
A function selector is four bytes of keccak256 hash of the signature of a function
(e.g., `bytes4(keccak256('supportsInterface(bytes4)'))`).
The signature is defined as the canonical expression of the basic prototype without parameter names and the return type.

We define the interface identifier as the XOR of all function selectors in the interface. This code example below shows how to calculate an interface identifier:

```solidity
pragma solidity ^0.4.24;
interface Solidity101 {
    function hello() external pure;
    function world(int) external pure;
}
contract Selector {
    function calculateSelector() public pure returns (bytes4) {
        Solidity101 i;
        return i.hello.selector ^ i.world.selector;
    }
}
```

Note: interfaces do not permit optional functions, therefore, the interface identifier will not include them.

### How a Contract Publishes the Interfaces it Implements

A contract that is compliant with this XIP shall implement the following interface (referred as `InterfaceIdentifier.sol`):

```solidity
pragma solidity ^0.4.24;
interface InterfaceIdentifier {
    /// @notice Query if a contract implements an interface
    /// @param interfaceID The interface identifier, as defined in this XIP.
    /// @dev Interface identifier is defined in this XIP. This function
    ///  uses less than 30,000 gas.
    /// @return `true` if the contract implements `interfaceID` and
    ///  `interfaceID` is not 0xffffffff, `false` otherwise.
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}
```

The interface identifier for this interface is `0x01ffc9a7`.
You can calculate this by running `bytes4(keccak256('supportsInterface(bytes4)'));` or using the `Selector` contract above.

The implementing contract will have a `supportsInterface` function, and it returns:

- `true` when `interfaceID` is `0x01ffc9a7` (`supportsInterface` itself)
- `false` when `interfaceID` is `0xffffffff`
- `true` for `interfaceID` this contract implements
- `false` for any other `interfaceID`

This function must return a bool and use at most 30,000 gas.

Implementation note: there are several logical ways to implement this function. Please see the example implementations and the discussion on gas usage.

### How to Query if a Contract Implements This XIP

1. The source contract makes a `STATICCALL` to the destination address with input data: `0x01ffc9a701ffc9a700000000000000000000000000000000000000000000000000000000` and gas 30,000. This corresponds to `contract.supportsInterface(0x01ffc9a7)`.
2. If the call fails or return false, the destination contract does not implement this XIP.
3. If the call returns true, a second call is made with input data `0x01ffc9a7ffffffff00000000000000000000000000000000000000000000000000000000`.
  This corresponds to `contract.supportsInterface(0xffffffff)`.
4. If the second call fails or returns true, the destination contract does not implement this XIP.
5. Otherwise it implements this XIP.

### How to Query if a Contract Implements any Given Interface

1. If you are not sure if the contract implements this XIP, use the above procedure to confirm.
2. If it does not implement this XIP, then you will have to see what methods it uses in other way.
3. If it implements this XIP then just call `supportsInterface(interfaceID)` to determine if it implements an interface you can use.
  
## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.-->
We tried to keep this specification as simple as possible. This implementation is also compatible with the current Solidity version.

## Backwards Compatibility
<!--All XIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The XIP must explain how the author proposes to deal with these incompatibilities. XIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
The mechanism described above (with `0xffffffff`) should work with most of the contracts previous to this standard to determine that they do not implement this XIP.

## Test Cases
<!--Test cases for an implementation are mandatory for XIPs that are affecting consensus changes.  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/xip-####/`.-->
Following is a contract that detects which interfaces other contracts implement. From @fulldecent and @jbaylina.

```solidity
pragma solidity ^0.4.24;
contract supportsInterfaceQuery {
    bytes4 constant InvalidID = 0xffffffff;
    bytes4 constant supportsInterfaceID = 0x01ffc9a7;
    function doesContractImplementInterface(address _contract, bytes4 _interfaceId) external view returns (bool) {
        uint256 success;
        uint256 result;
        (success, result) = noThrowCall(_contract, supportsInterfaceID);
        if ((success==0)||(result==0)) {
            return false;
        }
        (success, result) = noThrowCall(_contract, InvalidID);
        if ((success==0)||(result!=0)) {
            return false;
        }
        (success, result) = noThrowCall(_contract, _interfaceId);
        if ((success==1)&&(result==1)) {
            return true;
        }
        return false;
    }
    function noThrowCall(address _contract, bytes4 _interfaceId) constant internal returns (uint256 success, uint256 result) {
        bytes4 id = supportsInterfaceID;
        assembly {
                let x := mload(0x40)               // Find empty storage location using "free memory pointer"
                mstore(x, id)                      // Place signature at beginning of empty storage
                mstore(add(x, 0x04), _interfaceId) // Place first argument directly next to signature
                success := staticcall(
                                    30000,         // 30k gas
                                    _contract,     // To addr
                                    x,             // Inputs are stored at location x
                                    0x24,          // Inputs are 36 bytes long
                                    x,             // Store output over input (saves space)
                                    0x20)          // Outputs are 32 bytes long
                result := mload(x)                 // Load the result
        }
    }
}
```

## Reference Implementation
<!--An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification.  If the implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/xip-####/`.-->
This approach uses a `view` function implementation of `InterfaceIdentifier`. The execution cost is 586 gas for any input. But contract initialization requires storing each interface (`SSTORE` is 20,000 gas). The `MappingImplementation` contract is generic and reusable.

```solidity
pragma solidity ^0.4.24;
import "./InterfaceIdentifier.sol";
contract MappingImplementation is InterfaceIdentifier {
    /// @dev You must not set element 0xffffffff to true
    mapping(bytes4 => bool) internal supportedInterfaces;
    function MappingImplementation() internal {
        supportedInterfaces[this.supportsInterface.selector] = true;
    }
    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return supportedInterfaces[interfaceID];
    }
}
interface Simpson {
    function is2D() external returns (bool);
    function skinColor() external returns (string);
}
contract Lisa is MappingImplementation, Simpson {
    function Lisa() public {
        supportedInterfaces[this.is2D.selector ^ this.skinColor.selector] = true;
    }
    function is2D() external returns (bool){}
    function skinColor() external returns (string){}
}
```

Following is a `pure` function implementation of `InterfaceIdentifier`. The worst-case execution cost is 236 gas, but increases linearly with a higher number of supported interfaces.

```solidity
pragma solidity ^0.4.24;
import "./InterfaceIdentifier.sol";
interface Simpson {
    function is2D() external returns (bool);
    function skinColor() external returns (string);
}
contract Homer is InterfaceIdentifer, Simpson {
    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return
          interfaceID == this.supportsInterface.selector || // InterfaceIdentifier
          interfaceID == this.is2D.selector
                         ^ this.skinColor.selector; // Simpson
    }
    function is2D() external returns (bool){}
    function skinColor() external returns (string){}
}
```

With three or more supported interfaces (including this XIP itself as a required supported interface), the mapping approach (in every case) costs less gas than the pure approach (at worst case).


## References
- https://eips.ethereum.org/EIPS/eip-165
- https://solidity.readthedocs.io/en/develop/abi-spec.html#function-selector

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).
