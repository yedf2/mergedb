[中文版](https://github.com/yedf/mergedb/blob/master/README-cn.md)

# MergeDB: a better leveldb

MergeDB is developed and maintained by yedf.
It is built on earlier work on LevelDB by Sanjay Ghemawat (sanjay@google.com)
and Jeff Dean (jeff@google.com)

The interface of mergedb is the same as [leveldb](https://github.com/google/leveldb)

# Improvement

Benchmark run on my macbook shows following  

    ./db_bench --benchmarks=fillrandom --num=6000000 --threads=1 --value_size=1024 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --use_existing_db=0

### Write Speed

*   leveldb 15.6M/s
*   mergedb 25.7M/s

### Space Consumptionb

*   leveldb 4.4G
*   mergedb 4.1G

### About Benchmarks

This benchmarks runs on my macbook.  

*   CPU 2.7 GHz Intel Core i5 2 cores
*   Memory 8 GB 1867 MHz DDR3

Use of bloom filter with 15 bits will reduce 99.9% miss read, so both LevelDB and MergeDb 's read qps is about the same as iops of disk.  
Both leveldb and mergedb 's read speed is about 281 micros/op when the data is not in buffer.

    ./db_bench --benchmarks=readrandom --num=6000000 --threads=1 --value_size=1024 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --use_existing_db=1 --reads=20000

Testing a large db will take several days, so the above benchmarks only test a db with very few data.   
I simulate a large db for LevelDB by changing the write_buffer_size and level1 size to get a db with more levels(7 levels with only a little data in level 6).

    ./db_bench --benchmarks=fillrandom --num=6000000 --threads=1 --value_size=1024 --cache_size=104857600 --bloom_bits=10 --open_files=500000 --db=ldb --compression_ratio=1 --use_existing_db=0 --write_buffer_size=65536

Write Speed

*   leveldb 5.0M/s
*   mergedb 9.9M/s

# Algorithm

*   given:  
    * level_size(n) for size of level n  
    * max_size(n) for max size can be stored in level n
    * acc_size(n) for level_size(0) + level_size(1) + ... + level_size(n)  
    * N for largest non-empty level
*   max_size(n+1) / max_size(n) is changed from 10 to 2
*   Compaction will pickup levels from level 0 up to level k and generate a single larger level.  
    k is the max n for acc_size(n) < max_size(n)  
    For this strategy, writes(excluding the log and level0) for a full leveled (each level's size is equal to max_size(n)) db of n levels can be calculated:  
            writes(n) = writes(n-1) + writes-for-only-level(n)  
    Since level n can be constructed by a single compact of a full leveled db of n-1 levels. So  
            writes-for-only-level(n) = writes(n-1) + max_size(n)  
            write(n) = writes(n-1) + write(n-1) + max_size(n)
                = 2 * writes(n-1) + max_size(n)
                = 4 * writes(n-2) + 2 * max_size(n-1) + max_size(n)
                = 4 * writes(n-2) + 2 * max_size(n)
                = n * max_size(n)
                = n * size_of_db / 2
    Thus the write-amplification is n/2
*   Doing a compaction involving all data is a time cosuming operation.
    In order not to stop writting, we pickup several ajacent non-empty levels from outer(larger n).
    When outer compaction is in progress, inner compaction may start.
    This strategy will result in a little more writes.
*   The space can be 3X if we are overwriting records and level_size(N) is about the same as level_size(N-1).  
    In order to reduce space cosumption, we compact when sum_size(N, N-1, ..., N-4) / level_size(N) > 1.25  
    This strategy will result in about 3 extra writes for each record
