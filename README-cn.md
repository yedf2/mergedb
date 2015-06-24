# MergeDB 更快的leveldb

MergeDB基于Sanjay Ghemawat (sanjay@google.com)和Jeff Dean (jeff@google.com)的LevelDB

# 改进

在我的macbook上的测试对比

    ./db_bench --benchmarks=fillrandom --num=6000000 --threads=1 --value_size=1024 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --use_existing_db=0


### 写性能

*   leveldb 15.6M/s
*   mergedb 25.7M/s

### 磁盘空间使用

*   leveldb 4.4G
*   mergedb 4.1G

### 性能测试环境

macbook.  

*   CPU 2.7 GHz Intel Core i5 2 cores
*   Memory 8 GB 1867 MHz DDR3

15bit的bloomfilter可以将减少99.9%的无法命中读操作，因此LevelDB与MergeDB的读性能与磁盘的iops基本相等（当然数据在缓存中则读qps会大很多）。
当数据没有被缓存时，LevelDB和MergeDB的随机读测试结果为281.055 micros/op。命令如下

    ./db_bench --benchmarks=readrandom --num=6000000 --threads=1 --value_size=1024 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --write_buffer_size=2097152 --use_existing_db=1 --reads=20000

测试一个几百G的数据库读写操作需要几天的时间，这里通过修改levelDB的write_buffer_size参数和level n的size来让LevelDB产生更多的level（一共7层数据，level6上有一点点数据）。
可以类似的达到一个大数据库的效果

    ./db_bench --benchmarks=fillrandom --num=6000000 --threads=1 --value_size=1024 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --use_existing_db=0 --write_buffer_size=65536

写入性能

*   leveldb 5.0M/s
*   mergedb 9.9M/s

# 算法

*   函数定义:  
    * level_size(n) level n 的数据大小  
    * max_size(n) level n 可存放数据的最大大小
    * acc_size(n) = level_size(0) + level_size(1) + ... + level_size(n)  
    * N 有数据的最大层
*   max_size(n+1) / max_size(n) 由10改为2
*   选择level 0 到 level k进行压缩  
    k 是acc_size(k) < max_size(k)的最大值  
    这个策略之下，产生一个每层都达到最大数据量的db写入次数（不包括写入log和level0） 
        writes(n) = writes(n-1) + writes-for-only-level(n)  
    层级 n 的数据可以通过对level 0 到 level n-1 的所有数据进行一次压缩获得 
        writes-for-only-level(n) = writes(n-1) + max_size(n)  
        write(n) = writes(n-1) + write(n-1) + max_size(n)
            = 2 * writes(n-1) + max_size(n)
            = 4 * writes(n-2) + 2 * max_size(n-1) + max_size(n)
            = 4 * writes(n-2) + 2 * max_size(n)
            = n * max_size(n)
            = n * size_of_db / 2
    最后的写放大是n/2
*   对所有层级进行一次压缩会耗费大量的时间，期间如果停止写入操作是不可行的，
    因此每次我们都会从外层（较大的n）开始查找，找到连续非空的4层则进行压缩。  
    外层的压缩进行时，内层可以启动新的压缩。即外层压缩期间，可以接受的新写入数据量理想情况下可以达到外层压缩数据量的1/16.  
*   对于一个会重复写入数据的应用来书，当最外层N的数据量约等于N-1层的数据量时，则各层数据量分别为X X X/2 X/4 ...   
    X的数据占用了3X的空间。
    为了节约空间，每次检查最外5层，如果
        sum_size(N, N-1, ..., N-4)/level_size(N) > 1.25
    则进行一次压缩，保证最多只占用1.25X的空间

