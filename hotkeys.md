# 总结
redis-cli hotkeys实现是一个很消耗cpu 的操作

* redis-cli获取hotkeys工作过程如下：

```sh
1. scan 命令迭代 获取 key 键值
2. object freq key-name(获取每一个key的 访问频率)
3. 将每一次访问频率排序进行排序，获取最大访问频率的top k热键（代码这个k是16）
4. 重复上述步骤，遍历所有key，给每一个key的访问频率按照最大访问次数进行top k排序，找出hotkeys
```

* redis-server工作过程：

```sh
1. 服务端需要开启lfu算法，才能统计每一个key的访问频率
2. 开启lfu算法的前提是需要配置maxmemory参数（配置内存缓冲超过多少时，使用lfu淘汰算法）
3. redis-cli通过lfu的统计每一个key访问频率这个参数实现了hotkeys这个功能。
4. 如下会通过代码详细分析实现过程
```

# redis-cli hotkeys 使用
* server配置

```sh
maxmemory 100mb
maxmemory-policy allkeys-lfu
redis-cli hotkeys 使用
```

## hotkeys 命令执行

```sh
./redis-cli -p 6501 --hotkeys

# Scanning the entire keyspace to find hot keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.22%] Hot key '"1test953"' found so far with counter 2
[00.33%] Hot key '"1test558"' found so far with counter 2
[00.67%] Hot key '"1test962"' found so far with counter 2
[00.78%] Hot key '"2test73"' found so far with counter 2
[00.89%] Hot key '"1test854"' found so far with counter 2
[01.11%] Hot key '"1test25"' found so far with counter 2
[01.54%] Hot key '"2test186"' found so far with counter 2
[02.15%] Hot key '"2test300"' found so far with counter 2
[02.70%] Hot key '"0test48"' found so far with counter 2
[03.48%] Hot key '"2test91"' found so far with counter 2
[04.47%] Hot key '"1test397"' found so far with counter 2
[04.59%] Hot key '"2test966"' found so far with counter 2
[04.69%] Hot key '"2test913"' found so far with counter 2
[05.13%] Hot key '"1test975"' found so far with counter 2
[05.35%] Hot key '"1test128"' found so far with counter 2
[05.46%] Hot key '"2test313"' found so far with counter 2

-------- summary -------

Sampled 9202 keys in the keyspace!
hot key found with counter: 2	keyname: "1test953"
hot key found with counter: 2	keyname: "1test558"
hot key found with counter: 2	keyname: "1test962"
hot key found with counter: 2	keyname: "2test73"
hot key found with counter: 2	keyname: "1test854"
hot key found with counter: 2	keyname: "1test25"
hot key found with counter: 2	keyname: "2test186"
hot key found with counter: 2	keyname: "2test300"
hot key found with counter: 2	keyname: "0test48"
hot key found with counter: 2	keyname: "2test91"
hot key found with counter: 2	keyname: "1test397"
hot key found with counter: 2	keyname: "2test966"
hot key found with counter: 2	keyname: "2test913"
hot key found with counter: 2	keyname: "1test975"
hot key found with counter: 2	keyname: "1test128"
hot key found with counter: 2	keyname: "2test313"
```

# hotkeys 代码分析

* 主要通过redis-cli和redis-server实现原理来分析hotkeys的具体实现

```sh
1. redis-cli 遍历所有key得到访问频率进行最大top k的排序，然后找出hotkeys
2. redis-server通过lfu算法动态更新每一个key的访问频率
```

## redis-cli分析

```sh
1. scan 命令迭代 获取 keys键值
2. object freq key-name(获取每一个key的 访问频率)
3. 将每一次访问频率排序进行排序，获取最大访问频率的top k热键（代码这个k是16）
4. 重复上述步骤，遍历所有key，给每一个key的访问频率按照最大访问次数进行top k排序，找出hotkeys
```

## redis-server 代码实现

