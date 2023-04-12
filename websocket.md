# Websocket

### 1、sending的含义：

每个out task：

dispatch()的时候会在锁里抢sending。

抢到则设置自己的ready flag = true
抢不到则设置ready flag = false，并且往后放一个waiter_task以及一个out_task，其中waiter_task挂在channel的condition上。如下：

[out_task]->[waiter_task]->[out_task]

然后在done()的时候，如果发现out task自己的那个ready == false，表示这次没抢到sending，等待下次waiter完事之后再dispatch我。

所以理论上，有可能要这样来很多次😂

### 2、所以以上的方式，对与其他还要抢sending的人来说，有什么参考价值？

先回答：谁还要抢sending？ 答案是channel自己。它要被handle_terminate()或者client->deinit()的时候，都要二次dispatch()这个channel。channel内部由established这个flag来看我被dispatch()时要干嘛。所以，很显然这也是个重复多次的过程。


### 3、多层channel的派生及成员


`WFComplexChannel<MSG>`

有mutex、condition、sending

`WFChannel<MSG>`

有callback、process

`ChanRequest`

有state、error、established（1:yes; 0:no; ）、object、target


问题：他们分别负责什么层次？每个成员决定了什么流程？


### 4、如果client.deinit()同时被handle_terminate()，是否有channel先被delete的风险？
  
不会。由于delete是在deinit的series的callback中，我们下面分析deinit() 先抢到sending、handle_terminate() 在等的情况。

~~~
[ client.deinit() ]                       [ handle_terminate() ]          sending     established
            ｜                                     | 
            v                                      |                       false         1
抢到sending=true                                    v                       true
[ channel->dispatch()  ]                      [wait_task]
WFComplexChannel的话
如果this->object != NULL
调用WFChannel::dispatch()

ChannRequest::dispatch()
if (established == 1)
this->scheduler->shutdown(this);
它会从mpoller里删掉、
减entry的引用计数、
entry->target->release(0)、
调用上层实现的handle()、
最后release_conn(entry)。

这个handle()属于ChanRequest
设置 established=0                                 |                                        0
并调用subtask_done()
也就是进入了done()

WFComplexChannel::done()
如果established == 0
锁内sending=0，signal();                                                    false

WFCondition::signal()
拿出一个task，调用task->send()


WFMailboxTask::send()                             ｜
if (--value==0)
subtask_done()
done()
直接调用callback ———————————————————————>  wait_callback()
                                          如果established==0
signal()完后 <—————————————————————————   直接退出
解锁
调用WFChannel的callback
delete this。
~~~
