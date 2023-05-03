# Handcrank: a drivechain wannabe with no softfork

# TLDR in 2 sentences

Use a multisig with 51% of miners to emulate a covenant. The covenant lets miners vote on how to spend the money in the covenant address, which is all you need to make a drivechain.

# TLDR in 10 sentences

1. Alice makes a 51 of 51 multisig address with the most recent btc miners who mined a block
2. She deposits X btc to it after she gets the miners to create a “presigned withdrawal” covenant
3. She announces her covenant on a sidechain to get X “sidechain bitcoins”
4. She does stuff on the sidechain and (let’s suppose) loses ownership of her money
5. A new person claims they now own Alice’s deposit and says they want to withdraw it
6. The covenant lets 51% of miners send the coins anywhere after a delay of 3 months
7. Any bitcoin miner can check the new person’s claim and start sending them Alice’s deposit
8. During the delay, any miner who rejects the claim can reset the delay to add 6 more months
9. During the delay, the rightful owner is supposed to say where the money should *really* go
10. After the delay miners send the money to its rightful owner or take it if they never spoke up

# Background

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

Suppose 51% of miners run some software which embeds a fresh public key into the coinbase of every bitcoin block they mine. Any user at any time can select 51 pubkeys from the last 100 mined bitcoin blocks and use them to construct a 51 of 51 musig pubkey. With this pubkey in hand, the user can create a bitcoin “deposit address” which can only be spent from using a "mega signature" that is valid for the musig key. The user will soon send coins to this deposit address, but first they create a series of withdrawal transactions from the deposit address, which will be described several paragraphs from now.

## 2 & 3. She deposits X btc to it after she gets the miners to create a “presigned withdrawal” covenant and she announces her covenant on a sidechain to get X “sidechain bitcoins”

The user passes the unsigned transactions to all of the miners whose pubkeys she aggregated into the musig key, then awaits their signature “shards” so she can combine them into a bunch of mega signatures, with one or two mega signatures for each withdrawal transaction. The full set of withdrawal transactions will comprise a multisig covenant. To complete the covenant, at least 1 of the 51 miners must delete their shard of the musig key, that way the money in the musig address can only go to one of the addresses authorized by the limited number of mega signatures, and not go anywhere else.

When the covenant is ready (i.e. when the user has all the signatures), the user deposits their coins into the musig address and creates a corresponding message on the sidechain, paid for by subtracting a fee from the amount she put into the musig address. The message announces the user’s sidechain address, the signatories to the multisig covenant, and the full text of all of the withdrawal transactions (including all of the signatures) that comprise the multisig covenant. Sidechain nodes detect this message and validate (1) that 51% of miner pubkeys from the last 100 bitcoin blocks are indeed aggregated together in the musig address (2) that they signed all the withdrawal transactions and (3) that the withdrawal transactions all match the specification outlined below. If everything checks out, the user’s sidechain address is credited with an amount of “sidechain bitcoins” corresponding to however much she deposited into the musig address, minus the fee for announcing her message. If any of the above items do not check out, the message is considered invalid and is excluded from the sidechain, and the user's address is not credited.

So far, our user, and all other sidechain nodes, have trusted 51% of the miners to create a covenant, and they trust this covenant to properly hold the user's money for the duration of the time that money is on the sidechain. Here is more information about the withdrawal transactions signed by the miners.

## 4, 5, & 6. She does stuff on the sidechain and (let’s suppose) loses ownership of her money, a new person claims they now own Alice’s deposit and says they want to withdraw it, and the covenant lets 51% of miners send the coins anywhere after a delay of 3 months

The withdrawal transactions are a set of several bitcoin transactions. The first one has one mega signature which can only be used to send the depositor’s money into a timelocked “pending withdrawal” address. Every other withdrawal transaction, if broadcasted, does one of two things.

If the money is in the “pending withdrawal” address and the “spend sig” is passed in the witness data and 13,150 blocks (3 months) have passed since the money entered this address, the money goes to an anyone_can_spend address. If the money is in the “pending withdrawal” address and the “delay sig” is passed in the witness data and 1 block has passed since the money entered this address, the money goes into a “delayed withdrawal” address, which is identical to the current one except the timelock is 6 months (26,300 blocks) instead of 3. If a miner opts not to broadcast a withdrawal transaction, the transaction will continue to progress through its current timelock one block at a time.

Altogether, the miners signed 4 transactions: the first puts the money in the “pending withdrawal” address, the second can only move the money from the “pending withdrawal” address into an anyone_can_spend address after 3 months, the third can only prevent that by putting the money into the “delayed withdrawal” address, and the fourth can only move the money from the “delayed withdrawal” address into an anyone_can_spend address. Miners can delay the withdrawal a maximum possible amount of 9 months by waiting til the 3 month timelock is just about to expire and then put the money in the 6 month timelock. They can delay it for a lesser time by putting the money in the 6 month timelock earlier, or by never putting it in the 6 month timelock, in which case the money only has to wait 3 months. After the delay, the money can only enter an anyone_can_spend address.

