Runtime economics
=================

The economics of the runtime layer are intended to incentivize efficient allocation of computational resources, primarily CPU time, to users. Pending implementation of state pruning, use of disk space is treated as CPU time use and charged, irreversibly, per byte written. At mainnet launch, gas is allocated according to a first-in, first-out model for deploys, with a fixed price of 1 mote (1/10^9 part of a CSPR token) per 1 unit of gas.

Gas allocation
--------------

Any finite resource on a publicly accessible computer network must be rate-limited, as, otherwise, overuse of this resource is an attack vector. This is a minimal requirement for platform security. However, rate-limiting, implemented on our platform as a simple block gas limit with several lanes, leaves the platform with the problem of fairly and efficiently allocating the gas. We are continuing research on the optimal structure for spot and futures gas markets and interested readers should consult the relevant CEPs. In the meantime, this section outlines some basic features of the platform that would underpin any allocation scheme.

Consensus before execution & basics of payment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Highway protocol in its mainnet implementation reaches consensus on a block *before* the block is executed. This introduces a subtle difference from platforms like Ethereum. Deploys sent to the Casper network can only be selected on the basis of claimed, rather than used, gas. Consequently, to incentivize user-side optimization and to prevent exhaustion of block space by poorly optimized deploys, the platform provides no refunds for unused gas.

Additionally, because we allow complex payment code, there is a minimal amount of CSPR that must be present in the user account, to ensure that the payment computation is covered. Further balance checks may be introduced in the future.

Costs and limits
^^^^^^^^^^^^^^^^

Gas cost is a measure of relative time used by different primitive operations of the execution engine, which is also assumed to be additive. By additivity, we mean that the time to execute a program is approximately proportional the sum of gas expended by the opcodes and functions called within the program. Casper assigns gas costs both to primitive execution engine opcodes and to certain more complex *host side* functions that are callable from within the execution engine context. Gas costs for opcodes and host side functions are specified in the chainspec and may vary according to arguments.

It is expected that the current gas cost table will be refined over time to reflect time use more closely, with updates introduced in future upgrades. It is also anticipated that, with the introduction of state pruning, storage will come to be costed explicitly and separately from compute time.

Lanes
^^^^^

Block gas limit is split into two lanes, one for native transfers and one for general deploys. The number of transfers, which cost a fixed amount of gas, is governed directly by the *block_max_transfer_count* chainspec parameter (set to 2500 at mainnet launch).

Gas fees
--------

At mainnet launch, the price of gas is fixed at 1 mote per 1 unit of gas.

Fee allocation
^^^^^^^^^^^^^^

All fees from a particular block accrue to its proposer. This incentivizes non-empty block production and allows major dApps to execute their own deploys for free, provided they operate a validator node and are comfortable with the latency introduced by validator scheduling.

Spot pricing
^^^^^^^^^^^^

Please see `CEP #22 <https://github.com/CasperLabs/ceps/pull/22>`_ for one potential design of a gas spot market.

Futures pricing
^^^^^^^^^^^^^^^

Please see `CEP #17 <https://github.com/CasperLabs/ceps/pull/17>`_ for our draft proposal of a gas futures market.
