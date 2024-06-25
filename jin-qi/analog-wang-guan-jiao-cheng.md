---
description: 视频教程：https://youtu.be/0750UaRz92E
---

# 🤭 Analog-网关教程

### 合约部署

1. BranchlessMath.sol

```
// SPDX-License-Identifier: MIT
// Analog's Contracts (last updated v0.1.0) (src/utils/BranchlessMath.sol)

pragma solidity ^0.8.20;

/**
 * 
@dev
 Utilities for branchless operations, useful when a constant gas cost is required.
 */
library BranchlessMath {
    /**
     * 
@dev
 Returns the smallest of two numbers.
     */
    function min(uint256 x, uint256 y) internal pure returns (uint256) {
        return select(x < y, x, y);
    }

    /**
     * 
@dev
 Returns the largest of two numbers.
     */
    function max(uint256 x, uint256 y) internal pure returns (uint256) {
        return select(x > y, x, y);
    }

    /**
     * 
@dev
 If `condition` is true returns `a`, otherwise returns `b`.
     */
    function select(bool condition, uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            // branchless select, works because:
            // b ^ (a ^ b) == a
            // b ^ 0 == b
            //
            // This is better than doing `condition ? a : b` because:
            // - Consumes less gas
            // - Constant gas cost regardless the inputs
            // - Reduces the final bytecode size
            return b ^ ((a ^ b) * toUint(condition));
        }
    }

    /**
     * 
@dev
 If `condition` is true returns `a`, otherwise returns `b`.
     */
    function select(bool condition, address a, address b) internal pure returns (address) {
        return address(uint160(select(condition, uint256(uint160(a)), uint256(uint160(b)))));
    }

    /**
     * 
@dev
 If `condition` is true return `value`, otherwise return zero.
     */
    function selectIf(bool condition, uint256 value) internal pure returns (uint256) {
        unchecked {
            return value * toUint(condition);
        }
    }

    /**
     * 
@dev
 Unsigned saturating addition, bounds to UINT256 MAX instead of overflowing.
     * equivalent to:
     * uint256 r = x + y;
     * return r >= x ? r : UINT256_MAX;
     */
    function saturatingAdd(uint256 x, uint256 y) internal pure returns (uint256) {
        unchecked {
            x = x + y;
            y = 0 - toUint(x < y);
            return x | y;
        }
    }

    /**
     * 
@dev
 Unsigned saturating subtraction, bounds to zero instead of overflowing.
     * equivalent to: x > y ? x - y : 0
     */
    function saturatingSub(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            // equivalent to: a > b ? a - b : 0
            return (a - b) * toUint(a > b);
        }
    }

    /**
     * 
@dev
 Unsigned saturating multiplication, bounds to `2 ** 256 - 1` instead of overflowing.
     */
    function saturatingMul(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            uint256 c = a * b;
            bool success;
            assembly {
                // Only true when the multiplication doesn't overflow
                // (c / a == b) || (a == 0)
                success := or(eq(div(c, a), b), iszero(a))
            }
            return c | (toUint(success) - 1);
        }
    }

    /**
     * 
@dev
 Unsigned saturating division, bounds to UINT256 MAX instead of overflowing.
     */
    function saturatingDiv(uint256 x, uint256 y) internal pure returns (uint256 r) {
        assembly {
            r := div(x, y)
        }
    }

    /**
     * 
@dev
 Returns the ceiling of the division of two numbers.
     *
     * This differs from standard division with `/` in that it rounds towards infinity instead
     * of rounding towards zero.
     */
    function ceilDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        unchecked {
            // The following calculation ensures accurate ceiling division without overflow.
            // Since a is non-zero, (a - 1) / b will not overflow.
            // The largest possible result occurs when (a - 1) / b is type(uint256).max,
            // but the largest value we can obtain is type(uint256).max - 1, which happens
            // when a = type(uint256).max and b = 1.
            return selectIf(a > 0, ((a - 1) / b + 1));
        }
    }

    /**
     * 
@dev
 Cast a boolean (false or true) to a uint256 (0 or 1) with no jump.
     */
    function toUint(bool b) internal pure returns (uint256 u) {
        /// @solidity memory-safe-assembly
        assembly {
            u := iszero(iszero(b))
        }
    }

    /**
     * 
@dev
 Cast an address to uint256
     */
    function toUint(address addr) internal pure returns (uint256) {
        return uint256(uint160(addr));
    }
}
```

2. Primitives.sol

