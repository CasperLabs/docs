Consensus economics
===================

Highway consensus is a continuous, trust-less process that has a fixed set of validators engage in scheduled communication to advance the linear chain of finalized blocks, which represents the agreed upon history of changes to the global state. The fixed set of validators may change at each era boundary. The economics of this layer revolve around selection of sets of validators for each era and incentivization of participation according to the schedule.

Entry
-----

Several eras after genesis, the system settles into selecting a set of validators several eras in advance, governed by the *auction_delay* chainspec parameter (set to 1 at mainnet launch). This is done by means of a stake auction that selects a fixed number, governed by the *validator_slots* chainspec parameter, of highest bids at the time of the auction, conducted in the switch block. After *auction_delay* eras, the winning validators become the validating set for the new era.

Bids
^^^^

Each bid is a collection of tokens from a prospective or current validator and its delegators, considered in the auction as a single total. Bids and delegations can be freely increased, but withdrawal is subject to an unbonding period governed by the *unbonding_delay* chainspec parameter. Unbonding tokens are not considered during the auction. Consequently, the exact amount of the bid, which translates into protocol weight for winning validators, can vary within an era, with the bids visible in the switch block determining the winners.

Each bid is associated with a delegation rate, which can be changed at any time, and activity status. Both of these are described in subsequent sub-sections.

Delegation
^^^^^^^^^^

Delegation allows third parties to participate in consensus by adding weight to their preferred validators. Rewards received by validators are distributed in proportion to tokens bid and delegated, with the current or prospective validator responsible for the bid receiving a portion of the delegator rewards set by the delegation rate.

At mainnet launch, delegation is unrestricted. Interested readers should review `CEP #29 <https://github.com/CasperLabs/ceps/pull/29>`_, which proposes delegation caps.

Incentives
----------

Correct operation of the Highway protocol requires the economics of the platform to incentivize non-equivocation, for safety, and participation, for liveness. Participation consists of on-time block proposal and timely responses to block proposals.

Safety may be incentivized through slashing for equivocation. This feature is currently disabled, but may be reactivated in the future.

Participation is incentivized by scaling rewards for on-time proposals and responses, taking into account the speed of finalization. All rewards are added directly to the corresponding bids and delegations.

Participation
^^^^^^^^^^^^^

Issuance of new tokens and their distribution to validators incentivizes work even under low transaction load.

CSPR is issued at a fixed rate and distributed to validators (and, indirectly, delegators) in proportion to their stake. This is analogous to block rewards in Proof of Work blockchains, except for a couple of differences:

- The growth of CSPR supply is exponential
- Issuance takes into account slashed CSPR

With slashing disabled, we can compute block rewards starting with following formula

.. code-block::

   supply(i) = initial_supply * (1 + issuance_rate)^(tick_at_era_start(i) / ticks_per_year)

where :code:`i = 0, 1, ...` is the era's index, :code:`initial_supply` is the number of CSPR at genesis, :code:`issuance_rate` is the annual rate at which new CSPR is minted, and :code:`ticks_per_year = 365*24*60*60*1000 = 31_536_000_000`.

We introduce the *round issuance rate* (corresponding to the chainspec parameter *round_seigniorage_rate*)

.. code-block::

   round_issuance_rate = pow(1 + issuance_rate, 2^minimum_round_exponent / ticks_per_year) - 1

which is the annual issuance rate adjusted to a single round of length determined by chainspec parameter *minimum_round_exponent*. For illustration, an exponent of 14 corresponds to a round length of roughly 16 seconds.

Finally, the base round reward is computed as

.. code-block::

   base_round_reward(i) = round_issuance_rate * supply(i)

This value gives us the maximum amount of CSPR that the validators can collectively receive from a proposed block.

Distribution
~~~~~~~~~~~~~~~~~~~

Validators receive tokens for proposing and finalizing blocks, according to their performance. The concept of weight is crucial for understanding this distribution scheme:

- **Weight:** A validator's bonded stake, used in consensus
- **Assigned weight of a block/round:** The total stake of validators that are scheduled to participate on a block
- **Participated weight of a block/round:** The total stake of validators that actually end up participating, or sending messages to finalize a block before their respective rounds end

To determine eligibility, we look at **on-time finalization (OTF)**. Validators should finalize blocks on time, by sending required messages before their respective rounds end.

It should be noted that switch blocks are not visible to the issuance calculation (as it is performed in the switch block itself for each era) and, consequently, no tokens are issued for switch blocks.

Participation schedule
++++++++++++++++++++++

The participation schedule is segmented into rounds, which are in turn allocated dynamically according to the validators' exponents and a deterministic (randomized at era start) assignment of validators to milliseconds of an era. A validator with the round exponent :code:`n` has to participate in rounds that repeat every :code:`2^n` ticks.

Each validator is assessed according to their own round exponent. All assigned validators become eligible to receive tokens as long as the block gets finalized with messages that were sent within each validator's own round.

Eligibility
+++++++++++

Once a block has been proposed and enough time has passed, the history of messages can be examined to detect whether the block was indeed finalized on time, according to the conditions given above.

- If the block is *not* finalized on time, validators receive a fraction of the tokens, governed by the *reduced_reward_multiplier* chainspec parameter
- If the block is finalized on time, assigned validators share the reward proportionally to stake, regardless of whether they have sent messages or not

Inactivity
^^^^^^^^^^

Validators who send no messages during an entire era are marked as inactive and cease to participate in the auction until sending a special deploy that reactivates their bid.

Slashing
^^^^^^^^

Please review our `Equivocator Policy <https://github.com/CasperLabs/ceps/blob/master/text/0038-equivocator-policy.md>`_. We are currently conducting research into the utility of slashing as an incentive mechanism.

Founding validators
-------------------

Founding validators are subject to token lock-up, which prevents them from withdrawing any tokens from their bids for 90 days, then releases their genesis bid tokens in weekly steps, linearly, over the course of a further 90 days.
