

有同事问：通过如下两种方式重启 management 插件有何区别？

# 重置 management 插件数据库的两种方式

方式一：
```shell
$ rabbitmqctl eval 'exit(erlang:whereis(rabbit_mgmt_db), please_terminate).'
```

方式二：
```shell
$ rabbitmqctl eval 'application:stop(rabbitmq_management), application:start(rabbitmq_management).'
```

可以看到，两种方式都是通过 rabbitmqctl 的 eval 子命令调用 erlang 模块导出函数实现的**某种**重置功能；

那么哪种方式对系统的影响比较小呢？

# eval 子命令代码执行路径

官方对 eval 子命令的解释为：针对任意 Erlang 表达式进行求值计算；    
调用格式为
```shell
eval {expr}
```

在 `rabbitmqctl` 脚本中可以看到
```shell
RABBITMQ_USE_LONGNAME=${RABBITMQ_USE_LONGNAME} \
exec ${ERL_DIR}erl \
    -pa "${RABBITMQ_HOME}/ebin" \
    -noinput \
    -hidden \
    ${RABBITMQ_CTL_ERL_ARGS} \
    -boot "${CLEAN_BOOT_FILE}" \
    -sasl errlog_type error \
    -mnesia dir "\"${RABBITMQ_MNESIA_DIR}\"" \
    -s rabbit_control_main \
    -nodename $RABBITMQ_NODENAME \
    -extra "$@"
```

因此，最终是将 `eval 'xxx'` 传给了 `rabbit_control_main.erl` 模块进行处理，代码如下

```erlang
action(eval, Node, [Expr], _Opts, _Inform) ->
	%% 词法分析 eval 后指定的命令字串
    case erl_scan:string(Expr) of
        {ok, Scanned, _} ->
	        %% 基于词法分析后的内容得到调用序列信息
            case erl_parse:parse_exprs(Scanned) of
                {ok, Parsed} -> {value, Value, _} =
					                %% 进行 RPC 调用
					                %% 即在 Node 节点通过 erl_eval:exprs/1 执行
					                %% exit(erlang:whereis(rabbit_mgmt_db), please_terminate).
                                    unsafe_rpc(
                                      Node, erl_eval, exprs, [Parsed, []]), 
                                io:format("~p~n", [Value]),
                                ok;
                {error, E}   -> {error_string, format_parse_error(E)}
            end;
        {error, E, _} ->
            {error_string, format_parse_error(E)}
    end;
```

可见，eval 的执行最终是在目标 Node 上进行了 RPC 调用；

# 第一种方式

通过 `exit(erlang:whereis(rabbit_mgmt_db), please_terminate).` 重置数据库，其实就是向 `rabbit_mgmt_db` 进程发送了一个原因为 `please_terminate` 的退出信号；

在 erlang 官方文档中有如下说明

```erlang
exit(Pid, Reason) -> true
```

> Sends an exit signal with exit reason `Reason` to the process or port identified by `Pid`.
> 
> The following behavior applies if Reason is any term, except `normal` or `kill`:
>> - If Pid is not trapping exits, Pid itself exits with exit reason Reason.
>> - If Pid is trapping exits, the exit signal is transformed into a message `{'EXIT', From, Reason}` and delivered to the message queue of Pid.

从代码中可以看到，`rabbit_mgmt_db` 进程并没有对退出信号进行捕获，因此当其收到退出信号后，将直接退出执行；同时由于 `rabbit_mgmt_db` 进程是 `worker` 进程，且配置成 `permanent` 以保证总是被重启，故我们可以在 RabbitMQ 的日志中看到如下输出

输出 rabbit_mgmt_db 进程退出信息；
```shell
=SUPERVISOR REPORT==== 1-Aug-2016::17:32:09 ===
     Supervisor: {<0.333.0>,mirrored_supervisor_sups}
     Context:    child_terminated
     Reason:     please_terminate
     Offender:   [{pid,<0.338.0>},
                  {name,rabbit_mgmt_db},
                  {mfargs,{rabbit_mgmt_db,start_link,[]}},
                  {restart_type,permanent},
                  {shutdown,4294967295},
                  {child_type,worker}]
```

输出 rabbit_mgmt_db 重新被启动信息；
```shell
=INFO REPORT==== 1-Aug-2016::17:32:09 ===
Statistics database started.
```

通过上述日志信息，以及 RabbitMQ 内部进程实际变化情况，可以得出结论：上述过程中只有 `rabbit_mgmt_db` 进程发生了重启行为，其它进程没有任何变化，故对系统影响非常小；