## 7 & 8. Any bitcoin miner can check the new person’s claim and start sending them Alice’s deposit, and during the delay, any miner who rejects the claim can reset the delay to add 6 more months

An important feature of these withdrawal transactions is a miner's option to delay the withdrawal. Miners are expected to do this if they notice that another bitcoin miner started the withdrawal procedure without a corresponding message on the sidechain in which someone proved they own the deposit and announced where they want to withdraw it to. Without such a message, the withdrawal procedure should not be initiated by any miner, so if it was, that is suspicious, and extra time should be allotted to the withdrawal so that the rightful owner has time to prove their ownership and say where the money should *really* go. Miners who aren't paying attention to the sidechain shouldn't do anything – just let the transaction progress through its current timelock.

The following safeguard is in place to prevent users other than miners from trolling the process by broadcasting “delay” transactions in blocks where they are not called for (remember, all the withdrawal transactions are publicly visible on the sidechain): none of the withdrawal transactions pay mining fees. Therefore, according to the default policy of bitcoin core, nodes won't relay them and miners won't accept them. But miners who participate in the handcrank protocol can mine them because it's not against bitcoin's consensus rules to mine a feeless transaction, it’s only a default policy which can be overridden through RPC commands by handcrank software. As a result, miners who follow the handcrank protocol can make their own independent decision about which transactions to broadcast without influence from trolls.

## 9 & 10. During the delay, the rightful owner is supposed to say where the money should *really* go, and after the delay miners send the money to its rightful owner or take it if they never spoke up

Because miners can delay withdrawals, users of the sidechain can have confidence that miners get to decide between competing attempts to withdraw the same coins. Miners can force a withdrawal attempt to crawl along if it's not made by the rightful owner, leaving time for the rightful owner to say where the money should really go, at which point miners can hasten along the withdrawal and send the coins to the right place. Thus miners are in control of sidechain withdrawals.

51% of miners are trusted not to steal a user's coins. They could steal by colluding to keep their signature shards and use them to sign a transaction taking money straight out of the deposit address. This won’t work if at least one of the original 51% of miners was honest and deleted their shard, but that's not the only danger: at any time, the 51% can collude to (1) let transactions proceed quickly no matter what, (2) move any utxos that passed their delay period into an anyone_can_spend address, (3) immediately (in the same block) move those utxos into one or more of their own addresses, and (4) orphan any blocks containing transactions that try to get any utxos to their rightful owners through the expected procedure.

Users of the sidechain must trust miners to instead follow the expected procedure: make fraudulent withdrawals crawl through their timelocks until their rightful owners speak up. But (1) let rightful transactions proceed quickly, (2) move any utxos that passed their delay period into an anyone_can_spend address, (3) immediately (in the same block) move those utxos into the addresses specified by the rightful owners, and (4) orphan any blocks containing transactions that try to steal user funds from anyone_can_spend addresses. As long as 51% of miners follow this protocol, users of the sidechain should not lose funds. If less than 51% follow this protocol, users of the sidechain can lose their money forever, meaning they mistrusted the miners and should learn their lesson.

# Lifecycle of a handcrank deposit

1. Alice wants to put money on a handcrank sidechain
2. She gets the pubkeys of the last 100 bitcoin miners
3. She smooshes 51 of those pubkeys into a taproot address
4. She asks those 51 miners to sign a multisig covenant and then delete their keys
5. The covenant is a bunch of signed bitcoin transactions which move Alice's deposit out of the taproot address
6. Alice deposits X bitcoins into the taproot address
7. She posts a message on the sidechain announcing the covenant
8. Sidechain nodes validate the covenant but must trust that at least 1 of the 51 miners deleted their key
9. If the covenant checks out, Alice's sidechain address is credited with X "sidechain bitcoins"
10. On the sidechain, Alice transfers her money around and loses ownership of it
11. Some months or years later, a man named Bob posts an Ownership Proof on the sidechain to show he now owns Alice's deposit
12. He simultaneously posts a Withdrawal Request on the sidechain saying where to send the bitcoins
13. They are currently still sitting in the taproot address Alice made
14. A bitcoin miner verifies Bob’s Ownership Proof
15. The miner broadcasts the first covenant transaction from step 5
16. The covenant transaction moves Alice's deposit into a bitcoin address with a 3 month timelock
17. Other bitcoin miners who follow the handcrank protocol validate this first use of a covenant transaction to move Alice's deposit
18. If something's wrong (e.g. Bob’s Ownership Proof doesn't check out and he is NOT the rightful owner), they can reset the covenant timelock
19. The delay can be as long as 9 months and starts counting down again if any miner doesn't reset it
20. During this delay, the rightful owner may post a "valid" (according to sidechain rules) Withdrawal Request with accompanying Ownership Proof for miners to validate
21. When the delay ends a miner can move the money anywhere
22. But they are considered a thief unless they immediately move the money to the address in the Withdrawal Request
23. If the rightful owner never made a Withdrawal Request even after all the delays, the miner may move the money to an address of their choice
24. If in step 22 the money went to the wrong address, other miners orphan the block it happened in and replace it with one that does it right
25. End of protocol
