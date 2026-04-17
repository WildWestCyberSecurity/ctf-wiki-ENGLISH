# Blockchain Weaknesses

## Double-spending attack

The double-spending problem is a potential issue in electronic cash systems where the same funds are simultaneously paid to two recipients. Blockchain systems avoid the double-spending problem through a public ledger. When a user broadcasts a transaction, it is not immediately added to the blockchain. Instead, it waits for miners to package it into a block through mining. The recipient can consider the transaction valid only after confirming that the legitimate transaction has been added to the blockchain. Users protect themselves from double-spending fraud by waiting for confirmations when receiving payments on the blockchain. As the number of confirmations increases, the transaction becomes more irreversible.

However, blockchain systems cannot completely prevent double-spending. Attackers can still reverse transactions to achieve double-spending by using enormous hash computing resources to redo the proof of work for confirmed blocks and make their chain the longest chain. They can also carry out double-spending attacks against certain unconfirmed transactions. Currently, there are several common methods that can trigger double-spending attacks:

### 51% attack

A 51% attack refers to a miner attempting to control more than 50% of the hash rate (mining power) to achieve double-spending. In this attack, the attacker can prevent new transactions from being confirmed or reverse transactions that were already confirmed while they controlled the network.

Because PoW-based blockchains like Bitcoin follow the longest chain rule, when a miner discovers a longer chain on the network, they will abandon their current chain, copy the new longer chain entirely, and continue mining on that chain, while the shorter forked chain is discarded. Therefore, after completing a transaction with a merchant, the attacker forks the blockchain from before the transaction and uses their sufficient computing power to continuously mine a longer chain, causing the chain containing the transaction to be discarded, thus achieving double-spending.

Theoretically, the 51% attack cannot be prevented through technology alone; instead, it is avoided through economic principles by making the cost of achieving 51% of the network's computing power extremely high. An attacker with such powerful computing power would earn more from honest mining than from malicious behavior. However, for small-scale altcoins, the attack cost is relatively low. For example, the Ethereum Classic network has suffered multiple 51% attacks.

Reference: https://en.bitcoin.it/wiki/Weaknesses#Attacker_has_a_lot_of_computing_power

- Challenge Name: miniblockchain

### Finney attack

The Finney attack is named after Hal Finney, who was the first to describe the block withholding attack. This attack is a variant of the double-spending attack, primarily targeting merchants who accept 0-confirmation transactions.

The attacker pre-mines a block containing a transaction that sends funds to themselves but does not immediately broadcast it to the network. Instead, they spend the same coins in a transaction with a merchant that accepts 0-confirmation transactions. After the attacker obtains the merchant's exchanged goods but before transaction A is actually confirmed, they broadcast the previously pre-mined block to make the self-transfer transaction legitimate. At this point, the Bitcoin network will accept the valid block and invalidate the transaction to the merchant, ultimately achieving the double-spending objective.

0-confirmation refers to the state of a transaction that has been broadcast to the entire network and is about to be packaged into a block. Because the current block time on the blockchain is too slow and transaction confirmation requires a long waiting period, some merchants accept 0-confirmation transactions to save time — meaning you only need to broadcast the transaction information to the entire network without waiting for it to be packaged into a block.

### Race attack

The attacker broadcasts two conflicting transactions using the same funds in quick succession, but ultimately only one transaction gets confirmed. This attack mainly controls miner fees to achieve double-spending and also targets merchants that accept 0-confirmation transactions. The final result is that the transaction sent to the attacker's own address gets packaged and confirmed while the other payment transaction becomes invalid.  
The difference from the Finney attack is that the Finney attack is a 0-confirmation transaction vs. a conflicting block, while the race attack is a 0-confirmation transaction vs. a conflicting transaction.

### Vector76 attack

Also known as the one-confirmation attack, it is a combination of the Race attack and the Finney attack, allowing a transaction with one confirmation to still be reversed.

In this attack, a miner creates two nodes — one connected to the merchant's node and the other connected to well-connected nodes in the blockchain network. Then, the miner creates two transactions using the same funds: one sent to the merchant's address (called Transaction 1), and one sent to their own wallet address with a higher miner fee (called Transaction 2).

The attacker does not immediately broadcast these two transactions but instead mines on the branch containing Transaction 1. After the attacker mines a block, they broadcast Transaction 1 to the merchant's node and Transaction 2 to the other node. After Transaction 2 is considered valid, the attacker immediately broadcasts the block they previously mined on the Transaction 1 branch to the merchant. At this point, the merchant who accepts one-confirmation payments will confirm the transaction as successful.

Since Transaction 2 was sent to a node connected to more nodes, miners have a higher probability of mining a longer chain on this branch. In this case, Transaction 1 will be rolled back, thus achieving double-spending.

Reference:  
http://bitcointalk.org/index.php?topic=36788.msg463391#msg463391
http://www.reddit.com/r/Bitcoin/comments/2e7bfa/vector76_double_spend_attack/cjwya6x

## Block withholding attack

The simplest form of a block withholding attack is the Finney attack described above, but there are also block withholding attacks targeting [mining pools](https://academy.binance.com/zh/articles/mining-pools-explained).

The most common payment mechanism for mining pools is PPS (Pay-Per-Share). In this mechanism, miners receive a fixed reward for each "share" they contribute. A share is used to record the hash values contributed by miners — the share here is not a valid hash for the blockchain network but only a result matching the conditions set by the mining pool. Since finding a solution that meets the PoW blockchain system requirements is an extremely low-probability event for individual miners, mining pools set a reasonable threshold for miners to submit their work results (shares) to better measure their workload.

A block withholding attack occurs when a malicious miner finds a result that meets the mining pool's requirements but does not meet the Bitcoin system's requirements, and they normally submit the proof of work to the pool. However, once they find a result that meets the Bitcoin system's requirements — meaning they actually mined a block — they withhold this result privately and do not submit it to the pool, causing the pool to lose the corresponding reward. Block withholding attacks cause losses to both the miner and the pool: the miner simply doesn't receive the pool's shared reward, but the pool loses the block reward.

## Selfish-Mining attack

After mining a new block, the attacker hides it and does not publish it. Other honest miners, unaware of the new block's existence, continue mining on top of the old block. When the attacker mines a second block, they simultaneously publish both hidden blocks, creating a blockchain fork. As long as the attacker has mined one more block than the honest miners, the attacker's fork becomes the longest chain. Therefore, the chain that the honest miners were working on is invalidated because it is shorter than the attacker's forked chain. At this point, the attacker receives the corresponding rewards for mining two new blocks, while the honest miners' rewards are rolled back.

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.
