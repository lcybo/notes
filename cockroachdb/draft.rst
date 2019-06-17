*******
draft
*******

txn_interceptor
==================

.. code::

  interceptorAlloc struct {
    arr [7]txnInterceptor
    // rootTxn only
    txnHeartbeater
    // assign sequence numbers to requests, every write request is given a new seq number
    txnSeqNumAllocator
    // rootTxn only(only rootTnx has write intents), attach collected intents to EndTransaction
    txnIntentCollector
    // track all asynchronous writes, ensure all writes has been proposed before commit 
    // or follow-up overlapping read/write.
    txnPipeliner
    // if transaction experiences a retry error, the coordinator uses the Refresh and RefreshRange 
    // api to verify that no write has occurred to the spans more recently then the txn's original
    // timestamp.
    txnSpanRefresher
    txnCommitter
    txnMetricRecorder
    txnLockGatekeeper // not in interceptorStack array.
  }



consistenct
============

https://jepsen.io/consistency
https://www.cockroachlabs.com/blog/living-without-atomic-clocks/

transaction write
=================

.. code::

  // Determine the read and write timestamps for the write. For a
  // non-transactional write, these will be identical. For a transactional
  // write, we read at the transaction's original timestamp (forwarded by any
  // refresh timestamp) but write intents at its provisional commit timestamp.
  // See the comment on the txn.Timestamp field definition for rationale.
  readTimestamp := timestamp
  writeTimestamp := timestamp
  if txn != nil {
    readTimestamp = txn.OrigTimestamp
    readTimestamp.Forward(txn.RefreshedTimestamp)
	if readTimestamp != timestamp {
      return errors.Errorf("mvccPutInternal: txn's read timestamp %s does not match timestamp %s",
      txn.OrigTimestamp, timestamp)
    }
    writeTimestamp = txn.Timestamp
  }


timestamps in transaction
=========================

.. code::

  // The original timestamp at which the transaction started.
  OrigTimestamp hlc.Timestamp
  
  // Initial Timestamp + clock skew.
  MaxTimestamp hlc.Timestamp

  // The refreshed timestamp is the timestamp at which the transaction can commit without necessitating a serializable restart.
  RefreshedTimestamp hlc.Timestamp

  // The list maps NodeIDs to timestamps as observed from their local clock during this transaction.
  ObservedTimestamps []ObservedTimestamp


retry error
============

.. code::

  WriteTooOldError
  ReadWithinUncertaintyIntervalError

AsyncConsensus
==============

使用QueryIntentRequest确保commit之前所有的intent已经成功commit.

