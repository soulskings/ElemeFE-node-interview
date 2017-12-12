# child_process (子进程)

---

### 1. 基本概念

#### **1.1 linux进程**
对于操作系统而言,进程是分配资源的最小单位.

在windows系统下我们通过任务管理器或者服务看到的都是广义上的进程.
![](http://omrbgpqyl.bkt.clouddn.com/17-11-24/52543983.jpg)

在类Unix系统下我们通过`ps`进程查看进程.
![](http://omrbgpqyl.bkt.clouddn.com/17-11-24/66187398.jpg)

具体参数意义如下:
![](http://omrbgpqyl.bkt.clouddn.com/17-11-24/38089481.jpg)



#### **1.2 进程控制块(PCB)**

**PCB(process control block)**，进程控制块，其实是一个数据结构描述，通过它人们才能对系统的进程进行管理.

一般情况下，PCB中包含以下内容：
1. 进程标识符（内部，外部）
2. 处理机的信息（通用寄存器，指令计数器，PSW，用户的栈指针）
3. 进程调度信息（进程状态，进程的优先级，进程调度所需的其它信息，事件）
4. 进程控制信息（程序的数据的地址，资源清单，进程同步和通信机制，链接指针）

Linux的进程控制块为一个由结构`task_struct`所定义的数据结构,在创建一个新进程时,系统在内存中申请一个空的`task_struct`区，并输入相关信息(**信息很多,大概有:进程标识符、进程状态、进程优先级/调度策略等等...**)

新建进程流程如下:
![](http://omrbgpqyl.bkt.clouddn.com/17-11-24/31957772.jpg)


#### **1.3 僵尸进程与孤儿进程**

**僵尸进程**:Linux的进程都是由父进程创建的,当子进程死亡后,子进程一方面会释放占有的资源,一方面会保留一少部分信息交由父进程接管,但是如果父进程不接管已死亡的子进程留下的信息,那么子进程的pid就不会销毁,也就是说一个孩子虽然死了,但是父亲没有拿着户口本、身份证、死亡证明去派出所注销，这个孩子在户籍上就一直是活着的，成了所谓的僵尸。
如果一个父进程大量子进程死亡都没有接收死亡子进程信息的话，大量pid会被占用，但是pid总数有限，最终导致pid不足。

**孤儿进程**： 顾名思义，指的是父进程死亡后，由其创建的所有子进程全部成为了没有父进程的孤儿进程,这个时候孤儿进程会被`init`进程接管,你也可以理解为孩子失去双亲后被国家收养了,那个`init`进程就是国家,是创建所有进程的初始进程.

#### **1.4 IPC进程间通信**

**进程间通信(IPC，Inter-Process Communication)**，指至少两个进程或线程间传送数据或信号的一些技术或方法.

IPC是进程管理永远绕不开的话题,常见的通信技术如下:
![](http://omrbgpqyl.bkt.clouddn.com/17-11-25/46822480.jpg)

**1.4.1 管道(pipe)**

管道实际是用于进程间通信的一段**共享内存**，创建管道的进程称为管道服务器，连接到一个管道的进程为管道客户机。
*一个进程在向管道写入数据后，另一进程就可以从管道的另一端将其读取出来*。

**管道的特点：**
1. 管道是半双工的，数据只能向一个方向流动；需要双方通信时，需要建立起两个管道；
2. 只能用于父子进程或者兄弟进程之间（具有亲缘关系的进程）。
3. 单独构成一种独立的文件系统：管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它不属于某种文件系统，而是自立门户，单独构成一种文件系统，并且只存在与内存中。
4. 数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的末尾，并且每次都是从缓冲区的头部读出数据。

**管道的实现机制：**
管道是由内核管理的一个**缓冲区**，管道的一端连接一个进程的输出。这个进程会向管道中放入信息。同时一头连着一个进程的读取,当缓冲区没有信息时读取进程进入等待状态,当缓冲区信息已满时,输入进程会阻塞等待缓冲区释放,读取的顺序原则是**先进先出**.

**1.4.2 命名管道（FIFO）**
命名管道是一种特殊类型的文件，它在系统中以文件形式存在,这样克服了管道的弊端，**他可以允许没有亲缘关系的进程间通信**。

**1.4.3 消息队列（Message queues）**
MQ(Message queues) 用于在进程间传递数据。MQ 采用**链表** 来实现消息队列，该链表是由系统内核维护，系统中可能有很多的 MQ，每个 MQ 用消息队列描述符（消息队列 ID：qid）来区分，qid 是唯一的，用来区分不同的 MQ。在进行进程间通信时，一个进程将消息加到 MQ 尾端，另一个进程从消息队列中取消息,不一定以先进先出来取消息，也可以按照消息类型字段取消息，这样就实现了进程间的通信。如下 MQ 的模型：

![](http://omrbgpqyl.bkt.clouddn.com/17-11-25/36578196.jpg)

**与管道的比较**

1. 消息队列也可以独立于发送和接收进程而存在，而管道随着双方进程的死亡而消失。
2. 同时通过发送消息还可以避免命名管道的同步和阻塞问题，不需要由进程自己来提供同步方法。
3. 接收程序可以通过消息类型有选择地接收数据，而不是像命名管道中那样，只能默认地接收。

**1.4.4 共享内存（Share Memory）**
共享内存是在多个进程之间共享**内存区域**的一种进程间的通信方式，由IPC为进程创建的一个特殊地址范围，此时其他进程可以将同一段共享内存连接到自己的地址空间中。所有进程都可以访问共享内存中的地址，就好像它们是malloc分配的一样。如果一个进程向共享内存中写入了数据，所做的改动将立刻被其他进程看到。
共享内存是IPC**最快**的方式，因为共享内存方式的通信没有中间过程，而管道、消息队列等方式则是需要将数据通过中间机制进行转换。共享内存方式直接将某段内存段进行映射，多个进程间的共享内存是同一块的物理空间，仅仅映射到各进程的地址不同而已，因此不需要进行复制，可以直接使用此段空间。

![](http://omrbgpqyl.bkt.clouddn.com/17-11-25/22740017.jpg)

**1.4.5 UNIX域套接字（ Unix domain socket ）**
传统的套接字（Socket）是基于TCP/IP协议栈的，需要指定IP地址,从而用来进行不同机器的通信,但是对于同一台机器的通信来说这一套并不高效.
因此**Unix域套接字**，专门用来处理同一台机器进程通信问题。
![](http://omrbgpqyl.bkt.clouddn.com/17-11-25/63260591.jpg)


### 2. node的多进程

#### **2.1 子进程**

`child_process`模块可以进行子进程的创建.

**其中有这几个重要方法:**
**1.`spawn`:** spawn方法可以启动一个子进程执行命令,比如通过`shell`压缩文件、转换格式等等一系列操作。

```javascript
const { spawn } = require('child_process');
const ls = spawn('ls', ['-l', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.log(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```
上述代码我们读取了目标目录的文件列表,我们可以通过监听事件来对结果进行进一步操作.

**2.`exec`:** exec方法可以启动一个子进程执行命令,并缓冲产生的数据，当子进程完成后回调函数可以将其调用.

与`spawn`方法相比`exec`多了回调函数,更方便我们进一步操作.
```javascript
const { exec } = require('child_process');

const ls = exec('ls -l', (error, stdout, stderr) => {
  if (error) {
    console.error(error.stack);
    console.log('Error code: ' + error.code);
  }
  console.log('Child Process STDOUT: ' + stdout);
});
```


**3.`execFile`:** execFile方法可以执行一个外部应用，与`exec`类似，除了不衍生一个 shell。 而是，指定的可执行的 file 被直接衍生为一个新进程，这使得它比 `child_process.exec()` 更高效。。

```javascript
const { execFile } = require('child_process');
const child = execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) {
    throw error;
  }
  console.log(stdout);
});
```

**4.`fork`:** `fork`方法直接创建一个子进程，执行Node脚本，`fork('./child.js')` 相当于 `spawn('node', ['./child.js'])` 。与`spawn`方法不同的是，`fork`会在父进程与子进程之间，建立一个通信管道，用于进程之间的通信。

```javascript
const n = child_process.fork('./worker.js');
n.on('message', function(m) {
  console.log('PARENT got message:', m);
});
n.send({ hello: 'world' });
```

#### **2.2 进程间通信**

**2.2.1 node通信原理**
`fork`创建进程之后，会在父子进程之间建立IPC通信,并过message与send()等方法进行通信,这通信方法基于由node的管道技术实现,而这个管道技术不同于上文提到的操作系统的*管道*,node的管道技术由libuv提供,而在不同操作系统下具体实现不同:Windows下由命名管道实现,*nix下由UNIX域套接字实现.