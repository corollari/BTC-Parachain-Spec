.. _data-model:


Data Model
============

The BTC-Relay, as opposed to Bitcoin SPV clients, only stores a subset of information contained in block headers and does not store transactions. 
Specifically, only data that is absolutely necessary to perform correct verification of block headers and transaction inclusion is stored. 

Types
~~~~~

RawBlockHeader
..............

An 80 bytes long Bitcoin blockchain header.

*Substrate* ::

   pub type RawBlockHeader = [u8; 80];


Constants
~~~~~~~~~

DIFFICULTY_ADJUSTMENT_INTERVAL
..............................

The interval in number of blocks at which Bitcoin adjusts its difficulty (approx. every 2 weeks = 2016 blocks).

*Substrate* ::

  const DIFFICULTY_ADJUSTMENT_INTERVAL: u32 = 2016;

TARGET_TIMESPAN
...............

Expected duration of the different adjustment interval in seconds, ``1209600`` seconds (two weeks) in the case of Bitcoin.

*Substrate* ::

  const TARGET_TIMESPAN: u32 = 1209600;

TARGET_TIMESPAN_DIVISOR
.......................

Auxiliary constant used in Bitcoin's difficulty re-target mechanism. 

*Substrate* ::

  const TARGET_TIMESPAN_DIVISOR: u32 = 4;
   
UNROUNDED_MAX_TARGET
....................

The maximum difficulty target, :math:`2^{224}-1` in the case of Bitcoin. For more information, see the `Bitcoin Wiki <https://en.bitcoin.it/wiki/Target>`_.

*Substrate* ::

    const UNROUNDED_MAX_TARGET: U256 = U256([
    <u64>::max_value(),
    <u64>::max_value(),
    <u64>::max_value(),
    0x0000_0000_ffff_ffffu64,
  ]);

MAIN_CHAIN_ID
.............

Identifier of the Bitcoin main chain tracked in the ``ChainsIndex`` mapping. At any point in time, the ``BlockChain`` with this identifier is considered to be the main chain and will be used to transaction inclusion verification.

*Substrate* ::

    const MAIN_CHAIN_ID: u32 = 0;


STABLE_TRANSACTION_CONFIRMATIONS
.................................

Global security parameter (typically referred to as ``k`` in scientific literature), determining the umber of confirmations (in blocks) necessary for a transaction to be considered "stable". Stable thereby means that the probability of the transaction being excluded from the blockchain due to a fork is negligible. 

*Substrate* ::

    const STABLE_TRANSACTION_CONFIRMATIONS: u32 = 6;

Structs
~~~~~~~
  
BlockHeader
..................

Representation of a Bitcoin block header, as stored in the 80 byte byte representation in the Bitcoin block chain (contains **no additional metadata** - see :ref:`RichBlockHeader`). 
This struct is only used for parsing the 80 byte block header - not for storage! 

.. note:: Fields marked as [Optional] are not critical for the secure operation of BTC-Relay, but can be stored anyway, at the developers discretion. We omit these fields in the rest of this specification. 

.. tabularcolumns:: |l|l|L|

======================  =========  ========================================================================
Parameter               Type       Description
======================  =========  ========================================================================
``merkleRoot``          byte32     Root of the Merkle tree referencing transactions included in the block.
``target``              u256       Difficulty target of this block (converted from ``nBits``, see `Bitcoin documentation <https://bitcoin.org/en/developer-reference#target-nbits>`_.).
``timestamp``           timestamp  UNIX timestamp indicating when this block was mined in Bitcoin.
``hashPrevBlock``       byte32     Block hash of the predecessor of this block.
.                       .          .
``version``             i32        [Optional] Version of the submitted block.
``nonce``               u32        [Optional] Nonce used to solve the PoW of this block. 
======================  =========  ========================================================================

*Substrate* 

::

  #[derive(Encode, Decode, Default, Clone, PartialEq)]
  #[cfg_attr(feature = "std", derive(Debug))]
  pub struct BlockHeader<H256, U256, Timestamp> {
        merkle_root: H256,
        target: U256,
        timestamp: u64,
        hash_prev_block: H256,
        version: i32, 
        nonce: u32
  }

.. _blockHeader: 

RichBlockHeader
................

Representation of a Bitcoin block header containing additional metadata. This struct is used to store Bitcoin block headers. 

.. note:: Fields marked as [Optional] are not critical for the secure operation of BTC-Relay, but can be stored anyway, at the developers discretion. We omit these fields in the rest of this specification. 

.. tabularcolumns:: |l|l|L|

======================  =========    ========================================================================
Parameter               Type         Description
======================  =========    ========================================================================
``blockhash``           bytes32      Bitcoin's double SHA256 PoW block hash
``blockHeight``         u32          Height of this block in the Bitcoin main chain.
``chainRef``            u32          Pointer to the ``BlockChain`` struct in which this block header is contained.
``blockHeader``         BlockHeader  Associated parsed ``BlockHeader`` struct 
======================  =========    ========================================================================

*Substrate* 

::

  #[derive(Encode, Decode, Default, Clone, PartialEq)]
  #[cfg_attr(feature = "std", derive(Debug))]
  pub struct RichBlockHeader<H256, U256, Timestamp> {
        block_height: U256,
        chain_ref: U256,
        block_header: BlockHeader<H256, U256, Timestamp>,
  }



BlockChain
..........

Representation of a Bitcoin blockchain / fork.

.. tabularcolumns:: |l|l|L|

