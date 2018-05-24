# Auth Contract

* [Background](#Background)
* [Application contract calls rights management](#Application contract calls rights management)
* [Auth Contract interface design](#Auth Contract interface design)
* [Workflow](#Workflow)
* [Contract Example (C#)](#Contract Example (C#))

## Background

Currently, the function of smart contract can be called by anyone, which obviously does not meet the actual requirements. The basic idea of ​​role-based rights management is that each role can call a partial function, and each entity can be assigned multiple roles (the entity is identified by its ONT ID).

If the smart contract needs to add the rights management function, it must record the roles assigned in the contract, the functions that the role can call, which entity has this role, and so on. This work is tedious and can be managed by a system contract.

In the following, we say that the contract that requires rights management functions are *App Contract*, and the system contract described in this document are *Auth Contract*.

## Application contract calls rights management

The *Auth contract* is responsible for managing the function call permissions of the *application contract*.

- Record the administrator information of the application contract, i.e. record `contract -> adminOntId` (kv type data);

- Record all assigned roles of the application contract, and the list of functions that the role can call, i.e. `role -> []funcName`;

- 记录实体ONT ID的权限token（包含**有效时间**、**级别**、**角色**）列表，即`ontID -> [](role, expireTime, level)`。

- Record permission token of the entity's ONT ID (including **valid time**, **level**, **role**) list, i.e. `ontID -> [] (role, expireTime, level)`.


## Auth Contract interface design

### a. Set up the administrator of an application contract 

- Initialize the contract administrator

  ```json 
  bool initContractAdmin(byte[] adminOntID);
  ```

  This method should be called inside the application contract.

- Transfer contract management rights

	```json
	bool transfer(byte[] contractAddr, byte[] newAdminOntID, unsigned int keyNo);
	```
	contractAddr is the address of the target contract.
    When the management rights are completely transferred, newAdminOntID becomes the new administrator.
    
    This function must be called by the contract administrator, and the transaction signature is verified by the public key with the number keyNo of the adminOntID.
    
### b. Verify the contract call

- Verify the validity of the contract call

  	```json
  	bool verifyToken(byte[] contractAddr, byte[] callerOntID, byte[] funcName, unsigned int keyNo);
  	```

	Calling token by contract consists of three parts：the contract address, the caller's onID, the function name, and the public key number keyNo that initiated the call.
    
### c. Contract rights allocation
- Assign a function to a role

	```json
	bool assignFuncsToRole(byte[] contractAddr, byte[] adminOntID, byte[] role, string[] funcNames, unsigned int keyNo);
	```
	contractAddr is the address of the target contract, adminOntID is the administrator's ONT ID of the target contract, funcNames is the function name of the target contract, and keyNo is the public key number used by the administrator to initiate this call.
	
    This function must be called by the contract administrator, and it will automatically bind all functions to the role. if it is already bounded, the binding procedure automatically skip, and finally return true.

- Bind a role to a entity identity
	```json
	bool assignOntIDsToRole(byte[] contractAddr, byte[] adminOntId, byte[] role, object[] ontIDs, unsigned int keyNo);
	```

	This function must be called by the contract administrator. The ONT ID in the ontIDs array is assigned the `role` and finally returns true.
        In the current implementation, the level of the permission token is equal to 2 by default.

### d. Contract permission delegate
- Delegate contract calling rights to others
    ```json
    bool delegate(byte[] contractAddr, byte[] from, byte[] to, byte[] role, int period, int level, unsigned int keyNo);
    
	bool withdraw(byte[] contractAddr, byte[] initiator, byte[] delegate,  byte[] role, unsigned int keyNo);
    ```
    
    The role owner can delegate the role to others. `from` is the ONT ID of the transferor, `to` is the ONT ID of the delegator, `role` is the role of delegator, and the `period` parameter specifies the duration of the delegation. (use second as the unit).
   
      The delegator can delegate his role to more people, and the parameter `level` specifies the depth of the delegation level. E.g,
    - level = 1: The delegator cannot delegate his role to others; the current implementation only supports this situation.

     The role owner can withdraw the role delegation in advance. `initiator` is the initiator, `delegate` is the role delegator, and the initiator can withdraw the role from the delegator in advance.

## Auth Contract interface design

1. At initialization, the contract sets up the administrator by calling the `initContractAdmin` method;
2. The contract administrator assigns roles and binds functions that each role can call;
3. The contract administrator assigns roles to OntID;
4. Before the specific function of the contract is executed, you can first verify whether the contract caller has the permission to call, that is, verify whether the caller provides the token; After the verification passes, you can execute the specific function.

## Contract Example(C#)

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
	
 