```
// SPDX-License-Identifier: MIT
// Analog's Contracts (last updated v0.1.0) (src/Primitives.sol)

pragma solidity >=0.8.0;

import {BranchlessMath} from "./BranchlessMath.sol";

/**
 * 
@dev
 GmpSender is the sender of a GMP message
 */
type GmpSender is bytes32;

/**
 * 
@dev
 Tss public key
 * 
@param
 yParity public key y-coord parity, the contract converts it to 27/28
 * 
@param
 xCoord affine x-coordinate
 */
struct TssKey {
    uint8 yParity;
    uint256 xCoord;
}

/**
 * 
@dev
 Schnorr signature.
 * OBS: what is actually signed is: keccak256(abi.encodePacked(R, parity, px, nonce, message))
 * Where `parity` is the public key y coordinate stored in the contract, and `R` is computed from `e` and `s` parameters.
 * 
@param
 xCoord public key x coordinates, y-parity is stored in the contract
 * 
@param
 e Schnorr signature e component
 * 
@param
 s Schnorr signature s component
 */
struct Signature {
    uint256 xCoord;
    uint256 e;
    uint256 s;
}

/**
 * 
@dev
 GMP payload, this is what the timechain creates as task payload
 * 
@param
 source Pubkey/Address of who send the GMP message
 * 
@param
 srcNetwork Source chain identifier (for ethereum networks it is the EIP-155 chain id)
 * 
@param
 dest Destination/Recipient contract address
 * 
@param
 destNetwork Destination chain identifier (it's the EIP-155 chain_id for ethereum networks)
 * 
@param
 gasLimit gas limit of the GMP call
 * 
@param
 salt Message salt, useful for sending two messages with same content
 * 
@param
 data message data with no specified format
 */
struct GmpMessage {
    GmpSender source;
    uint16 srcNetwork;
    address dest;
    uint16 destNetwork;
    uint256 gasLimit;
    uint256 salt;
    bytes data;
}

/**
 * 
@dev
 Message payload used to revoke or/and register new shards
 * 
@param
 revoke Shard's keys to revoke
 * 
@param
 register Shard's keys to register
 */
struct UpdateKeysMessage {
    TssKey[] revoke;
    TssKey[] register;
}

/**
 * 
@dev
 Message payload used to revoke or/and register new shards
 * 
@param
 revoke Shard's keys to revoke
 * 
@param
 register Shard's keys to register
 */
struct Network {
    uint16 id;
    address gateway;
}

/**
 * 
@dev
 Status of a GMP message
 */
enum GmpStatus {
    NOT_FOUND,
    SUCCESS,
    REVERT,
    INSUFFICIENT_FUNDS,
    PENDING
}

/**
 * 
@dev
 EIP-712 utility functions for primitives
 */
library PrimitiveUtils {
    /**
     * 
@dev
 GMP message EIP-712 Type Hash.
     * Declared as raw value to enable it to be used in inline assembly
     * keccak256("GmpMessage(bytes32 source,uint16 srcNetwork,address dest,uint16 destNetwork,uint256 gasLimit,uint256 salt,bytes data)")
     */
    bytes32 internal constant GMP_MESSAGE_TYPE_HASH = 0xeb1e0a6b8c4db87ab3beb15e5ae24e7c880703e1b9ee466077096eaeba83623b;

    function toAddress(GmpSender sender) internal pure returns (address) {
        return address(uint160(uint256(GmpSender.unwrap(sender))));
    }

    function toSender(address addr, bool isContract) internal pure returns (GmpSender) {
        uint256 sender = BranchlessMath.toUint(isContract) << 160 | uint256(uint160(addr));
        return GmpSender.wrap(bytes32(sender));
    }

    // computes the hash of an array of tss keys
    function eip712hash(TssKey memory tssKey) internal pure returns (bytes32) {
        return keccak256(abi.encode(keccak256("TssKey(uint8 yParity,uint256 xCoord)"), tssKey.yParity, tssKey.xCoord));
    }

    // computes the hash of an array of tss keys
    function eip712hash(TssKey[] memory tssKeys) internal pure returns (bytes32) {
        bytes memory keysHashed = new bytes(tssKeys.length * 32);
        uint256 ptr;
        assembly {
            ptr := keysHashed
        }
        for (uint256 i = 0; i < tssKeys.length; i++) {
            bytes32 hash = eip712hash(tssKeys[i]);
            assembly {
                ptr := add(ptr, 32)
                mstore(ptr, hash)
            }
        }

        return keccak256(keysHashed);
    }

    // computes the hash of the fully encoded EIP-712 message for the domain, which can be used to recover the signer
    function eip712hash(UpdateKeysMessage memory message) internal pure returns (bytes32) {
        return keccak256(
            abi.encode(
                keccak256("UpdateKeysMessage(TssKey[] revoke,TssKey[] register)TssKey(uint8 yParity,uint256 xCoord)"),
                eip712hash(message.revoke),
                eip712hash(message.register)
            )
        );
    }

    function eip712TypedHash(UpdateKeysMessage memory message, bytes32 domainSeparator)
        internal
        pure
        returns (bytes32)
    {
        return _computeTypedHash(domainSeparator, eip712hash(message));
    }

    function eip712hash(GmpMessage memory message) internal pure returns (bytes32 id) {
        bytes memory data = message.data;
        /// @solidity memory-safe-assembly
        assembly {
            // keccak256(http://message.data)
            id := keccak256(add(data, 32), mload(data))

            // now compute the GmpMessage Type Hash without memory copying
            let offset := sub(message, 32)
            let backup := mload(offset)
            {
                mstore(offset, GMP_MESSAGE_TYPE_HASH)
                {
                    let offset2 := add(offset, 0xe0)
                    let backup2 := mload(offset2)
                    mstore(offset2, id)
                    id := keccak256(offset, 0x100)
                    mstore(offset2, backup2)
                }
            }
            mstore(offset, backup)
        }
    }

    function encodeCallback(GmpMessage calldata message, bytes32 domainSeparator)
        internal
        pure
        returns (bytes32 messageHash, bytes memory r)
    {
        bytes calldata data = message.data;
        /// @solidity memory-safe-assembly
        assembly {
            r := mload(0x40)

            // GmpMessage Type Hash
            mstore(add(r, 0x0004), GMP_MESSAGE_TYPE_HASH)
            mstore(add(r, 0x0024), calldataload(add(message, 0x00))) // message.source
            mstore(add(r, 0x0044), calldataload(add(message, 0x20))) // message.srcNetwork
            mstore(add(r, 0x0064), calldataload(add(message, 0x40))) // message.dest
            mstore(add(r, 0x0084), calldataload(add(message, 0x60))) // message.destNetwork
            mstore(add(r, 0x00a4), calldataload(add(message, 0x80))) // message.gasLimit
            mstore(add(r, 0x00c4), calldataload(add(message, 0xa0))) // message.salt

            // Copy http://message.data to memory
            let size := data.length
            mstore(add(r, 0x0104), size) // http://message.data length
            calldatacopy(add(r, 0x0124), data.offset, size) // http://message.data

            // Computed GMP Typed Hash
            messageHash := keccak256(add(r, 0x0124), size) // keccak(http://message.data)
            mstore(add(r, 0x00e4), messageHash)
            messageHash := keccak256(add(r, 0x04), 0x0100) // GMP eip712 hash
            mstore(0, 0x1901)
            mstore(0x20, domainSeparator)
            mstore(0x40, messageHash) // this will be restored at the end of this function
            messageHash := keccak256(0x1e, 0x42) // GMP Typed Hash

            // onGmpReceived
            size := and(add(size, 31), 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe0)
            size := add(size, 0xa4)
            mstore(add(r, 0x0064), 0x01900937) // selector
            mstore(add(r, 0x0060), size) // length
            mstore(add(r, 0x0084), messageHash) // GMP Typed Hash
            mstore(add(r, 0x00a4), calldataload(add(message, 0x20))) // http://msg.network
            mstore(add(r, 0x00c4), calldataload(add(message, 0x00))) // msg.source
            mstore(add(r, 0x00e4), 0x80) // http://msg.data offset

            size := and(add(size, 31), 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe0)
            size := add(size, 0x60)
            mstore(0x40, add(add(r, size), 0x40))
            r := add(r, 0x60)
        }
    }

    function eip712TypedHash(GmpMessage memory message, bytes32 domainSeparator)
        internal
        pure
        returns (bytes32 messageHash)
    {
        messageHash = eip712hash(message);
        messageHash = _computeTypedHash(domainSeparator, messageHash);
    }

    function _computeTypedHash(bytes32 domainSeparator, bytes32 messageHash) private pure returns (bytes32 r) {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0, 0x1901000000000000000000000000000000000000000000000000000000000000)
            mstore(0x02, domainSeparator)
            mstore(0x22, messageHash)
            r := keccak256(0, 0x42)
            mstore(0x22, 0)
        }
    }
}
```

