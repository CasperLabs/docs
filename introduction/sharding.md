# Heterogenous Side Chains

## Motivation

Existing blockchains are treated as a "single global computer", but this implicitly assumes there is a "one-size fits all" computer capable of meeting all use cases. This is obviously false as there are trade offs to be made in all aspects of the blockchain space that are often talked about \(e.g. speed, security, decentralization, censorship-resistance\), and not all use cases require \(or even can work with\) the extreme end of these trade-offs. For example, some industries \(e.g. finance, medicine\) have strict regulations about where data can be located; in particular forbidding the usage of a "global computer" to work with this data. Such regulations clearly limit the amount of decentralization which can be allowed, yet such industries would still benefit from the security and verifiability the blockchain offers. Similarly, transactions which are fast-paced but low value \(such as those at a typical coffee shop\) have a much higher requirement for speed as compared to those which are rare but high value \(e.g. purchasing of real estate or other high value, low liquidity assets\). This is the idea alluded to above -- some use cases may have a high throughput requirement, which may in turn become requirements on the node hardware, and then finally become a requirement of the chain which is designed for this use case. Note that a fast network connection is considered to be part of the hardware requirements for the purposes of this discussion \(it doesn't matter how may cores you have if you cannot tell the network about all the transactions you are processing\). Considering this example further, higher value transactions have a higher need for security compared with lower value transactions. Therefore, a side chain designed for high value transactions may prefer to have many, diverse validators, regardless of their hardware or geographic location. This brings us back to the point mentioned above that while node hardware can be a limiting factor in throughput of a network, side chains allow us to put constraints on the hardware in parts of the network where very high throughput is needed, while allowing other parts of the network to have any device be a node where very high decentralization is needed.

The ability to customize parts of the network to meet specific use cases is essential to delivering a platform on which all business sectors can find some value. Our goal with side chains is to enable that customization. Note that we have said nothing about removing the "global computer". There will be a root chain which looks similar to a network like Ethereum \(globally decentralized, reasonably open-access\). In the trade-off space, one can view it as preferring decentralization and security at the cost of speed \(since the rate at which data can be transferred around the globe is limited\). This is a customization which is completely allowed and in fact will play an important role, as this will be the form the "root chain" will take \(described in the following section\).

## Technical Details

A side chain is an independently replicated blockdag, with its own independent set of validators. In most respects, different side chains are independent instances of the CasperLabs protocol. The exception to this is the CasperLabs native token, which will be common throughout all chains. This allows clients to easily do business in multiple chains, or move between chains as needed. The way in which this is accomplished is by imposing a tree structure on the chains. There is a single _root chain_, which is governed by the community and is where the CasperLabs token mint lives \(see [the tokens page](../technical-framework/block-storage/token.md) for more information\). The root has zero or more _child chains_ who receive their supply of tokens from the root via smart contracts detailed below. The intuition is that the tokens available in a child chain are "backed" by tokens in its parent, all the way up to the root where the original mint is. This is similar to how countries were allowed to print money back when the "gold standard" was enforced. The purpose of this system is to ensure smaller chains do not compromise the security of the token supply because a parent chain will not accept more tokens to be transferred back than were originally allocated. Therefore, even if a child chain is corrupt and its validators are producing invalid blocks which forge tokens, these cannot impact the rest of the network because they can never be moved out into the parent chain.

Note that the interaction of moving tokens between children \(detailed below\) is the _only_ interaction between chains. There is no direct calling of contracts that live in another chain. To enable that would require an external service using off-chain triggers to make transactions in one chain be forwarded to another chain. Such a system is out of scope for this work.

### Communication between chains

Since different chains are independent blockdags, the only way they can communicate is as clients of one another. However, for security reasons, it is not sufficient to have one validator speak for the entire chain they are a part of; instead there must be assurance that the chain as a whole agrees to the communication and the message it conveys. For this purpose we introduce the k-of-n \(KON\) blessed contract, which functions as a voting mechanism for validators to agree with the sending of a message. At a high level, the contract recognizes some participants, each with some weight \(the total weight being `n`\), and has a threshold `k`, such that a proposal being voted on is only accepted if at least a weight of `k` \(out of the total `n`, hence the name\) agree. To save on the number of contract calls \(and avoid stuck proposals\), each proposal has a time-to-live \(TTL\). The proposal must reach the threshold before the TTL or it will be defeated. There are only votes in favour of a proposal, not votes against, because not voting is considered equivalent to voting no. This saves on the number of calls to the contract because nay-sayers do not need to call the contract to give their opinion. We sketch the API of the KON contract below.

```javascript
//weights are non-negative, whole numbers
type Weight = Nat
//time is measured in cumulative gas spent; a non-negative number
type Time = Nat

//Data type representing possible proposals to vote on.
//The two variants Update and Release correspond to the two
//types of actions which can result from the proposal being
//successful. The `id` field present in both Proposal variants
//is a unique identifier which is computed based on the deploy
//which triggered the voting to need to happen. For example,
//a the deploy in which a new validator bonds on the network
//would be used to derive the `id` of the corresponding 
//`Update` proposal which adds that new validator to the
//participants of the KON contract. The formula to determine the
//`id` will be `id = hash(blockhash + deployhash + seq)`, where
//`blockhash` is the hash of the block which includes the deploy, 
//`deployhash` is the hash of the deploy which initiated the need
//for the proposal, and `seq` is a sequential identifier (starting
//from zero) which is used if the same deploy causes multiple
//proposals to need to be created.
//Other fields in the proposal are described below.
data Proposal = 
  Update(
    id: Array[Byte],
    newParticipants: Map[PublicKey, Weight], 
    newThreshold: Weight,
    ttl: Time,
    yesWeight: Weight //weight that has voted for this proposal
  ) |
  Release(
    id: Array[Byte],
    amount: Nat, 
    destination: Address,
    //confirmation ID sent back to originating chain to confirm transaction
    confId: Array[Bytes],
    ttl: Time,
    yesWeight: Weight
  ) |
  Confirm(
    id: Array[Byte],
    confId: Array[Bytes],
    ttl: Time,
    yesWeight: Weight
  )

contract KON:
  var participants: Map[PublicKey, Weight]
  var threshold: Weight
  var activeProposals: Set[Proposal]

  //Note: all calls to the contract automatically prune the `activeProposals`
  //      removing the ones which have passed their TTL and applying the effects
  //      of those which have surpassed the threshold.

  //Proposal/vote to update the properties of this contract. The purpose of
  //not having a `vote` end-point is to prevent duplicate proposals from being
  //created. A call to `update` with a new set of parameters creates a new active
  //proposal, whereas a call with a set of parameters which is already active
  //is a vote for that proposal. `ttl` does not count as a parameter of the proposal,
  //in the case they differ the larger `ttl` is always chosen; this means a proposal
  //can be kept alive as long as there is sufficiently frequent voting activity on it.
  //Note: the current participants must meet the current threshold for
  //      the change to be applied. If an update proposal is successful then
  //      all other existing active proposals are now compared against the
  //      new standards to determine if they are successful.
  function update(
      id: Array[Byte],
      newParticipants: Map[PublicKey, Weight], 
      newThreshold: Weight,
      ttl: Time //proposal must be successful before `ttl`
    ): Unit

  //Proposal/vote to release tokens into a particular address. Once again
  //there is no `vote` end-point to avoid creating duplicate proposals.
  //More details about how `release` has access to tokens to send to
  //the destination address are given in the next section.
  function release(
    id: Array[Byte],
    amount: Nat, 
    destination: Address, 
    confId: Array[Byte],
    ttl: Time
  ): Unit

  //Proposal/vote to confirm a `release` of tokens has occurred.
  function confirm(
    id: Array[Byte],
    confId: Array[Byte],
    ttl: Time
  ): Unit
```

In order to prevent chains from needing to watch each other's DAGs, the KON contract containing the participants corresponding to validators in the parent chain lives in the child chain, and vice versa. To be clear, what this means is that if `KONp` is the KON contract which is in the parent chain \(note there will be a separate `KONp` contract for each different child that parent has\) and `KONc` is the KON contract which is in the child chain then

```text
participants(KONp) == validators in the child
participants(KONc) == validators in the parent
```

This is reversed on purpose to prevent these chains from needing to watch each other's blocks. A transfer from parent to child has a trigger originating in the parent, but an effect \(the new tokens appearing\) in the child. So the parent validators will be the participants controlling such a transfer since they already need to watch their own blocks, but the contract that they vote on will be in the child, because that is where the effect needs to happen. This is why `KONc` has the parent validators as its participants. A similar argument holds for why `KONp` has the child validators as participants.

The disadvantage of this approach is that chains need to keep each other up to date with bonding/unbonding events that occur via cross-chain `update` proposals to the KON contract. However, even though validators in one chain act as clients of the other chain, calls validators make to the KON contract will not be charged in the way user transactions normally are. It costs both parties resources to send and process KON transactions, but both are also compensated. In the case of a user transferring tokens from one chain to the other, the sender is compensated because the user pays to initiate the transaction in that chain, and the receiver is compensated because the value of their chain is increased by the influx of new tokens. In the case of bonding/unbonding there is no explicit compensation for either party; this is simply the overhead of continuing to do business with one another. This overhead should be low though because KON operations are reasonably simple \(and therefore do not consume a large number of resources\), and bonding/unbonding operations will be much less common than general user transactions.

### Moving tokens between chains

Each KON contract is paired with another contract for managing the tokens. This contract is different depending on the chain's role \(parent or child\). The parent chain has a `depository` contract, which holds the tokens that have been sent to the child. The child chain has a `localMint` contract which creates "new" tokens to correspond to those locked up in the parent's `depository`. The local mint also accepts tokens to burn which will be released from the depository in the parent \(corresponding to moving the tokens back to the parent chain\). The `release` API of the KON contract is the only mechanism for taking tokens out of the depository/local mint, while the API for giving tokens back to the depository/local mint \(thus initiating a cross-chain transfer\) is available to all users. Cross-chain transfer proceed via a two-phase-commit-like transaction sequence. When the user requests the transfer, the tokens are stored in a staging area until the transaction is confirmed by the receiving chain. The API for the `depository` contract is given below. Note that the same notation used on [the tokens page](../technical-framework/block-storage/token.md) is used again in the depository API definition.

```javascript
//Type alias for the tuple of information representing
//a pending deposit. It holds: a purse with the funds,
//the confirmation ID, the refund pointer and the TTL.
type PendingDeposit = (P[cl], Array[Byte], URef, Time)

//Lives in the parent chain
contract depository
  //Local CasperLabs-type purse where the tokens transferred 
  //to the child are held.
  var lockup: P[cl]
  //Set of transactions waiting to be confirmed. When `deposit` is called
  //a new `PendingDeposit` is created. This is eventually either
  //confirmed by the child validators and the funds are moved to the lockup,
  //or the `ttl` expires and the funds are returned to the user.
  var staging: Set[PendingDeposit]

  //Only accessible via the `release` endpoint of the corresponding 
  //KON contract (controlled by the child validators). Uses the `lockup` 
  //purse as the source of the tokens to send to the `destination`. 
  //If there are insufficient funds in the lockup
  //then whatever tokens remain are sent to the `destination`. This
  //prevents more tokens coming out of the depository than were put in.
  private function release(amount: Nat, destination: Address): Unit

  //Only accessible via the `confirm` endpoint of the corresponding
  //KON contract. Confirms a pending deposit, removing it from the staging
  //set and adding the funds to the lockup.
  private function confirm(confId: Array[Byte]): Unit

  //Instructs the validators to call the `release` API in the child chain
  //once this transaction is finalized. The confirmation ID is returned as
  //the result of this function. 
  function deposit(
    purse: P[cl], //purse containing funds to transfer
    destination: Address, //address of the account in other chain to give the tokens
    refund: URef, //pointer to contract/account where to return the funds if transaction fails
    ttl: Time //time for the transaction to be confirmed before it is refunded
  ): Array[Byte]
```

The API for the local mint contract is as follows:

```javascript
//Lives in the child chain
contract localMint
  //Local mint which is used to create the tokens that correspond
  //to those locked up in the depository.
  var mint: m(cl)
  //Set of transactions waiting to be confirmed. When `deposit` is called
  //a new `PendingDeposit` is created. This is eventually either
  //confirmed by the parent validators and the funds burned,
  //or the `ttl` expires and the funds are returned to the user.
  var staging: Set[PendingDeposit]

  //Only accessible via the `release` endpoint of the corresponding 
  //KON contract (controlled by the parent validators). The `mint` 
  //in this contract is used as the source of the tokens to send to 
  //`destination`.
  private function release(amount: Nat, destination: Address): Unit

  //Only accessible via the `confirm` endpoint of the corresponding
  //KON contract. Confirms a pending transfer, removing it from the staging
  //set and burning the funds (they do not need to be locked up because they
  //will simply be re-minted if the user transfers them back).
  private function confirm(confId: Array[Byte]): Unit

  //Instructs the validators to call the `release` API in the parent chain
  //once this transaction is finalized. The confirmation ID is returned as
  //the result of this function. 
  function deposit(
    purse: P[cl], //purse containing funds to transfer
    destination: Address, //address of the account in other chain to give the tokens
    refund: URef, //pointer to contract/account where to return the funds if transaction fails
    ttl: Time //time for the transaction to be confirmed before it is refunded
  ): Array[Byte]
```

To clarify the details of how cross-chain transactions work, we give the following examples.

**Example 1: Transferring tokens from parent to child** 1. User makes a transaction in the parent chain calling the `deposit` API of the `depository` contract \(paying for this operation in the parent\) 2. As validators see the transaction be finalized they call the `release` endpoint of the KON contract in the child with the appropriate parameters. This operation costs resources for both parent and child, but both are also being compensated. The parent is compensated by the paid transaction in 1. and the child by the new tokens entering their chain as a result \(increasing the value of their chain\) 3. When the threshold in KON contract is reached, the new tokens are added to the purse at the target address in the child chain \(this happens as a consequence of one of the KON calls in 2.\) 4. When validators in the child chain see that 3. has been finalized, they call the `confirm` endpoint of the KON contract in the parent with the appropriate parameters. 5. When the threshold in the KON contract is reached, the tokens are moved from the `staging` set of the `depository` and into the `lockup`. 6. If the `ttl` on the `PendingDeposit` expires before 5. occurs then the funds are returned to the user \(in the parent chain\).

**Example 2: Transferring tokens from child to parent** 1. User makes a transaction in the child chain calling the `deposit` API of the `localMint` contract \(paying for this operation in the child\) 2. As validators see the transaction be finalized they call the `release` endpoint of the KON contract in the parent with the appropriate parameters. This operation costs resources for both parent and child, but both are also being compensated. The child is compensated by the paid transaction in 1. and the parent by the new tokens entering their chain as a result \(increasing the value of their chain\) 3. When the threshold in KON contract is reached, the desired amount of tokens are taken from the depository and added to the purse at the target address in the parent chain \(this happens as a consequence of one of the KON calls in 2.\). Note: if insufficient funds are available in the depository then only as many tokens that remain in the depository are taken. 4. When validators in the parent chain see that 3. has been finalized, they call the `confirm` endpoint of the KON contract in the child with the appropriate parameters. 5. When the threshold in the KON contract is reached, the tokens are removed from the `staging` set of the `localMint` and burned \(they do not need to be kept in a lockup because they will simply be re-minted if the user transfers them back\). 6. If the `ttl` on the `PendingDeposit` expires before 5. occurs then the funds are returned to the user \(in the child chain\).

### Creating and destroying chains

Fundamentally, a chain only exists as part of the CasperLabs blockdag ecosystem because either \(a\) it is the root chain or \(b\) it has a corresponding KON+depository contract pair in a parent chain which is part of the ecosystem. The process of creating and destroying chain therefore amounts to creating and destroying KON+depository contracts \(the root chain is never created or destroyed, it simply _is_\). A chain is created as a child of the root chain by calling the _chain factory_ contract which produces the KON+depository corresponding to the chain. The factory takes as parameters the initial KON participants \(i.e. initial chain validators\), an initial endowment purse to seed the depository \(this is possibly empty, but non-empty if the genesis block includes pre-minted CasperLabs tokens\), and a chain identifier. The chain identifier is used to identify nodes which belong to each chain \(which is needed for validators to send KON transactions back and forth\). The chain factory will not allow two chain to be created with the same identifier. Since new chain \(other than the root\) are independent blockdags with their own genesis block, we cannot guarantee that new chain will include the chain factory in the same way the root has. However, it is encouraged that chains do so if they wish to have child chains of their own.

A chain is destroyed by telling the chain factory to destroy it. A chain can only be destroyed if its depository is empty.

### Chains and seigniorage

It is contemplated that seigniorage will be part of the CasperLabs platform to increase the token supply in a controlled way over time. To ensure uniform growth of the token supply, the seigniorage will also be applied to tokens which are locked up in a root chain depository \(thus increasing the tokens allocated to each child chain\). Again, because the child chains are independent from the root we cannot guarantee that they will pass on any seigniorage onto their own children \(if they choose to even allow them\).

## A Note of Caution

The independence of chains enables rich diversity in the ecosystem, however, also comes with some risk. Tokens which are transferred into a depository are technically under the control of the validators for that chain. There is no way to guarantee that the tokens will appear in the destination address as the protocol would dictate and no mechanism to re-obtain those tokens in the case those validators are indeed faulty. Never commit tokens to a chain that you do not trust.
