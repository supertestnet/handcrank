# Handcrank: a drivechain wannabe with no softfork

# TLDR of how handcrank works

1. Alice makes a 51 of 51 multisig address with the most recent btc miners who mined a block
2. She deposits X btc to it after she gets the miners to create a “presigned withdrawal” covenant
3. She announces her covenant on a sidechain to get X “sidechain bitcoins”
4. She does stuff on the sidechain and (let’s suppose) loses ownership of her money
5. A new person claims they now own Alice’s deposit and says they want to withdraw it
6. The covenant lets 51% of miners send the coins anywhere after a delay of 1.5 months
7. Any bitcoin miner can check the new person’s claim and start sending them Alice’s deposit
8. During the delay, any miner who rejects the claim can extend the delay up to 3 months
9. During the delay, the rightful owner is supposed to say where the money should *really* go
10. When the delay expires miners send the money to its owner or take it if they never spoke up

# Overview

I’ve spent some time simplifying the idea of drivechain to hopefully turn it into something feasible with current bitcoin script. Here are some of my simplifications:

```
Bip300 creates a new timelock. A bip300 timelock is similar to OP_CHECKSEQUENCEVERIFY,
which is the timelock where you can only spend your coins after waiting X blocks.
Bip300's new timelock is different because it lets any miner increase the value of X
by adding a message to their bitcoin block. A person enters a sidechain by depositing
coins into a “sidechain deposit address” that uses this timelock. If, later, two
different people try to spend those coins, the timelock gives miners time to decide
who should "win."

A miner who sides with the second person can increase the value of X for the first
person, delaying their withdrawal attempt, but leave the second person’s attempt
unhindered. Each miner can delay either withdrawal attempt, or both. Whichever
attempt is delayed by fewer miners will succeed and the other one will fail, unless
both attempts are delayed long enough to allow yet another attempt to succeed before
them. By this method, 51% of miners get to choose who can withdraw the coins first.
```

And:

```
The essence of a drivechain's "deposit address" is one where a withdrawal attempt
from that address can be delayed indefinitely if miners keep incrementing an integer,
but miners can only increment it once per bitcoin block, and incrementing it is
optional.
```

And:

```
Anyone can try to withdraw from a drivechain, but each withdrawal has a timed delay,
during which miners can check if the withdrawal is authorized by the drivechain’s
rules. If it’s not, any miner can extend the delay to let an authorized withdrawal
happen first.
```

In the third summary, the drivechain bip (bip300) lets miners extend a delay through the extensible timelock mechanism I mentioned in the first summary. The timelock extends when a miner puts a certain message into a newly mined bitcoin block. But I suspect there is another way to do it without a soft fork, by making withdrawals from the sidechain happen at a quick rate "naturally" and at a slower rate if miners explicitly choose to delay them.

I will now explain how. I call the following protocol handcrank because a handcrank is much less efficient than a motor powered drivechain but they accomplish a similar goal.

# Handcrank protocol

## 1. Alice makes a 51 of 51 multisig address with the most recent btc miners who mined a block

Suppose 51% of miners run some software which embeds a fresh public key into the coinbase of every bitcoin block they mine. Any user at any time can select 51 pubkeys from the last 100 mined bitcoin blocks and use them to construct a 51 of 51 musig pubkey. With this pubkey in hand, the user can create a bitcoin “deposit address” which can only be spent from using a "mega signature" that is valid for the musig key. The user will soon send coins to this deposit address, but first they create a series of withdrawal transactions *from* the deposit address, which will be described several paragraphs from now.

The user passes the unsigned transactions to all of the miners whose pubkeys she aggregated into the musig key, then awaits their signature “shards” so she can combine them into a bunch of mega signatures, with two mega signatures for each withdrawal transaction. The full set of withdrawal transactions will comprise a multisig covenant. To complete the covenant, at least 1 of the 51 miners must delete their shard of the musig key, that way the money in the musig address can only go to one of the addresses authorized by the limited number of mega signatures, and not go anywhere else.

