
CSPR Transfer Workflow
======================

This document describes a sample workflow for transferring tokens and verifying their transfer on a Casper network as of `Release-1.0.0 <https://github.com/CasperLabs/casper-node/tree/release-1.0.0>`_.

Requirements
^^^^^^^^^^^^

To follow the steps in this document, you will need:

1. A compatible client or SDK such as the `JavaScript SDK <https://www.npmjs.com/package/casper-client-sdk>`_, `Java SDK <https://github.com/cnorburn/casper-java-sdk>`_, or GoLang SDK (location forthcoming)
2. The public key (hex) of 2 accounts (one source account, one target account)
3. A node RPC endpoint to send the transaction
4. The transfer amount needs to be 2.5 CSPR or more
5. The transfer will cost 0.0001 CSPR (10000 motes)  

The Rust Casper Client
^^^^^^^^^^^^^^^^^^^^^^

In this document, we will use the Rust Casper client to transfer tokens. You can find the client on `crates.io <https://crates.io/crates/casper-client>`_. 

Run the commands below to install the Casper client on most flavors of Linux and macOS. You will need the nightly version of the compiler.

.. code-block:: bash

  rustup toolchain install nightly 
  cargo +nightly install casper-client

The Casper client can print out `help` information, which provides an up-to-date list of supported commands. 

.. code-block:: bash

    casper-client --help

**Important**: For each command, you can use `help` to get the up-to-date arguments and descriptions:

.. code-block:: bash

    casper-client <command> --help

Setting up Accounts on Testnet
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
We recommended starting your integration by using the `Casper testnet <https://docs.cspr.community/docs/testnet.html>`_.

Accounts for the testnet can be created using the Rust Casper client or `the Block Explorer <https://clarity-testnet-old.make.services/#/>`_.

You need to create one account for the source of the transfer and one for the target account.

Option 1: Account setup using the Casper client
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This option describes how you can use the Rust Casper client to set up your accounts.

Execute the following command to generate your keys:

.. code-block:: bash

    casper-client keygen .

The above command will create three files in the current working directory:

1. ``secret_key.pem`` - PEM encoded secret key
2. ``public_key.pem`` - PEM encoded public key
3. ``public_key_hex`` - Hex encoded string of the public key

**Important Note**: SAVE your keys to a safe place, preferably offline.

Next, go to `the Block Explorer <https://clarity-testnet-old.make.services/#/>`_ to upload your public key. Log in using your Github or Google account. 

**Important Note**: Do NOT, EVER, upload your secret key.

Now you are ready to fund your account and `request tokens <#fund-your-account>`_.

**Important Note**: Responses from the node contain ``AccountHashes`` instead of the direct hex encoded public key. For traceability, it is important to generate the account hash and store this value locally. The account hash is a ``Blake2B`` hash of the public hex key.

Option 2: Account setup using the Block Explorer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Start by creating an account on Clarity using the `Create Account <https://clarity-testnet-old.make.services/#/accounts>`_ link.

Save these three files for each account and note the location where they are downloaded. We recommend moving your keys to a safe location, preferrably offline.

1. ``<Account-Name>_secret_key.pem`` - PEM encoded secret key
2. ``<Account-Name>_public_key.pem`` - PEM encoded public key
3. ``<Account-Name>_public_key_hex`` - Hex encoded string of the public key

**Note**: You need the `<Account-Name>_public_key_hex` in order to verify your account balance when querying the blockchain later.

Fund your Account
^^^^^^^^^^^^^^^^^

Next, you need to fund the source account using the ``[Request tokens]`` button on the `Faucet Page <https://clarity-testnet-old.make.services/#/faucet>`_ to receive tokens.

Acquire Node IP Address
^^^^^^^^^^^^^^^^^^^^^^^

You can get an IP address of a node on the network by visiting the `Peers Page <https://testnet.cspr.live/tools/peers>`_. You will see a list of peers, and you can select the IP of any peer on the list.

**Note**: If the selected peer is blocking the port, pick a different peer and try again.

You also have the option to run your own un-bonded peer on the network. Follow the `Casper How-To Guides <https://docs.cspr.community/>`_ for the testnet or mainnet, and skip the last step, which bonds the node to the network.

Transfer Funds
^^^^^^^^^^^^^^

Clients can communicate with nodes on the network via JSON-RPC requests sent to a node's RPC endpoint ``http://<peer-ip-address>:7777``. JSON-RPC requests include transfers which are a special type of deploy.