======================  ==============  ========================================================================
Parameter               Type            Description
======================  ==============  ========================================================================
``chainId``             U256            Unique identifier for faster lookup in ``ChainsIndex``
``chain``               Map<U256,H256>  Mapping of ``blockHeight`` to ``blockHash``, which points to a ``RichBlockHeader`` entry in ``BlockHeaders``.
``startHeight``         U256            Starting/lowest block height in the ``chain`` mapping. Used to determine the forking point during chain reorganizations.
``maxHeight``           U256            Max. block height in the ``chain`` mapping. Used for ordering in the ``Chains`` priority queue.
``noData``              Vec<U256>       List of block heights in ``chain`` referencing block hashes of ``RichBlockHeader`` entries in ``BlockHeaders`` which have been flagged as ``noData`` by Staked Relayers.
``invalid``             Vec<U256>       List of block heights in ``chain`` referencing block hashes of ``RichBlockHeader`` entries in ``BlockHeaders`` which have been flagged as ``invalid`` by Staked Relayers.
======================  ==============  ========================================================================

*Substrate* 

::

  #[derive(Encode, Decode, Default, Clone, PartialEq)]
  #[cfg_attr(feature = "std", derive(Debug))]
  pub struct RichBlockHeader<H256, Timestamp> {
        chainId: U256,
        chain: BTreeMap<U256,H256>,
        startHeight: U256,
        maxHeight: U256,
        noData: Vec<U256>, 
        invalid: Vec<U256>
  }


Data Structures
~~~~~~~~~~~~~~~

BlockHeaders
............

Mapping of ``<blockHash, RichBlockHeader>``, storing all verified Bitcoin block headers (fork and main chain) submitted to BTC-Relay.

*Substrate* ::

  BlockHeaders: map H256 => RichBlockHeader<U256, H256, Moment>;


Chains
.........

Priority queue of ``BlockChain`` elements, **ordered by** ``maxHeight`` (**descending**).
The ``BlockChain`` entry with the most significant ``maxHeight`` value (i.e., topmost element) in this mapping is considered to be the Bitcoin *main chain*.

The exact choice of data structure is left to the developer. We recommend to use a heap, which allows re-balancing (changing the priority/order of items while in the heap). Specifically, we require the following operations to be available:

  * ``max`` ... returns the item with the maximum value (as used for sorting).
  * ``insert`` ... inserts a new item, maintaining ordering in relation to other items.
  * ``delete`` ... removes an item.
  * ``find`` ... returns an item with a given index (by sorting key and stored value).
  * ``update`` ... [Optional] modifies the sorting key of an item and updates ordering if necessary (incrementing ``maxHeight`` of a BlockChain entry). Can be implemented using ``delete`` and ``insert``.

.. attention:: If two ``BlockChain`` entries have the same ``maxHeight``, do **not** change ordering! 

.. note:: The assumption for ``Chains`` is that, in the majority of cases, block headers will be appended to the *main chain* (longest chain), i.e., the ``BlockChain`` entry at the most significant position in the queue/heap. Similarly, transaction inclusion proofs (:ref:`verifyTransaction`) are only checked against the *main chain*. This means, in the average case lookup complexity will be O(1). Furthermore, block headers can only be appended if they (i) have a valid PoW and (ii) do not yet exist in ``BlockHeaders`` - hence, spamming is very costly and unlikely. Finally, blockchain forks and re-organizations occur infrequently, especially in Bitcoin. In principle, optimizing lookup costs should be prioritized, ideally O(1), while inserting of new items and re-balancing can even be O(n). 

.. *Substrate* ::
  // ideally:
  // Chains: PriorityQueue<BlockChain, Ord>;
  // alternative:
  Chains: BinaryHeap<BlockChain, Ord>;
  impl Ord for BlockChain {
    fn cmp(&self, other: &BlockChain) -> Ordering {
    other.maxHeight.cmp(&self.maxHeight)
    // Keeps ordering if equal ("first seen" as in Bitcoin)
    }
  }
  // Also needs to be implemented for BinaryHeap
  impl PartialOrd for BlockChain {
    fn partial_cmp(&self, other: &BlockChain) -> Option<Ordering> {
        Some(self.cmp(other))
    }
  }
  

.. attention:: ``PriorityQueue`` is **currently not** natively supported in Substrate. A Rust implementation can be found `here <https://docs.rs/priority-queue/0.7.0/priority_queue/>`_, which has O(1) lookup and O(log(n)) re-balancing. This functionality can be emulated using a ``LinkedList`` by maintaining ordering upon insertion (worst case O(n), but will be O(1) is most cases as explained above). In theory, this can also be implemented using a ``BinaryHeap`` by deleting and re-inserting ``BlockChain`` entries when necessary.


ChainsIndex
............

Auxiliary mapping of ``BlockChain`` structs to unique identifiers, for faster read access / lookup ``<U256, BlockChain>``, 

*Substrate* ::

  ChainsIndex: map U256 => BlockChain<H256>;

BestBlock
.........

32 byte Bitcoin block hash (double SHA256) identifying the current blockchain tip, i.e., the ``RichBlockHeader`` with the highest ``blockHeight`` in the ``BlockChain`` entry, which has the most significant ``height`` in the ``Chains`` priority queue (topmost position). 

*Substrate* ::

  BestBlock: H256;


.. note:: Bitcoin uses SHA256 (32 bytes) for its block hashes, transaction identifiers and Merkle trees. In Substrate, we hence use ``H256`` to represent these hashes.

BestBlockHeight
...............

Integer representing the maximum block height (``height``) in the ``Chains`` priority queue. This is also the ``blockHeight`` of the ``RichBlockHeader`` entry pointed to by ``BestBlock``.

*Substrate* ::

  BestBlockHeight: U256;


ChainCounter
.................

Integer increment-only counter used to track existing BlockChain entries.
Initialized with 1 (0 is reserved for ``MAIN_CHAIN_ID``).
*Substrate* ::

  ChainCounter: U256 = 1;






