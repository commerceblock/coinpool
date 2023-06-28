# Coinpool/Joinpool concepts

## Coinpools

N > 2 participants lock their funds in a single UTXO, can transact (with existing other members) within this UTXO off-chain, can send funds to new members with an on-chain tx, and are able to withdraw their coins from the pool unilaterally at any time. Can be thought of as a multiparty payment channel. 

https://coinpool.dev/v0.1.pdf

Coins can be unilaterally withdrawn via taproot MAST paths. A covenant mechanism is required to ensure the unilateral withdrawal transaction has outputs that reflect the pool ownership.  

### Requirements

The current design of coinpool require consensus soft-forks to add two new sighash (`SIGHASH_ANYPREVOUT`, and `SIGHASH_GROUP`) and a new script op code `OP_MERKLESUB`. 

`OP_MERKLESUB` is an opcode that enables covenants. It enforces the spending transaction output to be the same as the Taproot output being spent. 

### Limitations

Main limitation is the interactivity requirement. All current members of the pool are required to sign updates to the state. If any participant is not responsive, remaining members must withdraw on-chain. 

### Interactivity/Trust

Some solutions to reduce the interactivity
requirements. Pools members could hold nLocktime timelocked kick-out transactions to evict offline
users who do not respond before expiration.

Others possibilities:

- Non-interactive Off-chain Pool Partitions
- Trusted coordinator
- Trust-minimized : Partition Statements

https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-April/020370.html

## Current rule versions

With the current bitcoin consensus rules, taproot has been enabled. This means that `P2TR` addresses and UTXOs can be created, enabling both TR spending paths and Schnorr signature spends. 

The use of Schnorr signatures enables efficient threhold signature schemes for relative large groups of pubkeys. 

### P2TR MuSig

One potential scheme that could be implimented with the current bitcoin consensus rules involves a party of `N > 2` participants paying coins to a P2TR address which is a single pubkey shared by all participants (`N of N`). This UTXO can then only be spent from if all `N` participants agree on the spending transaction and co-sign with the `N of N` keys via the MuSig protocol. 

Before depositing coins into the shared P2TR address (generated via a MuSig round), a sequence of 'backout transactions' are signed by all participants which reflect the ownership of coins deposited. The backout tx(s) enable withdrawal of funds if any other participants are unresponsive or refuse to sign an orderly withdrawal. 

Orderly withdrawals, or new coins being added to the pool, or payments (transfer of ownership) to new participants outside the pool, are performed by generating a new shared pubkey (for the new group, `N + M - P` where `M` is new members and `P` is withdrawn members) and corresponding set of backout transactions reflecting the new ownership. 

### Backout trees

The simplest backout transaction concept is for a single transaction (without timelock) that simply reflects ownership of coins in the pool. Each participant would hold a copy of this backout transaction and could broadcast it at any point to unilaterally withdraw their share of the coins. This however presents a substantial greifing risk to the other participants - as all other coins would be 'withdrawn' at the same time. 

In addition, this mass/enforced withdrawal would need to happen any time any member of the group became unresponsive. 

The greifing risk could be minimised with a fee penalty for the backout. The transaction fee for this tx would be required to be set conservatively high to preempt future mempool congestions (and would be larger due to the size of the tx), therefore it would always make econimic sense for any participant to do a cooperative withdrawal if possible. 

The risk of a participant being unresponsive could be mitigated with a tree/set of backouts multilaterally removing a single or multiple particiapnts, and the depth of this tree would be chosen as a balance between efficiency and risk. For each participant a shared pubkey would be generated for the remaining `N-1` particiapnts, and a tx created a signed paying the balance minus the withdrawal to this pubkey (P2TR) and the particiapants withdrawal to their key. Also this new shared pubkey would have a full backout tx spending from it. 

So for `N` particiapnts there would need to be `N` new shared pubkeys generated and `2N` signatures for each on-chain state update. But a single non-responsive particiapt could be evicted from the group. 

### Decrementing timelocks

Off-chain transfers within the group can be performed in the same way described in the chainpool paper but with decrementing `nLocktime` timelocks replacing the Eltoo mechanism (which requires the `SIGHASH_ANYPREVOUT` soft fork). The initial backup `nLocktime` would be set to a future block height and then decremented with each state update (requiring all participants sign) in the same way as mercury statechains. This would have to downside of setting a timelimit to the onchain coinpool UTXO and requiring a delay before unilateral backouts could be confirmed. 

### Coodinator roles

