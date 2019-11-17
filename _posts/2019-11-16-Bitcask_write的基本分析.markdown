---
layout:     post
title:      "Bitcask中write的基本分析"
subtitle:   "write K/V"
date:       2019-11-16 15:00:00
author:     "StarryMoon"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - bitcask
    - K/V store
    - write
---

## Bitcask

> bitcask 是一个日志型的基于hash表结构的key-value存储模型

- 日志型数据文件； 所谓的日志型文件，就是append only，所有的写操作只追加到文件的末尾，不修改文件已有的数据。在bitcask模型中，数据文件是日志型只增不减的写入文件，数据文件默认的大小是10M，当文件大小增加到相应的限制时，就会产生一个新的文件，老的文件变成只读模式。在bitcask模型中，任意时间点下，只有一个文件是可写的，即active data file，其他已经达到限制大小的文件，称之为older data file。如下图：
   ![20191116-01](/img/in-post/20191116/20191116-01.jpg)
- 基于hash表的索引；为了提高查找效率，bitcask在内存建立了基于hash表的索引数据结构。hash表的结构如下图所示：
   ![20191116-02](/img/in-post/20191116/20191116-02.jpg)
  
- hint文件

|Namespace类型|系统调用参数|隔离域|
|---|---|---|
|Mount Namespace|CLONE_NEWNS|文件系统|
|UTS Namespac|CLONE_NEWUTS|主机名|
|IPC Namespace|CLONE_NEWIPC|进程间通信|
|PID Namespace|CLONE_NEWPID|进程ID|
|Network Namespace|CLONE_NEWNET|网络栈|
|User Namespace|CLONE_NEWUSER|用户权限|

## Cgroup

> Linux Cgroups提供了对一组进程及将来子进程的资源限制、控制和统计的能力，这些资源包括CPU、内存、存储、网络等。Cgroups中主要有三个组件：

 > * cgroup
   一个cgroup包含一组进程；
 > * subsystem
   一组资源控制的模块；
 > * hierarchy
   树状的组织结构；

基本原则：

 1. 一个subsystem只能附加到一个hierarchy上面；
 2. 一个hierarchy可以附加多个subsystem;
 3. 一个进程可以作为多个cgroup的成员，但是这些cgroup必须在不同的hierarchy中。

### 理解：

  -  cgroup的形式是一个文件系统，挂载相应类型的文件系统之后，就可以构建一个hierarchy;
  -  为了便于理解，可以认为一个subsystem对应一个hierarchy,即cpu、memory等分别是不同hierarchy;
    
  -  *Linux中默认挂载(/sys/fs/cgroup)了cgroups文件系统。在Docker的实现中，Docker Daemon会在(/sys/fs/cgroup)下的每一个子系统(即hierarchy)目录中创建一个名为docker的控制组，然后再docker控制组里面，再为每个容器建立一个以容器id为名称的容器控制组，这个容器里的所有进程的进程号都会写到该控制组的tasks中。*
  ![20171204-05](/img/in-post/20171204/20171204-05.png)
  ![20171204-01](/img/in-post/20171204/20171204-01.png)
  ![20171204-03](/img/in-post/20171204/20171204-03.png)
cpu和memory下分别都存在docker控制组
![20171204-04](/img/in-post/20171204/20171204-04.png)
/sys/fs/cgroup下的每个目录都是独立的hierarchy，也是根cgroup，创建docker控制组即为创建子cgroup。

## 简单容器创建

```go

 const cgroupMemoryHierarchyMount = "/sys/fs/cgroup/memory"
 func main() {
    if os.Args[0] == "proc/self/exe" {
 
         fmt.Printf("current pid %d", syscall.Getpid())
         fmt.Println()
         cmd := execCommand("sh", "-c", `stress --vm-bytes 200m --vm-keep -m 1`)
         cmd.SysProcAttr = &syscall.SysProcAttr{}
         cmd.Stdin = os.Stdin
         cmd.Stdout = os.Stdout
         cmd.Stderr = os.Stderr
         if err := cmd.Run(); err != nil {
              fmt.Println(err)
              os.Exit(1)
         }
     }
 
     cmd := exec.Command("/proc/self/exe")
     cmd.SysProcAttr = &syscall.SysProcAttr{
           Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
     }
 
     cmd.Stdin = os.Stdin
     cmd.Stdout = os.Stdout
     cmd.Stderr = os.Stderr
 
     if err := cmd.Start(); err != nil {
         fmt.Println("ERROR", err)
         os.Exit(1)
    }else {
         fmt.Printf("%v", cmd.Process.Pid)
 
         os.Mkdir(path.Join(cgroupMemoryHierarchyMount, "testmemorylimit"), 0755)
 
         ioutil.WriteFile(path.Join(cgroupMemoryHierarchyMount, "testmemorylimit", "tasks"),   []byte(strconv.Itoa(cmd.Process.Pid)), 0644)
 
         ioutil.WriteFile(path.Join(cgroupMemoryHierarchyMount, "testmemorylimit", "memory.limit_in_bytes"), []byte("100  m"), 0644)
     }
     cmd.Process.Wait()
  }
   
```
---

![20171204-06](/img/in-post/20171204/20171204-06.png)

> exec.Command() 执行外部命令
SysProcessAttr 新进程空间的属性
cmd.Start() 开始运行
cmd.Run() 执行完毕后返回
/proc/self/exe 链接到当前执行进程

