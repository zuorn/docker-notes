# Docker 学习笔记（二） —— 使用 Dockerfile 定制镜像

> Dockerfile 是一个文本文件，其内包含了一条条的 指令\(Instruction\)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

#### Dockerfile 指令

**FROM 指定基础镜像**

ROM 就是指定 基础镜像，Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

FROM scratch 表示一个空白镜像。所以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

**RUN 执行命令**

RUN 指令是用来执行命令行命令的。由于命令行的强大能力，RUN 指令在定制镜像时是最常用的指令之一。其格式有两种：

* shell 格式：RUN &lt;命令&gt;，就像直接在命令行中输入的命令一样。刚才写的 Dockerfile 中的 RUN 指令就是这种格式
* exec 格式：RUN \["可执行文件", "参数1", "参数2"\]，这更像是函数调用中的格式。

需要注意的是，Dockerfile 中每一个指令都会建立一层，所以每个Run都会创建一层镜像，如果有多个命令可以尽量写在一个RUN里面。可以使用`&&` 串联起来

**COPY 复制文件**

* COPY \[--chown=:\] &lt;源路径&gt;... &lt;目标路径&gt;
* COPY \[--chown=:\] \["&lt;源路径1&gt;",... "&lt;目标路径&gt;"\]

&lt;目标路径&gt; 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 WORKDIR 指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

使用 COPY 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。 在使用该指令的时候可以加上 --chown=: 选项来改变文件的所属用户及所属组。

```text
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```

**ADD 更高级的复制**

ADD 和 COPY 的格式和性质都是基本一致的。区别在于：

* 比如 &lt;源路径&gt; 可以是一个 URLDocker 引擎会试图去下载这个链接的文件放到 &lt;目标路径&gt; 去。下载后的文件权限自动设置为 600。
* &lt;源路径&gt; 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 &lt;目标路径&gt; 去

如果权限需要调整的话就需要另外的一层RUN，所以并不推荐使用。 对于复制文件，应该尽可能的使用COPY。

**CMD 容器启动命令**

与RUN相似，有两种格式：

* shell 格式：CMD &lt;命令&gt;
* exec 格式：CMD \["可执行文件", "参数1", "参数2"...\]

容器就是进程，所以在启动容器的时候需要指定所运行的程序及参数。CMD 指令就是用于指定默认容器主进程的启动命令的。

在运行时可以指定新的命令来替代镜像设置中的这个默认命令。

需要注意的是 Docker 不是虚拟机，**容器中的命令都应该以前台执行**而不是像虚拟机、物理机里面那样，用 systemd 去启动后台服务，**容器内没有后台服务的概念**。

对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

**栗子：**

* 错误的写法

```text
CMD service nginx start
```

* 正确的写法

```text
CMD ["nginx", "-g", "daemon off;"]
```

**ENTRYPOINT 入口点**

ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定。

当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为：

```text
<ENTRYPOINT> "<CMD>"
```

ENTRYPOINT的作用：

* 应用运行前的准备工作，比如mysql镜像在启动时执行相关的sql，可以写一个脚本，然后放入 ENTRYPOINT 中去执行，这个脚本会将接到的参数（也就是 ）作为命令，在脚本最后执行。

**ENV 设置环境变量**

格式：

* ENV  
* ENV = =...

支持环境变量的指令：ADD、COPY、ENV、EXPOSE、LABEL、USER、WORKDIR、VOLUME、STOPSIGNAL、ONBUILD。

**ARG 构建参数**

格式：ARG &lt;参数名&gt;\[=&lt;默认值&gt;\]

构建参数和 ENV 的效果一样，都是设置环境变量。所不同的是，ARG 所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。但是不要因此就使用 ARG 保存密码之类的信息，因为 docker history 还是可以看到所有值的。

Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg &lt;参数名&gt;=&lt;值&gt; 来覆盖。

**VOLUME 定义匿名卷**

格式：

