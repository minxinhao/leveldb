- [1. LevelDB](#1-leveldb)
  - [1.1. include/leveldb](#11-includeleveldb)
    - [1.1.1. DB类](#111-db%e7%b1%bb)
    - [1.1.2. Options类](#112-options%e7%b1%bb)
    - [1.1.3. WriteOption](#113-writeoption)
    - [1.1.4. Slice](#114-slice)
    - [1.1.5. WriteBatch](#115-writebatch)
    - [1.1.6. ReadOptions](#116-readoptions)
    - [1.1.7. Iterator](#117-iterator)
      - [1.1.7.1. CleanupNode](#1171-cleanupnode)
          - [1.1.7.1.1. cleanupFunction](#11711-cleanupfunction)
    - [1.1.8. SnapShot](#118-snapshot)
    - [1.1.9. cache.h](#119-cacheh)
    - [1.1.10. Comparator.h](#1110-comparatorh)
    - [1.1.11. dumpfile.h](#1111-dumpfileh)
    - [1.1.12. Env.h](#1112-envh)
    - [1.1.13. filter_policy.h](#1113-filterpolicyh)
    - [1.1.14. TableBuildher.h](#1114-tablebuildherh)
    - [1.1.15. Table.h](#1115-tableh)
    - [结语](#%e7%bb%93%e8%af%ad)

# 1. LevelDB

重新开始阅读LevelDB。之前的文档保留在https://github.com/minxinhao/learn_notes.这次偏重底层实际代码的阅读。

## 1.1. include/leveldb

### 1.1.1. DB类

1. Open 接受一个Options参数和一个string name，打开一个数据库，存放在dbptr中
2. Put 接受一个WriteOptions和封装了key/value的Slice。写入一个key/value对。
3. Delete 接受一个WriteOptions和封装了keySlice。删除一个key/value对。如果key不存在不报错。
4. Write 批量执行一系列的put和delete操作。
5. Get 接受一个ReadOptions和封装了key/value的Slice。读取一个key/value对。不存在对应的key会报错。
6. NewIterator 返回DB中所有数据的一个迭代器
7. GetSnapshot 返回当前数据库的快照
8. GetProperty 暂时不知道作用
9. CompactRange 手动compact


### 1.1.2. Options类

控制DB行为的类，作为参数传递给DB::open。

1. comparator Comparator* DB用来比较key大小的类
2. create_if_missing bool
3. error_if_exist bool
4. paranoid_checks bool 检查数据正确性
5. env Env* 封装了文件操作、线程调用等
6. info_log Logger* 存放输出的错误信息
7. write_buffer_size size_t 
8. max_open_files int
9. block_cache Cache*
10. block_size size_t
11. block_restart_interval int
12. max_file_size size_t
13. compression CompressionType
14. reuse_logs bool
15. filter_policy FilterPolicy

### 1.1.3. WriteOption

控制写行为的类，只有一个sync参数

1. sync bool 当sync为true的时候，只有计算机崩溃的时候会丢失一些写。线程崩溃的时候并不会丢失数据。东sync为true的时候，会调用底层WritableFile::Sync()。

### 1.1.4. Slice

基本的数据类型，在LevelDB中用来传递数据。

1. data_ const char*
2. size_ size_t

### 1.1.5. WriteBatch

1. rep_ string 存放操作的数据和类型


### 1.1.6. ReadOptions

1. verify_checksums bool
2. fill_cache bool
3. snapshot Snapshot*

### 1.1.7. Iterator

1. cleanup_head_ CleanupNode

#### 1.1.7.1. CleanupNode

1. function cleanupFunction
2. arg1 void*
3. arg2 void*
4. next CleanupNode*

###### 1.1.7.1.1. cleanupFunction


     using CleanupFunction = void (*)(void* arg1, void* arg2);

### 1.1.8. SnapShot

没有定义。

### 1.1.9. cache.h

定义cache类型要实现的方法。

1. insert 插入给定的key/value。同时传入在淘汰时要调用的函数.
2. Lookup 查找给定的key
3. Release 释放lookup返回的句柄
4. Value 返回句柄指向的value
5. Erase 删除给定的key
6. Prune 清空所有非活动项

### 1.1.10. Comparator.h

定义DB使用的Comparator类型.

1. Compare 基本的比较方法
2. Name 返回比较器的名字
3. FindShortestSeparator 返回两个字符串最长公共前缀
4. FindShortSuccessor 返回给定key的后继字符串

### 1.1.11. dumpfile.h

将给定名字的文件写入一个给定的WritableFile。

1. DumpFile 拷贝函数。调用一系列的WritableFile.append()

### 1.1.12. Env.h

1. WritableFile 定义了LevelDB中进行文件操作的接口。
2. Logger 定义了Log文件的接口。
3. FileLock 定义了文件锁
4. EnvWrapper 封装了一层的Env结构

### 1.1.13. filter_policy.h

定义了FilterPolicy类型。

### 1.1.14. TableBuildher.h

定义了文件和Table结构的交互类型：TableBuilder类型。

### 1.1.15. Table.h

定义了LevelDB中使用的sstable类型Table。

### 结语
到这里就阅读完了include/levelddb中的所有文件。这个文件夹下是levelDB中要用到的各种顶层类型的定义，抽象程度比较高。接下来看具体的实习以及底层的类型。