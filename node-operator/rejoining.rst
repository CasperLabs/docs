
Re-Joining the Network
======================

If your validator node was ejected, the following steps would help you re-join the network.

Choose the `latest release tag <https://github.com/casper-network/casper-node/tags>`_ and re-build the smart contracts required for `bonding <https://docs.casperlabs.io/en/latest/node-operator/bonding.html>`_:  

* Navigate to the ``casper-node`` directory
* Check out the latest release branch
* Re-build the contracts required for bonding

.. code-block:: bash

   cd casper-node
   git checkout release-X.X.X
   make setup-rs
   make build-client-contracts

Run the following transaction to re-activate your bid and re-join the network:

.. code-block:: bash

   casper-client put-deploy \
    --secret-key /etc/casper/validator_keys/secret_key.pem  \
    --chain-name casper  \
    --session-path ~/casper/casper-node/target/wasm32-unknown-unknown/release/activate_bid.wasm  \ 
    --payment-amount 300000000  \
    --session-arg "validator_public_key:public_key='$(cat /etc/casper/validator_keys/public_key_hex)'"
