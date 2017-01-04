This tutorial will walk through setting up two Parity nodes locally and sending transactions between them. If you want to learn more about different parameters specified in this tutorial please refer to the [[Chain specification]] page.

Each node will act as an authority on the network issuing blocks when necessary, there will be also one additional user account with a high initial balance.

## 1. Get Parity

## 2. Choose your chain

We will run a chain with Authority Round consensus engine.

`"gasLimitBoundDivisor"` which determines gas limit adjustment has the usual Ethereum value  
`"stepDuration"` is set to at least 5 second block times  
`"authorities"` is empty right now since we are yet to create our authority accounts  

`"params"` standard for most chains

`"genesis"` has some standard values for the Authority Round consensus

`"accounts"` contains the standard Ethereum builtin contracts, these should be included to use the Solidity contract writing language

```
{
    "name": "DemoPoA",
    "engine": {
        "authorityRound": {
            "params": {
                "gasLimitBoundDivisor": "0x400",
                "stepDuration": "5",
                "authorities" : []
            }
        }
    },
    "params": {
        "maximumExtraDataSize": "0x20",
        "minGasLimit": "0x1388",
        "networkID" : "0x2323"
    },
    "genesis": {
        "seal": {
            "authorityRound": {
                "step": "0x0",
                "signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
            }
        },
        "difficulty": "0x20000",
        "gasLimit": "0x5B8D80"
    },
    "accounts": {
        "0x0000000000000000000000000000000000000001": { "balance": "1", "builtin": { "name": "ecrecover", "pricing": { "linear": { "base": 3000, "word": 0 } } } },
        "0x0000000000000000000000000000000000000002": { "balance": "1", "builtin": { "name": "sha256", "pricing": { "linear": { "base": 60, "word": 12 } } } },
        "0x0000000000000000000000000000000000000003": { "balance": "1", "builtin": { "name": "ripemd160", "pricing": { "linear": { "base": 600, "word": 120 } } } },
        "0x0000000000000000000000000000000000000004": { "balance": "1", "builtin": { "name": "identity", "pricing": { "linear": { "base": 15, "word": 3 } } } }
    }
}
```

Save the above under `demo-spec.json`.

## 3. Setting up the two nodes

Now that a bare bones chain specification is complete the two nodes can be set up. Parity stores accounts for each chain with a different genesis hash in a separate folder, so in order to create the accounts in the correct one we will need to run with the `--chain` option.
Normally the two nodes would be started on separate machines, however since we are using the same one we will need to resolve some possible collisions:  
- `-d` determines the directory that a Parity instance uses for data and keys
- `--port` determines the port via which Parity communicates with other nodes
- `--jsonrpc-port` is the RPC port
- `--ui-port` is the port used by the secure UI
- `--dapps-port` is the port used by Parity Dapps
We will also want to expose all RPC apis to make interacting with the nodes easier `--jsonrpc-apis web3, eth, net, personal, parity, parity_set, traces, rpc, parity_accounts`.

Putting it all together gives us the following command to start Parity:
```
parity  --chain demo-spec.json -d /tmp/parity0 --port 30300 --jsonrpc-port 8540 --ui-port 8180 --dapps-port 8080 --jsonrpc-apis web3, eth, net, personal, parity, parity_set, traces, rpc, parity_accounts
```

Since the command is becoming rather clunky we can use a config files instead, which are passed using `--config` option. Node 0 will have this config file saved under `node0.toml`:
```
[parity]
chain = "demo-spec.json"
db_path = "/tmp/parity0"
[network]
port = 30300
[rpc]
port = 8540
apis = ["web3", "eth", "net", "personal", "parity", "parity_set", "traces", "rpc", "parity_accounts"]
[ui]
port = 8180
[dapps]
port = 8080
```

