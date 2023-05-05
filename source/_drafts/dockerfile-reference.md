---
title: dockerfile-reference
tags: docker
---

Docker可以通过读取`dockerfile`中的指令自动构建镜像。`dockerfile`是一个文本文档，其中包含了所有用户可以在命令行调用用于构建镜像的指令。用户可以使用`docker build`创建一个可以连续执行多个命令的自动化构建。

本文介绍了所有可以用在`dockerfile`中的命令。读完本文后可以通过阅读[最佳实践](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)获取面向技巧的指导。

## 用法

`docker build`命令从`dockerfile`和上下文中构建一个镜像。构建上下文是特定`path`或`url`中的一系列文件。`path`是本地文件系统的一个目录。`url`是git仓库。

上下文是递归处理的。所以`path`包括它的任意子目录，`url`包括该仓库以及子模块。下面的示例展示了使用当前目录作为上下文的构建命令：

```sh
$ docker build .
Sending build context to Docker daemon 6.51MB
...
```

构建是docker守护进程执行的，不是命令行。构建程序的第一件事是将整个上线文（递归的）发送给docker守护进程。大多数情况下最好用空目录作为上下文，将`dockerfile`文件放在该目录。仅添加构建`dockerfile`所需的文件。

> ***warning:***不要使用根目录作为`path`，这会导致构建程序将所有硬盘数据发送给docker守护进程

