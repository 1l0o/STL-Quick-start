---
show: step
version: 1.0
enable_checker: true
---

# 守护进程 Daemon 实现服务器

## 一、实验说明

#### 1.1 内容

本实验将基于Daemon守护进程实现服务器。

#### 1.2 实验环境

- Python3
- Xfce 终端

## 二、实验准备

#### 2.1 现在位置

![2-2.1-1](https://doc.shiyanlou.com/document-uid731737labid7232timestamp1532688104740.png/wm)

#### 2.2 C/S架构

##### 2.2.1 概述

Redis服务是一种C/S模型，提供请求－响应式协议的TCP服务，所以当客户端请求发出，服务端处理并返回结果到客户端。

##### 2.2.2 本实验中的C/S

本实验将完成C/S模型中的Server。

#### 2.3 Daemon 守护进程

##### 2.3.1 概述

系统为了某些功能必须要提供一些服务（不论是系统本身还是网络方面），但服务的提供总是需要进程的运行，实现这个服务的程序我们就称之为daemon。

守护进程(daemon)是一类在后台运行的特殊进程，用于执行特定的系统任务。很多守护进程在系统引导的时候启动，并且一直运行直到系统关闭。另一些只在需要的时候才启动，完成任务后就自动结束。

用户使守护进程独立于所有终端是因为，在守护进程从一个终端启动的情况下，这同一个终端可能被其他的用户使用。例如，用户从一个终端启动守护进程后退出，然后另外一个人也登录到这个终端。用户不希望后者在使用该终端的过程中，接收到守护进程的任何错误信息。同样，由终端键人的任何信号(例如中断信号)也不应该影响先前在该终端启动的任何守护进程的运行。虽然让服务器后台运行很容易(只要shell命令行以&结尾即可)，但用户还应该做些工作，让程序本身能够自动进入后台，且不依赖于任何终端。

>可以认为daemon就是service，两者不必区分的太清楚。因为达成某个service需要daemon在后台支持，没有daemon就没有service。

从守护进程的概念可以看出，系统所运行的每一种服务，都必须运行一个监听某个端口连接所发生的守护进程。例如，redis服务的端口就是`6379`。


##### 2.3.2 使用Daemon

本实验使用守护进程，就是通过它创建一个在后台运行的仿redis服务器。

在此基础上，将其端口设定成原本redis服务器的端口`6379`，并停用掉redis原本的服务。

这样做了以后，我们在终端里使用redis-cli连接的便是我们自己创建的服务器。


## 三、代码实现

因为我们要创建一个完整规范的项目，所以我们一开始就要求规范的文件目录。

进入`Code`文件夹，创立`PyRedis`文件夹，再在其中创建`src`文件夹，在 `src` 文件夹里创建`daemon.py`文件。

### 3.1 实验环境


使用`vim`或`gedit`来打开`daemon.py`文件，准备开始编写代码。

实现`daemon`的方法用很多，可以使用函数实现，也可以通过类实现。在本训练营中，方便大家熟悉面向对象的编程方法，选择使用类来实现`daemon`。

创建`Daemon`类，并导入一些必要的模块:

```python
import sys, os, time, atexit, signal

class Daemon:
    def __init__(self, pidfile):
        self.pidfile = pidfile # pidfile是控制进程的文件
```

```checker
- name: 检查是否存在文件夹 PyRedis/src
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/Code/PyRedis/src/
  error: | 
    我们发现您还没有创建文件夹 /home/shiyanlou/Code/PyRedis/src/
```
### 3.2 实现daemon

创建daemonize守护进程方法:

```python 
    def daemonize(self):
```
>3.2小节接下来的代码，都是在这个方法里面写。
>
>实现方法基于《Advanced Programming in The Unix Environment  Section》中关于守护进程的实现规范

#### 3.2.1 创建子进程，终止父进程

由于守护进程是脱离控制终端的，因此首先创建子进程，终止父进程，使得程序在shell终端里造成一个已经运行完毕的假象。之后所有的工作都在子进程中完成，而用户在shell终端里则可以执行其他的命令，从而使得程序以僵尸进程形式运行，在形式上做到了与控制终端的脱离。

下面直接放上代码:

```python
        try:
            pid = os.fork() #生成子进程
            if pid > 0:
                sys.exit(0)  #父进程退出
        except OSError as err:
            sys.stderr.write('fork #1 failed: {0}\n'.format(err))
            sys.exit(1)
```
代码讲解：为了达到上述的目的，我们调用`fork`创建了子进程，使用`exit`使父进程退出。除此之外，我们使用了异常处理机制来捕获错误。

#### 3.2.2 在子进程中创建新会话

setsid函数用于创建一个新的会话，并担任该会话组的组长。调用setsid有三个作用：让进程摆脱原会话的控制、让进程摆脱原进程组的控制和让进程摆脱原控制终端的控制。

在调用fork函数时，子进程全盘拷贝父进程的会话期(session，是一个或多个进程组的集合)、进程组、控制终端等，虽然父进程退出了，但原先的会话期、进程组、控制终端等并没有改变，因此，那还不是真正意义上使两者独立开来。setsid函数能够使进程完全独立出来，从而脱离所有其他进程的控制。

使用fork创建的子进程也继承了父进程的当前工作目录。由于在进程运行过程中，当前目录所在的文件系统不能卸载，因此，把当前工作目录换成其他的路径，如“/”或“/tmp”等。改变工作目录的常见函数是chdir。

文件创建掩码是指屏蔽掉文件创建时的对应位。由于使用fork函数新建的子进程继承了父进程的文件创建掩码，这就给该子进程使用文件带来了诸多的麻烦。因此，把文件创建掩码设置为0，可以大大增强该守护进程的灵活性。设置文件创建掩码的函数是umask，通常的使用方法为umask(0)。

实现代码很简单：

```python
		os.chdir('/') #修改工作目录   
		os.setsid()  #设置新的会话连接  
		os.umask(0)  #重新设置文件创建权限  
```



#### 3.2.3 获得真正的守护进程

虽然当前关闭了和终端的联系，但是后期可能会误操作打开了终端。

第二次fork将确保进程重新打开控制终端，并且产生子-孙进程，而子进程退出后孙进程将成为真正的守护进程。

代码与之前的类似：

```python
        try:
            pid = os.fork()  #第二次fork，禁止进程打开终端
            if pid > 0:
                # exit from second parent
                sys.exit(0) #第二个父进程退出
        except OSError as err:
            sys.stderr.write('fork #2 failed: {0}\n'.format(err))
            sys.exit(1)
```

#### 3.2.4 关闭文件描述符

用fork新建的子进程会从父进程那里继承一些已经打开了的文件。这些被打开的文件可能永远不会被守护进程读或写，但它们一样消耗系统资源，可能导致所在的文件系统无法卸载。所以我们要关闭文件描述符。


```python
		# 进程已经是守护进程了，重定向标准文件描述符
        sys.stdout.flush()
        sys.stderr.flush()
        si = open(os.devnull, 'r')
        so = open(os.devnull, 'a+')
        se = open(os.devnull, 'a+')
		
		#dup2函数原子化关闭和复制文件描述符
        os.dup2(si.fileno(), sys.stdin.fileno())         
        os.dup2(so.fileno(), sys.stdout.fileno())
        os.dup2(se.fileno(), sys.stderr.fileno())
```

#### 3.2.5 注册退出函数

首先，在`daemonize`方法外，写一个删除函数,并在`daemonize`注册它：

```
# 注意是在daemonize方法外写
    def delpid(self): # 删除pid
        os.remove(self.pidfile)
```

接着，**回到`daemonize`方法内，**

```python
 		# 注册删除pid函数
        atexit.register(self.delpid)
        # 根据文件pid判断是否存在进程
        pid = str(os.getpid())
        # 写入pidfile
        with open(self.pidfile, 'w+') as f:
            f.write(pid + '\n')
```

`atexit.register()`，通过此函数，注册了程序退出时的回调函数，确保了关闭守护进程时不会使其成为幽灵进程。

### 3.3 启动/关闭/重启 进程

对于守护进程，我们当然还需要实现启动/关闭/重启它。

#### 3.3.1 启动进程

启动进程的思路是先检查pid文件是否存在，若存在，则代表守护进程正在运行，并提示错误。若不存在，则启动守护进程。

```python
    def start(self):
        '''
        启动守护进程
        '''
        # 首先，检查pid文件是否存在以探测守护进程守护已运行
        try:
            with open(self.pidfile, 'r') as pf:
                pid = int(pf.read().strip()) 
        except IOError:
            pid = None

        if pid: # pid文件存在，代表守护进程已经在运行
            message = "pidfile {0} already exist. " + \
                      "Daemon already running?\n"
            sys.stderr.write(message.format(self.pidfile))
            sys.exit(1)

        # 守护进程没有运行，启动守护进程
        self.daemonize()
        self.run()
```

#### 3.3.2 关闭进程

关闭进程的思路是先检查pid文件是否存在，若不存在，则提示无守护进程在运行。若存在，则用`kill`方法来关闭进程。

```python
    def stop(self):
        '''
        关闭守护进程
        '''
        # 从pid文件中获取pid
        try:
            with open(self.pidfile, 'r') as pf:
                pid = int(pf.read().strip())
        except IOError:
            pid = None

        if not pid:  # 守护进程没有运行
            message = "pidfile {0} does not exist. " + \
                      "Daemon not running?\n"
            sys.stderr.write(message.format(self.pidfile))
            return  # not an error in a restart

        # 使用kill来关闭进程
        try:
            while 1:
                os.kill(pid, signal.SIGTERM) # 信号
                time.sleep(0.1)
        except OSError as err:
            e = str(err.args)
            if e.find("No such process") > 0:
                if os.path.exists(self.pidfile):
                    os.remove(self.pidfile)
            else:
                print(str(err.args))
                sys.exit(1)
```

#### 3.3.3 重启进程

重启就很简单啦，先关闭再启动就搞定。

```python
    def restart(self):
        '''
        重启守护进程
        '''
        self.stop() # 关闭守护进程
        self.start() # 启动守护进程
```

### 3.4 运行方法

给守护进程一个运行`run`方法，这个方法我们不用着急编写。

```python
    def run(self):
        '''
		当你使用子类来继承Daemon，你可以重构这个方法，
        他可以通过start()和restart()方法来成为守护进程。
        '''
```

现在，我们的守护进程类就编写完成，而且具有非常不错的扩展性。不只是本训练营的`PyRedis`可以使用，你可以通过基础守护进程类重写run方法，来搭建你想搭建的任何服务！

```checker
- name: 检查是否存在文件 daemon.py
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/Code/PyRedis/src/daemon.py
  error: | 
    我们发现您还没有创建文件 /home/shiyanlou/Code/PyRedis/src/daemon.py
```



## 四、测试

现在我们新建一个`testDaemon.py`来测试我们创建的守护进程。


#### 4.1 重写run方法
现在我们在`src`目录下创建`testDaemon.py`文件

写入我们的测试类，继承自`daemon.py`中的Daemon，并构造新的run()方法：


```python
import os, sys, time

from daemon import Daemon

PID_FILE = '/tmp/testDaemon.pid'

class TestDaemon(Daemon):
    '''
    测试类
    测试守护进程是否可以正常运行
    '''
    def run(self):
        while True:
            sys.stdout.write('%s:hello world\n' % (time.ctime(),))
            sys.stdout.flush()
            time.sleep(1)
```

除此之外，在`daemon.py`中更改`stdout`的保存路径，这样方便我们查看实验结果：


```python
so = open('/tmp/test.txt', 'a+')
```



#### 4.2 构造main函数

现在来写main函数，这个在这个函数里，我们主要是对类进行实例化，还有实现对命令行不同参数的读取来实现start/stop/restart操作。

```python
if __name__ == '__main__':
    '''
    在终端中输入 python testDaemon.py start/stop/restart 来控制守护进程。
    '''
    daemon = TestDaemon(PID_FILE) # 实例化对象

    if len(sys.argv) == 2:
        if 'start' == sys.argv[1]:
            print("Starting daemon...")
            daemon.start()
        elif 'stop' == sys.argv[1]:
            print("Stopping daemon...")
            daemon.stop()
        elif 'restart' == sys.argv[1]:
            print("Restarting daemon...")
            daemon.restart()
        else:
            print("Unknown command")
            sys.exit(2)
        sys.exit(0)
    else:
        print("usage: {0} start|stop|restart".format(sys.argv[0]))
        sys.exit(2)
```

```checker
- name: 检查是否存在文件 testDaemon.py
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/Code/PyRedis/src/testDaemon.py
  error: | 
    我们发现您还没有创建文件 /home/shiyanlou/Code/PyRedis/src/test_daemon.py
```

#### 4.3 测试


在目录中打开终端，输入以下命令：

![2-4.3-1](https://doc.shiyanlou.com/document-uid731737labid7232timestamp1532688113509.png/wm)

输入`cat /tmp/test.txt`来查看输出的结果： 

![2-4.3-2](https://doc.shiyanlou.com/document-uid731737labid7232timestamp1532688119958.png/wm)

做到现在，我们完成了`PyRedis`的服务器，接下来我们将做的是将服务器的功能一一实现。




## 五、完整代码

本实验的完整代码放在Github上，项目地址：[Exp2](https://github.com/KeepitReal555/PyRedis/tree/Exp2)

你也可以直接通过终端下载：

```
$ git clone -b Exp2 https://github.com/KeepitReal555/PyRedis.git
```

## 六、总结

本实验通过守护进程实现了PyRedis的服务器。在本实验中，我们了解了守护进程的概念，并学会用Python3来实现守护进程。

本节内容对初学者来说具有很大的难度，希望同学们认真学习并消化理解。

#### 延伸阅读

[守护进程-百度百科](https://baike.baidu.com/item/守护进程/966835?fr=aladdin)





