# RabbitMQ内存监控

RabbitMQ 监控用了多少，可用多少，能过算法计算出可以存多少数据在内存，要落盘多少到磁盘。


### Memory Monitor流程

![image](../images/memory_monitor_1.png)


#### 1. RabbitMQ在启动的时候会启动一个  rabbit_memory_monitor 进程


```
%% rabbit_memory_monitor
-rabbit_boot_step({rabbit_memory_monitor,
                   [{description, "memory monitor"},
                    {mfa,         {rabbit_sup, start_restartable_child,
                                   [rabbit_memory_monitor]}},
                    {requires,    rabbit_alarm},
                    {enables,     core_initialized}]}).
```


rabbit_memory_monitor 是保存所有队列进程的 内存可持续时间及条数等信息。

每个队列进程主动上报内存的一些情况等信息到  rabbit_memory_monitor  进程。

并且在monitor进程启动初始化的时候，会进行注册一个定时器，每2500ms 进行检次一下内存等情况

代码如下：


```
{ok, TRef} = timer:send_interval(?DEFAULT_UPDATE_INTERVAL, update),
```


每个队列进程在启动初始的时候，会往 rabbit_memory_monitor 进程发送一条注册信息

init_it2 方法里


```
ok = rabbit_memory_monitor:register(
	   self(), {rabbit_amqqueue,
				set_ram_duration_target, [self()]}),
```


当然在进程被关闭的时候，也会往 rabbit_memory_monitor 进程发送一注销的信息
terminate_shutdown 方法里


```
rabbit_memory_monitor:deregister(self()),
```


#### 2. 队列进程数据上报

以下2种情况队列进程会数据上报

###### A. 队列接收新消息，ACk等操作的时候

在队列接收到新消息，ack，推送消息给消费者的时候都会调用 noreply 或者 reply 方法。在这2个方法里，每次调用都会执行一个定时器  ，5秒后往当前进程发送指令 update_ram_duration。

让当前队列进程去获取一次当前队列内存使用情况上报给Monitor进程

###### B. 队列进程进入休眠之前


##### 队列进程触发 update_ram_duration 指令逻辑

```
handle_info(update_ram_duration, State = #q{backing_queue = BQ,
											backing_queue_state = BQS}) ->
	{RamDuration, BQS1} = BQ:ram_duration(BQS),
	DesiredDuration =
		rabbit_memory_monitor:report_ram_duration(self(), RamDuration),
	BQS2 = BQ:set_ram_duration_target(DesiredDuration, BQS1),
	%% Don't call noreply/1, we don't want to set timers
	{State1, Timeout} = next_state(State#q{rate_timer_ref      = undefined,
										   backing_queue_state = BQS2}),
	{noreply, State1, Timeout};


RamDuration 是根据每个当前队列的消息量及速率等 信息算出一个 平均持续时间
ram_duration(State) ->
	State1 = #vqstate { rates = #rates { in      = AvgIngressRate,
										 out     = AvgEgressRate,
										 ack_in  = AvgAckIngressRate,
										 ack_out = AvgAckEgressRate },
						ram_msg_count      = RamMsgCount,
						ram_msg_count_prev = RamMsgCountPrev,
						ram_pending_ack    = RPA,
						qi_pending_ack     = QPA,
						ram_ack_count_prev = RamAckCountPrev } =
						  update_rates(State),
	
	%% 获得内存中等待ack的消息数量
	RamAckCount = gb_trees:size(RPA) + gb_trees:size(QPA),
	
	Duration = %% (msgs+acks) / ((msgs+acks)/sec) == sec
		case lists:all(fun (X) -> X < 0.01 end,
					   [AvgEgressRate, AvgIngressRate,
						AvgAckEgressRate, AvgAckIngressRate]) of
			true  -> infinity;
			%% (内存中的消息数量 + 内存中等待ack的消息数量) / (4 * (消息进入速率 + 消息出的速率 + 消息进入ack速率 + 消息出ack速率)) = 平均的持续时间
			false -> (RamMsgCountPrev + RamMsgCount +
						  RamAckCount + RamAckCountPrev) /
						 (4 * (AvgEgressRate + AvgIngressRate +
								   AvgAckEgressRate + AvgAckIngressRate))
		end,
	
	{Duration, State1}.
```


==队列内存平均持续时间 = (内存中的消息数量 + 内存中等待ack的消息数量) / (4 * (消息进入速率 + 消息出的速率 + 消息进入ack速率 + 消息出ack速率))==

速率请参考速率算法章

#### 3. Monitor进程统计所有队列的 平均持续时间

rabbit_memory_monitor 进程接收到 队列进程发送过来 队列的平均持续时间。进行累加或者相减进行统计等信息。

逻辑如下：


```
handle_call({report_ram_duration, Pid, QueueDuration}, From,
			State = #state { queue_duration_sum = Sum,
							 queue_duration_count = Count,
							 queue_durations = Durations,
							 desired_duration = SendDuration }) ->
	%% 查找出带队列进程的信息
	[Proc = #process { reported = PrevQueueDuration }] =
		ets:lookup(Durations, Pid),
	
	gen_server2:reply(From, SendDuration),
	
	{Sum1, Count1} =
		case {PrevQueueDuration, QueueDuration} of
			{infinity, infinity} -> {Sum, Count};
			{infinity, _}        -> {Sum + QueueDuration,    Count + 1};
			{_, infinity}        -> {Sum - PrevQueueDuration, Count - 1};
			{_, _}               -> {Sum - PrevQueueDuration + QueueDuration,
									 Count}
		end,
	%% 将队列对应的信息更新后插入ETS表
	true = ets:insert(Durations, Proc #process { reported = QueueDuration,
												 sent = SendDuration }),
	{noreply, State #state { queue_duration_sum = zero_clamp(Sum1),
							 queue_duration_count = Count1 }};
```


