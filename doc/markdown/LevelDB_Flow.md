# LevelDB整体运行流程

为了从整体上全面地描述和了解levelDB的大致运行流程，这里结合[参考](https://dirtysalt.github.io/html/leveldb.html#org080d22f)和源码来整理LevelDB的主要运行流程。

## Open

LevelDB运行的第一步是打开一个DB对象，调用的方法是DB::Open。

```cpp
Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr) {
  *dbptr = nullptr;

  DBImpl* impl = new DBImpl(options, dbname); //创建DBImpl实例
  impl->mutex_.Lock();
  VersionEdit edit;
  // Recover handles create_if_missing, error_if_exists
  bool save_manifest = false;
  Status s = impl->Recover(&edit, &save_manifest);//调用impl的恢复，从log文件中构建table文件
  if (s.ok() && impl->mem_ == nullptr) {
    // Create new log and a corresponding memtable.
    uint64_t new_log_number = impl->versions_->NewFileNumber();
    WritableFile* lfile;
    s = options.env->NewWritableFile(LogFileName(dbname, new_log_number),
                                     &lfile);
    if (s.ok()) {
      edit.SetLogNumber(new_log_number);
      impl->logfile_ = lfile;
      impl->logfile_number_ = new_log_number;
      impl->log_ = new log::Writer(lfile);
      impl->mem_ = new MemTable(impl->internal_comparator_);
      impl->mem_->Ref();
    }
  }
  if (s.ok() && save_manifest) {
    edit.SetPrevLogNumber(0);  // No older logs needed after recovery.
    edit.SetLogNumber(impl->logfile_number_);
    s = impl->versions_->LogAndApply(&edit, &impl->mutex_); //清空之前的log文件
  }
  if (s.ok()) {
    impl->DeleteObsoleteFiles();
    impl->MaybeScheduleCompaction();
  }
  impl->mutex_.Unlock();
  if (s.ok()) {
    assert(impl->mem_ != nullptr);
    *dbptr = impl;
  } else {
    delete impl;
  }
  return s;
};
``` 

可以看见DB::Open里面创建来一个impl对象，然后进行

1. recover 读取现有的current文件来读取mainfest文件，然后读取(data)log文件来恢复已有的数据
2. 然后创建一个新的(data)log文件和对应的memtable。
3. 如果要保存？ 清空之前的log文件，并应用。
4. 删除多余的文件。
5. 进行可能的调度(memtable可能超过大小限制)。
6. 返回impl对象

## impl::Recover

Recover函数根据当前目录下来的文件来进行恢复，并设置传入的edit参数和save_manifest参数。
主要逻辑为：

1. 创建dbname_文件夹。（忽略当前文件夹已经存在的错误，没有看到具体的代码但是根据后面的代码认为应该会同时创建LockFile等一系列的文件）
2. 对当前DB进行加锁。
3. 如果current文件不存在（说明这是第一次创建DB），调用NewDB函数，创建空白的manifest文件和curretn文件，并将创建的manifest写入current。并且根据相应的参数选择是否进行报错。
4. 从当前的CURRENT里面的manifest文件恢复versions_。[versions_.Recover](#versionsetrecover)
5. 然后判断时候根据已有的manifest文件恢复错误，将没有加入manifest文件的log文件记录加入versions_。

> 写到这里才反应过来，在Open和Recover等顶层函数中也进行加锁是为了保证整个DB在作为一个模块倍外部并行操作时候的正确性。之前只注意到了DB内部进行的并行操作需要加锁。

```cpp
Status DBImpl::Recover(VersionEdit* edit, bool* save_manifest) {
  mutex_.AssertHeld();

  // Ignore error from CreateDir since the creation of the DB is
  // committed only when the descriptor is created, and this directory
  // may already exist from a previous failed creation attempt.
  env_->CreateDir(dbname_);
  assert(db_lock_ == nullptr);
  Status s = env_->LockFile(LockFileName(dbname_), &db_lock_);
  if (!s.ok()) {
    return s;
  }

  if (!env_->FileExists(CurrentFileName(dbname_))) {
    if (options_.create_if_missing) {
      s = NewDB();//创建一个新的mainifest，然后将current指向这个mainifest文件
      if (!s.ok()) {
        return s;
      }
    } else {
      return Status::InvalidArgument(
          dbname_, "does not exist (create_if_missing is false)");
    }
  } else {
    if (options_.error_if_exists) {
      return Status::InvalidArgument(dbname_,
                                     "exists (error_if_exists is true)");
    }
  }

  s = versions_->Recover(save_manifest); //从当前的CURRENT里面的manifest文件恢复versions_。如果重用已有manifest失败，则设置save_manifest为true。
  if (!s.ok()) {
    return s;
  }
  SequenceNumber max_sequence(0);

  // Recover from all newer log files than the ones named in the
  // descriptor (new log files may have been added by the previous
  // incarnation without registering them in the descriptor).
  //
  // Note that PrevLogNumber() is no longer used, but we pay
  // attention to it in case we are recovering a database
  // produced by an older version of leveldb.
  // 恢复那些还没有写入manifest的(data)log文件。
  const uint64_t min_log = versions_->LogNumber();
  const uint64_t prev_log = versions_->PrevLogNumber();
  std::vector<std::string> filenames;
  s = env_->GetChildren(dbname_, &filenames);
  if (!s.ok()) {
    return s;
  }
  std::set<uint64_t> expected;
  versions_->AddLiveFiles(&expected); //获取所有level的table文件。
  uint64_t number;
  FileType type;
  std::vector<uint64_t> logs;
  for (size_t i = 0; i < filenames.size(); i++) {//获取所有比minlog大的log文件和正在进行compact的文件。
    if (ParseFileName(filenames[i], &number, &type)) {
      expected.erase(number);
      if (type == kLogFile && ((number >= min_log) || (number == prev_log)))
        logs.push_back(number);
    }
  }
  if (!expected.empty()) {//如果存在没有写入manifest的log文件，说明之前发生了错误。
    char buf[50];
    snprintf(buf, sizeof(buf), "%d missing files; e.g.",
             static_cast<int>(expected.size()));
    return Status::Corruption(buf, TableFileName(dbname_, *(expected.begin())));
  }

  // Recover in the order in which the logs were generated
  std::sort(logs.begin(), logs.end());
  for (size_t i = 0; i < logs.size(); i++) {
    s = RecoverLogFile(logs[i], (i == logs.size() - 1), save_manifest, edit,
                       &max_sequence);
    if (!s.ok()) {
      return s;
    }

    // The previous incarnation may not have written any MANIFEST
    // records after allocating this log number.  So we manually
    // update the file number allocation counter in VersionSet.
    versions_->MarkFileNumberUsed(logs[i]);
  }

  if (versions_->LastSequence() < max_sequence) {
    versions_->SetLastSequence(max_sequence);
  }

  return Status::OK();
}
```


> 从这里可以知道删除旧的immutable的时候并不会删除对应的log文件？我想先弄清楚这个问题，所以先看他的put逻辑，这样可以搞清楚写的流程。[put](#put)
> 删除文件只有在Compaction的时候根据版本号进行。在删除旧immutable的时候并不会删除log文件。

## VersionSet::Recover

将当前的 CURRENT里面的manifest文件记录的log文件恢复到versions_中。

1. 将current文件的内容读到一个string中。根据current文件存储的manifest文件的名字打开manifest文件。
2. 从manifest文件中一次读取记录来构造一个edit，并检查comparator，应用edit到builder后，设置lo_number以及prev、next、last_sequence。（其实设置的后三个参数不是很明白？versionset的链表？）
3. 从builder中构建versions。然后设置上述的几个参数
4. 尝试重用manifest文件，并根据结果设置save_manifest参数。

> Version::Builder类还没有看。

```cpp
Status VersionSet::Recover(bool* save_manifest) {
  struct LogReporter : public log::Reader::Reporter {
    Status* status;
    void Corruption(size_t bytes, const Status& s) override {
      if (this->status->ok()) *this->status = s;
    }
  };

  // Read "CURRENT" file, which contains a pointer to the current manifest file
  std::string current;
  Status s = ReadFileToString(env_, CurrentFileName(dbname_), &current);
  if (!s.ok()) {
    return s;
  }
  if (current.empty() || current[current.size() - 1] != '\n') {
    return Status::Corruption("CURRENT file does not end with newline");
  }
  current.resize(current.size() - 1);

  std::string dscname = dbname_ + "/" + current;
  SequentialFile* file;
  s = env_->NewSequentialFile(dscname, &file);
  if (!s.ok()) {
    if (s.IsNotFound()) {
      return Status::Corruption("CURRENT points to a non-existent file",
                                s.ToString());
    }
    return s;
  }

  bool have_log_number = false;
  bool have_prev_log_number = false;
  bool have_next_file = false;
  bool have_last_sequence = false;
  uint64_t next_file = 0;
  uint64_t last_sequence = 0;
  uint64_t log_number = 0;
  uint64_t prev_log_number = 0;
  Builder builder(this, current_);

  {
    LogReporter reporter;
    reporter.status = &s;
    log::Reader reader(file, &reporter, true /*checksum*/,
                       0 /*initial_offset*/);
    Slice record;
    std::string scratch;
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {
      VersionEdit edit;
      s = edit.DecodeFrom(record);
      if (s.ok()) {
        if (edit.has_comparator_ &&
            edit.comparator_ != icmp_.user_comparator()->Name()) {
          s = Status::InvalidArgument(
              edit.comparator_ + " does not match existing comparator ",
              icmp_.user_comparator()->Name());
        }
      }

      if (s.ok()) {
        builder.Apply(&edit);
      }

      if (edit.has_log_number_) {
        log_number = edit.log_number_;
        have_log_number = true;
      }

      if (edit.has_prev_log_number_) {
        prev_log_number = edit.prev_log_number_;
        have_prev_log_number = true;
      }

      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;
        have_next_file = true;
      }

      if (edit.has_last_sequence_) {
        last_sequence = edit.last_sequence_;
        have_last_sequence = true;
      }
    }
  }
  delete file;
  file = nullptr;

  if (s.ok()) {
    if (!have_next_file) {
      s = Status::Corruption("no meta-nextfile entry in descriptor");
    } else if (!have_log_number) {
      s = Status::Corruption("no meta-lognumber entry in descriptor");
    } else if (!have_last_sequence) {
      s = Status::Corruption("no last-sequence-number entry in descriptor");
    }

    if (!have_prev_log_number) {
      prev_log_number = 0;
    }

    MarkFileNumberUsed(prev_log_number);
    MarkFileNumberUsed(log_number);
  }

  if (s.ok()) {
    Version* v = new Version(this);
    builder.SaveTo(v);
    // Install recovered version
    Finalize(v);
    AppendVersion(v);
    manifest_file_number_ = next_file;
    next_file_number_ = next_file + 1;
    last_sequence_ = last_sequence;
    log_number_ = log_number;
    prev_log_number_ = prev_log_number;

    // See if we can reuse the existing MANIFEST file.
    if (ReuseManifest(dscname, current)) {
      // No need to save new manifest
    } else {
      *save_manifest = true;
    }
  }

  return s;
}
```

## Put

Put和Delete的主要逻辑在Write函数中。这里我不清楚的实际上是对writers_队列进行的处理不清楚，同时感觉上存在但是没有找到的多线程感到奇怪（没有想明白w.cv的传递关系）。主要逻辑是

1. WrtiteBacth封装成的Writer加入到writes_队列。然后等待writes_队列中其他任务完成而进行到之前添加的Writer。
2. 调用MakeRoomForWrite来确保memtable未满，如果满了则转换成immutable以及转换成level0文件。（从这个函数还可以知道level0的生成是在MaybeScheduleCompaction中执行的）
3. 分别将writebatch的内容写入log和memtable。
4. 从writers_队列中取出之前插入的writer。然后向下一个writer的cv发送激活信号。

> 后来想明白了w.cv的传递就发生在4中。

```cpp
Status DBImpl::Put(const WriteOptions& o, const Slice& key, const Slice& val){
  return DB::Put(o, key, val);
}

Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;
  batch.Put(key, value); //将put操作封装成一个writebatch类型
  return Write(opt, &batch);
}

Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  Writer w(&mutex_); //DBImpl::Writer类型
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

  MutexLock l(&mutex_);
  writers_.push_back(&w);//添加到写队列中
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait(); //这里需要调用信号量来判断之前的写是否完成，说明写是开了多线程的？
  }
  if (w.done) {
    return w.status;
  }

  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(updates == nullptr);//这个函数是为当前的写入做一系列的判断和操作，保证当前的memtable有足够的空间来插入数据。
  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  if (status.ok() && updates != nullptr) {  // nullptr batch is for compactions
    WriteBatch* updates = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(updates, last_sequence + 1);
    last_sequence += WriteBatchInternal::Count(updates);

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      mutex_.Unlock();
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));//写入log文件
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(updates, mem_);//写入memtable
      }
      mutex_.Lock();
      if (sync_error) {
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
    if (updates == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }

  //应该就是取出前面插入的writer。但是为什么会出现front不等于当前构造的writer的情况。
  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

## DBImpl::MaybeScheduleCompaction

根据上述的分析，从imuutable memtable转换成level0的文件发生在MaybeScheduleCompaction中。准确的来说应该是所有的Compaction都在这个函数里面？

> 分析完这个模块（MinorCompaction？），我认为应该继续其他几种类型的compaction。这样一方面的判断替换底层level结构是要替换的所有模块？以及觉得自己还是没有想明白加入了version的put和delete等。因为在写入到底层的level后进行的更新操作在leveldb中是通过version已经compaction来进行的。我如果要实现和他一样的功能，还需要提供一样的version接口。这样一路往下回溯，我需要提供差不多所有模块的重构代码。

> 暂时就进行简单重构。直接写入level0（write0）的时候同时写入我的ComboTree。读的时候在读过memtable和imutable memtable后直接读我的combotree。

> 现在的方案的只在原有代码中加入我们的代码，尽量不该原有的逻辑和代码。这样是为了前期的修改方便，在后期需要进行测试和修改的时候还要继续看所有和底层模块相关的？感觉工作量很大。


```cpp
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (background_compaction_scheduled_) { //如果已经被调度了，没有必要重新调度
    // Already scheduled
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // DB is being deleted; no more background compactions
  } else if (!bg_error_.ok()) {
    // Already got an error; no more changes
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
    // No work to be done
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}
```

需要继续分析[env_->Schedule]()，看怎么调度的MinorCompaction和其他Compaction。

## Env::Sechedule

```cpp
void Schedule(void (*f)(void*), void* a) override {
    return target_->Schedule(f, a);
  }
```

```cpp
void Schedule(void (*f)(void*), void* a) override {
    return target_->Schedule(f, a);
  }
```

> 发现要弄清楚这几个thread，还得深入env结构？

后来打算还是暂时将schedule当作一个封装的接口调用，按照注释说的来理解和使用这个函数。转头去看传给schedule的参数BGWork。

```cpp
Arrange to run "(*function)(arg)" once in a background thread.

"function" may run in an unspecified thread. Multiple functions
added to the same Env may run concurrently in different threads.
I.e., the caller may not assume that background work items are
serialized.
```

BGWork函数用来调用db上的BackgroundCall函数。

```cpp
void DBImpl::BGWork(void* db) {
  reinterpret_cast<DBImpl*>(db)->BackgroundCall();
}
```

BackgroundCall其实就是封装了一个BackgroundCompaction。进行了额外的错误处理和是否需要继续进行Compaction的判断。

```cpp
void DBImpl::BackgroundCall() {
  MutexLock l(&mutex_);
  assert(background_compaction_scheduled_);
  if (shutting_down_.load(std::memory_order_acquire)) {
    // No more background work when shutting down.
  } else if (!bg_error_.ok()) {
    // No more background work after a background error.
  } else {
    BackgroundCompaction();
  }

  background_compaction_scheduled_ = false;

  // Previous compaction may have produced too many files in a level,
  // so reschedule another compaction if needed.
  MaybeScheduleCompaction();
  background_work_finished_signal_.SignalAll();
}
```

BackgrouCompaction是具体进行各种Compaction的地方。
```cpp
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != nullptr) { //imm_不为Null，需要进行Minor Compaction
    CompactMemTable(); //将imm_转换成level0文件。第一次看我重点关注这里的compaction。
    return;
  }

  Compaction* c;
  bool is_manual = (manual_compaction_ != nullptr);
  InternalKey manual_end;
  if (is_manual) { //存在manual compaction，优先进行manual compaction。 这部分先忽略过
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == nullptr);
    if (c != nullptr) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
    Log(options_.info_log,
        "Manual compaction at level-%d from %s .. %s; will stop at %s\n",
        m->level, (m->begin ? m->begin->DebugString().c_str() : "(begin)"),
        (m->end ? m->end->DebugString().c_str() : "(end)"),
        (m->done ? "(end)" : manual_end.DebugString().c_str()));
  } else {
    c = versions_->PickCompaction();
  }

  Status status;
  if (c == nullptr) {
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {
    // Move file to next level
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    c->edit()->DeleteFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size, f->smallest,
                       f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    VersionSet::LevelSummaryStorage tmp;
    Log(options_.info_log, "Moved #%lld to level-%d %lld bytes %s: %s\n",
        static_cast<unsigned long long>(f->number), c->level() + 1,
        static_cast<unsigned long long>(f->file_size),
        status.ToString().c_str(), versions_->LevelSummary(&tmp));
  } else {
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    CleanupCompaction(compact);
    c->ReleaseInputs();
    DeleteObsoleteFiles();
  }
  delete c;

  if (status.ok()) {
    // Done
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // Ignore compaction errors found during shutting down
  } else {
    Log(options_.info_log, "Compaction error: %s", status.ToString().c_str());
  }

  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    if (!status.ok()) {
      m->done = true;
    }
    if (!m->done) {
      // We only compacted part of the requested range.  Update *m
      // to the range that is left to be compacted.
      m->tmp_storage = manual_end;
      m->begin = &m->tmp_storage;
    }
    manual_compaction_ = nullptr;
  }
}
```


## DBImpl::CompactMemTable

这里进行Minor Compaction，和我目前的设计方案联系最紧密。主要操作逻辑在WriteLevel0Table函数中，后续进行删除log文件的操作和设置imm_等操作。

```cpp
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  Status s = WriteLevel0Table(imm_, &edit, base);//MinorCompaction的主要逻辑部分都在这里
  base->Unref();

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }

  // Replace immutable memtable with the generated Table
  if (s.ok()) {//删除log文件
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {//设置imm_和相关参数
    // Commit to the new state
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);
    DeleteObsoleteFiles();
  } else {
    RecordBackgroundError(s);
  }
}
```

## DBImpl::WriteLevel0Table

主要操作就是创建一个table文件，然后加到edit中，然后选择一个level放置新生成的文件。
如果要继续分析leveldb的话，就顺着buildtable往下看。不然的话，实际上将combotree的代码加入到这里就行了。由于之前已经看过table相关的代码了，所以觉得继续看看buildtable。

> 现在进一步分析觉得到buildtable中进行添加可能会更好。但是这样的话是单线程的。
> 后续考虑再来实现多线程的版本。

```cpp
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {//创建新的table文件
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);

  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```


## BuildTable


```cpp
Status BuildTable(const std::string& dbname, Env* env, const Options& options,
                  TableCache* table_cache, Iterator* iter, FileMetaData* meta) {
  Status s;
  meta->file_size = 0;
  iter->SeekToFirst();

  std::string fname = TableFileName(dbname, meta->number);
  if (iter->Valid()) {
    WritableFile* file;
    s = env->NewWritableFile(fname, &file);
    if (!s.ok()) {
      return s;
    }

    TableBuilder* builder = new TableBuilder(options, file);
    meta->smallest.DecodeFrom(iter->key());
    for (; iter->Valid(); iter->Next()) { //加入key
      Slice key = iter->key();
      meta->largest.DecodeFrom(key);
      builder->Add(key, iter->value());
      //在这里同步的向comboTree中加入key
      //
    }

    // Finish and check for builder errors
    s = builder->Finish();
    if (s.ok()) {
      meta->file_size = builder->FileSize();
      assert(meta->file_size > 0);
    }
    delete builder;

    // Finish and check for file errors
    if (s.ok()) {
      s = file->Sync();
    }
    if (s.ok()) {
      s = file->Close();
    }
    delete file;
    file = nullptr;

    if (s.ok()) {
      // Verify that the table is usable
      Iterator* it = table_cache->NewIterator(ReadOptions(), meta->number,
                                              meta->file_size);
      s = it->status();
      delete it;
    }
  }

  // Check for input iterator errors
  if (!iter->status().ok()) {
    s = iter->status();
  }

  if (s.ok() && meta->file_size > 0) {
    // Keep it
  } else {
    env->DeleteFile(fname);
  }
  return s;
}
```