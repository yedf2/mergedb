## MergeDB: a leveldb variant with faster write speed

MergeDB is developed and maintained by yedf.
It is built on earlier work on LevelDB by Sanjay Ghemawat (sanjay@google.com)
and Jeff Dean (jeff@google.com)

The interface of mergedb is the same as [leveldb](https://github.com/google/leveldb)
except for some options added which will be introduced below.

This code only change the compaction strategy in leveldb and achieve a faster write speed. In original leveldb, it suffers a great write amplification in random write. Take a db with 7 levels as an example, in order to write 1M data, there isa write to log, a write to level0, and about 11 writes to move up a level, that is about 66 write to move to top level. In mergedb, the total write amplification will be about 15, thus speed up the write opertions dramactically. It is espetially suitable for write heavy applications.

## Performance Benchmarks

Currently, I don't get a idea machine to perform following tests. Anyone interested in doing a test with a large database (maybe 800G) is appriciated.

### Hard Disk Tests

On an aliyun ECS with

  * 2 * Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz
  * 4G memory
  * a disk of size 20G with write speed about 45MB/s (observed by command 'cp /dev/zeror ./test')

  * 2 Million keys; each key is of size 16 bytes, each value is of size 4K bytes.
  * total database size is 8G without snappy.

### Test. Bulk Load of keys in sequential order
The database is empty at the beginning of this benchmark run and gradually fills up. No data is being read when the data load is in progress.

    mergedb: MB/sec (total data size GB)
    leveldb: 145.569 micros/op;   26.9 MB/s (total data size 7.9GB)

Here is the command for the test
    ./db_bench --benchmarks=fillseq --num=2000000 --threads=1 --value_size=4096 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --write_buffer_size=2097152 --use_existing_db=0

### Test. Random Write
Measure performance to randomly overwrite all keys into the database. The database was first created by sequentially inserting all keys. The results here do not measure the sequential-insertion phase, it measures only second part of the test that overwrites keys in random order. 
Notice, mergedb does better when the database is larger(but i don't get an idea machine)

    mergedb: MB/sec (total data size GB)
    leveldb: 2130.180 micros/op;    1.8 MB/s (total data size 12GB)

Here is the command for the test
    ./db_bench --benchmarks=overwrite --num=2000000 --threads=1 --value_size=4096 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --write_buffer_size=2097152 --use_existing_db=1
    
### Test3. Random Read
Measure random read performance of a database with all keys. The database was first created by sequentially inserting keys. Once the load is complete, we drop the file cache(or else most read will be in cache), then the benchmark randomly picks a key and issues a read request. The above measure measurement does not include the data loading part, it measures only the part that issues the random reads to database. 

    mergedb: 
    leveldb: 5068.182 micros/op    197.31 op/s

Here is the command for the test
    echo 1 > /proc/sys/vm/drop_caches; ./db_bench --benchmarks=readrandom --num=2000000 --threads=1 --value_size=4096 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --write_buffer_size=2097152 --use_existing_db=1 --reads=20000