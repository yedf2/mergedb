## MergeDB 更快的leveldb

MergeDB基于Sanjay Ghemawat (sanjay@google.com)和Jeff Dean (jeff@google.com)的LevelDB

LevelDB写放大问题严重影响了应用的写性能。对于一个普通的数据库写操作，LevelDB需要写一次到Log，一次到level0，11次写才能够让数据所在的level上升到level+1。假设数据库有7层，那么数据到达level6需要1+1+6*11=68次写。而MergeDB使用新的层级策略和压缩策略只需要大约15次写。

## 性能测试

由于没有合适的机器做测试，只在阿里云上得一台普通服务器上进行了简单的测试。对于更大的数据库，性能的差别更加明显

## 普通硬盘测试

测试机为一台阿里云的ECS：

  * 2 * Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz
  * 4G 内存
  * 磁盘写入速度大约为45M/s（通过命令'cp /dev/zero ./test'观测）
  * 2M条记录，key为16字节，value为4K字节
  * 数据库总大小为8G，snappy没有打开

### 随机写测试

测试随机写性能，数据库初始时为空，写入的过程中，没有读操作

    mergedb: 853.131 micros/op;    4.6 MB/s
    leveldb: 1913.242 micros/op;    2.0 MB/s

LevelDB与MergeDb的命令相同，如下：

    ./db_bench --benchmarks=fillrandom --num=2000000 --threads=1 --value_size=4096 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --write_buffer_size=2097152 --use_existing_db=0

### 随机读测试

测试随机读性能，数据库由前一个随机写创建。由于数据库较小，读取时会有很大比例在缓冲当中，因此测试读取之前我清空了缓存。读取的数目为2w，按照一般硬盘的速度，每秒100-200个随机io，2M条记录需要几个小时。

    mergedb: 6587.220 micros/op   151.80 op/s
    leveldb: 6493.976 micros/op   153.98 op/s

命令如下：

    echo 1 > /proc/sys/vm/drop_caches; ./db_bench --benchmarks=readrandom --num=2000000 --threads=1 --value_size=4096 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --write_buffer_size=2097152 --use_existing_db=1 --reads=20000
