.. _events:

Events
======

Initialized
--------------------

If the first block header was stored successfully, emit an event with the stored block’s height and the (PoW) block hash.

*Event Signature*

``Initialized(blockHeight, blockHash)``

*Parameters*

* ``blockHeight``: height of the current block submission.
* ``blockHash``: hash of the current block submission.

*Functions*

* :ref:`initialize`

*Substrate* ::

  Initialized(U256, H256);

StoreMainChainHeader
--------------------

If the block header was stored successfully, emit an event with the stored block’s height and the (PoW) block hash.

*Event Signature*

``StoreMainChainHeader(blockHeight, blockHash)``

*Parameters*

* ``blockHeight``: height of the current block submission.
* ``blockHash``: hash of the current block submission.

*Functions*

* :ref:`storeBlockHeader`

*Substrate* ::

  StoreMainChainHeader(U256, H256);

StoreForkHeader
---------------

If the submitted block header is on a fork, emit an event with the fork’s id, block height and the (PoW) block hash.

*Event Signature*

``StoreForkHeader(forkId, blockHeight, blockHash)``

*Parameters*

* ``forkId``: unique identifier of the tracked fork.
* ``blockHeight``: height of the current block submission.
* ``blockHash``: hash of the current block submission.

*Functions*

* :ref:`storeForkBlockHeader`

*Substrate* ::

  StoreForkHeader(U256, U256, H256);

ChainReorg
---------- 

If the submitted block header on a fork results in a reorganization (fork longer than current main chain), emit an event with the block hash of the new highest block, the new maximum block height and the depth of the fork

*Event Signature*

``ChainReorg(newChainTip, blockHeight, forkDepth)``

*Parameters*

* ``newChainTip``: hash of the new highest block.
* ``blockHeight``: new maximum block height (block height of fork tip).
* ``forkDepth``: depth of the fork (number of block after diverging from previous main chain).

*Functions*

* :ref:`storeForkBlockHeader`

*Substrate* ::

  ChainReorg(H256, U256, U256);

VerifyTransaction
-----------------

If the verification of the transaction inclusion proof was successful, emit an event for the given transaction identifier (``txId``), block height (``txBlockHeight``), and the specified number of ``confirmations``.

*Event Signature*

``VerifyTransaction(txId, blockHeight, confirmations)``

*Parameters*

* ``txId``: the hash of the transaction.
* ``txBlockHeight``: height of block of the transaction.
* ``confirmations``: number of confirmations requested for the transaction verification.

*Functions*

* :ref:`verifyTransactionInclusion`

*Substrate* ::

  VerifyTransaction(H256, U256, U256);





ValidateTransaction
---------------------

If parsing and validation of the given raw transaction was successful, emit an event specifying the ``txId``, the ``paymentValue``, the ``recipientBtcAddress`` and the ``opReturnId``.

*Event Signature*

``ValidateTransaction(txId, paymentValue, recipientBtcAddress, opReturnId)``

*Parameters*

* ``txId``: the hash of the transaction.
* ``paymentValue``: integer value of BTC sent in the (first) *Payment UTXO* of transaction.
* ``recipientBtcAddress``: 20 byte Bitcoin address of recipient of the BTC in the (first) *Payment UTXO*.
* ``opReturnId``: 32 byte hash identifier expected in OP_RETURN (replay protection).

*Functions*

* :ref:`validateTransaction`

*Substrate* ::

  ValidateTransaction(H256, U256, H160, H256);


