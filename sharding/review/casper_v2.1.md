# Full Casper chain v2.1

## WORK IN PROGRESS!!!!1!!1! by @vbuterin - 20180721 21:48 (UTC+8)

This is the work-in-progress document describing the specification for the Casper+Sharding chain, version 2.1.

In this protocol, there is a central PoS chain which stores and manages the current set of active PoS validators. The only mechanism available to become a validator initially is to send a transaction on the existing PoW main chain containing 32 ETH. When you do so, as soon as the PoS chain processes that block, you will be queued, and eventually inducted as an active validator until you either voluntarily log out or you are forcibly logged out as a penalty for misbehavior.

The primary source of load on the PoS chain is **attestations**. An attestation has a double role:

1. It attests to some parent block in the beacon chain
2. It attests to a block hash in a shard (a sufficient number of such attestations create a "cross-link", confirming that shard block into the main chain).


Every shard (e.g. there might be 1024 shards in total) is itself a PoS chain, and the shard chains are where the transactions and accounts will be stored. The cross-links serve to "confirm" segments of the shard chains into the main chain, and are also the primary way through which the different shards will be able to talk to each other.

Note that one can also consider a simpler "minimal sharding algorithm" where cross-links are simply hashes of proposed blocks.

Note: the python code at https://github.com/ethereum/beacon_chain does not reflect all of the latest changes. If there is a discrepancy, this document is likely to reflect the more recent changes.

### Terminology

* **Validator** - a participant in the Casper/sharding consensus system. You can become one by depositing 32 ETH into the Casper mechanism.
* **Active validator set** - those validators who are currently participating, and which the Casper mechanism looks to produce and attest to blocks, cross-links and other consensus objects.
* **Committee** - a (pseudo-) randomly sampled subset of the active validator set. When a committee is referred to collectively, as in "this committee attests to X", this is assumed to mean "some subset of that committee that contains enough validators that the protocol recognizes it as representing the committee".
* **Proposer** - the validator that creates a block
* **Attester** - a validator that is part of a committee that needs to sign off on a block.
* **Beacon chain** - the central PoS chain that is the base of the sharding system.
* **Shard chain** - one of the chains on which transactions take place and account data is stored.
* **Cross-link** - a set of signatures from a committee attesting to a block in a shard chain, which can be included into the beacon chain. Cross-links are the main means by which the beacon chain "learns about" the updated state of shard chains.
* **Epoch** - a period of 64 blocks.
* **Finalized**, **justified** - see Casper FFG finalization here: https://arxiv.org/abs/1710.09437

### Constants

* **SHARD_COUNT** - a constant referring to the number of shards. Currently set to 1024.
* **ETH_SUPPLY_CAP** - self-explanatory. Currently set to 2<sup>27</sup> ~= 134 million.
* **DEPOSIT_SIZE** - 32 ETH
* **MAX_VALIDATOR_COUNT** - `ETH_SUPPLY_CAP / DEPOSIT_SIZE = 4194304`
* **EPOCH_LENGTH** - 64 blocks
* **SLOT_DURATION** - 8 seconds
* **MIN_COMMITTEE_SIZE** - 128
* **END_EPOCH_GRACE_PERIOD** - 8 blocks

### Main chain changes

This PoS/sharding proposal can be implemented separately from the existing PoW main chain. Only two changes to the PoW main chain are required (and the second one is technically not strictly necessary).

* On the PoW main chain a contract is added; this contract allows you to deposit `DEPOSIT_SIZE` ETH; the `deposit` function also takes as arguments (i) `pubkey` (bytes), (ii) `withdrawal_shard_id` (int), (iii)  `withdrawal_addr` (address) and (iv) `randao_commitment` (bytes32)
* PoW Main chain clients will implement a method, `prioritize(block_hash, value)`. If the block is available and has been verified, this method sets its score to the given value, and recursively adjusts the scores of all descendants. This allows the PoS beacon chain's finality gadget to also implicitly finalize main chain blocks.

### Beacon chain

The beacon chain is the "main chain" of the PoS system. The beacon chain's main responsibilities are:

* Store and maintain the set of active, queued and exited validators
* Process cross-links (see above) 
* Process its own block-by-block consensus, as well as the FFG finality gadget

Here are the fields that go into every beacon chain block:

```python
fields = {
    # Hash of the parent block
    'parent_hash': 'hash32',
    # Slot number (for the PoS mechanism)
    'slot_number': 'int64',
    # Randao commitment reveal
    'randao_reveal': 'hash32',
    # Attestation votes
    'attestation_votes': [AttestationVote],
    # Reference to main chain block
    'main_chain_ref': 'hash32',
    # Hash of the active state
    'active_state_hash': 'hash32',
    # Hash of the crystallized state
    'crystallized_state_hash': 'hash32',
}
```