```sh
1. 服务端的是通过开启lfu算法，进行的访问频率统计，redis-cli通过获取每一个key 的访问频率进行最大top k排序找出hotkeys
2. 通过文档查询可知，lfu算法的实现是通过lru算法改进过来的，而且redis里面的lru和lfu有很多代码复用的情况，为了方便理解我会从lru算法开始讲解，然后过度到lfu算法
3. 如下会通过讲解redis的这两种实现算法，来分析具体原理
```

### LRU(least recently used)

* 缓冲达到上限时，优先淘汰最近最少使用缓存机制
* 已知的实现方式都是： 双链表 + map实现

#### redis LRU实现

* Redis的LRU算法并非完整的实现（它会尝试运行一个近似LRU的算法）
* 比较节约内存

lru实现分为两步:

* 更新key的访问时间戳
* 内存满时，淘汰cache

每一个键被访问时，都会更新这个时间戳

```c
typedef struct redisObject {
    ...
    // 单位s
	// 记录每一个key的最近访问时间
    unsigned lru:LRU_BITS; //LRU_BITS为24bit 就是存放每个 key 的访问时间
    ...
} robj;

// lru更新获取时间戳如下（redis mstime()/LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;）
// lru =  LRU_CLOCK();

#define LRU_BITS 24
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */
#define LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */

/* Return the LRU clock, based on the clock resolution. This is a time
 * in a reduced-bits format that can be used to set and check the
 * object->lru field of redisObject structures. */
unsigned int getLRUClock(void) {
    return (mstime()/LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;
}

/* This function is used to obtain the current LRU clock.
 * If the current resolution is lower than the frequency we refresh the
 * LRU clock (as it should be in production servers) we return the
 * precomputed value, otherwise we need to resort to a system call. */
unsigned int LRU_CLOCK(void) {
    unsigned int lruclock;
    if (1000/server.hz <= LRU_CLOCK_RESOLUTION) {
        atomicGet(server.lruclock,lruclock);
    } else {
        lruclock = getLRUClock();
    }
    return lruclock;
}
```

淘汰算法

```sh
1. 内存满时
2. Redis随机选择maxmemory_samples数量的key
3. 然后计算这些key的空闲时间(idle time)
4. Redis 维护一个固定大小的pool(最小堆排序，始终保留最大的空闲时间key)
5. 当满足条件时(比pool中的某些键的空闲时间还大)就可以进pool
6. pool更新之后，就淘汰pool中空闲时间最大的键。
```

```sh
# 随机采样点，默认是五个
maxmemory-samples 5
```

redis-lru算法效果图

