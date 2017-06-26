
### Summary

The [blockhash refactoring EIP](http://github.com/ethereum/EIPs/pull/210) allows blocks to directly link to blocks much older than themselves, making all blocks in the chain relatively tightly connected to each other. This allows for very efficient light client proofs that do not require light clients to verify an entire header chain; instead, light clients can verify a subchain containing "key blocks" whose ethash mining result hits an especially low target, and gain a probabilistic assurance that the longest chain contains these key blocks.

### Proof format

We can define a "probabilistic total-difficulty proof" as an RLP list as follows:

    [header1, proof1, header2, proof2, ...]
    
Where each header is a block header, and each proof[i] is a Merkle branch from header[i] to the hash of header[i + 1]. More formally, the proof is an RLP list where each element contains as a substring the hash of the next element, and the last element is the hash of the next header; the elements are taken from the branch of the state tree in header[i] that points to the hash of header[i + 1] that is available in the storage of the BLOCKHASH_CONTRACT_ADDR.

The proof serves to verify that the headers are linked to each other in the given order, and that the given chain has an approximate total difficulty equal to `sum(2 ** 256 / mining_result[i])` where `mining_result[i]` is the result of running the ethash verification function on the given header. A node producing a proof will take case to create a proof that contains as many low-mining-result blocks as possible; a specific algorithm would be to look for all "key blocks" whose mining result is less than 1/50000 of maximum for a valid block allowed by the most recent block difficulty, and then if these blocks do not have a direct connection because they are not an even multiple of 256 or 65536 apart, it would find "glue blocks" to link between them; for example, linking 3904322 to 3712498 might go through 3735552 (multiple of 65536, directly linked in 3904322) and 3712512 (multiple of 256, directly linked in 3735552), and finally 3712512 links directly to 3712498.

### Light client algorithm

Suppose that a light client receives a proof as above. The light client can try to answer this question: suppose that there is an attacker with the same amount of hashpower as the main chain. Given some probability threshold p (say, 1 in 1 million), how many blocks would the main chain be able to grow with probability 1-p in the same amount of time that the attacker makes their proof?

For example, let's suppose that we have a proof 8 blocks long, with mining results `[8937, 3047, 2601, 1612, 273, 3278, 1220, 1942]` (in a real-world example, all of these numbers would of course be very large values somewhere around 2**200). We can set up a thought experiment where an attacker and the "main chain" create blocks in parallel; the question asked is, what is the number TD such that while the main chain creates a number of blocks with the given total difficulty TD, the attacker can come up with at least 8 blocks with mining result less than or equal to 8937? The answer comes from the Poisson distribution's cumulative distribution function formula; the answer is, enough TD to create 81 blocks of mining result less than 1 million. Note that this carries high inefficiency: the simulation that actually gave that output involved 800 blocks of mining result less than 1 million. We can alleviate this problem by choosing a higher proof length, say 56; this would get us to the point where we cross the threshold at N = 400.

Now, we can get to our algorithm. A light client starts off knowing the genesis, and asks for probabilistic total-difficulty proofs. It stores a map `height_diffs: {(block_hash_from, block_hash_to): min length}`, which starts off as `{genesis: 0}`. When it receives a proof, it sets `height_diffs[(LAST_BLOCK, FIRST_BLOCK)] = max(height_diffs[(LAST_BLOCK, FIRST_BLOCK)], min_length(PROOF))`, where `min_length` is a function that computes the min length as above. Then, at any point the algorithm can find a min-height of any given block by applying any pathfinding algorithm ([Bellman-Ford](https://en.wikipedia.org/wiki/Bellman%E2%80%93Ford_algorithm) is ideal) to get the total min-length from the genesis to that block. The block with the highest min-height is taken to be the head.

Note that it is expected that proofs for the portion of the chain further back in the tail will be more "rarefied", including only perhaps one block per 20000 heights, but closer to the head the proofs will become "tighter", as there will be fewer blocks between the start of the proof and the head, and so the distance between successive blocks would need to be smaller for the proof to still contain enough blocks to have a reasonably high min-length. Once a client asks for proofs starting from a header that is less than ~250 blocks from the head, it may become reasonable to simply give it the entire sub-chain, ie. `[header[n], header[n+1], ... , header[n + 250]]`; the light client can interpret a sub-chain as a proof with a min-length that is simply equal to the total difficulty of the sub-chain.

### Estimated complexity

Suppose that there are 1 million blocks between the genesis (actually, we can use the Metropolis hardfork block as a "genesis", as this algorithm does not work before the hard fork anyway) and the head. Suppose that a client wants to be secure against attackers with up to half the hashpower of the main network with p = 0.999999. Then, the client would want to ensure that *any* subchain from the genesis to the head has a min-length of at least half its claimed length. We already roughly know from above that it takes 64 headers to get to that level. As an approximation, the client can ask for such a subchain for blocks 0....500000, then 500000...750000, and so forth. At 999750, it can stop, and simply take 250 block headers. In total, this would entail 12 proofs (768 headers), plus 768 Merkle branches, plus another 250 headers. A branch on average takes ~1000 bytes, a header takes ~500 bytes, so this would be ~1.27 MB, and would require 1018 ethash verifications from 30 epochs (so an additional 30 cache generations).