* VOLUME \["&lt;路径1&gt;", "&lt;路径2&gt;"...\]
* VOLUME &lt;路径&gt;

容器运行时，应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷（VOLUME）中。在Dockerfile中，可以事先指定某些目录为挂载为你命卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。

```text
VOLUME /data
```

这里的 /data 目录就会在运行时自动挂载为匿名卷，任何想 /data 中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。当然，运行时可以覆盖这个挂载设置。比如：

```text
docker run -d -v mydata:/data xxxx
```

在这行命令中，就使用了 mydata 这个命名卷挂载到了 /data 这个位置，替代了 Dockerfile 中定义的匿名卷的挂载配置。

**EXPOSE 声明端口**

格式：

* EXPOSE &lt;端口1&gt; \[&lt;端口2&gt;...\]。

EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。

**WORKDIR 指定工作目录**

格式：WORKDIR &lt;工作目录路径&gt;

使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），**以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录**。

如果需要改变以后各层的工作目录的位置，应该使用 WORKDIR 指令。

**USER 指定当前用户**

格式：

* USER 指定当前用户

USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER 则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。

和 WORKDIR 一样，USER 只是切换到指定的用户而已，这个用户必须是事先建立好的，否则无法切换。

**HEALTHCHECK 健康检查**

格式：

* HEALTHCHECK \[选项\] CMD &lt;命令&gt;：设置检查容器健康状况的命令
* HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令

当在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting，在 HEALTHCHECK 指令检查成功后变为 healthy，如果连续一定次数失败，则会变为 unhealthy。

HEALTHCHECK 支持下列选项：

* --interval=&lt;间隔&gt;：两次健康检查的间隔，默认为 30 秒；
* --timeout=&lt;时长&gt;：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
* --retries=&lt;次数&gt;：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。

和 CMD, ENTRYPOINT 一样，HEALTHCHECK 只可以出现一次，如果写了多个，只有最后一个生效。

**ONBUILD 为他人做嫁衣裳**

格式：

* ONBUILD &lt;其它指令&gt;。

ONBUILD 是一个特殊的指令，它后面跟的是其它指令，比如 RUN, COPY 等，而这些指令，**在当前镜像构建时并不会被执行。只有当以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。**

#### docker build 构建命令

格式：

```text
docker build [OPTIONS] PATH | URL | -
```

参数：

* -build-arg=\[\] :设置镜像创建时的变量；
* --cpu-shares :设置 cpu 使用权重；
* --cpu-period :限制 CPU CFS周期；
* --cpu-quota :限制 CPU CFS配额；
* --cpuset-cpus :指定使用的CPU id；
* --cpuset-mems :指定使用的内存 id；
* --disable-content-trust :忽略校验，默认开启；
* -f :指定要使用的Dockerfile路径；
* --force-rm :设置镜像过程中删除中间容器；
* --isolation :使用容器隔离技术；
* --label=\[\] :设置镜像使用的元数据；
* -m :设置内存最大值；
* --memory-swap :设置Swap的最大值为内存+swap，"-1"表示不限swap；
* --no-cache :创建镜像的过程不使用缓存；
* --pull :尝试去更新镜像的新版本；
* --quiet, -q :安静模式，成功后只输出镜像 ID；
* --rm :设置镜像成功后删除中间容器；
* --shm-size :设置/dev/shm的大小，默认值是64M；
* --ulimit :Ulimit配置。
* --tag, -t: 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签。
* --network: 默认 default。在构建期间设置RUN指令的网络模式

**镜像构建上下文（Context）**

docker build 命令最后有一个 .。. 表示当前目录，而 Dockerfile 就在当前目录，因此不少初学者以为这个路径是在指定 Dockerfile 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，你可能会发现，这是在指定 **上下文路径**。那么什么是上下文呢？

首先我们要理解 docker build 的工作原理。Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引擎变得轻而易举。

当我们进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要将一些本地文件复制进镜像，比如通过 COPY 指令、ADD 指令等。而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