![avatar](https://redis.io/images/redisdoc/lru_comparison.png)

* Theoretical LRU: 理想中的LRU算法
* Approx LRU Redis 2.8 5 Samples （旧版本的算法实现）
* Approx LRU Redis 3.0 5 Samples (新版本的算法实现)
* Approx LRU Redis 3.0 10 Samples （抽样数增加）
图片颜色

* 浅灰色带是被驱逐的对象。
* 灰色带是未被驱逐的对象。
* 绿色带是添加的对象。

抽样数问题:

* 随着抽样数增加，redis-lru算法越接近理想中的lru算法。但是抽样数增加会导致性能损耗

#### lru算法问题
LRU 看上去是最合理的，毕竟最近使用的数据更有可能会被再次访问，但看看如下的例子你会发现问题：

```sh
~代表时间秒
| 代表抽样时刻

~~~~~A~~~~~A~~~~~A~~~~A~~~~~A~~~~~A~~|
~~R~~R~~R~~R~~R~~R~~R~~R~~R~~R~~R~~R~|
~~~~~~~~~~C~~~~~~~~~C~~~~~~~~~C~~~~~~|
~~~~~D~~~~~~~~~~D~~~~~~~~~D~~~~~~~~~D|
A 访问频率间隔5秒
R访问频率间隔2秒
C访问频率间隔10秒
D访问频率间隔10秒
在|这个点去抽样:

A 空闲了2秒
R空闲了1秒
C空闲了6秒
D空闲了0秒，显然这个抽样统计时有问题的。
LFU(least frequently used)
访问次数最少的最应该被逐出
```

## lfu实现分为两步

* 更新key的访问频率
* 内存满时，淘汰cache

redis-lfu实现特点

* 复用lru结构体
* 访问次数特别大的 key 可能以后都不再访问了，但是因为访问次数大而一直占用着内存不被淘汰，需要一个方法来逐步“驱除”（有点 LRU的意思），最简单的就是逐步衰减访问次数
* 某些 key 访问次数可能非常之大，理论上可以无限大，但实际上我们并不需要精确的访问次数，所有redis里面的counter是一个近似计数算法；


```c
typedef struct redisObject {
    ...
    // 16 bit 上一次递减时间,分钟级别的时间戳
    // 访问次数特别大的 key 可能以后都不再访问了，但是因为访问次数大而一直占用着内存不被淘汰，需要一个方法来逐步“驱除”（有点 LRU的意思），最简单的就是逐步衰减访问次数
    // 8 bit 访问次数，0 ~ 255
    // 某些 key 访问次数可能非常之大，理论上可以无限大，但实际上我们并不需要精确的访问次数；
    // redis对于访问次数做了特殊处理
    unsigned lru:LRU_BITS; //LRU_BITS为24bit 就是存放每个 key 的访问时间
    ...
} robj;
```

配置参数

```sh
# counter 累计因子，影响访问次数counter计数
# 默认为10.可以看到,一个key访问一千万次以后counter值才会到达255.factor值越小, counter越灵敏
lfu-log-factor 10
# 默认为1
# 衰减时间，每一分钟counter - 1
lfu-decay-time 1
```

lfu-log-factor对访问次数影响

* factor越大随着访问次数增加counter变化的越缓慢

```sh
* factor 对 counter 大小影响
+--------+------------+------------+------------+------------+------------+
| factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
+--------+------------+------------+------------+------------+------------+
| 0      | 104        | 255        | 255        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 1      | 18         | 49         | 255        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 10     | 10         | 18         | 142        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 100    | 8          | 11         | 49         | 143        | 255        |
+--------+------------+------------+------------+------------+------------+
```

lfu代码

```c
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);//首先计算是否需要将counter衰减
    counter = LFULogIncr(counter);//根据上述返回的counter计算新的counter
    val->lru = (LFUGetTimeInMinutes()<<8) | counter; //robj中的lru字段只有24bits,lfu复用该字段。高16位存储一个分钟数级别的时间戳，低8位存储访问计数
}

#define	RAND_MAX	0x7fffffff

// int rand(void);
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;//原来保存的时间戳
    unsigned long counter = o->lru & 255; //原来保存的counter
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    //server.lfu_decay_time默认为1,每经过一分钟counter衰减1
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;//如果需要衰减,则计算衰减后的值
    return counter;
}
 
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;//counter最大只能存储到255,到达后不再增加
    double r = (double)rand()/RAND_MAX;//算一个随机的小数值
    double baseval = counter - LFU_INIT_VAL;//新加入的key初始counter设置为LFU_INIT_VAL,为5.不设置为0的原因是防止直接被逐出
    if (baseval < 0) baseval = 0;
    double p = 1.0/(baseval*server.lfu_log_factor+1);//server.lfu_log_facotr默认为10
    if (r < p) counter++;//可以看到,counter越大,则p越小，随机值r小于p的概率就越小。换言之,counter增加起来会越来越缓慢
    return counter;
}
 
unsigned long LFUGetTimeInMinutes(void) {
    return (server.unixtime/60) & 65535;//获取分钟级别的时间戳
}
```

淘汰算法

* 内存满时
* Redis随机选择maxmemory_samples数量的key
* 然后计算这些key的（255 - key的访问频率）值
* Redis 维护一个固定大小的pool(最小堆排序，始终保留最大的（255 - key的访问频率）值)
* 当满足条件时(比pool中的某些键的空闲时间还大)就可以进pool
* pool更新之后，就淘汰pool中空闲时间最大的键。

总结

* redis-cli的hotkeys就是通过lfu，获取key的访问频率进行最大top k排序计算出hotkeys

## lru 和 lfu 算法压测对比

* maxmemory 50m
* 同一个服务端不同算法

### lfu

```sh
$ ./redis-cli -p 6501 --lru-test 1000000
135000 Gets/sec | Hits: 134655 (99.74%) | Misses: 345 (0.26%)
139000 Gets/sec | Hits: 138622 (99.73%) | Misses: 378 (0.27%)
139250 Gets/sec | Hits: 138876 (99.73%) | Misses: 374 (0.27%)
130500 Gets/sec | Hits: 130146 (99.73%) | Misses: 354 (0.27%)
135000 Gets/sec | Hits: 134660 (99.75%) | Misses: 340 (0.25%)
133750 Gets/sec | Hits: 133396 (99.74%) | Misses: 354 (0.26%)
134250 Gets/sec | Hits: 133912 (99.75%) | Misses: 338 (0.25%)
132250 Gets/sec | Hits: 131930 (99.76%) | Misses: 320 (0.24%)
128750 Gets/sec | Hits: 128435 (99.76%) | Misses: 315 (0.24%)
133000 Gets/sec | Hits: 132682 (99.76%) | Misses: 318 (0.24%)
132000 Gets/sec | Hits: 131649 (99.73%) | Misses: 351 (0.27%)
133750 Gets/sec | Hits: 133422 (99.75%) | Misses: 328 (0.25%)
134250 Gets/sec | Hits: 133922 (99.76%) | Misses: 328 (0.24%)
131000 Gets/sec | Hits: 130676 (99.75%) | Misses: 324 (0.25%)
135500 Gets/sec | Hits: 135143 (99.74%) | Misses: 357 (0.26%)
134000 Gets/sec | Hits: 133687 (99.77%) | Misses: 313 (0.23%)
```

### lru

```sh
$ ./redis-cli -p 6501 --lru-test 1000000
122250 Gets/sec | Hits: 121906 (99.72%) | Misses: 344 (0.28%)
121000 Gets/sec | Hits: 120651 (99.71%) | Misses: 349 (0.29%)
126250 Gets/sec | Hits: 125903 (99.73%) | Misses: 347 (0.27%)
128750 Gets/sec | Hits: 128355 (99.69%) | Misses: 395 (0.31%)
129250 Gets/sec | Hits: 128883 (99.72%) | Misses: 367 (0.28%)
124500 Gets/sec | Hits: 124148 (99.72%) | Misses: 352 (0.28%)
126750 Gets/sec | Hits: 126355 (99.69%) | Misses: 395 (0.31%)
125750 Gets/sec | Hits: 125399 (99.72%) | Misses: 351 (0.28%)
132250 Gets/sec | Hits: 131868 (99.71%) | Misses: 382 (0.29%)
133250 Gets/sec | Hits: 132867 (99.71%) | Misses: 383 (0.29%)
118250 Gets/sec | Hits: 117912 (99.71%) | Misses: 338 (0.29%)
129000 Gets/sec | Hits: 128632 (99.71%) | Misses: 368 (0.29%)
129500 Gets/sec | Hits: 129175 (99.75%) | Misses: 325 (0.25%)
125750 Gets/sec | Hits: 125398 (99.72%) | Misses: 352 (0.28%)
```

### 总结

* 从Gets/sec 和 Misses来看 lfu更优
* 由于机器性能不够，没有进行极限压测，数据仅供参考

# 文档链接

* https://redis.io/topics/lru-cache