###### 1). 假如当前队列，以前没上报过数据并且，当前上报的数据都是为空，则不进行处理

###### 2). 第一次上报数据，则进程统计累加，并且 Count+1

###### 3). 假如以前上报过并且，这一次上报数据为空，则把上一次上报的数据给减掉，并且Count-1

###### 4). 假如以前上报有数据，并且这一次也有上报数据，则 减去上一次上报的 并加上这一次上报的,Count不变

每一次上报完，都要保存一下当前上报的数据为最后上报的数据，用于下一次上报统计使用。Count 在这里只是一个统计上报有效数据的队列总数使用。

#### 4. monitor进程检测内存等使用情况 并将队列平均内存占比下发给各队列

monitor进程注册 的定时器，每隔2500ms会往当前进程发送一条检测的信息。这个是在monitor进程启动时候做的


```
handle_info(update, State) ->
	{noreply, internal_update(State)};
```


internal_update 方法里会获取当前 erlang 还虚拟机使用的内存占比，并且和 rabbitmq.config配置文件中的 vm_memory_high_watermark_paging_ratio 参数进行对比

###### 1). < vm_memory_high_watermark_paging_ratio 不做任何处理


###### 2). >  vm_memory_high_watermark_paging_ratio


平均每个队列内存持续时间 =  内存持续时间总和 / 队列数 / 虚拟机内存使用率

平均每个队列内存持续时间，在RabbitMQ里是一个概念，并不是完全是最大内存使用多少的概念，而是将内存，条数等一些信息转换出来的一个数字。

并且往所有队列进程中发送消息 队列平均最大可持续时间。


```
inform_queues(ShouldInform, DesiredDurationAvg, Durations) ->
	true =
		ets:foldl(
		  fun (Proc = #process{reported = QueueDuration,
							   sent     = PrevSendDuration,
							   callback = {M, F, A}}, true) ->
				   case ShouldInform(PrevSendDuration, DesiredDurationAvg)
							andalso ShouldInform(QueueDuration, DesiredDurationAvg) of
					   true  -> ok = erlang:apply(
									   M, F, A ++ [DesiredDurationAvg]),
								ets:insert(
								  Durations,
								  Proc#process{sent = DesiredDurationAvg});
					   false -> true
				   end
		  end, true, Durations).
```


 
其实在这个监控过程中，还有考滤到磁盘是否报警等情况，假如磁盘都告警了，是不会往所有队列发送内存占比情况的

monitor进程下发队列平均最大可持续时间的时候，是通过 队列进程在注册到监控进程的时候指定下发的方法名 set_ram_duration_target 进行下发的。

##### 5. 队列进程获取到Monitor进程下发的最大可持续时间

队列进程在接收到Monitor进程下发最大可持续时间，则修改数 target_ram_count 及触发一次将一部分数据落盘的操作

**target_ram_count**： 在队列进程中是记录当前队列进程最大可用于存放多少条数据于内存中。这个值会在 set_ram_duration_target  里进行修改。

==target_ram_count =  in,out,ack_in,ack_out 几个速率之和 * 队列平均最大可持续时间==


```
set_ram_duration_target(
  DurationTarget, State = #vqstate {
									rates = #rates { in      = AvgIngressRate,
													 out     = AvgEgressRate,
													 ack_in  = AvgAckIngressRate,
													 ack_out = AvgAckEgressRate },
									target_ram_count = TargetRamCount }) ->
	%% 获得所有的速率之和
	Rate =
		AvgEgressRate + AvgIngressRate + AvgAckEgressRate + AvgAckIngressRate,
	%% 计算最新的内存中最多的消息数量
	TargetRamCount1 =
		case DurationTarget of
			infinity  -> infinity;
			_         -> trunc(DurationTarget * Rate) %% msgs = sec * msgs/sec
		end,
	%% 得到最新的内存中最多的消息数
	State1 = State #vqstate { target_ram_count = TargetRamCount1 },
	a(case TargetRamCount1 == infinity orelse
			   (TargetRamCount =/= infinity andalso
					TargetRamCount1 >= TargetRamCount) of
		  true  -> State1;
		  %% 如果最新的内存中的消息数量小于老的内存中的消息数量，则立刻进行减少内存中的消息的操作
		  false -> reduce_memory_use(State1)
	  end).
```



在修改完 target_ram_count 之后，会进行一次判断当前队列内存消息条数 是否大于 target_ram_count ，进行一次刷盘操作。

其实set_ram_duration_target 方法调用不只会由monitor进程触发。

在队列进程休眠这前，上报数据给Monitor上报数据的时候，Monitor进程会读取当前最近一次计算的队列平均可持续时间，并立马返回给队列进程。

这个时候队列进程拿到数据之后，也会再行一次判断是否需要刷盘操作，刷完之后再进入休眠操作。

### 小结

每个rabbitmq节点都有一个全局的monitor进程。

所有队列进程在初始化的时候都会往当前节点的monitor发送一个注册消息。

每个队列进程每隔5秒(也有可能5s里一次性发送N条)的往monitor进程发送 当前队列内存持续时间信息，让monitor进程统计。

monitor进程每隔2500ms 获取当前erlang虚拟机内存在系统层占比，并且和配置文件vm_memory_high_watermark_paging_ratio 参数进行对。

并且往各个队列发送平均占比情况下，让队列进程进行把一些数据持久化到磁盘等操作
