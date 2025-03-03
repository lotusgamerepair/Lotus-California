// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract ThirdwebGateway is Ownable {
    using ECDSA for bytes32;

    event TransferStart(
        bytes32 indexed clientId,
        bytes32 indexed transactionId,
        address indexed from,
        address target,
        bytes data
    );

    event TransferEnd(
        bytes32 indexed clientId,
        bytes32 indexed transactionId,
        address indexed to,
        uint256 amount
    );

    event FeePayout(
        bytes32 indexed clientId,
        bytes32 indexed transactionId,
        address indexed recipient,
        uint256 amount
    );

    address public operator;
    bytes32 private DOMAIN_SEPARATOR;

    // EIP-712 typehash for ForwardRequest
    bytes32 private constant FORWARD_REQUEST_TYPEHASH =
        keccak256(
            "ForwardRequest(bytes32 clientId,bytes32 transactionId,address target,uint256 value,bytes data,address feeToken,uint256 feeTotalBps,address[] payees,uint256[] sharesBps,uint256 deadline)"
        );

    constructor(address _operator) {
        operator = _operator;
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
                keccak256("Thirdweb Gateway"),
                keccak256("1"),
                block.chainid,
                address(this)
            )
        );
    }

    function setOperator(address _newOperator) external onlyOwner {
        operator = _newOperator;
    }

    function forwardTransfer(
        bytes32 clientId,
        bytes32 transactionId,
        address target,
        uint256 value,
        bytes calldata data,
        address feeToken,
        uint256 feeTotalBps,
        address[] calldata payees,
        uint256[] calldata sharesBps,
        uint256 deadline,
        bytes calldata signature
    ) external payable {
        require(msg.value == value, "Value mismatch");
        require(block.timestamp <= deadline, "Transaction expired");

        bytes32 digest = hashForwardRequest(
            clientId,
            transactionId,
            target,
            value,
            data,
            feeToken,
            feeTotalBps,
            payees,
            sharesBps,
            deadline
        );

        address signer = digest.recover(signature);
        require(signer == operator, "Invalid signature");

        emit TransferStart(clientId, transactionId, msg.sender, target, data);

        if (feeToken == address(0)) {
            _handleNativeTransfer(clientId, transactionId, target, value, data, feeTotalBps, payees, sharesBps);
        } else {
            _handleERC20Transfer(clientId, transactionId, target, value, data, feeToken, feeTotalBps, payees, sharesBps);
        }
    }

    function _handleNativeTransfer(
        bytes32 clientId,
        bytes32 transactionId,
        address target,
        uint256 value,
        bytes calldata data,
        uint256 feeTotalBps,
        address[] calldata payees,
        uint256[] calldata sharesBps
    ) private {
        uint256 initialBalance = address(this).balance - msg.value;
        (bool success, ) = target.call{value: value}(data);
        require(success, "Call failed");
        
        uint256 received = address(this).balance - initialBalance;
        _distributeFees(clientId, transactionId, received, feeTotalBps, payees, sharesBps, address(0));
        
        uint256 remaining = address(this).balance;
        payable(msg.sender).transfer(remaining);
        emit TransferEnd(clientId, transactionId, msg.sender, remaining);
    }

    function _handleERC20Transfer(
        bytes32 clientId,
        bytes32 transactionId,
        address target,
        uint256 value,
        bytes calldata data,
        address feeToken,
        uint256 feeTotalBps,
        address[] calldata payees,
        uint256[] calldata sharesBps
    ) private {
        IERC20 token = IERC20(feeToken);
        uint256 initialBalance = token.balanceOf(address(this));
        
        (bool success, ) = target.call{value: value}(data);
        require(success, "Call failed");
        
        uint256 received = token.balanceOf(address(this)) - initialBalance;
        _distributeFees(clientId, transactionId, received, feeTotalBps, payees, sharesBps, feeToken);
        
        uint256 remaining = token.balanceOf(address(this));
        token.transfer(msg.sender, remaining);
        emit TransferEnd(clientId, transactionId, msg.sender, remaining);
    }

    function _distributeFees(
        bytes32 clientId,
        bytes32 transactionId,
        uint256 received,
        uint256 feeTotalBps,
        address[] calldata payees,
        uint256[] calldata sharesBps,
        address feeToken
    ) private {
        require(payees.length == sharesBps.length, "Invalid fee configuration");
        require(payees.length > 0, "No fee recipients");
        
        uint256 totalFee = (received * feeTotalBps) / 10000;
        require(totalFee <= received, "Fee exceeds amount");
        
        uint256 totalShares;
        for (uint256 i = 0; i < sharesBps.length; i++) {
            totalShares += sharesBps[i];
        }
        require(totalShares == 10000, "Invalid shares sum");
        
        if (feeToken == address(0)) {
            for (uint256 i = 0; i < payees.length; i++) {
                uint256 amount = (totalFee * sharesBps[i]) / 10000;
                payable(payees[i]).transfer(amount);
                emit FeePayout(clientId, transactionId, payees[i], amount);
            }
        } else {
            IERC20 token = IERC20(feeToken);
            for (uint256 i = 0; i < payees.length; i++) {
                uint256 amount = (totalFee * sharesBps[i]) / 10000;
                token.transfer(payees[i], amount);
                emit FeePayout(clientId, transactionId, payees[i], amount);
            }
        }
    }

    function hashForwardRequest(
        bytes32 clientId,
        bytes32 transactionId,
        address target,
        uint256 value,
        bytes calldata data,
        address feeToken,
        uint256 feeTotalBps,
        address[] calldata payees,
        uint256[] calldata sharesBps,
        uint256 deadline
    ) public view returns (bytes32) {
        bytes32 dataHash = keccak256(data);
        bytes32 payeesHash = keccak256(abi.encode(payees));
        bytes32 sharesHash = keccak256(abi.encode(sharesBps));
        
        bytes32 structHash = keccak256(
            abi.encode(
                FORWARD_REQUEST_TYPEHASH,
                clientId,
                transactionId,
                target,
                value,
                dataHash,
                feeToken,
                feeTotalBps,
                payeesHash,
                sharesHash,
                deadline
            )
        );
        
        return keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash));
    }

    function withdraw(address payable recipient, address token, uint256 amount) external onlyOwner {
        if (token == address(0)) {
            recipient.transfer(amount);
        } else {
            IERC20(token).transfer(recipient, amount);
        }
    }

    function emergencyWithdraw(address payable recipient, address token, uint256 amount) external {
        require(msg.sender == operator, "Unauthorized");
        if (token == address(0)) {
            recipient.transfer(amount);
        } else {
            IERC20(token).transfer(recipient, amount);
        }
    }

    receive() external payable {}
}