## 2 & 3. She deposits X btc to it after she gets the miners to create a “presigned withdrawal” covenant and she announces her covenant on a sidechain to get X “sidechain bitcoins”

When the covenant is ready (i.e. when the user has all the signatures), the user deposits their coins into the musig address and creates a corresponding message on the sidechain, paid for by subtracting a fee from the amount she put into the musig address. The message announces the user’s sidechain address, the signatories to the multisig covenant, and the full text of all of the withdrawal transactions (including all of the signatures) that comprise the multisig covenant. Sidechain nodes detect this message and validate (1) that 51% of miner pubkeys from the last 100 bitcoin blocks are indeed aggregated together in the musig address (2) that they signed all the withdrawal transactions and (3) that the withdrawal transactions all match the specification outlined below. If everything checks out, the user’s sidechain address is credited with an amount of “sidechain bitcoins” corresponding to however much she deposited into the musig address, minus the fee for announcing her message. If any of the above items do not check out, the message is considered invalid and is excluded from the sidechain, and the user's address is not credited.

So far, our user, and all other sidechain nodes, have trusted 51% of the miners to create a covenant, and they trust this covenant to properly hold the user's money for the duration of the time that money is on the sidechain. Here is more information about the withdrawal transactions signed by the miners.

## 4, 5, & 6. She does stuff on the sidechain and (let’s suppose) loses ownership of her money, a new person claims they now own Alice’s deposit and says they want to withdraw it, and the covenant lets 51% of miners send the coins anywhere after a delay of 1.5 months

The withdrawal transactions are a set of several thousand nearly identical bitcoin transactions. The first one is special, it only has one mega signature, which can only be used to send the depositor’s money into an initial “withdrawal address.” Every other withdrawal transaction is signed by two mega signatures. If a withdrawal transaction is broadcasted with one of its corresponding mega signatures as a witness element, it does one of two things.

If the "spend sig" is passed in the witness data and 13,150 blocks (1.5 months) have passed since the money entered the sidechain, the money goes to an anyone_can_spend address. If the "delay sig" is passed in the witness data and 1 block has passed since the money entered this address, the money goes straight back into the current address, but the timelock on the "spend sig" is now incremented by 1, thus delaying the withdrawal slightly. If a miner opts not to broadcast a withdrawal transaction, the transaction will continue to progress through its current timelock one block at a time.

The miners signed 13,150 such transactions (with 2 signatures for each, creating 26,300 mega signatures), so a user's funds might be locked up for as many as 26,300 blocks, or about 3 months, before it goes to an anyone_can_spend address. But if miners do not hinder its progress, withdrawals can be processed in only 13,150 blocks i.e. 1.5 months.

## 7 & 8. Any bitcoin miner can check the new person’s claim and start sending them Alice’s deposit, and during the delay, any miner who rejects the claim can extend the delay up to 3 months

An important feature of these withdrawal transactions is a miner's option to delay the withdrawal. Miners are expected to do this if they see two or more competing messages on bitcoin's blockchain in which people tell them to withdraw the same coins to different addresses. Miners who see such conflicting messages are expected to delay the withdrawal until they discern which withdrawal to process. Miners who aren't paying attention to the sidechain shouldn't do anything – just let the transaction progress through its current timelock.

The following safeguard is in place to prevent users other than miners from trolling the process by constantly broadcasting all of the “delay” transactions (which, remember, are publicly visible on the sidechain): none of the withdrawal transactions (neither the “delay” transactions nor the “spend” transactions) pay mining fees. Therefore miners running default bitcoin core software won't accept them. Miners who participate in the handcrank protocol won’t accept them either because they are supposed to make their own independent decision about which transactions to broadcast, not take suggestions from others.

## 9 & 10. During the delay, the rightful owner is supposed to say where the money should *really* go, and when the delay expires miners send the money to its owner or take it if they never spoke up