The ``transfer`` command below demonstrates how to transfer from a source account to a target account using the Rust client by sending a request to the selected node's RPC endpoint.

**Transfer Requirements**:

1. Each transfer amount needs to be 2.5 CSPR or more
2. Each transfer costs 0.0001 CSPR (10000 motes). The transfer cost is fixed, and a different ``payment-amount`` will be ignored

When sending a transfer, **use the required ``transfer-id`` field in the request to identify and track transactions** in your platform. For example, you might store transactions in a database in a Transaction table. The primary key of this table could be a TransactionID. You can set the ``transfer-id`` in the transfer request below to be the TransactionID from your database table. This way, you can use the ``transfer-id`` field to tag the transaction and correlate it to your back-end storage.

**Note**: ``transfer-id`` is a ``u64`` field and can be set using the required ``--transfer-id`` flag in the example command below.

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``transfer-id`` - <64-BIT INTEGER> Required user-defined transfer id
- ``node-address`` - <HOST:PORT> Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]
- ``amount`` - <512-BIT INTEGER> The number of motes to transfer
- ``secret-key`` - Path to secret key file
- ``chain-name`` - Name of the chain, to avoid the deploy from being accidentally or maliciously included in a different chain
  - The *chain-name* for testnet is **casper-test**
  - The *chain-name* for mainnet is **casper**
- ``target-account`` - <HEX STRING> Hex-encoded public key of the account from which the main purse will be used as the target

::

    casper-client transfer \
        --id 1 \
        --transfer-id 123456789012345 \
        --node-address http://<peer-ip-address>:7777 \
        --amount <amount-to-transfer> \
        --secret-key <source-account-secret-key>.pem \
        --chain-name casper \
        --target-account <hex-encoded-target-account-public-key>

**Important response fields:**

- ``"result"."deploy_hash"`` - the address of the executed transfer, needed to look up additional information about the transfer

**Note**: Save the returned `deploy_hash` from the output to query information about the transfer deploy later.

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 1,
      "jsonrpc": "2.0",
      "method": "account_put_deploy",
      "params": {
        "deploy": {
          "approvals": [
            {
              "signature": "130 chars",
              "signer": "010f50b0116f213ef65b99d1bd54483f92bf6131de2f8aceb7e3f825a838292150"
            }
          ],
          "hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
          "header": {
            "account": "010f50b0116f213ef65b99d1bd54483f92bf6131de2f8aceb7e3f825a838292150",
            "body_hash": "da35b095640a403324306c59ac6f18a446dfcc28faf753ce58b96b635587dd8e",
            "chain_name": "casper-net-1",
            "dependencies": [],
            "gas_price": 1,
            "timestamp": "2021-04-20T18:04:40.333Z",
            "ttl": "1h"
          },
          "payment": {
            "ModuleBytes": {
              "args": [
                [
                  "amount",
                  {
                    "bytes": "021027",
                    "cl_type": "U512",
                    "parsed": "10000"
                  }
                ]
              ],
              "module_bytes": ""
            }
          },
          "session": {
            "Transfer": {
              "args": [
                [
                  "amount",
                  {
                    "bytes": "0400f90295",
                    "cl_type": "U512",
                    "parsed": "2500000000"
                  }
                ],
                [
                  "target",
                  {
                    "bytes": "8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
                    "cl_type": {
                      "ByteArray": 32
                    },
                    "parsed": "8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668"
                  }
                ],
                [
                  "id",
                  {
                    "bytes": "00",
                    "cl_type": {
                      "Option": "U64"
                    },
                    "parsed": null
                  }
                ]
              ]
            }
          }
        }
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 1,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "deploy_hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713"
      }
    }

.. raw:: html

    </details>

|

Deploy Status
~~~~~~~~~~~~~

Once a transaction (deploy) has been submitted to the network, it is possible to check its execution status using ``get-deploy``. 

If the ``execution_results`` in the output are null, the transaction hasn't run yet. Transactions are finalized upon execution.

**Important request fields:**

- ``id`` - <STRING OR INTEGER> JSON-RPC identifier, applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``node-address`` - <HOST:PORT>Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]

::

    casper-client get-deploy \
          --id 2 \
          --node-address http://<peer-ip-address>:7777 \
          <deploy-hash>


**Important response fields:**

