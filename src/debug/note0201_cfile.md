
path： storage/innobase/row/row0upd.cc

更新聚簇索引， 就是更新 innodb b+ 树的叶子节点   
row_upd_clust_step

```

/** Updates the clustered index record.
 @return DB_SUCCESS if operation successfully completed, DB_LOCK_WAIT
 in case of a lock wait, else error code */
static MY_ATTRIBUTE((warn_unused_result)) dberr_t
    row_upd_clust_step(upd_node_t *node, /*!< in: row update node */
                       que_thr_t *thr)   /*!< in: query thread */
{
  dict_index_t *index;
  btr_pcur_t *pcur;
  ibool success;
  dberr_t err;
  mtr_t mtr;
  rec_t *rec;
  mem_heap_t *heap = NULL;
  ulint offsets_[REC_OFFS_NORMAL_SIZE];
  ulint *offsets;
  ibool referenced;
  ulint flags = 0;
  trx_t *trx = thr_get_trx(thr);
  rec_offs_init(offsets_);

  index = node->table->first_index();

  referenced = row_upd_index_is_referenced(index, trx);

  pcur = node->pcur;

  /* We have to restore the cursor to its position */

  mtr_start(&mtr);

  /* Disable REDO logging as lifetime of temp-tables is limited to
  server or connection lifetime and so REDO information is not needed
  on restart for recovery.
  Disable locking as temp-tables are not shared across connection. */
  if (index->table->is_temporary()) {
    flags |= BTR_NO_LOCKING_FLAG;
    mtr.set_log_mode(MTR_LOG_NO_REDO);

    if (index->table->is_intrinsic()) {
      flags |= BTR_NO_UNDO_LOG_FLAG;
    }
  }
```
  /* If the restoration does not succeed, then the same
  transaction has deleted the record on which the cursor was,
  and that is an SQL error. If the restoration succeeds, it may
  still be that the same transaction has successively deleted
  and inserted a record with the same ordering fields, but in
  that case we know that the transaction has at least an
  implicit x-lock on the record. */

  / *如果修复不成功,那么相同的事务的记录删除光标,这是一个SQL错误
  。如果修复成功,它可能仍然是相同的事务先后删除和插入一个记录排序字段,
  但在这种情况下我们知道至少有一个事务隐式独占锁的记录。* /
```
  ut_a(pcur->m_rel_pos == BTR_PCUR_ON);

  ulint mode;

  DEBUG_SYNC_C_IF_THD(thr_get_trx(thr)->mysql_thd,
                      "innodb_row_upd_clust_step_enter");

  if (dict_index_is_online_ddl(index)) {
    ut_ad(node->table->id != DICT_INDEXES_ID);
    mode = BTR_MODIFY_LEAF | BTR_ALREADY_S_LATCHED;
    mtr_s_lock(dict_index_get_lock(index), &mtr);
  } else {
    mode = BTR_MODIFY_LEAF;
  }

  success = btr_pcur_restore_position(mode, pcur, &mtr);

  if (!success) {
    err = DB_RECORD_NOT_FOUND;

    mtr_commit(&mtr);

    return (err);
  }

  rec = btr_pcur_get_rec(pcur);
  offsets = rec_get_offsets(rec, index, offsets_, ULINT_UNDEFINED, &heap);

  if (!node->has_clust_rec_x_lock) {
    err = lock_clust_rec_modify_check_and_lock(flags, btr_pcur_get_block(pcur),
                                               rec, index, offsets, thr);
    if (err != DB_SUCCESS) {
      mtr_commit(&mtr);
      goto exit_func;
    }
  }

  ut_ad(lock_trx_has_rec_x_lock(thr_get_trx(thr), index->table,
                                btr_pcur_get_block(pcur),
                                page_rec_get_heap_no(rec)));

  /* NOTE: the following function calls will also commit mtr */

  if (node->is_delete) {
    err = row_upd_del_mark_clust_rec(flags, node, index, offsets, thr,
                                     referenced, &mtr);

    if (err == DB_SUCCESS) {
      node->state = UPD_NODE_UPDATE_ALL_SEC;
      node->index = index->next();
    }

    goto exit_func;
  }
```
  /* If the update is made for MySQL, we already have the update vector
  ready, else we have to do some evaluation: */

  / *如果更新为MySQL,我们已经更新矢量准备好了,我们要做一些其他的评估:* /
```
  if (UNIV_UNLIKELY(!node->in_mysql_interface)) {
    /* Copy the necessary columns from clust_rec and calculate the
    new values to set */
    row_upd_copy_columns(rec, offsets, index, UT_LIST_GET_FIRST(node->columns));
    row_upd_eval_new_vals(node->update);
  }

  if (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE) {
    err = row_upd_clust_rec(flags, node, index, offsets, &heap, thr, &mtr);
    goto exit_func;
  }

  row_upd_store_row(trx, node, trx->mysql_thd,
                    thr->prebuilt ? thr->prebuilt->m_mysql_table : NULL);

  if (row_upd_changes_ord_field_binary(index, node->update, thr, node->row,
                                       node->ext)) {
```

    /* Update causes an ordering field (ordering fields within
    the B-tree) of the clustered index record to change: perform
    the update by delete marking and inserting.
    TODO! What to do to the 'Halloween problem', where an update
    moves the record forward in index so that it is again
    updated when the cursor arrives there? Solution: the
    read operation must check the undo record undo number when
    choosing records to update. MySQL solves now the problem
    externally! */

    / *更新导致一个排序字段(排序字段内b - tree)聚集索引记录的变化:执行更新删除标记和插入。待办事项!要做什么“万圣节问题”,一个更新将记录在指数再次更新当光标到达那里?解决方案:读操作时必须检查undo记录撤销数量选择记录更新。MySQL可以解决现在的问题外部!* /


```    

    err =
        row_upd_clust_rec_by_insert(flags, node, index, thr, referenced, &mtr);

    if (err != DB_SUCCESS) {
      goto exit_func;
    }

    node->state = UPD_NODE_UPDATE_ALL_SEC;
  } else {
    err = row_upd_clust_rec(flags, node, index, offsets, &heap, thr, &mtr);

    if (err != DB_SUCCESS) {
      goto exit_func;
    }

    node->state = UPD_NODE_UPDATE_SOME_SEC;
  }

  node->index = index->next();

exit_func:
  if (heap) {
    mem_heap_free(heap);
  }
  return (err);
}
```