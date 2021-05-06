
.. _economics:

Overview of economics
=========

Casper's economic activity can be conceptualized as taking place on four distinct layers: consensus, runtime, ecosystem, and the macroeconomy. Each layer, consensus and up, provides a foundation for the activity taking place on the next layer. A trust-less platform requires that proper incentives be provided to participants operating each of these layers to ensure that they work together to unlock the platform's value.

We cannot yet provide formal game-theoretic results for our incentive mechanisms, but interested readers can follow our progress with the `Economics of the Casper Blockchain <https://github.com/CasperLabs/Casper-economics-paper>`_ paper, which will be periodically updated to summarize ongoing research.

This section of our online documentation is intended only to familiarize the user with our core economics features rather than describe their precise implementation and user interface. Some of the features may not be currently active.

Consensus
---------

The consensus layer of our platform runs on the Highway flavor of CBC-Casper. The distinguishing characteristics of this protocol are its safety and liveness guarantees. Specifically, under the assumptions made in the `Highway protocol paper <https://github.com/CasperLabs/highway>`_, blocks in the canonical history cannot be reverted, and new blocks continue to be added to this history indefinitely. The assumptions, however, require that a large portion of validators remain online and honest; this assumed behavior must be incentivized for the platform to remain secure and live.

When discussing consensus, we default to considering it "one era at a time," unless expressly stated otherwise, in keeping with the Highway paper. Recall that each era is, effectively, a separate instance of the protocol.

Agents (consensus layer)
^^^^^^^^^^^^^^^^^^^^^^^^

*Validators* are responsible for maintaining platform security by building an ever-growing chain of finalized blocks, backing this chain's security with their stakes. Their importance (often referred to as "weight") both to protocol operation and security is, in fact, equal to their stake, which includes both their own and delegated tokens.

*Delegators* are users who participate in the platform's security by delegating their tokens to validators, which adds to their weight, and collecting a part of the rewards proportional to their delegations, net of a cut ("delegation rate") that is collected by the validator.

*Operators* can be designated by validators to perform the computational tasks required by the protocol, with the validator's stake now vouching for the operator's correctness.

Incentives (consensus layer)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The *auction* determines the composition of the validator set for each era of the protocol. It is a "first-price" (winning bids become stakes) auction with a fixed number of spots, chosen to balance security with performance (generally, the platform will run slower with more validators). Because rewards are proportional to the stake, it is expected that this competitive mechanism will provide a powerful impetus for staking as many tokens as possible.

*Slashing* ensures that the platform remains safe by burning all tokens staked by validators who equivocate.

*Rewards* (per era) are issued to validators who perform, at their nominal pace, in such a way as to make timely progress on block finalization. These rewards are shared with delegators proportionally to their contributions, net of a cut taken by the validator.

*Evictions* deactivate validators who fail to participate in an era, disabling their bid and suspending their participation until they signal readiness to resume participation by invoking a particular entry point in the auction contract.

Runtime
---------

The runtime layer encompasses deployment and execution of smart contracts, session code, and other activity that performs computation on the global state. This suggests potential markets for finite platform resources, such as markets for computing time and storage. Such markets could ensure that resources are allocated to their highest-value uses. Currently, however, we limit ourselves to metering compute time, measured as gas. Gas can be conceptualized as relative time use of different WASM operations and host-side functions. Use of storage is also presently assigned a gas cost. We do not currently have a pricing mechanism for metered gas, although an outstanding Casper Improvement Proposal (`CEP #22 <https://github.com/CasperLabs/ceps/pull/22>`_) suggests the implementation of a first-price gas auction similar to Ethereum's. The initial mainnet deploy selection mechanism will be based on FIFO.

We expect to continue work on runtime resource markets, particularly gas futures (`CEP #17 <https://github.com/CasperLabs/ceps/pull/17>`_) after mainnet launch.

Agents (consensus layer)
^^^^^^^^^^^^^^^^^^^^^^^^^

*Validators* again play a vital role in this layer since protocol operation includes construction and validation of new blocks, consisting of deploys that change the global state, which the validators also maintain.

*Users* execute session and contract code using the platform's computational resources

Incentives (consensus layer)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Transaction fees*, or charges for gas use, ensure that the users compensate validators for performing their computations. Transaction fees are awarded to the block creator. Because we expect to launch with FIFO ordering of deploys, it can be assumed that one unit of gas will be priced at one mote until future changes to deploy orders are implemented.

Ecosystem
---------

The ecosystem layer encompasses dApp design and operation. CasperLabs maintains multiple partnerships with prospective dApp developers, and we anticipate devoting significant resources to research the economics of prospective dApps.

Macroeconomy
------------

Casper's macroeconomics refers to the activity in the cryptocurrency markets, where CSPR can be treated as one cryptoasset among many rather than a computational platform. Our token economics are different from those of "digital gold" tokens like Bitcoin, designed to be scarce. Our tokens are minted from a fixed starting basis, which is accounted for by tokens distributed to genesis validators, employees, community and held for future distributions. The total supply of tokens grows at a fixed annual percentage rate from this basis, net of slashed tokens.

The inflationary nature of our macroeconomics has two significant advantages over enforced scarcity. Inflation incentivizes token holders to stake or delegate their tokens, a behavior we explicitly support with our delegation feature. Additionally, spending tokens on real economic activity on the platform is relatively more attractive to hoarding tokens in anticipation of speculative gain.

.. toctree::
   :maxdepth: 2
   :hidden:

   Overview <self>
   consensus.rst
   runtime.rst
   economicsoverview.rst