The beacon chain state is split into two parts, _active state_ and _crystallized state_.

Here's the active state:

```python
fields = {
    # Total quantity of wei that attested for the most recent checkpoint
    'total_attester_deposits': 'int64',
    # Who attested
    'attester_bitfield': 'bytes'
}
```

Here's the crystallized state:

```python
fields = {
    # List of active validators
    'active_validators': [ValidatorRecord],
    # List of joined but not yet inducted validators
    'queued_validators': [ValidatorRecord],
    # List of removed validators pending withdrawal
    'exited_validators': [ValidatorRecord],
    # The permutation of validators used to determine who
    # cross-links what shard in this epoch
    'current_epoch_shuffling': ['int24'],
    # The current epoch
    'current_epoch': 'int64',
    # The last justified epoch
    'last_justified_epoch': 'int64',
    # The last finalized epoch
    'last_finalized_epoch': 'int64',
    # The current dynasty
    'current_dynasty': 'int64',
    # The next shard that cross-linking assignment will start from
    'next_shard': 'int16',
    # The current FFG checkpoint
    'current_checkpoint': 'hash32',
    # Records about the most recent crosslink `for each shard
    'crosslink_records': [CrosslinkRecord],
    # Total balance of deposits
    'total_deposits': 'int256',
    # Used to select the committees for each shard
    'crosslink_seed': 'hash32',
    # Last epoch the crosslink seed was reset
    'crosslink_seed_last_reset': 'int64'
}
```

Each ValidatorRecord is an object containing information about a validator:

```python
fields = {
    # The validator's public key
    'pubkey': 'int256',
    # What shard the validator's balance will be sent to 
    # after withdrawal
    'withdrawal_shard': 'int16',
    # And what address
    'withdrawal_address': 'address',
    # The validator's current RANDAO beacon commitment
    'randao_commitment': 'hash32',
    # Current balance
    'balance': 'int64',
    # Dynasty where the validator can
    # (be inducted | be removed | withdraw their balance)
    'switch_dynasty': 'int64'
}
```

And a CrosslinkRecord contains information about the last fully formed crosslink to be submitted into the chain:

```python
fields = {
    # What epoch the crosslink was submitted in
    'epoch': 'int64',
    # The block hash
    'hash': 'hash32'
}
```

### Beacon chain processing

Processing the beacon chain is fundamentally similar to processing a PoW chain in many respects. Clients download and process blocks, and maintain a view of what is the current "canonical chain", terminating at the current "head". However, because of the beacon chain's relationship with the existing PoW chain, and because it is a PoS chain, there are differences.

For a block on the beacon chain to be processed by a node, three conditions have to be met:

* The parent pointed to by the `parent_hash` has already been processed and accepted
* The main chain block pointed to by the `main_chain_ref` has already been processed and accepted
* The node's local clock time is greater than or equal to the minimum timestamp as computed by `GENESIS_TIME + slot_number * SLOT_DURATION`

If these three conditions are not met, the client should delay processing the block until the three conditions are all satisfied.

Block production is significantly different because of the proof of stake mechanism. A client simply checks what it thinks is the canonical chain when it should create a block, and looks up what its slot number is; when the slot arrives, it either proposes or attests to a block as required.

### Beacon chain fork choice rule

The beacon chain uses the Casper FFG fork choice rule of "favor the chain containing the highest-epoch justified checkpoint". To choose between chains that are all descended from the same justified checkpoint, the chain uses "recursive proximity to justification" to choose a checkpoint, then uses GHOST within an epoch.

RPJ works as follows. First, start at the latest justified checkpoint. Then, choose the descendant checkpoint that has the most attestations supporting it. Repeat until you get to a checkpoint with no children.

For example:

![](https://yuml.me/8d100380.png =225x600)

## Beacon chain state transition function

We now define the state transition function. At the high level, the state transition is made up of two parts:

1. The epoch transition, which happens only if `slot_count // 64 > epoch_number`, and affects the crystallized state and active state
2. The per-block processing, which happens every block (if during an epoch transition block, it happens after the epoch transition), and affects the active state only

The epoch transition generally focuses on changes to the validator set, including adjusting balances and adding and removing validators, as well as processing cross-links and setting up FFG checkpoints, and the per-block processing generally focuses on verifying aggregate signatures and saving temporary records relating to the in-block activity in the active state.

### Helper functions

We start off by defining some helper algorithms. First, an algorithm for pseudorandomly shuffling the validator set based on some seed:

```python
def get_shuffling(seed, validator_count):
    assert validator_count <= 16777216
    rand_max = 16777216 - 16777216 % validator_count
    o = list(range(validator_count)); source = seed
    i = 0
    while i < validator_count:
        source = blake(source)
        for pos in range(0, 30, 3):
            m = int.from_bytes(source[pos:pos+3], 'big')
            remaining = validator_count - i
            if remaining == 0:
                break
            if validator_count < rand_max:
                replacement_pos = (m % remaining) + i
                o[i], o[replacement_pos] = o[replacement_pos], o[i]
                i += 1
    return o
```

The following algorithm is used to split up validators into groups at the start of every epoch, determining at what height they can make attestations and what shard they are making crosslinks for:
     
```python
def get_cutoffs(validator_count):
    height_cutoffs = [0]
    # EPOCH_LENGTH // phi
    cofactor = 39
    STANDARD_COMMITTEE_SIZE = MAX_VALIDATOR_COUNT // SHARD_COUNT
    # If there are not enough validators to fill a minimally
    # sized committee at every height, skip some heights
    if validator_count < EPOCH_LENGTH * MIN_COMMITTEE_SIZE:
        height_count = validator_count // MIN_COMMITTEE_SIZE or 1
        heights = [(i * cofactor) % EPOCH_LENGTH
                   for i in range(height_count)]
    # If there are enough validators, fill all the heights
    else:
        height_count = EPOCH_LENGTH
        heights = list(range(EPOCH_LENGTH))
            
    filled = 0
    for i in range(EPOCH_LENGTH - 1):
        if not i in heights:
            height_cutoffs.append(height_cutoffs[-1])
        else:
            filled += 1
            height_cutoffs.append(filled * validator_count // height_count)
    height_cutoffs.append(validator_count)
    
    # For the validators assigned to each height, split them up
    # into committees for different shards. Do not assign the
    # last 8 heights in an epoch to any shards.
    shard_cutoffs = [0]
    for i in range(EPOCH_LENGTH - 8):
        size = height_cutoffs[i+1] - height_cutoffs[i]
        shards = (size + STANDARD_COMMITTEE_SIZE - 1) // STANDARD_COMMITTEE_SIZE
        pre = shard_cutoffs[-1]
        for j in range(1, shards+1):
            shard_cutoffs.append(pre + size * j // shards)
        
    return height_cutoffs, shard_cutoffs
```

This splits up the validator set into groups for each height, and within each height into groups for different shards. Note that if the validator set is too small to support a committee at every height, some heights are simply left "inactive".

The `current_shuffling` is recalculated at the start of a dynasty transition.

### Per-block processing

The per-block processing is broken down into three parts.

#### Checking partial crosslink records

A block can have 0 or more `AggregateVote` objects, where each `AggregateVote` object has the following fields:

```python
fields = {
    'height': 'int16',
    'parent': 'hash32',
    'checkpoint_hash': 'hash32',
    'shard_id': 'int16',
    'shard_block_hash': 'hash32',
    'notary_bitfield': 'bytes',
    'aggregate_sig': ['int256']
}
```

For each one of these votes:

* Verify that `height <= slot_number`
* Verify that `height >= slot_number = slot_number % EPOCH_LENGTH`
* Use `get_cutoffs` above to get the index cutoffs for the heights and the shards. If `height_in_epoch < EPOCH_LENGTH - 8`, verify that `si = (shard_id - next_shard) % SHARD_COUNT` is a valid shard index for that height (ie. check `height_cutoffs[height_in_epoch] <= shard_cutoffs[si] < height_cutoffs[height_in_epoch + 1])`). Otherwise, verify `shard_id == 65535` and `shard_block_hash == 0`.
* If `height_in_epoch < EPOCH_LENGTH - 8`, let `start = shard_cutoffs[si]` and `end = shard_cutoffs[si+1]`. Otherwise, let `start = height_cutoffs[height_in_epoch]` and `end = height_cutoffs[height_in_epoch + 1]`
* Verify that `len(notary_bitfield) == ceil_div8(end - start)`, where `ceil_div8 = (x + 7) // 8`. Verify that bits `end-start....` and higher, if present (ie. `end-start` is not a multiple of 8), are all zero
* Take all indices `0 <= i < end-start` where the ith bit of `notary_bitfield` equals 1, take `start+i` as the indices of those validators, extract their pubkeys, and add them all together to generate the group pubkey.
* Verify that the `checkpoint_hash` matches the `current_checkpoint`
* Verify that the `aggregate_sig` verifies using the group pubkey and `hash(height_in_epoch + parent + checkpoint_hash + shard_id + shard_block_hash)` as the message.
* AND all indices taken above into the `attester_bitfield`. Add the `balance` of any newly added validators into the `total_attester_deposits`.

