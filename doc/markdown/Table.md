# Table

阅读Table文件夹中关于sstable实现的有关内容。

## block_builder

> 和table_builder是两个类，不要弄混了。

在[overview](./overview.md)中已经描述过了，sstable只按照block来进行组织的。这里的block_builder用来构建大的Data Block结构。存放必要的信息。

1. options_ Options 构造的参数
2. buffer_ string 缓存区
3. restarts_ vector<uint32_t> 记录每个restart pointer的偏移
4. counter_ 记录在上一个restart pointer后又有几个entry了。
5. finished 标识写入是否结束了。
6. last_key_ 当前block的最后一个key(最大的key)

## Block