为了在构建上下文使用文件，`dockerfile`通过特定的指令指向文件，例如`copy`指令。通过添加`.dockerignore`文件指定排除的文件和目录提高构建性能。查看[如何创建`.dockerignore`文件](https://docs.docker.com/engine/reference/builder/#dockerignore-file)文档获取更多信息。

通常情况下`dockerfile`位于上下文的根目录下。可以在执行`docker build`时通过`-f`指定文件系统上任意位置的`dockerfile`。

```
$ docker build -f /path/to/dockerfile .
```

如果构建成功可以指定保存新景象的仓库和标签：

```
$ docker build -t shykes/myapp .
```

可以在`build`指令后添加多个`-t`参数将构建后的镜像标记到多个仓库中。

```
$ docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .
```

docker守护进程执行`dokcerfile`中的指令前，会对`dockerfile`进行初步检查，如果有语法错误的话，返回错误：
```
$ docker build -t test/myapp .

Sending build context to Docker daemon 2.048 kB
Error response from daemon: Unknown instruction: RUNCMD
```

docker守护进程逐行执行`dockerfile`中的指令，必要的话，在最后输出你的新镜像ID之前将每个指令的结果提交到一个新的镜像。docker守护进程会自动清理发送的上下文。

注意每个指令都是独立运行，导致新的镜像被创建-所以`RUN cd /tmp`对下一行指令没有任何影响。

只要可能，docker会重用中间镜像（缓存）显著加速`docker build`的处理。控制台输出中的`Using cache`表明使用了缓存。（通过[最佳实践](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)了解更多信息）

```
$ docker build -t svendowideit/ambassador .

Sending build context to Docker daemon 15.36 kB
Step 1/4 : FROM alpine:3.2
 ---> 31f630c65071
Step 2/4 : MAINTAINER SvenDowideit@home.org.au
 ---> Using cache
 ---> 2a1c91448f5f
Step 3/4 : RUN apk update &&      apk add socat &&        rm -r /var/cache/
 ---> Using cache
 ---> 21ed6e7fbb73
Step 4/4 : CMD env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
 ---> Using cache
 ---> 7ea8aef582cc
Successfully built 7ea8aef582cc
```

构建缓存仅对有本地父链的镜像生效。意思是之前构建的镜像或者整个镜像链通过`docker load`加载。如果你想使用特定镜像的构建缓存，可以指定`--cache-from`选项。指定了`--cache-from`的镜像不需要有父链并且可能在其他仓库（registry）拉取。

构建完成后，可以将镜像推送到仓库（registry）。

## 构建套件

从18.09版本开始，docker支持[moby/buildkit](https://github.com/moby/buildkit)项目支持的用于执行构建的新后端。

新的后端构建套件相较老的实现提供了许多好处。

- 检测并跳过未用到的构建阶段
- 并行构建独立的构建阶段
- 构建上下文的两次构建仅增量传输有改动的文件
- 在构建上下文中检测并跳过传输未用到的文件
- 使用外部实现了许多新特性的dockerfile
- 避免其他API的副作用（中间镜像和容器）
- 优先考虑构建缓存以自动修剪

为了使用构建套件后端，需要调用`docker build`前设置环境变量`DOCKER_BUILDKIT=1`。

## 格式

下面是`dockerfile`的格式

```
# 注释
INSTRUCTION arguments
```

指令是大小写不敏感的。但是，习惯上是将指令写为大写来更简单的将他们与参数区分。

docker按顺序执行`dockerfile`中的指令。`dockerfile`必须以`FROM`指令开始。但是可以在解析器指令，注释，以及全局作用域的变量之后。`FROM`指令指定了正在构建的父镜像。`dockerfile`中的`FROM`指令前仅可能有一个或多个`ARG`指令用户定义`FROM`行使用的参数。

docker吧`#`开头的行当做注释，除非是个合法的解析器指令（参考下方解析器指令）。其他地方的`#`会被当做参数。合法的语句如下：

```
# 注释
RUN echo 'we are running some # of cool things'
```

`dockerfile`指令被执行前注释行会被移除，意味着下面示例中的的注释不会被`echo`命令处理，下面的两个示例是等价的：

```
RUN echo hello \
# comment
world
```

```
RUN echo hello \
world
```

注释中不支持行继续符（命令后面的“\”）。

> ***注意空格***<br>
> 为了向后兼容，注释和指令前面的空格会被忽略，但并不不鼓励。下面的示例中，开头的空格不会被保留，因此下面的两个示例是等价的：
> 
```
        # this is a comment-line
    RUN echo hello
RUN echo world
```
```
# this is a comment-line
RUN echo hello
RUN echo world
```

> 但是，注意指令和参数中的空格，例如`RUN`命令中的空格会被保留。

## 解释器指令

解释器指令是可选的，影响`dockerfile`中后续指令的行为。解释器指令不会在构建中增加层，也不会在构建步骤中展示。解释器指令是一种特殊的注释方式，形如`# directive=value`。每个指令仅能被使用一次。

一旦注释，空行，或者构建指令被执行，docker将忽略解释器指令。而是会将任何格式化为解释器指令的内容视为注释，并且不会校验是否可能是解释器指令。因此所有的解释器指令必须在`dockerfile`的最上面。

解释器指令大小写不敏感。习惯上用小写。解释器指令后习惯上会加一个空行。解释器指令不支持行继续符。

行继续符导致的非法：
```
# direc \
tive=value
```

指令出现两次导致非法：
```
# directive=value1
# directive=value2

FROM ImageName
```

出现在构建指令后，被当做注释：
```
FROM ImageName
# directive=value
```

出现在非解释器指令的注释后，被当做注释：
```
# About my dockerfile
# directive=value
FROM ImageName
```

不能识别的指令也会被当做注释，因此即使后面的指令是正确的指令，也会被当做注释：
```
# unknowndirective=value
# knowndirective=value
```

解释器指令中允许没有行继续符的空格。因此下面的指令是等价的：
```
#directive=value
# directive =value
#	directive= value
# directive = value
#	  dIrEcTiVe=value
```

docker支持下面两个解释器指令：
- syntax
- escape

### syntax
```
# syntax=[remote image reference]
```

例如：

```
# syntax=docker/dockerfile
# syntax=docker/dockerfile:1.0
# syntax=docker.io/docker/dockerfile:1
# syntax=docker/dockerfile:1.0.0-experimental
# syntax=example.com/user/repo:tag@sha256:abcdef...
```

这个特性只有在使用构建套件后端时可用。

`syntax`指令定义了用来构建当前`dockerfile`的`dockerfile`构建器的位置。构建套件后台允许无缝的使用构建器的外部实现，这些构建器以docker镜像的形式发布，并在容器的沙箱环境执行。

自定义dockerfile实现允许：
- 自动获取错误修复，并且无需更新守护程序
- 确保所有用户使用同样的实现构建dockerfile
- 无需更新守护程序使用最新特性
- 试用实验性或第三方特性

#### 官方发布

docker在docker hub的`docker/dockerfile`仓库下发布用来构建dockerfiles的官方镜像版本。新镜像有两个发布频道：稳定版和实验版

稳定版通道遵循语义版本控制。例如：

- `docker/dockerfile:1.0.0` - 仅允许不可变版本1.0.0
- `docker/dockerfile:1.0` - 允许版本1.0.*
- `docker/dockerfile:1` - 允许版本1.*.*
- `docker/dockerfile:latest` - latest release on stable channel

发布的时候，实验频道在稳定版本的基础上使用主要和次要版本号进行增量版本控制。例如：

- docker/dockerfile:1.0.1-experimental - only allow immutable version 1.0.1-experimental
- docker/dockerfile:1.0-experimental - latest experimental releases after 1.0
- docker/dockerfile:experimental - latest release on experimental channel

应该根据需要选择最适合的频道，如果仅仅是需要修复bug，应该使用`docker/dockerfile:1.0`。如果想要使用实验特性，则需要使用实验频道。如果使用实验版本，新版本发布可能不会向后兼容，因此最好使用不可变的全版本号。

### escape

```
# escape=\ (backslash)
OR
# escape=` (backtick)
```

`escape`指令设置`dockerfile`中的转义字符。如果没有设置，默认是`\`。

`转义字符`用来转义字符和行。这就允许`dockerfile`中的指令可以跨多行。注意，不管`dockerfile`中是否包含`escape`解释器指令，转义在`RUN`命令都不会生效，除非是在行尾。

将转义字符设置为"\`"在Windows平台非常有用，因为Windows的目录路分隔符是"\\"。"\`"与window powershell是一致的。

下面的示例在Windows上将会导致不明显的失败。末尾的第二个"\\"将会被当做新行的转义，而不是第一个"\\"的转义。同理，第三行结尾的"\\"也一样，假设真的当做一个指令执行，会导致将其当做行继续符。结果是将dockerfile中的第二行和第三行当做一个指令。

```
FROM microsoft/nanoserver
COPY testfile.txt c:\\
RUN dir c:\
```

结果：

```
PS C:\John> docker build -t cmd .
Sending build context to Docker daemon 3.072 kB
Step 1/2 : FROM microsoft/nanoserver
 ---> 22738ff49c6d
Step 2/2 : COPY testfile.txt c:\RUN dir c:
GetFileAttributesEx c:RUN: The system cannot find the file specified.
PS C:\John>
```

上面问题的一个解决方案是在`COPY`和`dir`中使用"/"。然而，这种语法最好的情况下只是让人感到困惑，因为不是windows的原生的目录分隔符，而最糟糕的是，并非所有Windows都支持"/"作为目录分隔符，因此容易出错。

通过增加`escape`解释器指令，下面的`dockerfile`在`windows`上使用平台原生文件路径语法成功实现预期。

```
# escape=`

FROM microsoft/nanoserver
COPY testfile.txt c:\
RUN dir c:\
```

## 环境替换

环境变量（通过`ENV`语句定义`）也可以作为变量用在特定指令中被`dockerfile`解析。转义也可以通过在字面上将类似变量的语法包含到语句中来处理。

在`dockerfile`中用`$variable_name`或`${variable_name}`来标记环境变量。这两种写法是相等的，大括号语法通常用来解决变量名不带空格的变量名问题比如`${foo}_bar`。

`${varialbe_name}`写法还支持下面几个标准的`bash`修饰符：

- `${variable:-word}`表示如果设置了`variable`变量，结果为设置的值，否则值为`word`。
- `${variable:+word}`表示如果设置了`variable`变量，结果是`word`，否则值为空字符串

上面的例子中，`word`可以是任何职，包括附加的环境变量。

变量前加`\\`可以进行转义：例如`\$foo`或者`${foo}`会在字面上分别替换成`$foo`和`${foo}`。

例如（解析后的表达式在`#`后面）：

```
FROM busybox
ENV FOO=/bar
WORKDIR ${FOO}   # WORKDIR /bar
ADD . $FOO       # ADD . /bar
COPY \$FOO /quux # COPY $FOO /quux
```

`dockerfile`中的以下指令支持环境变量：

- ADD
- COPY
- ENV
- EXPOSE
- FROM
- LABEL
- STOPSIGNAL
- USER
- VOLUME
- WORKDIR
- ONBUILD （当与上面任意一个支持的指令结合）

环境变量替换在整个指令中的值是一样的。换句话说，例如：

```
ENV abc=hello
ENV abc=bye def=$abc
ENV ghi=$abc
```

结果`def`的值是`hello`，而不是`bye`。而`ghi`的值是`bye`因为不是设置`abc=bye`指令的一部分。

## .dockerignore文件