Extend the list of `AggregateVote` objects in the `active_state` , ordering the new additions by `shard_block_hash`.

Verify that the `AggregateVote` objects include the first attester at that height (ie. `height_cutoffs[height_in_epoch]`); this attester will be the proposer of the block.

### Epoch transitions

When the current block height (ie. the height in the active state) is 0 mod SHARD_COUNT, we execute an epoch transition, which breaks down into several parts. Note that all validator balance changes only take effect after all of these parts finish.

#### Calculate rewards for FFG votes

* If `total_attester_deposits * 3 >= total_deposits * 2`, set `crystallized_state.justified_epoch` to equal `crystallized_state.current_epoch` (note: this is still the previous epoch at this point in the computation). If this happens, and the justified epoch was previously `crystallized_state.current_epoch - 1`, set the `crystallized_state.finalized_epoch` to equal that value.
* Compute the `online_reward` and `offline_penalty` based on the Casper FFG incentive and quadratic leak rules (not yet fully specified)
* Add the `online_reward` to every validator who participated in the last epoch, and subtract the `offline_penalty` from everyone who did not

### Calculate rewards for crosslinks

Repeat for every shard S:

* Take the set of `shard_block_hash` values that have been proposed for S (call this ROOTS)
* Take the ROOT in ROOTS such that the total deposit size of the subset of validators that have voted for `shard_block_hash` ROOT and `shard_id` S is maximal; call this BESTROOT
* For every validator that voted for BESTROOT, increment their balance by `online_crosslink_reward`. For everyone else who was assigned to a shard (ie. not the proposers of the last 8 blocks), decrement their balance by `offline_crosslink_penalty`
* If the size of the subset that voted * 3 >= `total_deposits * 2`, and `crosslink_records[shard_id].epoch < last_finalized_epoch`, set `crosslink_records[shard_id]` to `{epoch: current_epoch, hash: BESTROOT}`


### Reshuffling

If `current_epoch - crosslink_seed_last_reset` is an exact power of two, then using `temp_seed = blake(crosslink_seed + bytes([log2(current_epoch - crosslink_seed_last_reset)]))`, set  `current_shuffling = get_shuffling(temp_seed, len(new_active_validators))`. Note that all of the temp committees are predictable from the `crosslink_seed`, but their positioning stretches out exponentially (eg. 10001 to 10002, 10002 to 10004, 10004 to 10008, 10008 to 10016...).

The exponential backoff ensures that, if there has not been a crosslink for N epochs, the next committee will have ~N/2 epochs to process through and verify the ~N epochs worth of history for that shard, so the amount of work per second that the committee nodes need to perform is bounded.

### Dynasty transition

If the last two epochs were both justified, then:

* Set `crystallized_state.dynasty += 1`
* Set `crosslink_seed = blake(crosslink_seed)` # Pending a real RNG
* Go through all `queued_validators`, in order from first to last. Any validators whose `switch_dynasty` is equal to or earlier than the new dynasty are immediately added to the end of `active_validators` (setting `switch_dynasty` to the highest possible integer), up to a maximum of `(len(crystallized_state.active_validators) // 30) + 1` (notice that because `queued_validators` is sorted, it suffices to check the queued validators at the beginning until either you've processed that many, or you reach one whose `switch_dynasty` is later than the current dynasty)
* Go through all `active_valdidators`, and move any with either (i) balance equal to <= 50% of their initial balance, or (ii) `switch_dynasty` equal to or less than the new current dynasty, to `exited_validators`, moving up to a maximum of `(len(crystallized_state.active_validators) // 30) + 1` validators

### Miscellaneous

* Increment the current epoch
* Set the `current_checkpoint` to the hash of the previous block
* Reset the FFG voter bitfield and AggregateVote list


-------

Note: this is ~80% complete. Main sections missing are:

* Logic for the formats of shard chains, who proposes shard blocks, etc (in an initial release, if desired we could make cross-links just be Merkle roots of blobs of data; in any case, one can philosophically view the whole point of the shard chains as being a coordination device for choosing what blobs of data to propose as crosslinks)
* Logic for inducting queued validators from the main chain
* Penalties for signing or attesting to non-canonical-chain blocks (update: may not be necessary, see https://ethresear.ch/t/attestation-committee-based-full-pos-chains/2259)
* Slashing conditions
* Logic for withdrawing deposits to shards
* Per-validator proofs of custody
