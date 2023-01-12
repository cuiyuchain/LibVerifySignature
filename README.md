# LibVerifySignature数字签名验签合约库

在FISCO BCOS原有验签库LibDecode.sol以及对应文档的基础上，做以下升级：

1.基于内联汇编处理字节运算的优势降低gas消耗为原来的64%

2.将适配Solidity版本由0.4.25提升至适配于FISCO BCOS 2.9.1的0.6.10版本

3.将原文档中基于geth+web3获取数字签名的部分修改为基于WeBASE-Front获取

4.修复了原合约只能对基于以太坊节点与web3生成的签名进行验签的Bug

5.修复了原合约还需要额外比较生成地址与公钥的缺陷，直接输出验签结果（原合约并不符合公钥验签的逻辑）

6.修正部分函数名的语义逻辑，避免与ABI解码混淆

7.基于FISCO BCOS EVM的特性，优化了椭圆曲线算法中恢复标识符的运算

8.增加了对于验签者身份的限定，更符合实际应用场景

9.标注中文注释，便于理解

文档正文如下：

## 合约源码

### 完整库嵌入主合约VerifySignature.sol

```solidity
pragma solidity ^0.6.10;
/**
* @author cuiyu
* @title  数字签名验签
**/
contract VerifySignature {
    // 参数0 验签者地址  参数1 签名者地址  参数3 消息哈希值  参数4 数字签名
    function verifySignature(address verifier, address publicKey, bytes32 hash, bytes memory signature) public pure returns (bool)
    {   
        //签名者与验签者不可相同
        require(verifier != publicKey, "invalid verifier");

        // 检查数字签名的长度是否为65字节（65 = 1 + 32 + 32）
        require(65 == signature.length, "invalid signature length!");

        // 因为需要调用ecrecover预编译函数，而ecrecover需要的参数包括数字签名的三个片段(r, s, v)
        bytes32 r; // signature的前32个字节  椭圆曲线数字签名输出
        bytes32 s; // signature的第33字节到第64字节  椭圆曲线数字签名输出
        uint8 v;   // signature的第65字节  恢复标识符，数值为27或28
        // 在椭圆曲线算法中，依靠r和s可计算出曲线上的多个点，因此会恢复出两个不同的公钥及其地址，因此需要基于v值进行选择

        string memory prefix = "\x19Ethereum Signed Message:\n32"; //前缀，可理解为ecrecover函数使用的格式要求

        // 内联汇编在处理字节运算时，具有较低的gas消耗       
        assembly {
            r := mload(add(signature, 32))         
            s := mload(add(signature, 64))
            // 在fisco bcos的EVM实现中，v在内部只是0x0或0x1，需要通过加27进行调整
            v := add(byte(0, mload(add(signature, 96))), 27) 

            if eq(or(eq(v, 27), eq(v, 28)), 0) { stop() }         
        }
        
        // ecrecover预编译函数可以计算出签名者地址，通过与publicKey进行比对，即可得到验签结果
        return (ecrecover(keccak256(abi.encodePacked(prefix, hash)), v, r, s) == publicKey);
        
    }  

    function testVerifySignature(address signerToVerify, bytes32 hash, bytes memory signature) public view returns (bool result) {
        return verifySignature(msg.sender, signerToVerify, hash, signature);
    }    

}

```

### 调库方式

LibVerifySignature.sol

```solidity
pragma solidity ^0.6.10;
/**
* @author cuiyu
* @title  数字签名验签
**/
library LibVerifySignature {
    // 参数0 验签者地址  参数1 签名者地址  参数3 消息哈希值  参数4 数字签名
    function verifySignature(address verifier, address publicKey, bytes32 hash, bytes memory signature) public pure returns (bool)
    {   
        //签名者与验签者不可相同
        require(verifier != publicKey, "invalid verifier");

        // 检查数字签名的长度是否为65字节（65 = 1 + 32 + 32）
        require(65 == signature.length, "invalid signature length!");

        // 因为需要调用ecrecover预编译函数，而ecrecover需要的参数包括数字签名的三个片段(r, s, v)
        bytes32 r; // signature的前32个字节  椭圆曲线数字签名输出
        bytes32 s; // signature的第33字节到第64字节  椭圆曲线数字签名输出
        uint8 v;   // signature的第65字节  恢复标识符，数值为27或28
        // 在椭圆曲线算法中，依靠r和s可计算出曲线上的多个点，因此会恢复出两个不同的公钥及其地址，因此需要基于v值进行选择

        string memory prefix = "\x19Ethereum Signed Message:\n32"; //前缀，可理解为ecrecover函数使用的格式要求

        // 内联汇编在处理字节运算时，具有较低的gas消耗       
        assembly {
            r := mload(add(signature, 32))         
            s := mload(add(signature, 64))
            // 在fisco bcos的EVM实现中，v在内部只是0x0或0x1，需要通过加27进行调整
            v := add(byte(0, mload(add(signature, 96))), 27) 

            if eq(or(eq(v, 27), eq(v, 28)), 0) { stop() }         
        }
        
        // ecrecover预编译函数可以计算出签名者地址，通过与publicKey进行比对，即可得到验签结果
        return (ecrecover(keccak256(abi.encodePacked(prefix, hash)), v, r, s) == publicKey);
        
    }      

}


```