While node 1 will have config saved under `node1.toml`:
```
[parity]
chain = "demo-spec.json"
db_path = "/tmp/parity1"
[network]
port = 30301
[rpc]
port = 8541
apis = ["web3", "eth", "net", "personal", "parity", "parity_set", "traces", "rpc", "parity_accounts"]
[ui]
port = 8181
[dapps]
port = 8081
```
Alternative config files can be generated [here](https://ethcore.github.io/parity-config-generator/).

We will create three accounts: two authorities and one user account. There are three main ways to create accounts, pick one that suits you best:
#### RPC
Start the node 0 using `parity --config node0.toml`.

RPC can accessed via `web3`, `parity.js` or simply using `curl`. This will create the first authority address:
```
curl -H "Content-Type: application/json" -X POST --data '{"jsonrpc":"2.0","method":"parity_newAccountFromPhrase","params":["node0", "node0"],"id":0}' localhost:8540
```
Returned address should be `0x00bd138abd70e2f00903268f3db08f2d25677c9e`.

The user address:
```
curl -H "Content-Type: application/json" -X POST --data '{"jsonrpc":"2.0","method":"parity_newAccountFromPhrase","params":["user", "user"],"id":0}' localhost:8540
```
Returned address should be `0x004ec07d2329997267ec62b4166639513386f32e`.

Now start the other node with `parity --config node1.toml` and create the second authority account:
```
curl -H "Content-Type: application/json" -X POST --data '{"jsonrpc":"2.0","method":"parity_newAccountFromPhrase","params":["node1", "node1"],"id":0}' localhost:8541
```
Returned address should be `0x00aa39d30f0d20ff03a22ccfc30b7efbfca597c2`.

#### UI
1. Start the node 0 using `parity --config node0.toml`
2. Go to `localhost:8080` in your browser and go through the initial setup
3. Click on "NEW ACCOUNT"
4. Pick "Recover account from recovery phrase", this will ensure the address will be the same as in this tutorial
5. Use the phrase "node0" and password "node0"
6. The new created account should have address `0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e`
7. Now recover from another phrase "user" and password "user" which will lead to account `0x004ec07d2329997267Ec62b4166639513386F32E`
8. Start the node 1 using `parity --config node1.toml`
9. Go to `localhost:8081`
10. Again recover from phrase "node1" and password "node1" which will create account `0x00Aa39d30F0D20FF03a22cCfc30B7EfbFca597C2`

#### `parity account new`
You can also create an account without starting Parity using:
```
parity account new --config node0.toml
```
This does not give you control over what address will be created, so for the remainder of the tutorial we will stick with the accounts created by the previous methods.

## 4. Complete the chain specification

If the account were created from phrases we should have the following setup now:
- node 0 has the `0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e` authority account and `0x004ec07d2329997267Ec62b4166639513386F32E` user account
- node 1 has the `0x00Aa39d30F0D20FF03a22cCfc30B7EfbFca597C2` authority account

The accounts can be now added to the spec file. Open `demo-spec.json` back up and add the authorities we have just created to the `"authorities"` array:
```
"authorities" : [
    "0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e",
    "0x00Aa39d30F0D20FF03a22cCfc30B7EfbFca597C2"
]
```

Then add our user account to the genesis, so that we have some balance to send around:
```
"0x004ec07d2329997267Ec62b4166639513386F32E": { "balance": "100000000" }
```

The complete `demo-spec.json` should look like this:
```
{
    "name": "DemoPoA",
    "engine": {
        "authorityRound": {
            "params": {
                "gasLimitBoundDivisor": "0x400",
                "stepDuration": "5",
                "authorities" : [
                    "0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e",
                    "0x00Aa39d30F0D20FF03a22cCfc30B7EfbFca597C2"
                ]
            }
        }
    },
    "params": {
        "maximumExtraDataSize": "0x20",
        "minGasLimit": "0x1388",
        "networkID" : "0x2323"
    },
    "genesis": {
        "seal": {
            "authorityRound": {
                "step": "0x0",
                "signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
            }
        },
        "difficulty": "0x20000",
        "gasLimit": "0x5B8D80"
    },
    "accounts": {
        "0x0000000000000000000000000000000000000001": { "balance": "1", "builtin": { "name": "ecrecover", "pricing": { "linear": { "base": 3000, "word": 0 } } } },
        "0x0000000000000000000000000000000000000002": { "balance": "1", "builtin": { "name": "sha256", "pricing": { "linear": { "base": 60, "word": 12 } } } },
        "0x0000000000000000000000000000000000000003": { "balance": "1", "builtin": { "name": "ripemd160", "pricing": { "linear": { "base": 600, "word": 120 } } } },
        "0x0000000000000000000000000000000000000004": { "balance": "1", "builtin": { "name": "identity", "pricing": { "linear": { "base": 15, "word": 3 } } } },
        "0x004ec07d2329997267Ec62b4166639513386F32E": { "balance": "100000000" }
    }
}
```

## 5. Run the nodes

Now that the spec is complete we can start the two nodes that will participate in the chain.


Putting it all together 

## 6. Connect them together

In order to ensure nodes are connected to each other we will need to know their [enode addresses](https://github.com/ethereum/wiki/wiki/enode-url-format). There are two ways to obtain them:  
- RPC
- UI
- startup console output

## 7. Send transactions

## 8. Further development