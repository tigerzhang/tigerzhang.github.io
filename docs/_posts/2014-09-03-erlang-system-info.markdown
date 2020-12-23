---
layout: post
title: erlang-system-info
date: '2014-09-03 11:51:41'
---

获取 Erlang 系统信息：

```erlang
SchedId = erlang:system_info(scheduler_id),SchedNum = erlang:system_info(schedulers),ProcCount = erlang:system_info(process_count),ProcLimit = erlang:system_info(process_limit),ProcMemUsed = erlang:memory(processes_used),ProcMemAlloc = erlang:memory(processes),MemTot = erlang:memory(total),io:format(“abormal termination: ”“~n Scheduler id: ~p”“~n Num scheduler: ~p”“~n Process count: ~p”“~n Process limit: ~p”“~n Memory used by erlang processes: ~p”“~n Memory allocated by erlang processes: ~p”“~n The total amount of memory allocated: ~p”,[SchedId, SchedNum, ProcCount, ProcLimit,ProcMemUsed, ProcMemAlloc, MemTot]).
```

原文：http://blog.yufeng.info/archives/tag/erlangsystem_info