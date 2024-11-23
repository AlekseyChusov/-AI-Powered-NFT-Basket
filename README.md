pragma solidity ^0.8.4;

import "@openzeppelin/contracts/utils/introspection/IERC165.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/interfaces/IERC1271.sol";
import "@openzeppelin/contracts/utils/cryptography/SignatureChecker.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IERC6551Account {
    receive() external payable;

    function token() external view returns (uint256 chainId, address tokenContract, uint256 tokenId);

    function state() external view returns (uint256);

    function isValidSigner(address signer, bytes calldata context) external view returns (bytes4 magicValue);
}

interface IERC6551Executable {
    function execute(address to, uint256 value, bytes calldata data, uint8 operation)
        external
        payable
        returns (bytes memory);
}

contract ERC6551Account is IERC165, IERC1271, IERC6551Account, IERC6551Executable {
    uint256 immutable deploymentChainId = block.chainid;
    uint256 public state;

    struct Asset {
        address tokenAddress;
        uint256 amount;
    }

    Asset[] public assets;

    receive() external payable {}

    function execute(address to, uint256 value, bytes calldata data, uint8 operation)
        external
        payable
        virtual
        returns (bytes memory result)
    {
        require(_isValidSigner(msg.sender), "Invalid signer");
        require(operation == 0, "Only call operations are supported");

        ++state;

        (bool success, bytes memory returnedData) = to.call{value: value}(data);
        if (!success) {
            assembly {
                revert(add(returnedData, 32), mload(returnedData))
            }
        }
        result = returnedData; //  Присваиваем значение result здесь
    }

    function isValidSigner(address signer, bytes calldata) external view virtual returns (bytes4) {
        return _isValidSigner(signer) ? IERC6551Account.isValidSigner.selector : bytes4(0);
    }

    function isValidSignature(bytes32 hash, bytes memory signature) external view virtual returns (bytes4 magicValue) {
        return SignatureChecker.isValidSignatureNow(owner(), hash, signature)
            ? IERC1271.isValidSignature.selector
            : bytes4(0);
    }

    function supportsInterface(bytes4 interfaceId) external pure virtual returns (bool) {
        return
            interfaceId == type(IERC165).interfaceId ||
            interfaceId == type(IERC6551Account).interfaceId ||
            interfaceId == type(IERC6551Executable).interfaceId;
    }

    function token() public view virtual returns (uint256, address, uint256) {
        bytes memory footer = new bytes(0x60);

        assembly {
            extcodecopy(address(), add(footer, 0x20), 0x4d, 0x60)
        }

        return abi.decode(footer, (uint256, address, uint256));
    }

    function owner() public view virtual returns (address) {
        (uint256 chainId, address tokenContract, uint256 tokenId) = token();
        if (chainId != deploymentChainId) return address(0);

        return IERC721(tokenContract).ownerOf(tokenId);
    }

    function _isValidSigner(address signer) internal view virtual returns (bool) {
        return signer == owner();
    }

    function addAsset(address _tokenAddress, uint256 _amount) public {
        require(owner() == msg.sender, "Only owner can add assets");
        require(assets.length < 10, "Maximum 10 assets allowed");

        assets.push(Asset(_tokenAddress, _amount));
    }

    function getAssets() public view returns (Asset[] memory) {
        return assets;
    }
}