- ``"result"."execution_results"[0]."transfers[0]"`` - the address of the executed transfer that the source account initiated. We will use it to look up additional information about the transfer
- ``"result"."execution_results"[0]."block_hash"`` - contains the block hash of the block that included our transfer. We will require the `state_root_hash` of this block to look up information about the accounts and their balances

**Note**: Transfer addresses use a ``transfer-`` string prefix.

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
    "id": 2,
    "jsonrpc": "2.0",
    "method": "info_get_deploy",
    "params": {
      "deploy_hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713"
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 2,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "deploy": {
          "approvals": [
            {
              "signature": "130 chars",
              "signer": "010f50b0116f213ef65b99d1bd54483f92bf6131de2f8aceb7e3f825a838292150"
            }
          ],
          "hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
          "header": {
            "account": "010f50b0116f213ef65b99d1bd54483f92bf6131de2f8aceb7e3f825a838292150",
            "body_hash": "da35b095640a403324306c59ac6f18a446dfcc28faf753ce58b96b635587dd8e",
            "chain_name": "casper-net-1",
            "dependencies": [],
            "gas_price": 1,
            "timestamp": "2021-04-20T18:04:40.333Z",
            "ttl": "1h"
          },
          "payment": {
            "ModuleBytes": {
              "args": [
                [
                  "amount",
                  {
                    "bytes": "021027",
                    "cl_type": "U512",
                    "parsed": "10000"
                  }
                ]
              ],
              "module_bytes": ""
            }
          },
          "session": {
            "Transfer": {
              "args": [
                [
                  "amount",
                  {
                    "bytes": "0400f90295",
                    "cl_type": "U512",
                    "parsed": "2500000000"
                  }
                ],
                [
                  "target",
                  {
                    "bytes": "8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
                    "cl_type": {
                      "ByteArray": 32
                    },
                    "parsed": "8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668"
                  }
                ],
                [
                  "id",
                  {
                    "bytes": "00",
                    "cl_type": {
                      "Option": "U64"
                    },
                    "parsed": null
                  }
                ]
              ]
            }
          }
        },
        "execution_results": [
          {
            "block_hash": "7c7e9b0f087bba5ce6fc4bd067b57f69ea3c8109157a3ad7f6d98b8da77d97f9",
            "result": {
              "Success": {
                "cost": "10000",
                "effect": {
                  "operations": [
                    {
                      "key": "hash-d13610d5930fdab36fc25838bc8b4b77fdb4859755dd628c2d30e2a6dfc86a8c",
                      "kind": "Read"
                    },
                    {
                      "key": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
                      "kind": "Read"
                    },
                    {
                      "key": "balance-39b6cc617efddbcc5e989c9eb73ddb5d825bb1070309e7429c029826074e038a",
                      "kind": "Read"
                    },
                    {
                      "key": "balance-9e90f4bbd8f581816e305eb7ea2250ca84c96e43e8735e6aca133e7563c6f527",
                      "kind": "Write"
                    },
                    {
                      "key": "deploy-ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
                      "kind": "Write"
                    },
                    {
                      "key": "balance-34ec8bcae2675d16bad7e8ba10fada1e50dacf3935ce3b12c25a5bf000fefc76",
                      "kind": "Write"
                    },
                    {
                      "key": "transfer-8d81f4a1411d9481aed9c68cd700c39d870757b0236987bb6b7c2a7d72049c0e",
                      "kind": "Write"
                    },
                    {
                      "key": "hash-1e13f06cb64bcbf46348dc53c35444da5afc956cfd764cbc3399dc71692e0bd8",
                      "kind": "Read"
                    },
                    {
                      "key": "balance-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e",
                      "kind": "Write"
                    }
                  ],
                  "transforms": [
                    {
                      "key": "balance-39b6cc617efddbcc5e989c9eb73ddb5d825bb1070309e7429c029826074e038a",
                      "transform": "Identity"
                    },
                    {
                      "key": "deploy-ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
                      "transform": {
                        "WriteDeployInfo": {
                          "deploy_hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
                          "from": "account-hash-b0049301811f23aab30260da66927f96bfae7b99a66eb2727da23bf1427a38f5",
                          "gas": "10000",
                          "source": "uref-9e90f4bbd8f581816e305eb7ea2250ca84c96e43e8735e6aca133e7563c6f527-007",
                          "transfers": [
                            "transfer-8d81f4a1411d9481aed9c68cd700c39d870757b0236987bb6b7c2a7d72049c0e"
                          ]
                        }
                      }
                    },
                    {
                      "key": "hash-1e13f06cb64bcbf46348dc53c35444da5afc956cfd764cbc3399dc71692e0bd8",
                      "transform": "Identity"
                    },
                    {
                      "key": "transfer-8d81f4a1411d9481aed9c68cd700c39d870757b0236987bb6b7c2a7d72049c0e",
                      "transform": {
                        "WriteTransfer": {
                          "amount": "2500000000",
                          "deploy_hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
                          "from": "account-hash-b0049301811f23aab30260da66927f96bfae7b99a66eb2727da23bf1427a38f5",
                          "gas": "0",
                          "id": null,
                          "source": "uref-9e90f4bbd8f581816e305eb7ea2250ca84c96e43e8735e6aca133e7563c6f527-007",
                          "target": "uref-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e-004",
                          "to": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668"
                        }
                      }
                    },
                    {
                      "key": "balance-34ec8bcae2675d16bad7e8ba10fada1e50dacf3935ce3b12c25a5bf000fefc76",
                      "transform": {
                        "AddUInt512": "10000"
                      }
                    },
                    {
                      "key": "hash-d13610d5930fdab36fc25838bc8b4b77fdb4859755dd628c2d30e2a6dfc86a8c",
                      "transform": "Identity"
                    },
                    {
                      "key": "balance-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e",
                      "transform": {
                        "AddUInt512": "2500000000"
                      }
                    },
                    {
                      "key": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
                      "transform": "Identity"
                    },
                    {
                      "key": "balance-9e90f4bbd8f581816e305eb7ea2250ca84c96e43e8735e6aca133e7563c6f527",
                      "transform": {
                        "WriteCLValue": {
                          "bytes": "0ee0bff9d5085bc138938d44c64d31",
                          "cl_type": "U512",
                          "parsed": "999999999999999999999994999980000"
                        }
                      }
                    }
                  ]
                },
                "transfers": [
                  "transfer-8d81f4a1411d9481aed9c68cd700c39d870757b0236987bb6b7c2a7d72049c0e"
                ]
              }
            }
          }
        ]
      }
    }

