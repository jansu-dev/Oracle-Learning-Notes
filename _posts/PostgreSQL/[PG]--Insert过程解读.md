---
title:[PG]--Insert过程解读
date:2020-09-16
---



##### [PG]--Insert过程解读

- [插入调试](插入调试)

- [页头数据布局](页头数据布局)

- [插入过程概览](插入过程概览)

  

### 插入调试

![image.png](http://cdn.lifemini.cn/dbblog/20200915/38e54a4b54c94ffabc7357e253d4ecac.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/9d6091fa9d8448bb8cfeadbe31f27720.png)
ip_blkid 是块号；
ip_posid 是块内的本条记录的序号；

![image.png](http://cdn.lifemini.cn/dbblog/20200915/972c00cee17f40a8a9e273d3b31cdde1.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/dbdc9d496cff43d7b1869119b5b75807.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/a91c288a85b94b4ca479c7d17d73a561.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/fca3daa3caf44a77a5c92aec23758c62.png)

```
2648        *forknum = bufHdr->tag.forkNum;
(gdb) p *bufHdr
$14 = {
  tag = {
    rnode = {
      spcNode = 1663, 
      dbNode = 16384, 
      relNode = 16385
    }, 
    forkNum = MAIN_FORKNUM, 
    blockNum = 0
  }, 
  buf_id = 361, 
  state = {
    value = 3549691906
  }, 
  wait_backend_pid = 0, 
  freeNext = -2, 
  content_lock = {
    tranche = 53, 
    state = {
      value = 1627389952
    }, 
    waiters = {
      head = 122, 
      tail = 122
    }
  }
}
```

![image.png](http://cdn.lifemini.cn/dbblog/20200915/c463db5d351947d7a7d147a41d2f1396.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/84ecc6fd2f3b4188846f62e20f2a591b.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/625823d375ad4849ae4da2fff62d64f1.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/5a5a27a6407a4dccba8c6dcd40557508.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/422ddd538f214dfe91eceb7e320cbf1d.png)

![image.png](http://cdn.lifemini.cn/dbblog/20200915/ee4e67b17bd44c41ac72858e2070dbdf.png)



### 页头数据布局

![image.png](http://cdn.lifemini.cn/dbblog/20200915/9f1769caac5540b5b622a91d0cd36f6e.png)

```
// src/include/storage/itemid.h

typedef struct ItemIdData
{
    unsigned    lp_off:15,        /* offset to tuple (from start of page) */
                lp_flags:2,        /* state of line pointer, see below */
                lp_len:15;        /* byte length of tuple */
} ItemIdData;

/*
 * lp_flags has these possible states.  An UNUSED line pointer is available
 * for immediate re-use, the other states are not.
 */
#define LP_UNUSED        0        /* unused (should always have lp_len=0) */
#define LP_NORMAL        1        /* used (should always have lp_len&gt;0) */
#define LP_REDIRECT        2        /* HOT redirect (should have lp_len=0) */
#define LP_DEAD            3        /* dead, may or may not have storage */
```



### 插入过程概览

```
Breakpoint 1, PageAddItemExtended () at bufpage.c:210
210        if (phdr->pd_lower < SizeOfPageHeaderData ||
(gdb) 
222        limit = OffsetNumberNext(PageGetMaxOffsetNumber(page));
(gdb) 
225        if (OffsetNumberIsValid(offsetNumber))
(gdb) 
250            if (PageHasFreeLinePointers(phdr))
(gdb) 
284        if ((flags & PAI_IS_HEAP) != 0 && offsetNumber > MaxHeapTuplesPerPage)
(gdb) 
305        if (lower > upper)
(gdb) 
311        itemId = PageGetItemId(phdr, offsetNumber);
(gdb) 
313        if (needshuffle)
(gdb) 
318        ItemIdSetNormal(itemId, upper, size);
(gdb) 
335        memcpy((char *) page + upper, item, size);
(gdb) 
338        phdr->pd_lower = (LocationIndex) lower;
(gdb) 
339        phdr->pd_upper = (LocationIndex) upper;
(gdb) 
341        return offsetNumber;
(gdb) 
RelationPutHeapTuple (relation=relation@entry=0x7fc14796e810, 
    buffer=buffer@entry=362, tuple=tuple@entry=0x25d0a98, 
    token=token@entry=false) at hio.c:56
56        if (offnum == InvalidOffsetNumber)
(gdb) 
60        ItemPointerSet(&(tuple->t_self), BufferGetBlockNumber(buffer), offnum);
(gdb) 
67        if (!token)
(gdb) 
72            item->t_ctid = tuple->t_self;
(gdb) 
heap_insert (relation=relation@entry=0x7fc14796e810, 
    tup=tup@entry=0x25d0a98, cid=cid@entry=0, options=options@entry=0, 
    bistate=bistate@entry=0x0) at heapam.c:1916
1916        if (PageIsAllVisible(BufferGetPage(buffer)))
(gdb) 
1936        MarkBufferDirty(buffer);
(gdb) 
1939        if (!(options & HEAP_INSERT_SKIP_WAL) && RelationNeedsWAL(relation))
(gdb) 
1944            Page        page = BufferGetPage(buffer);
(gdb) 
1952            if (RelationIsAccessibleInLogicalDecoding(relation))
(gdb) 
1960            if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber &&
(gdb) 
1967            xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
(gdb) 
1969            if (all_visible_cleared)
(gdb) 
1971            if (options & HEAP_INSERT_SPECULATIVE)
(gdb) 
1980            if (RelationIsLogicallyLogged(relation) &&
(gdb) 
1987            XLogBeginInsert();
(gdb) 
1988            XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
(gdb) 
1990            xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
(gdb) 
1992            xlhdr.t_hoff = heaptup->t_data->t_hoff;
(gdb) 
1999            XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
(gdb) 
2000            XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
(gdb) 
2002            XLogRegisterBufData(0,
(gdb) 
2007            XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
(gdb) 
2009            recptr = XLogInsert(RM_HEAP_ID, info);
(gdb) 
2011            PageSetLSN(page, recptr);
(gdb) 
2014        END_CRIT_SECTION();
(gdb) 
2016        UnlockReleaseBuffer(buffer);
(gdb) 
2017        if (vmbuffer != InvalidBuffer)
(gdb) 
2026        CacheInvalidateHeapTuple(relation, heaptup, NULL);
(gdb) 
2029        pgstat_count_heap_insert(relation, 1);
(gdb) 
2035        if (heaptup != tup)
(gdb) 
heapam_tuple_insert (relation=0x7fc14796e810, slot=0x25d0938, cid=0, 
    options=0, bistate=0x0) at heapam_handler.c:258
258        ItemPointerCopy(&tuple->t_self, &slot->tts_tid);
(gdb) 
260        if (shouldFree)
(gdb) 
ExecInsert () at nodeModifyTable.c:591
591                if (resultRelInfo->ri_NumIndices > 0)
(gdb) 
597        if (canSetTag)
(gdb) 
600            setLastTid(&slot->tts_tid);
(gdb) 
609        ar_insert_trig_tcs = mtstate->mt_transition_capture;
(gdb) 
610        if (mtstate->operation == CMD_UPDATE && mtstate->mt_transition_capture
(gdb) 
627        ExecARInsertTriggers(estate, resultRelInfo, slot, recheckIndexes,
(gdb) 
630        list_free(recheckIndexes);
(gdb) 
644        if (resultRelInfo->ri_WithCheckOptions != NIL)
(gdb) 
648        if (resultRelInfo->ri_projectReturning)
(gdb) 
350        return slot;
(gdb) 
ExecModifyTable (pstate=0x25cfe40) at nodeModifyTable.c:2218
2218                    if (proute)
(gdb) 
2240            if (slot)
(gdb) 
2062            ResetPerTupleExprContext(estate);
(gdb) 
2069            if (pstate->ps_ExprContext)
(gdb) 
2072            planSlot = ExecProcNode(subplanstate);
(gdb) 
2074            if (TupIsNull(planSlot))
(gdb) 
2077                node->mt_whichplan++;
(gdb) 
2078                if (node->mt_whichplan < node->mt_nplans)
(gdb) 
2248        estate->es_result_relation_info = saved_resultRelInfo;
(gdb) 
2253        fireASTriggers(node);
(gdb) 
2255        node->mt_done = true;
(gdb) 
2257        return NULL;
(gdb) 
2243                return slot;
(gdb) 
ExecutePlan (execute_once=<optimized out>, dest=0x25efd40, 
    direction=<optimized out>, numberTuples=0, sendTuples=<optimized out>, 
    operation=CMD_INSERT, use_parallel_mode=<optimized out>, 
    planstate=0x25cfe40, estate=0x25cfab0) at execMain.c:1652
1652            if (TupIsNull(slot))
(gdb) 
1703        if (!(estate->es_top_eflags & EXEC_FLAG_BACKWARD))
(gdb) 
1704            (void) ExecShutdownNode(planstate);
(gdb) 
1706        if (use_parallel_mode)
(gdb) 
standard_ExecutorRun (queryDesc=0x25b6cb0, direction=<optimized out>, 
    count=0, execute_once=<optimized out>) at execMain.c:378
378        if (sendTuples)
(gdb) 
381        if (queryDesc->totaltime)
(gdb) 
384        MemoryContextSwitchTo(oldcontext);
(gdb) 
114        return old;
(gdb) 
ProcessQuery (plan=<optimized out>, 
    sourceText=0x24f5180 "insert into t1 values (1,'jan_test');", 
    params=0x0, queryEnv=0x0, dest=0x25efd40, 
    completionTag=0x7fff842dfe60 "") at pquery.c:166
166        if (completionTag)
(gdb) 
170            switch (queryDesc->operation)
(gdb) 
180                    snprintf(completionTag, COMPLETION_TAG_BUFSIZE,
(gdb) 
183                    break;
(gdb) 
203        ExecutorFinish(queryDesc);
(gdb) 
204        ExecutorEnd(queryDesc);
(gdb) 
206        FreeQueryDesc(queryDesc);
(gdb) 
FreeQueryDesc (qdesc=0x25b6cb0) at pquery.c:111
111        UnregisterSnapshot(qdesc->snapshot);
(gdb) 
112        UnregisterSnapshot(qdesc->crosscheck_snapshot);
(gdb) 
115        pfree(qdesc);
(gdb) 
pfree (pointer=0x25b6cb0) at ../../../../src/include/utils/memutils.h:127
127        context = *(MemoryContext *) (((char *) pointer) - sizeof(void *));
(gdb) 
1035        context->methods->free_p(context, pointer);
(gdb) 
AllocSetFree (context=0x25b6ba0, pointer=0x25b6cb0) at aset.c:990
990        AllocChunk    chunk = AllocPointerGetChunk(pointer);
(gdb) 
1005        if (chunk->size > set->allocChunkLimit)
(gdb) 
1039            int            fidx = AllocSetFreeIndex(chunk->size);
(gdb) 
1041            chunk->aset = (void *) set->freelist[fidx];
(gdb) 
1051            set->freelist[fidx] = chunk;
(gdb) 
PortalRunMulti (portal=portal@entry=0x255ae30, 
    isTopLevel=isTopLevel@entry=true, 
    setHoldSnapshot=setHoldSnapshot@entry=false, dest=dest@entry=0x25efd40, 
    altdest=altdest@entry=0x25efd40, 
    completionTag=completionTag@entry=0x7fff842dfe60 "INSERT 0 1")
    at pquery.c:1299
1299                if (log_executor_stats)
(gdb) 
1302                TRACE_POSTGRESQL_QUERY_EXECUTE_DONE();
(gdb) 
1337            if (lnext(stmtlist_item) != NULL)
(gdb) 
1345            MemoryContextDeleteChildren(portal->portalContext);
(gdb) 
1229        foreach(stmtlist_item, portal->stmts)
(gdb) 
1349        if (active_snapshot_set)
(gdb) 
1350            PopActiveSnapshot();
(gdb) 
1364        if (completionTag && completionTag[0] == '\0')
(gdb) 
PortalRun (portal=portal@entry=0x255ae30, 
    count=count@entry=9223372036854775807, 
    isTopLevel=isTopLevel@entry=true, run_once=run_once@entry=true, 
    dest=dest@entry=0x25efd40, altdest=altdest@entry=0x25efd40, 
    completionTag=0x7fff842dfe60 "INSERT 0 1") at pquery.c:800
800                    MarkPortalDone(portal);
(gdb) 
832        PG_END_TRY();
(gdb) 
834        if (saveMemoryContext == saveTopTransactionContext)
(gdb) 
838        ActivePortal = saveActivePortal;
(gdb) 
839        if (saveResourceOwner == saveTopTransactionResourceOwner)
(gdb) 
843        PortalContext = savePortalContext;
(gdb) 
845        if (log_executor_stats && portal->strategy != PORTAL_MULTI_QUERY)
(gdb) 
848        TRACE_POSTGRESQL_QUERY_EXECUTE_DONE();
(gdb) 
850        return result;
(gdb) 
exec_simple_query (
    query_string=0x24f5180 "insert into t1 values (1,'jan_test');")
    at postgres.c:1223
1223            receiver->rDestroy(receiver);
(gdb) 
1225            PortalDrop(portal, false);
(gdb) 
1227            if (lnext(parsetree_item) == NULL)
(gdb) 
1238                if (use_implicit_block)
(gdb) 
1240                finish_xact_command();
(gdb) 
1265            EndCommand(completionTag, dest);
(gdb) 
1067        foreach(parsetree_item, parsetree_list)
(gdb) 
1273        finish_xact_command();
(gdb) 
1278        if (!parsetree_list)
(gdb) 
1284        switch (check_log_duration(msec_str, was_logged))
(gdb) 
1300        if (save_log_statement_stats)
(gdb) 
1303        TRACE_POSTGRESQL_QUERY_DONE(query_string);
(gdb) 
1305        debug_query_string = NULL;
(gdb) 
PostgresMain (argc=<optimized out>, argv=argv@entry=0x251ef68, 
    dbname=<optimized out>, username=<optimized out>) at postgres.c:4084
4084            send_ready_for_query = true;    /* initially, or after error */
(gdb) 
4096            doing_extended_query_message = false;
(gdb) 
4102            MemoryContextSwitchTo(MessageContext);
(gdb) 
4103            MemoryContextResetAndDeleteChildren(MessageContext);
(gdb) 
4105            initStringInfo(&input_message);
(gdb) 
4111            InvalidateCatalogSnapshotConditionally();
(gdb) 
4126            if (send_ready_for_query)
(gdb) 
4128                if (IsAbortedTransactionBlockState())
(gdb) 
4141                else if (IsTransactionOrTransactionBlock())
(gdb) 
4157                    ProcessCompletedNotifies();
(gdb) 
4165                    if (notifyInterruptPending)
(gdb) 
4168                    pgstat_report_stat(false);
(gdb) 
4170                    set_ps_display("idle", false);
(gdb) 
4171                    pgstat_report_activity(STATE_IDLE, NULL);
(gdb) 
4174                ReadyForQuery(whereToSendOutput);
(gdb) 
4175                send_ready_for_query = false;
(gdb) 
4189            firstchar = ReadCommand(&input_message);
```