3. IGateway.sol

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {GmpSender} from "./Primitives.sol";

/**
* 
@dev
 Required interface of an Gateway compliant contract
*/

/*
At Address - 0x000000007f56768de3133034fa730a909003a165

submitMessage:
gasLimit 30000
data     0x01
sepolia address 0xF871c929bE8Cd8382148C69053cE5ED1a9593EA7 and net 7
shibuya address 0xB5D83c2436Ad54046d57Cd48c00D619D702F3814 and net 5
*/

interface IGateway {
    event GmpCreated(
        bytes32 indexed id,
        bytes32 indexed source,
        address indexed destinationAddress,
        uint16 destinationNetwork,
        uint256 executionGasLimit,
        uint256 salt,
        bytes data
    );

    function networkId() external view returns (uint16);

    function deposit(GmpSender sender, uint16 sourceNetwork) external payable;

    function depositOf(GmpSender sender, uint16 sourceNetwork) external view returns (uint256);

    function submitMessage(
        address destinationAddress,
        uint16 destinationNetwork,
        uint256 executionGasLimit,
        bytes calldata data
    ) external payable returns (bytes32);
}
```



At Address - 0x000000007f56768de3133034fa730a909003a165

submitMessage: gasLimit 30000&#x20;

data 0x01&#x20;

sepolia address 0xF871c929bE8Cd8382148C69053cE5ED1a9593EA7 and net 7&#x20;

shibuya address 0xB5D83c2436Ad54046d57Cd48c00D619D702F3814 and net 5
