# Auth Contract

* [Background](#Background)
* [Application contract call rights management](#Application contract call rights management)
* [AuthContract interface design](#AuthContractinterfacedesign)
* [使用流程](#使用流程)
* [合约示例(C#版)](#合约示例(C#版))


## Background

当前，智能合约的函数可以被任何人调用，这显然不符合现实要求。基于角色的权限管理的基本思想是，每个角色可以调用部分函数，每个实体可以被赋予多种角色（实体是由其ONT ID来标识）。

如果智能合约需要增加权限管理功能，那就必须记录合约中分配的角色，以及角色可调用的函数，哪些实体具有该角色等等信息。这个工作比较繁琐，可交由一个系统合约来管理。

在下文中，我们称所有需要权限管理功能的合约为*应用合约*（App Contract），此文档描述的系统合约为*Auth合约*（Auth Contract）。

## 应用合约调用权限管理

*Auth合约*负责管理*应用合约*的函数调用权限。

- 记录应用合约的管理员信息，即记录`contract -> adminOntId`（kv类型数据）；

- 记录应用合约的所有已分配的角色，及角色对应可调用的函数列表，即`role -> []funcName`；

- 记录实体ONT ID的权限token（包含**有效时间**、**级别**、**角色**）列表，即`ontID -> [](role, expireTime, level)`。

## AuthContract接口设计

### a. 设置应用合约管理员

- 初始化合约管理员

  ```json 
  bool initContractAdmin(byte[] adminOntID);
  ```

  此方法应在应用合约内部调用。

- 转让合约管理权

	```json
	bool transfer(byte[] contractAddr, byte[] newAdminOntID, unsigned int keyNo);
	```
	contractAddr是目标合约的地址，
    将管理权完全转让，newAdminOntID成为新管理员。
    
    此函数必须由合约管理员调用，即将会以adminOntID名下编号为keyNo的公钥来验证交易签名是否合法。

### b. 验证合约调用

- 验证合约调用token的有效性
  	```json
  	bool verifyToken(byte[] contractAddr, byte[] callerOntID, byte[] funcName, unsigned int keyNo);
  	```

	合约调用token包含三个部分：合约地址，调用者的ontID，函数名，以及发起此次调用的公钥编号keyNo。
    
### c. 合约权限分配
- 为角色分配函数
	```json
	bool assignFuncsToRole(byte[] contractAddr, byte[] adminOntID, byte[] role, string[] funcNames, unsigned int keyNo);
	```
	contractAddr是目标合约的地址，adminOntID是目标合约的管理员ONT ID，role是角色，funcNames是目标合约中的函数名，keyNo是管理员发起此次调用所使用的公钥编号。
	
    必须由合约管理者调用，将所有函数自动绑定到role，若已经绑定，自动跳过，最后返回true。

- 绑定角色到实体身份
	```json
	bool assignOntIDsToRole(byte[] contractAddr, byte[] adminOntId, byte[] role, object[] ontIDs, unsigned int keyNo);
	```

	必须由合约管理者调用，ontIDs数组中的ONT ID被分配`role`角色，最后返回true。
	在当前实现中，权限token的级别level默认等于2。

### d. 合约权限代理
- 将合约调用权代理给其他人
    ```json
    bool delegate(byte[] contractAddr, byte[] from, byte[] to, byte[] role, int period, int level, unsigned int keyNo);
    
	bool withdraw(byte[] contractAddr, byte[] initiator, byte[] delegate,  byte[] role, unsigned int keyNo);
    ```
    
    角色拥有者可以将角色代理给其他人，`from`是转让者的ONT ID，`to`是代理人的ONT ID，`role`表示要代理的角色，`period`参数指定委托任期时间（以second为单位）。
   
    代理人可以再次将其角色代理给更多的人，`level`参数指定委托层次深度。例如，
     - level = 1: 此时代理人就无法将其角色再次代理出去；当前实现只支持此情况。

     角色拥有者可以提前将角色代理提前撤回，`initiator`是发起者，`delegate`是角色代理人，initiator将代理给delegate的角色提前撤回。


## 使用流程

1. 合约在初始化时通过调用`initContractAdmin`方法，设置此合约的管理员身份；
2. 合约管理者分配角色，并绑定角色可以调用的函数；
3. 合约管理者为OntID分配角色；
4. 合约的具体函数在执行之前，可以首先验证合约调用者是否拥有调用的权限，即验证该调用者是否提供了token；验证通过之后，可以执行具体函数。

## 合约示例(C#版)

```json
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;
using Neo.SmartContract.Framework.Services.System;
using System;
using System.ComponentModel;
using System.Numerics;

namespace Example
{
    public struct initContractAdminParam
    {
        public byte[] adminOntID;
    }

    public struct verifyTokenParam
    {
        public byte[] contractAddr;
        public byte[] caller;
        public byte[] fn;
        public int keyNo;
    }

    public class AppContract : SmartContract
    {
        public static readonly byte[] adminOntID = { 0x01, 0x02 };

        [Appcall("ff00000000000000000000000000000000000006")]//ScriptHash
        public static extern byte[] AuthContract(string op, object[] args);

        public static Object Main(string operation, object[] token, object[] args)
        {
            if (operation == "init") return init(args);
            
            
            if (operation == "foo")
            {
                //we need to check if the caller is authorized to invoke foo
                if (!verifyToken(operation, token)) return false;

                return foo(args);
            }

            return false; 
        }

        public static bool foo(object[] args)
        {
            return true;
        }

        public static bool init(object[] args)
        {
            object[] _args = new object[1]; 

            initContractAdminParam param;
            param.adminOntID = (byte[]) args[0];

            _args[0] = Neo.SmartContract.Framework.Helper.Serialize(param);
            byte[] ret = AuthContract("InitContractAdmin", _args);

            return ret[0] == 1;
        }

        public static bool verifyToken(string operation, object[] token)
        {
            object[] _args = new object[1];

            verifyTokenParam param;
            param.contractAddr = ExecutionEngine.ExecutingScriptHash;
            param.fn = operation.AsByteArray();
            param.caller = (byte[])token[0];
            param.keyNo = (int)token[1];

            _args[0] = param.Serialize();
            byte[] ret = AuthContract("verifyToken", _args);

            return ret[0] == 1;
        }
    }
}



```
	
 