发信号前

![停止 rabbit_mgmt_db 进程前](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E4%BB%85%E5%81%9C%E6%AD%A2rabbit_mgmt_db%E7%9A%84%E6%96%B9%E5%BC%8F_1.png "停止 rabbit_mgmt_db 进程前")

发信号后
![停止 rabbit_mgmt_db 进程后](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E4%BB%85%E5%81%9C%E6%AD%A2rabbit_mgmt_db%E7%9A%84%E6%96%B9%E5%BC%8F_2.png "停止 rabbit_mgmt_db 进程后")

对比发现，只有 `rabbit_mgmt_db` 进程 pid 发生了变化；

# 第二种方式

有了上面的解释，第二种方式的代价就很容易进行对比了；

命令中的 `application:stop(rabbitmq_management)` 其实是将 management 插件所创建的整个进程树全部停止掉；再通过 `application:start(rabbitmq_management).` 新建整个进程树；因此，代价大大滴～～

重启 management 插件前的**进程**信息

![重启 management 插件前的进程信息](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E9%87%8D%E5%90%AFmanagement%E6%8F%92%E4%BB%B6%E5%89%8D%E7%9A%84%E8%BF%9B%E7%A8%8B%E4%BF%A1%E6%81%AF.png "重启 management 插件前的进程信息")

重启 management 插件前的**树**信息

![重启 management 插件前的树信息](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E9%87%8D%E5%90%AFmanagement%E6%8F%92%E4%BB%B6%E5%89%8D%E7%9A%84%E6%A0%91%E4%BF%A1%E6%81%AF.png "重启 management 插件前的树信息")


重启 management 插件后的**进程**信息

![重启 management 插件后的进程信息](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E9%87%8D%E5%90%AFmanagement%E6%8F%92%E4%BB%B6%E5%90%8E%E7%9A%84%E8%BF%9B%E7%A8%8B%E4%BF%A1%E6%81%AF.png "重启 management 插件后的进程信息")

重启 management 插件后的**树**信息

![重启 management 插件后的树信息](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E9%87%8D%E5%90%AFmanagement%E6%8F%92%E4%BB%B6%E5%90%8E%E7%9A%84%E6%A0%91%E4%BF%A1%E6%81%AF.png "重启 management 插件后的树信息")


# 结论


方式一：
```shell
$ rabbitmqctl eval 'exit(erlang:whereis(rabbit_mgmt_db), please_terminate).'
```

仅重置 management 插件中的 rabbit_mgmt_db 进程，即维护统计数据库的进程，对 RabbitMQ 的影响较小；但该方式会在 SASL 日志中输出 `SUPERVISOR REPORT` 错误报告信息；

> ⚠️ 在 cluster 环境下，仅执行一次该命令，不会导致 management 插件统计数据库发生节点迁移；但在连续执行 N 次该命令后（达到 supervisor 中指定的子进程规范里规定的次数时），会（概率性？）导致 rabbit_mgmt_db 进程在 cluster 节点上的迁移；

同时会输出如下信息表明重启成功
```shell
=INFO REPORT==== 2-Aug-2016::17:52:11 ===
Statistics database started.
```


方式二：
```shell
$ rabbitmqctl eval 'application:stop(rabbitmq_management), application:start(rabbitmq_management).'
```

会重启 management 插件这个应用，对 RabbitMQ 的影响较大；

从如下日志和源码中可以看到，这种重启行为至少会进行如下操作
- 重建用于记录基于 HTTP API 进行访问时的日志文件（如果设置了 http_log_dir 的话）；
- 重建 management 插件的监听 socket（基于 listener 配置） 
- 重建 management 插件对应的整个 erlang 进程树结构；

此外，还会重建 `rabbit_web_dispatch_sup_15672` 和其下的 N 个子进程；

> ⚠️ 在 cluster 环境下，仅执行一次该命令，就可以导致 management 插件统计数据库发生节点迁移；

```shell
=INFO REPORT==== 2-Aug-2016::16:44:16 ===
opening log file: "/Users/sunfei/workspace/WGET/rabbitmq_server-3.6.1/var/log/rabbitmq/access.log.2016_08_02_08"

=INFO REPORT==== 2-Aug-2016::16:44:16 ===
Management plugin started. Port: 15672

=INFO REPORT==== 2-Aug-2016::16:44:16 ===
Statistics database started.
```