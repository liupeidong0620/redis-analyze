# 总结
redis-cli的bigkeys方式是一个很消耗cpu的操作，cli会和server通信遍历所有key，然后获取每个key的value大小，进行排序算出bigkeys。

# bigkeys 分析
## redis-cli使用

```sh
./redis-cli -p 6501 --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far '"lru:305423"' with 5 bytes
[14.26%] Biggest string found so far '"test1"' with 65 bytes
[48.19%] Biggest string found so far '"tesrt"' with 183 bytes

-------- summary -------

Sampled 601246 keys in the keyspace!
Total key length in bytes is 5901340 (avg len 9.82)

Biggest string found '"tesrt"' has 183 bytes

0 lists with 0 items (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
601246 strings with 3006515 bytes (100.00% of keys, avg size 5.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

## 代码逻辑

```sh
1. scan 迭代

2. 获取 key 键值

3. type key-name

4. 获取每一个key的类型

5. 然后通过每一个key 对应的value大小

typeinfo type_string = { "string", "STRLEN", "bytes" };
typeinfo type_list = { "list", "LLEN", "items" };
typeinfo type_set = { "set", "SCARD", "members" };
typeinfo type_hash = { "hash", "HLEN", "fields" };
typeinfo type_zset = { "zset", "ZCARD", "members" };
typeinfo type_stream = { "stream", "XLEN", "entries" };


重复上述步骤，遍历所有key，找出bigkeys
```
