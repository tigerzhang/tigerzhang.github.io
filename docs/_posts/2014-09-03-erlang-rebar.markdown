---
layout: post
title: Erlang 工具：rebar
date: '2014-09-03 11:52:28'
---

#创建工程
```
$ ./rebar create-app appid=dummy_proj
==> dummy_proj (create-app)
Writing src/dummy_proj.app.src
Writing src/dummy_proj_app.erl
Writing src/dummy_proj_sup.erl
```


#编译
编辑 rebar.conf
```
{require_min_otp_vsn, “R15”}.
{erl_opts, [{i, “include”},
{src_dirs, [“src”, “test”]}]}.
{sub_dirs, [“rel”]}.
{lib_dirs,[“lib”, “plugins”]}.
{deps_dir, [“deps”]}.
{erl_opts, [debug_info, fail_on_warning]}.
```
编译：
```
$ ./rebar compile
==> dummy_proj (compile)
Compiled src/dummy_proj_sup.erl
Compiled src/dummy_proj.erl
Compiled src/dummy_proj_server.erl
==> rel (compile)
==> dummy_proj (compile)
```

#运行
```
$ erl -pa ebin/ -pa deps/*/ebin/ -s dummy_proj
```

#发布
用 Erlang 的发布工具，可以把整个 Erlang 的运行环境全部放到一个目录下。
```
$ mkdir rel
$ cd rel/
$ ../rebar create-node nodeid=dummynode
==> rel (create-node)
Writing reltool.config
Writing files/erl
Writing files/nodetool
Writing files/dummynode
Writing files/app.config
Writing files/vm.args
```

#配置文件
如果存在 files/sys.config，rebar 会优先用作应用的配置文件，如果不存在，使用 files/app.config。我们一般习惯使用 app.config。

```
$ mv files/sys.config files/app.config
```

删除 files/reltool.config 中：
```
{copy, “files/sys.config”, “releases/\{\{rel_vsn\}\}/sys.config”},
```
添加：
```
{template, “files/app.config”, “etc/app.config”}
$ ./rebar generate==> rel (generate)
$ cd rel/dummynode
```

>新版的 rebar 会优先使用 sys.config，我们也会逐渐迁移到 sys.config，现在的原则是不同时用sys.config, app.config
>如果使用 sys.config，则不要删除 sys.config。
>_注意_：当 sys.config 存在时，app.config 不能生效。

#用 rebar 生成发布文件
修改 rel/reltool.config, 为依赖的 app 指定路径，增加 ../deps 到 lib_dirs：
```
{lib_dirs, ["../deps"]},
```

修改 rel/reltool.config，为 dummynode 这个应用指定路径：
```
{app, dummynode, [{mod_cond, app}, {incl_cond, include}, {lib_dir, ".."}]}
```
lib_dir 是包含 dummynode 应用的 ebin 目录的路径。这里 .. 代表 rel 目录的上层目录。

＃ 运行
以 console 方式运行

`$ ./bin/dummynode console`

以 daemon 方式运行

`$ ./bin/dummynode start`

连接到已经运行的 daemon

`$ ./bin/dummynode attach`

Ctrl-D 退出