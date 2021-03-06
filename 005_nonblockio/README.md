# 005_nonblockio

## 1. 阻塞 IO

通过前两篇的学习，你学会了使用 `read` 函数从文件读取数据，也掌握了如何使用 `lseek` 修改偏移指针。也许你从没意识到 `read` 函数是如何工作的，毕竟从本地文件读取数据是一件相当自然的操作。但是让 `read` 从标准输入（你的控制台）读取数据会怎样呢？

有一个非常简单的命令 `cat`，当你在终端输入它的时候，它会一直阻塞在那里，什么也不输出。实际上 `cat` 命令内部的实现默认就是从你的标准输入读取数据。当你在屏幕输入一行文字后，`cat` 就会把你输入的数据读取出来，再通过 `write` 输出到标准输出上。

```shell
$ cat
hello # 你的输入，然后回车
hello
```

当你使用 `read` 函数读取普通文件的时候，它总能在有限的时间内返回。但是如果你从设备，比如终端（标准输入设备）读数据，只要没有遇到换行符(‘\n’)，read 一定会“堵”在那而不返回。还有比如从网络读数据，如果网络一直没有数据到来，read 函数也会一直堵在那而不返回。

`read` 的这种行为，称为 block，中文叫阻塞。如果一个进程阻塞在 `read` 函数，操作系统会把这个进程从运行态修改为阻塞态，同时将其加入一个称之为“等待队列”的数据结构中。一旦资源被满足（有数据到来），操作系统会将其重新设置成就绪态，并随时准备调度它运行。

有很多操作 IO 的函数都可能会被阻塞，比如 `write`, `recv`, `send` 等。

## 2. 任务

最终你的文件夹是这样的。

```shell
allen/
├── Makefile
├── README.md
├── block_io_ex.c
└── nonblock_io.c
```

### 2.1 阻塞 IO，从终端读取数据

在 `allen` 文件夹下，有一个程序 `block_io.c`，请你使用 `make` 程序编译，生成程序 `block_io`。该程序可以打开一个文件或者设备，读取其中的数据，并将所有的小写字母转换成大写字母，输出到你的屏幕。例如：

```shell
$ ./block_io README.md
```

接下来，你可以打开你的终端设备，从中读取数据

```shell
$ ./block_io /dev/tty
```

**注：`/dev/tty` 表示你使用的终端设备**

当你执行上面的命令后，你会发现程序卡在那里。接下来，随意输入一些字母并回车，`block_io` 会把你的输入的小写字母转换成大写字母，并继续等待你的输入。如果你想结束它，可以按下 `CTRL D` 组合键。

**`CTRL D` 组合键会产生一个 EOF，当接收者收到 EOF 的时候，read 就会返回 0**

第一个任务非常简单，你也不用写程序，只需要操作一下即可。

### 2.2 阻塞 IO，从有名管道文件读取数据

当你使用 `make` 命令的时候，你会发现同时生成一个 `allen.fifo` 文件，接下来，请你再打开一个新的控制台（控制台 b），你当前的控制台，后面称之为 a。

在 b 中输入以下命令：

```shell
$ cat > allen.fifo
```

在 a 中输入以下命令：

```shell
$ ./block_io allen.fifo
```

好了，剩下的操作，就是在 b 中输入一些数据。你会发现在 a 中会接收到你从 b 输入的数据。最后你可以在 b 中按下 `CTRL D` 组合键，退出。


### 2.3 阻塞 IO，同时从终端和有名管道读取数据

接下来，我希望你能改造 `block_io` 程序，能同时从终端以及有名管道读取数据，并将接受到的小写字母转换成大写。

你的新程序命名为 `block_io_ex.c`。在你写这个程序之前，请先运行我写好的 `block_io_ex`，在 a 中输入以下命令：

```shell
$ ./block_io_ex /dev/tty allen.fifo
```

在 b 中输入：

```shell
$ cat > allen.fifo
```