Because miners can delay withdrawals, users of the sidechain can have confidence that miners get to decide between competing attempts to withdraw the same coins. Miners can force a withdrawal attempt to crawl along if it's not made by the rightful owner, leaving time for the rightful owner to say where the money should really go, at which point miners can hasten along the withdrawal and send the coins to the right place. Thus miners are in control of sidechain withdrawals.

51% of miners are trusted not to steal a user's coins. They could steal by colluding to keep their signature shards and use them to sign a transaction taking money straight out of the deposit address. This won’t work if at least one of the original 51% of miners was honest and deleted their shard, but that's not the only danger: at any time, the 51% can collude to (1) let transactions proceed quickly no matter what, (2) move all utxos into the anyone_can_spend address with the spend sigs as quickly as possible, (3) immediately (in the same block) move them into one or more of their own addresses, and (4) orphan any blocks containing transactions that try to get any utxos to their rightful owners through the expected procedure.

Users of the sidechain must trust miners to instead follow the expected procedure: make fraudulent withdrawals crawl through their timelocks until their rightful owners speak up. But (1) let rightful transactions proceed quickly, (2) move the utxos into the anyone_can_spend address with the spend sigs, (3) immediately (in the same block) move them into the addresses specified by the rightful owners, and (4) orphan any blocks containing transactions that try to steal user funds. As long as 51% of miners follow this protocol, users should not lose funds. If less than 51% follow this protocol, users of the sidechain can lose their money forever, meaning they mistrusted the miners and should learn their lesson.

# Lifecycle of a handcrank deposit

1. Alice wants to put money on a handcrank sidechain
2. She gets the pubkeys of the last 100 bitcoin miners
3. She smooshes 51 of those pubkeys into a taproot address
4. She asks those 51 miners to sign a multisig covenant and then delete their keys
5. The covenant is a bunch of signed bitcoin transactions which move Alice's deposit out of the taproot address
6. The covenant includes a bunch of signatures called Delay Sigs and Spend Sigs
7. Alice deposits X bitcoins into the taproot address
8. She posts a message on the sidechain announcing the covenant
9. Sidechain nodes validate the covenant but must trust that at least 1 of the 51 miners deleted their key
10. If the covenant checks out, Alice's sidechain address is credited with X "sidechain bitcoins"
11. On the sidechain, Alice transfers her money around and loses ownership of it
12. Some months or years later, a man named Bob posts an Ownership Proof on the sidechain to show he now owns Alice's deposit
13. He simultaneously posts a Withdrawal Request on the sidechain saying where to send the bitcoins
14. They are currently still sitting in the taproot address Alice made
15. Bob posts the hash of his Withdrawal Request on bitcoin's blockchain
16. A bitcoin miner verifies Bob’s Ownership Proof
17. The miner broadcasts the first covenant transaction from step 5
18. The covenant transaction moves Alice's deposit into a bitcoin address with a 1.5 month timelock
19. The covenant transaction includes the hash of Bob’s Withdrawal Request
20. Other bitcoin miners who follow the handcrank protocol validate this first use of a covenant transaction to move Alice's deposit
21. If something's wrong (e.g. Bob’s Ownership Proof doesn't check out and he is NOT the rightful owner), they use Delay Sigs to increase the covenant timelock 1 block at a time
22. The maximum delay is 3 months and starts counting down again if any miner doesn't increase it
23. During this delay, the rightful owner may post a "valid" (according to sidechain rules) Withdrawal Request with accompanying Ownership Proof for miners to validate
24. Until that happens or the 3 months are up (whichever comes first), miners keep delaying
25. When miners stop delaying, the countdown continues at its regular pace
26. When the countdown expires a miner can move the money with the Spend Sig to an anyone_can_spend address
27. In the same block's next transaction, they must move the money to the address in the Withdrawal Request
28. If the rightful owner never made a Withdrawal Request even after all the delays, the miner moves the money to an address of their choice
29. If in step 27 the money went to the wrong address, other miners orphan the block it happened in and replace it with one that does it right
30. End of protocol