.. raw:: html

    </details>

|


State Root Hash
~~~~~~~~~~~~~~~~

We will use the ``block_hash`` to query and retrieve the block that contains our deploy. Afterward, we will retrieve the root hash of the global state trie for this block, also known as the block's ``state_root_hash``. We will use the ``state_root_hash`` to look up various values, like the source and destination account and their balances.

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``node-address`` <HOST:PORT> Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]
- ``block-identifier`` - <HEX STRING OR INTEGER> Hex-encoded block hash or height of the block. If not given, the last block added to the chain as known at the given node will be used

::

    casper-client get-block \
          --id 3 \
          --node-address http://<peer-ip-address>:7777 \
          --block-identifier <block-hash> \

**Important response fields:**

- ``"result"."block"."header"."state_root_hash"`` - contains the root hash of the global state trie for this block

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 3,
      "jsonrpc": "2.0",
      "method": "chain_get_block",
      "params": {
        "block_identifier": {
          "Hash": "7c7e9b0f087bba5ce6fc4bd067b57f69ea3c8109157a3ad7f6d98b8da77d97f9"
        }
      }
    }


**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 3,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "block": {
          "body": {
            "deploy_hashes": [],
            "proposer": "012c6775c0e9e09f93b9450f1c5348c5f6b97895b0f52bb438f781f96ba2675a94",
            "transfer_hashes": [
              "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713"
            ]
          },
          "hash": "7c7e9b0f087bba5ce6fc4bd067b57f69ea3c8109157a3ad7f6d98b8da77d97f9",
          "header": {
            "accumulated_seed": "50b8ac019b7300cd1fdeec050310e61b900e9238aa879929745900a91bd0fc4f",
            "body_hash": "224076b19c04279ae9b97f620801d5ff40ba64f431fe0d5089ef7cb84fdff45a",
            "era_end": null,
            "era_id": 0,
            "height": 8,
            "parent_hash": "416f339c4c2ff299c64a4b3271c5ef2ac2297bb40a477ceacce1483451a4db16",
            "protocol_version": "1.0.0",
            "random_bit": true,
            "state_root_hash": "cfdbf775b6671de3787cfb1f62f0c5319605a7c1711d6ece4660b37e57e81aa3",
            "timestamp": "2021-04-20T18:04:42.368Z"
          },
          "proofs": [
            {
              "public_key": "010f50b0116f213ef65b99d1bd54483f92bf6131de2f8aceb7e3f825a838292150",
              "signature": "130 chars"
            },
            {
              "public_key": "012c6775c0e9e09f93b9450f1c5348c5f6b97895b0f52bb438f781f96ba2675a94",
              "signature": "130 chars"
            },
            {
              "public_key": "018d5da83f22c9b65cdfdf9f9fdf9f7c98aa2b8c7bcf14bf855177bbb9c1ac7f0a",
              "signature": "130 chars"
            },
            {
              "public_key": "01b9088b92c8a8d592f6ec8c3e8153d7c55fc0c38b5999a214e37e73a2edd6fe0f",
              "signature": "130 chars"
            },
            {
              "public_key": "01b9e3484d96d5693e6c5fe789e7b28972aa392b054a76d175f079692967f604de",
              "signature": "130 chars"
            }
          ]
        }
      }
    }

