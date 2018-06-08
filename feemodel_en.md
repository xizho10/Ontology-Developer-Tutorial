
<h1 align="center">Ontology smart contract fee model</h1>

## 1. GAS Limit
Gas limit is used to perform step counting in the opcode process when executing smart contract. In theory, the more complex the smart contract, the higher the amount of gas limit required, and the lowest number of Gas limit on the Ontology transaction is 30000.


## 2. GAS Price

The gas price is to price the opcode. The higher the gas price, the more priority the consensus node package the transaction.

## 3. Transaction Fee

The transaction fee is the product of gas limit and gas price. The actual transaction fee is divided into the following three situations:

### 1. The number of steps executing opcode equals to gas limit


**. transaction fee =  gas price * gas limit**

### 2. The number of steps executing opcode greater than gas limit

**transaction fee =  gas price * gas limit**

The transaction failed but gas will not be refunded

### 3. The number of steps executing opcode less than gas limit
**transaction fee =  gas price * (opcode实际消耗)**

Excess gas will be refunded


## 4. opcode 定价

| Function         | Gas Consumption |
| ---------------- | --------- |
| GetHeader        | 100       |
| GetBlock         | 200       |
| GetTransaciton   | 100       |
| GetContract      | 100       |
| Deploy contract  | 10000000  |
| Migrate contract | 10000000  |
| Get storage      | 100       |
| Put storage      | 1000/KB   |
| Delete storage   | 100       |
| Checkwitness     | 200       |
| Checksig         | 200       |
| AppCall          | 10        |
| TailCall         | 10        |
| SHA1             | 10        |
| SHA256           | 10        |
| HASH160          | 20        |
| HASH256          | 20        |
| Ordinary OPCODE       | 1         |




## Example

Use Ontology CLI to initiate a transfer transaction to demonstrate how to use gas price and gas limit. Ensure that there is sufficient ONG


- Check balance：

```
./Ontology asset balance TA7FwLmuX6qMcWTgZtUxt6tjzFgfaBM5sz
```

![image](./images/transferbefore.png)

- Transfer：
```
./Ontology asset transfer --from TA7FwLmuX6qMcWTgZtUxt6tjzFgfaBM5sz  --to TA7FwLmuX6qMcWTgZtUxt6tjzFgfaBM5sz  --amount 1000 --gasprice 5 --gaslimit 40000
```

![image](./images/transferafter.png)

You can see that the transfer amount = 1000 ONT, gas price = 5, gas limit = 40000, the final consumption of gas = 1787019.99985 - 1787019.9997 = 0.00015