TestVerifySignature.sol

```solidity
pragma solidity ^0.6.10;
import "./LibVerifySignature.sol";
contract TestVerifySignature {
    
    function testVerifySignature(address signerToVerify, bytes32 hash, bytes memory signature) public returns (bool result) {
        return LibVerifySignature.verifySignature(msg.sender, signerToVerify, hash, signature);
    }
}

```



## 使用方法

基于WeBASE-Front获取交易日志

![image](https://user-images.githubusercontent.com/87604354/211952533-2aa83344-7c33-4edd-b5fd-dfcaa77c00db.png)

记录交易hash值`0x4998cf9cdb953e35d309bd43c9152cf9984b55784f52742f129853eac1ed1e9c`
![image](https://user-images.githubusercontent.com/87604354/211952553-98bbb077-27a1-44bf-8b01-60541207b640.png)

基于以下AddPreFix.sol合约增加前缀

```solidity
pragma solidity ^0.6.10;
/**
* @author cuiyu
* @title  增加前缀
**/
contract AddPreFix {
    
    function addPreFix(bytes32 hash) public pure returns(bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    } 
        
}

```

得到增加前缀后的hash值`0x5caae3d75a157e2155346acea1f63c8010cba794435378c314c87d762ab4c995`

基于WeBASE-Front的数字签名工具，传入刚刚生成的hash，生成签名

![image](https://user-images.githubusercontent.com/87604354/211952587-c0474617-7426-4e93-b19d-d0638f86ca69.png)

依次将r s v组合为最终数字签名，s需要去掉开头的0x，当v为27时，用00表示v，参与组合；当v为28时，用01表示v，参与组合，得到最终数字签名：0x3569d5e2b508ebdb43bd4ebd90470405cf78d2d83058f51a394d8c8b8c98770805f9ba46a185ab6691ec680aad0c54c97ac61545cab5d857282b26b8718e701e00

获取签名时的用户地址：0xf0c757dd9393584a22b20a6b2e31597bb3e29938

![image](https://user-images.githubusercontent.com/87604354/211952609-8a2fc3bb-b9c9-48ea-920c-086f3f7c6573.png)

部署验签合约，调用testVerifySignature方法

![image](https://user-images.githubusercontent.com/87604354/211952619-c0616f1b-d4b9-4a8f-9c8b-1ee817f44f25.png)

依次传参：

0xf0c757dd9393584a22b20a6b2e31597bb3e29938

0x4998cf9cdb953e35d309bd43c9152cf9984b55784f52742f129853eac1ed1e9c

0x3569d5e2b508ebdb43bd4ebd90470405cf78d2d83058f51a394d8c8b8c98770805f9ba46a185ab6691ec680aad0c54c97ac61545cab5d857282b26b8718e701e00

![image](https://user-images.githubusercontent.com/87604354/211952631-45f8671c-4d7b-4666-9df0-453a771975cb.png)

验签成功

## 参考文献

https://github.com/ethereum/go-ethereum/issues/3731

https://web3dao-cn.github.io/solidity-example/signature/

https://ethereum.stackexchange.com/questions/15766/what-does-v-r-s-in-eth-gettransactionbyhash-mean

https://baijiahao.baidu.com/s?id=1681361604605948145

https://smartdev-doc.readthedocs.io/zh_CN/latest/docs/WeBankBlockchain-SmartDev-Contract/api/default/crypto/LibDecode.html

 