.. raw:: html

    </details>

|


Query the Source Account
~~~~~~~~~~~~~~~~~~~~~~~~

Next, we will query for information about the ``Source`` account, using the global-state hash of the block containing our transfer and the public key of the target account.

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``node-address`` - <HOST:PORT> Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]
- ``state-root-hash`` - <HEX STRING> Hex-encoded hash of the state root
- ``key`` - <FORMATTED STRING or PATH> The base key for the query. This must be a properly formatted public key, account hash, contract address hash, URef, transfer hash or deploy-info hash.

::

    casper-client query-state \
      --id 4 \
      --node-address http://<peer-ip-address>:7777 \
      --state-root-hash <state-root-hash> \
      --key <hex-encoded-source-account-public-key>

**Important response fields:**

- ``"result"."stored_value"."Account"."main_purse"`` - the address of the main purse containing the sender’s tokens. This purse is the source of the tokens transferred in this example

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 4,
      "jsonrpc": "2.0",
      "method": "state_get_item",
      "params": {
        "key": "account-hash-b0049301811f23aab30260da66927f96bfae7b99a66eb2727da23bf1427a38f5",
        "path": [],
        "state_root_hash": "cfdbf775b6671de3787cfb1f62f0c5319605a7c1711d6ece4660b37e57e81aa3"
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 4,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "merkle_proof": "2228 chars",
        "stored_value": {
          "Account": {
            "account_hash": "account-hash-b0049301811f23aab30260da66927f96bfae7b99a66eb2727da23bf1427a38f5",
            "action_thresholds": {
              "deployment": 1,
              "key_management": 1
            },
            "associated_keys": [
              {
                "account_hash": "account-hash-b0049301811f23aab30260da66927f96bfae7b99a66eb2727da23bf1427a38f5",
                "weight": 1
              }
            ],
            "main_purse": "uref-9e90f4bbd8f581816e305eb7ea2250ca84c96e43e8735e6aca133e7563c6f527-007",
            "named_keys": []
          }
        }
      }
    }

.. raw:: html

    </details>


|


Query the Target Account
~~~~~~~~~~~~~~~~~~~~~~~~~

We will repeat the previous step to query information about the target account. 

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``state-root-hash`` - <HEX STRING> Hex-encoded hash of the state root
- ``key`` - <FORMATTED STRING or PATH> The base key for the query. This must be a properly formatted public key, account hash, contract address hash, URef, transfer hash or deploy-info hash.

::

    casper-client query-state \
          --id 5 \
          --state-root-hash <state-root-hash> \
          --key <hex-encoded-target-account-public-key>

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 5,
      "jsonrpc": "2.0",
      "method": "state_get_item",
      "params": {
        "key": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
        "path": [],
        "state_root_hash": "cfdbf775b6671de3787cfb1f62f0c5319605a7c1711d6ece4660b37e57e81aa3"
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 5,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "merkle_proof": "2228 chars",
        "stored_value": {
          "Account": {
            "account_hash": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
            "action_thresholds": {
              "deployment": 1,
              "key_management": 1
            },
            "associated_keys": [
              {
                "account_hash": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668",
                "weight": 1
              }
            ],
            "main_purse": "uref-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e-007",
            "named_keys": []
          }
        }
      }
    }

.. raw:: html

    </details>

