# redis test

* redis 单元测试架构分析
* tclsh 脚本实现的框架

## 运行

### make 运行

```Makefile
# 单元测试
make test

# redis 模块化编程测试
make test-modules

# 没有研究
# todo
# 应该哨兵模式的测试
make test-sentinel

```

### shell 运行

* make 里面的执行，就是执行这些sh脚本

```sh
$ ls runtest*
runtest           runtest-cluster   runtest-moduleapi runtest-sentinel
```

```sh
sh runtest

# cluster 集群测试
sh runtest-cluster

sh runtest-moduleapi

sh runtest-sentinel


# 具体使用细节

sh runtest-xxx --help
```

### 运行单实例测试

```sh

# 查看有哪些单元测试
sh runtest --list-tests

# 运行其中一个测试
sh --single 单实例列表

# runtest-cluster 单实例运行
# 
$ ls tests/cluster/tests/

sh --single 单实例文件名

```

## 单元测试源码

* runtest-xxx shell脚本调用 tests目录下的tclsh脚本
* tclsh 脚本运行

## test_helper.tcl

### 框架逻辑

```sh

# 1. 入口函数

test_server_main {
    ...
    # 启动一个server 接收命令
    socket -server accept_test_clients  -myaddr 127.0.0.1 $clientport
    # 启动多个client 实例
    # Start the client instances
    set ::clients_pids {}
    if {$::external} {
        set p [exec $tclsh [info script] {*}$::argv \
            --client $clientport &]
        lappend ::clients_pids $p
    } else {
        set start_port $::baseport
        set port_count [expr {$::portcount / $::numclients}]
        for {set j 0} {$j < $::numclients} {incr j} {
            # 执行脚本，启动一个clients
            set p [exec $tclsh [info script] {*}$::argv \
                --client $clientport --baseport $start_port --portcount $port_count &]
            lappend ::clients_pids $p
            incr start_port $port_count
        }
    }

    ....
}


#2 server 做的事情
# 回调函数
accept_test_clients
# fd 读取数据
# 
# 判断 单元测试 执行状态
# 和所有的client fd 建立连接
read_from_test_client
# 发送单元测试文件名称到client上，去执行
signal_idle_client

#3 client 做的事情
# test_server_main 执行tests/test_helper.tcl脚本,启动clients

# 等待server 发送单元测试命令，执行
test_client_main {
    set ::test_server_fd [socket localhost $server_port]
    fconfigure $::test_server_fd -encoding binary
    send_data_packet $::test_server_fd ready [pid]
    while 1 {
        set bytes [gets $::test_server_fd]
        set payload [read $::test_server_fd $bytes]
        foreach {cmd data} $payload break
        if {$cmd eq {run}} { # 执行测试脚本文件
            execute_test_file $data
        } elseif {$cmd eq {run_code}} {
            foreach {name filename code} $data break
            execute_test_code $name $filename $code
        } else {
            error "Unknown test client command: $cmd"
        }

}

```

### 单元测试例子

``` sh
# 单元测试文件存放目录
# tests/unit
```

#### unit/limits.tcl 例子

```sh
# 运行
sh runtest --single unit/limits
```

```sh
# start_server 执行函数 启动一个redis server
# tags 日志打印标签
# overrides 覆盖配置参数

start_server {tags {"limits network"} overrides {maxclients 10}} {
    # start_server 函数中执行 如下语句
    if {$::tls} {
        set expected_code "*I/O error*"
    } else {
        set expected_code "*ERR max*reached*"
    }
    # 执行test 函数
    test {Check if maxclients works refusing connections} {
        # test 函数执行如下语句
        set c 0
        catch {
            while {$c < 50} {
                incr c
                set rd [redis_deferring_client]
                $rd ping
                $rd read
                after 100
            }
        } e # 捕捉错误 
        assert {$c > 8 && $c <= 10}
        set e
    } $expected_code # 这个地方会进行错误匹配，看是否是指定的错误
}
```

## redis cluster 集群测试

```sh
# 执行脚本
sh runtest-cluster

# 实际执行tclsh脚本
# tests/cluster/run.tcl
```

### tests/cluster/run.tcl

```sh
proc main {} {
    parse_options
    # 启动redis_server 实例
    # 组建集群
    spawn_instance redis $::redis_base_port $::instances_count {
        "cluster-enabled yes"
        "appendonly yes"
        "gossip-validation-enabled yes"
    }
    # 运行测试脚本
    # tests/cluster/tests/ 测试脚本存放在这个目录下
    run_tests

    cleanup
    end_tests
}

# spawn_instance 定义在 tests/instances.tcl
# run_tests 执行source file 加载文件
```

### 00-base.tcl

* 执行
```sh
sh runtest-cluster --single 00-base.tcl
```

* y源码
```sh
source "../tests/includes/init-tests.tcl"

if {$::simulate_error} {
    test "This test will fail" {
        fail "Simulated error"
    }
}

# 执行test 函数
test "Different nodes have different IDs" {
    set ids {}
    set numnodes 0
    foreach_redis_id id {
        incr numnodes
        # Every node should just know itself.
        set nodeid [dict get [get_myself $id] id]
        assert {$nodeid ne {}}
        lappend ids $nodeid
    }
    set numids [llength [lsort -unique $ids]]
    assert {$numids == $numnodes}
}

test "It is possible to perform slot allocation" {
    cluster_allocate_slots 5
}
.....

```
