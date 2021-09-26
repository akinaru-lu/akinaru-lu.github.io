---
layout: post
title: unicorn:restart踩坑记
date: 2021-09-25 23:36:21 +0900
categories: tech
tags: ruby rails
---

> Unicorn是Ruby生态圈的Web服务器。最近在工作上使用Unicorn的时候踩了一些坑，在此梳理一下。

目前公司里使用Capistrano来进行Ruby on Rails的部署，但是在部署之后遇到了没有读取新的环境变量的问题。然后谷歌了一圈，了解了Unicorn的进程模型和信号处理，Dotenv库是如何处理环境变量的，等等。最后的解决方法很简单，改一行代码就搞定了，但是这个过程花了不少时间。

### 多进程模型
单master-多worker模式，由master加载源代码，各个worker fork自master，所以是master的一份复制。实际的负载也是由worker来处理的。而worker进程间的负载均衡则交给操作系统去做[<sup>1</sup>](#refer1)。

```
I, [#65753]  INFO -- : worker=0 ready
I, [#65748]  INFO -- : master process ready
I, [#65754]  INFO -- : worker=1 ready
I, [#65755]  INFO -- : worker=2 ready
I, [#65756]  INFO -- : worker=3 ready
```

上述日志说unicorn启动后pid为65748的master进程和pid为65753, 65754, 65755, 65756的四个worker进程被创建了。然后在逻辑代码里打印`Process.pid`，可以看到打印的都是worker的pid，顺序随机。

### Unicorn优雅部署
发送USR2信号给旧master，新master启动，然后从新master fork出新worker。到此还不算完，因为旧的进程还没停止，所以需要发送WINCH给旧的master来退出旧的worker，最后再发送QUIT给旧master，这样旧master和旧worker才全部退出，到此才算部署成功[<sup>2</sup>](#refer2)。

###### 进程派生关系
```
unicorn master (old)
\_ unicorn worker[0]
\_ unicorn worker[1]
\_ unicorn worker[2]
\_ unicorn worker[3]
\_ unicorn master (new)
   \_ unicorn worker[0]
   \_ unicorn worker[1]
   \_ unicorn worker[2]
   \_ unicorn worker[3]
```

###### 日志
```
I, [#65753]  INFO -- : worker=0 ready
I, [#65748]  INFO -- : master process ready
I, [#65754]  INFO -- : worker=1 ready
I, [#65755]  INFO -- : worker=2 ready
I, [#65756]  INFO -- : worker=3 ready
--- # kill -s USR2 65748
I, [#65821]  INFO -- : worker=0 ready
I, [#65807]  INFO -- : master process ready
I, [#65822]  INFO -- : worker=1 ready
I, [#65823]  INFO -- : worker=2 ready
I, [#65824]  INFO -- : worker=3 ready
--- # kill -s WINCH 65748
I, [#65748]  INFO -- : gracefully stopping all workers
I, [#65748]  INFO -- : reaped #<Process::Status: pid 65755 exit 0> worker=2
I, [#65748]  INFO -- : reaped #<Process::Status: pid 65754 exit 0> worker=1
I, [#65748]  INFO -- : reaped #<Process::Status: pid 65753 exit 0> worker=0
I, [#65748]  INFO -- : reaped #<Process::Status: pid 65756 exit 0> worker=3
--- # kill -s QUIT 65748
I, [#65748]  INFO -- : master complete
```

我一开始在本地测试的时候由于只给old master发送了USR2信号，所以刷新页面后有时会看到旧版本的内容。

不过Capistrano的做法是在`unicorn.rb`的after_fork hook里添加如下代码[<sup>3</sup>](#refer3)。

```rb
# unicorn.rb

before_fork do |server, worker|
  old_pid = "#{server.config[:pid]}.oldbin"
  if File.exist?(old_pid) && old_pid != server.pid
    begin
      sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH => e
      Rails.logger.error e
    end
  end
end
```

在新master fork worker之前会触发这个hook。这样就是发送`TTOU`给旧master，一个一个地减少旧worker的数量，只剩最后最后一个的时候发送`QUIT`把旧master和worker都退出了。不一次性退出所有worker的目的是为了更平滑地部署。上述过程结束后通过`ps`命令检查新master的pid可以发现其父进程pid变为1了。

值得注意的是，因为新的master是fork自旧的master的，所以环境变量也会继承过来。所以在新master启动后需要覆盖更新原有的环境变量以确保加载到最新的值。只不过那些用不到被删掉的环境变量还会继续留在进程里无法prune，有洁癖的话只能冷启动unicorn，完全停止再重新启动了。


###### 覆盖更新环境变量
```rb
# application.rb

# 不要用Dotenv.load，因为这不覆盖已存在的ENV
Dotenv.overload(*env_files)
```

### 感想
虽然这段时间踩了unicorn不少的坑，但我们最终是要部署到K8S上去的，优雅部署的工作都交给K8S去做了，就没太多unicorn的事了。果然这一个人的命运啊。。。

---

### 参考

<div id="refer1"></div>
- [1] [Configuring Puma, Unicorn and Passenger for Maximum Efficiency](https://www.speedshop.co/2017/10/12/appserver.html)

<div id="refer2"></div>
- [2] [unicorn/SIGNALS](https://github.com/defunkt/unicorn/blob/master/SIGNALS)

<div id="refer3"></div>
- [3] [wiki/rails3/unicorn](https://tachesimazzoca.github.io/wiki/rails3/unicorn.html)