| 
Get Source Account Balance
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that we have the source purse address, we can get its balance using the ``get-balance`` command. In the following sample output, the balance of the source account is 5000000000 motes.

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``node-address`` - <HOST:PORT> Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]
- ``state-root-hash`` - <HEX STRING> Hex-encoded hash of the state root
- ``purse-uref`` - <FORMATTED STRING> The URef under which the purse is stored. This must be a properly formatted URef "uref-<HEX STRING>-<THREE DIGIT INTEGER>"

::

    casper-client get-balance \
          --id 6 \
          --node-address http://<peer-ip-address>:7777 \
          --state-root-hash <state-root-hash> \
          --purse-uref <source-account-purse-uref>

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 6,
      "jsonrpc": "2.0",
      "method": "state_get_balance",
      "params": {
        "purse_uref": "uref-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e-007",
        "state_root_hash": "cfdbf775b6671de3787cfb1f62f0c5319605a7c1711d6ece4660b37e57e81aa3"
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 6,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "balance_value": "5000000000",
        "merkle_proof": "2502 chars"
      }
    }

.. raw:: html

    </details>

| 
Get Target Account Balance
~~~~~~~~~~~~~~~~~~~~~~~~~~

Similarly, now that we have the address of the target purse, we can get its balance. 

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``node-address`` - <HOST:PORT> Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]
- ``state-root-hash`` - <HEX STRING> Hex-encoded hash of the state root
- ``purse-uref`` - <FORMATTED STRING> The URef under which the purse is stored. This must be a properly formatted URef "uref-<HEX STRING>-<THREE DIGIT INTEGER>"

::

    casper-client get-balance \
          --id 7 \
          --node-address http://<peer-ip-address>:7777 \
          --state-root-hash <state-root-hash> \
          --purse-uref <target-account-purse-uref>

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 7,
      "jsonrpc": "2.0",
      "method": "state_get_balance",
      "params": {
        "purse_uref": "uref-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e-007",
        "state_root_hash": "cfdbf775b6671de3787cfb1f62f0c5319605a7c1711d6ece4660b37e57e81aa3"
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 7,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "balance_value": "5000000000",
        "merkle_proof": "2502 chars"
      }
    }

.. raw:: html

    </details>

|
Query Transfer Details
~~~~~~~~~~~~~~~~~~~~~~

We will use the ``transfer-<address>`` to query more details about the transfer.

**Important request fields:**

