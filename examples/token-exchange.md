#  Token兑换服务dApp

Token兑换服务dApp,提供Ontology上Oep5规范的token相互兑换的服务，收取少量手续费。

## 贡献者

* Lucas
* Mickey

## 功能介绍

* 用户可以选择需要兑换的Oep5的token
* 服务商可以设置token兑换的手续费

## 页面示例

主页面：

![token-exchange](images/token-exchange.jpg)

兑换页面：

![token-exchange2](images/token-exchange2.jpg)

用户确认签名页面：

![token-exchange3](images/token-exchange3.jpg)

## 合约

Token互换合约实现的功能是：对于符合oep5标准的Token实现互换

### 用法

1. 调用SetContractHash方法将TokenA和TokenB的合约hash保存到互换合约中

> Note: TokenA和TokenB的合约地址要先进行反转，然后在调用SetContractHash

2. 调用GetContractHash方法查询TokenA和TokenB的合约地址信息

> Note:查询结果应该是TokenA和TokenB反转后的地址

3. 调用SetTokenBase方法设置TokenA和TokenB的互换比例
4. 调用GetTokenBase查询互换比例
5. 调用SetFeeRate设置手续费率
6. 调用GetFeeRate查询手续费率
7. 将TokenA发行的代币一部分打给互换合约地址，将TokenB发行的代币一部分打给互换合约地址（也就是各种代币在互换合约中预存一部分Token，用于用户兑换）

> Note: 往互换合约中打钱时，一定要现将互换合约的地址进行反转，再打钱

8. 持有TokenA的用户调用Exchange方法兑换TokenB

### 合约源代码
```c#
using Ont.SmartContract.Framework;
using Ont.SmartContract.Framework.Services.Ont;
using Ont.SmartContract.Framework.Services.System;
using System;
using System.ComponentModel;
using System.Numerics;

namespace ExchangeContract
{
    public class ExchangeContract : SmartContract
    {
        // 合约管理员，设置Token兑换比率，手续费等操作
        //public static readonly byte[] admin = "AT4dGijAQSMcN5mj2q7ePTWcKFEnG1qeu3".ToScriptHash();
        public static readonly byte[] admin = {123,212,116,117,198,162,213,5,136,227,129,61,35,74,2,183,243,175,138,83};

        public static readonly byte[] basePrefix = "base_".AsByteArray();

        public delegate object OEP5Contract(string method, object[] args);

        public static Object Main(string operation, params object[] args)
        {
            if (operation == "Exchange")
            {
                if (args.Length != 5) return false;
                byte[] from = (byte[])args[0];
                byte[] to = (byte[])args[1];
                string fromSymbol = (string)args[2];
                string toSymbol = (string)args[3];
                ulong value = (ulong)args[4];
                return Exchange(from, to, fromSymbol, toSymbol, value);
            }


            // 设置token兑换比率，比如1TokenA:2TokenB,symbol=tokena, base=1,symbol=tokenb,base=2
            if (operation == "SetTokenBase")
            {
                if (args.Length != 2) return false;
                string symbol = (string)args[0];
                BigInteger value = (BigInteger)args[1];
                return SetTokenBase(symbol, value);
            }

            // 获取Token的兑换比率（百分比）
            if (operation == "GetTokenBase")
            {
                if (args.Length != 1) return false;
                string symbol = (string)args[0];
                return GetTokenBase(symbol);
            }
            // 设置手续费百分比率，例如5%则percentage=5
            if (operation == "SetFeeRate")
            {
                if (args.Length != 1) return false;
                ulong percentage = (ulong)args[0];
                return SetFeeRate(percentage);
            }

            // 获取当前手续费比率（百分比）
            if (operation == "GetFeeRate")
            {
                return GetFeeRate();
            }
            // 设置合约Hash
            if (operation == "SetContractHash")
            {
                if (args.Length != 2) return false;
                string contractKey = (string)args[0];
                byte[] hash = (byte[])args[1];
                return SetContractHash(contractKey, hash);
            }
            // 获取指定商户的合约hash
            if (operation == "GetContractHash")
            {
                if (args.Length != 1) return false;
                string contractKey = (string)args[0];
                return GetContractHash(contractKey);
            }
            return false;
        }
        // from应该是兑换目的token发起方的账户，to 是目的token的回收账户
        // fromSymbol是发起方token的符号，比如TokenA,"tokena", toSymbol同理
        // value 是兑换token的数量
        public static bool Exchange(byte[] from, byte[] to, string fromSymbol, string toSymbol, ulong value)
        {
            if (value < 0) return false;
            if (from.Length != 20 || to.Length != 20) return false;

            byte[] fromHash = GetContractHash(fromSymbol);
            byte[] toHash = GetContractHash(toSymbol);
            if (fromHash.Length != 20 || toHash.Length != 20) return false;

            byte[] contract = ExecutionEngine.ExecutingScriptHash;

            StorageContext context = Storage.CurrentContext;

            ulong fromBase = (ulong)Storage.Get(context, basePrefix.Concat(fromSymbol.AsByteArray())).AsBigInteger();
            ulong toBase = (ulong)Storage.Get(context, basePrefix.Concat(toSymbol.AsByteArray())).AsBigInteger();

            if (fromBase <= 0 || toBase <= 0)
            {
                Runtime.Notify("Invalid token convertion base.");
                return false;
            }

            ulong feeRate = (ulong)Storage.Get(context, "feeRate").AsBigInteger();

            if (feeRate < 0)
            {
                Runtime.Notify("Invalid fee rate.");
                return false;
            }

            ulong toValue = (value * toBase / fromBase) * (100 - feeRate) / 100;

            if (!TransferOEP5(from, to, fromHash, value)) throw new Exception();

            if (!TransferOEP5(contract, from, toHash, toValue)) throw new Exception();

            return true;
        }

        public static bool SetTokenBase(string symbol, BigInteger value)
        {
            if (value <= 0) return false;
            if (!Runtime.CheckWitness(admin)) return false;

            byte[] baseKey = basePrefix.Concat(symbol.AsByteArray());
            Storage.Put(Storage.CurrentContext, baseKey, value);
            return true;
        }

        public static BigInteger GetTokenBase(string symbol)
        {
            return Storage.Get(Storage.CurrentContext, basePrefix.Concat(symbol.AsByteArray())).AsBigInteger();
        }

        public static bool SetFeeRate(ulong percentage)
        {
            if (percentage < 0 || percentage >= 100) return false;
            if (!Runtime.CheckWitness(admin)) return false;

            Storage.Put(Storage.CurrentContext, "feeRate", percentage);
            return true;
        }

        public static BigInteger GetFeeRate()
        {
            return Storage.Get(Storage.CurrentContext, "feeRate").AsBigInteger();
        }

        public static bool SetContractHash(string key, byte[] hash)
        {
            if (!Runtime.CheckWitness(admin)) return false;
            if (key == "" || key.Length == 0 || hash.Length != 20) return false;

            Storage.Put(Storage.CurrentContext, key, hash);
            return true;
        }

        public static byte[] GetContractHash(string key)
        {
            return Storage.Get(Storage.CurrentContext, key);
        }

        private static bool TransferOEP5(byte[] from, byte[] to, byte[] assetID, BigInteger amount)
        {
            var args = new object[] { from, to, amount };
            var contract = (OEP5Contract)assetID.ToDelegate();
            if (!(bool)contract("transfer", args)) return false;
            return true;
        }
    }
}
```