剩下的，请你结合 2.1 和 2.2，在 a 和 b 中都输入一些数据，看看会怎样？接下来根据你的理解，仿照写一个 `block_io_ex` 出来。

### 2.4 非阻塞 IO

有了 2.3 的基础后，相信你已经发现，每次当你从终端 a 输入数据，程序并处理后，你会发现，下次再输入数据，程序就卡在那里不动了。这时候，你再去终端 b 输入数据，发现又能继续处理之前未处理完的 a 中的数据。

**a 和 b 之间，总是互相阻塞。** 2.3 面临的问题，实际上是这样的：

```c
while(1) {
  阻塞 read(设备1);
  处理设备1数据;

  阻塞 read(设备2);
  处理设备2数据;
}
```

阻塞 IO 让你的程序无法同时处理两个设备的数据。想象一下，如果是写网络程序，后果是怎样的？你希望不同的客户端之间互相被阻塞吗？显然不是。

聪明的同学，会发现，可以使用多线程来解决，为每个打开的设备开一个线程处理不就行了吗？是的，没错。但是，在很久很久以前，Linux 是不支持多线程程序的。况且我们还没有学习多线程技术，就放弃这个选择吧。

咱们这里如果能让 `read` 不阻塞，那该多好。如果把上面的代码改成这样：

```c
while(1) {
  非阻塞 read(设备1);
  if (设备1有数据){
    处理设备1数据;
  }

  非阻塞 read(设备2);
  if (设备2有数据) {
    处理设备2数据;
  }
}
```

说不定能解决问题呢！

好了，我已经为你提前写好一个类似的可以运行的程序 `nonblock_io`，它的使用方法和 2.3 中的是一样的，但是它看起来解决了 2.3 中的问题。

接下来，请你改写 `block_io_ex`，写一个 `nonblock_io` 程序，完成上面的功能。在 2.4.1 和 2.4.2 中你可以找到一些提示。

#### 2.4.1 非阻塞 IO 如何设置

你需要关注的问题是，如何让 `read` 不阻塞。下面是一段把描述符设置为非阻塞的函数，我已经为你封装好了，你可以直接使用:

```c
int set_nonblock(int fd) {
    int old = fcntl(fd, F_GETFL);
    return fcntl(fd, F_SETFL, old | O_NONBLOCK);
}
```

这里有一个比较核心的函数 `fcntl`，首先使用 `F_GETFL` 命令把旧的 flags 标记取出来，然后再加上 `O_NONBLOCK` 标记，重新设置回去，就这么简单。

#### 2.4.2 阻塞为什么是在描述符上设置？

有同学会问，为什么要把阻塞标记设置在描述符上，而不是提供一个非阻塞版本的 `read` 函数，或者给 `read` 再加一个阻塞和非阻塞的参数来控制呢？

这两种方案都是可行的，但是 Linux 选择了前者。或者说，标准委员会规定了，使用前面的用法。这意味着，使用前者方案也许会存在更多的优势。

`read` 系统调用一旦检查到资源还未准备好，同时当前 fd 被打上了 `O_NONBLOCK` 标记，就会立即返回，而不是把进程转换成阻塞态。同时，返回 -1，`errno` 设置成 `EAGAIN` 或者 `EWOULDBLOCK`。(`EWOULDBLOCK` 的值可能和 `EAGAIN` 是一样的，也可能是不一样的，因此你最好同时观察这两个值)。


### 2.5 问答

- block_io 为什么会阻塞？
- block_io_ex 为什么无法并发处理两个不同设备来的数据?
- 查阅手册，`fcntl` 各个参数以及返回值的含义是什么？
- 你觉得非阻塞 IO 解决了什么问题？
- nonblock_io 会有什么问题？

请将题目和答案，保存在 README.md 中。

### 2.6 组内讨论

剩下的疑问，群内互相讨论。

## 3. 参考资料

[文件IO-阻塞与非阻塞IO](https://blog.csdn.net/q1007729991/article/details/52663574)