- ``id`` - <STRING OR INTEGER> Optional JSON-RPC identifier applied to the request and returned in the response. If not provided, a random integer will be assigned
- ``node-address`` - <HOST:PORT> Hostname or IP and port of node on which HTTP service is running [default:http://localhost:7777]
- ``state-root-hash`` - <HEX STRING> Hex-encoded hash of the state root
- ``key`` - <FORMATTED STRING or PATH> The base key for the query. This must be a properly formatted public key, account hash, contract address hash, URef, transfer hash or deploy-info hash.

::

    casper-client query-state \
          --id 8 \
          --node-address http://<peer-ip-address>:7777 \
          --state-root-hash <state-root-hash> \
          --key transfer-<address>

.. raw:: html

    <details>
    <summary>Explore the JSON-RPC request and response generated.</summary>

**JSON-RPC Request**:

.. code-block:: json

    {
      "id": 8,
      "jsonrpc": "2.0",
      "method": "state_get_item",
      "params": {
        "key": "transfer-8d81f4a1411d9481aed9c68cd700c39d870757b0236987bb6b7c2a7d72049c0e",
        "path": [],
        "state_root_hash": "cfdbf775b6671de3787cfb1f62f0c5319605a7c1711d6ece4660b37e57e81aa3"
      }
    }

**JSON-RPC Response**:

.. code-block:: json

    {
      "id": 8,
      "jsonrpc": "2.0",
      "result": {
        "api_version": "1.0.0",
        "merkle_proof": "924 chars",
        "stored_value": {
          "Transfer": {
            "amount": "2500000000",
            "deploy_hash": "ec2d477a532e00b08cfa9447b7841a645a27d34ee12ec55318263617e5740713",
            "from": "account-hash-b0049301811f23aab30260da66927f96bfae7b99a66eb2727da23bf1427a38f5",
            "gas": "0",
            "id": null,
            "source": "uref-9e90f4bbd8f581816e305eb7ea2250ca84c96e43e8735e6aca133e7563c6f527-007",
            "target": "uref-6f4026262a505d5e1b0e03b1e3b7ab74a927f8f2868120cf1463813c19acb71e-004",
            "to": "account-hash-8ae68a6902ff3c029cea32bb67ae76b25d26329219e4c9ceb676745981fd3668"
          }
        }
      }
    }

.. raw:: html

    </details>

|
Here we can see more information about the transfer we conducted: its deploy hash, the account which executed the transfer, the source and target purses, and the target account. Using this additional information, we can verify that our transfer was executed successfully.

Other Available RPCs
^^^^^^^^^^^^^^^^^^^^

The example above uses JSON-RPC calls to execute and then verify the transfer. There are additional JSON-RPC calls that you can make to address other use cases.

The following command lists all the JSON-RPC calls that the node supports:

::

    casper-client list-rpcs --node-address http://<peer-ip-address>:7777

The endpoint returns an OpenRPC compliant document that describes all the JSON-RPC calls available and provides examples for the RPCs. Please be sure to query this specific endpoint as it provides up-to-date information on interacting with the RPC endpoint.


FAQ
^^^
This section covers frequently asked questions and our recommendations.

Deploy Processing
~~~~~~~~~~~~~~~~~
**Question**: How do I know that a deploy was finalized?

**Answer**: If a deploy was executed, then it has been finalized. If the deploy status comes back as null, that means the deploy has not been executed yet. Once the deploy executes, it is finalized, and no other confirmation is needed. Exchanges that are not running a read-only node must also keep track of `finality signatures <#finality-signatures>`_ to prevent any attacks from high-risk nodes.

Finality Signatures
~~~~~~~~~~~~~~~~~~~
**Question**: When are finality signatures needed?

**Answer**: Finality signatures are confirmations from validators that they have executed the transaction. Exchanges should be asserting finality by collecting the weight of two-thirds of transaction signatures. If an exchange runs a read-only node, it can collect these finality signatures from its node. Otherwise, the exchange must assert finality by collecting finality signatures and have proper monitoring infrastructure to prevent a Byzantine attack. 

Suppose an exchange connects to someone else's node RPC to send transactions to the network. In this case, the node is considered high risk, and the exchange must assert finality by checking to see how many validators have run the transactions in the network.

The EventStore
~~~~~~~~~~~~~~
**Question**: What is the EventStore? 

**Answer**: The CasperLabs/event-store has been deprecated and is incompatible with the node event stream. It is best to monitor deploy processing status via polling port 9999, which is the event stream port of a node: ``http://<peer-ip-address>:9999``. Push the events of interest into a database for future reference. In this process, you can also get the associated finality signatures of the block of interest.

deploy_hash vs. transfer_hash
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
**Question**: How is a deploy_hash different than a transfer_hash?

**Answer**: Essentially, there is no difference between a `deploy_hash` and a `transfer_hash` since they are both deploy transactions. However, the platform is labeling the subset of deploys which are transfers, to filter transfers from other types of deploys. In other words, a `transfer_hash` is a native transfer, while a `deploy_hash` is another kind of deploy.

account-hex vs. account-hash
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
**Question**: Should a customer see the account-hex or the account-hash?

**Answer**: Exchange customers or end-users only need to see the `account-hex`. They do not need to know the `account_hash`. The `account_hash` is needed in the backend to verify transactions. Store the `account-hash` to query and monitor the account. Customers do not need to know this value, so to simplify their experience, we recommend storing both values and displaying only the `account-hex`.

Example Deploy
~~~~~~~~~~~~~~
**Question**: Can you provide an example of a deploy?

**Answer**: You can find a *testDeploy* reference in `GitHub <https://github.com/casper-ecosystem/casper-client-sdk/blob/master/test/lib/DeployUtil.test.ts#L5>`_.

Operating with Keys
~~~~~~~~~~~~~~~~~~~
**Question**: How should we work with the PEM keys?

**Answer**: The `Keys API <https://casper-ecosystem.github.io/casper-client-sdk/modules/_lib_keys_.html>`_ provides methods for `Ed25519` and `Secp256K1` keys. Also, review the tests in `GitHub <https://github.com/casper-ecosystem/casper-client-sdk/blob/master/test/lib/Keys.test.ts#L39>`_ and the `Working with Keys <https://docs.casperlabs.io/en/latest/dapp-dev-guide/keys.html>`_ documentation.