## contract instruction

* Set the Token base to calculate the conversion ratio of different Tokens.

```
public static bool SetTokenBase(string symbol, BigInteger value)
```

|parameter name|parameter description|
|:--|:--|
|symbol|Token symbol|
|value|Token base|


For example, if the Token base of TokenA is set to 1, and the Token base of TokenB is set to 2, the ratio of Token1 and Token2 when redeeming is 1:2.

* query Token base

```
public static BigInteger GetTokenBase(string symbol)
```

parameter description

|parameter name|parameter description|
|:--|:--|
|symbol|Token symbol|


* setting fee

```
public static bool SetFeeRate(ulong percentage)
```

parameter description

|parameter name|parameter description|
|:--|:--|
|percentage|fee|


For example, percentage=5, indicating that the handling fee is 5%.

* query fee

```
public static BigInteger GetFeeRate()
```

* setting Token contract hash

```
 public static bool SetContractHash(string key, byte[] hash)
```

parameter description

|parameter name|parameter description|
|:--|:--|
|key|Token contract name|
|hash|Token contract hash|

Note: When setting the Token contract address, you need to reverse the Token address.

* query contract hash

```
public static byte[] GetContractHash(string key)
```

parameter description

|parameter name|parameter description|
|:--|:--|
|key|Token name|


* exchanging of different Tokens

```
public static bool Exchange(byte[] from, byte[] to, string fromSymbol, string toSymbol, ulong value)
```

parameter description

|parameter name|parameter description|
|:--|:--|
|from|Redeem the account of the Token originator|
|to|Purpose Token's recycling account|
|fromSymbol|The symbol of the originator Token|
|toSymbol|Token symbol to be redeemed|
|value|Number of redemptions|

