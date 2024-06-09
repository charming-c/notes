# Shell 脚本语法

`shell`可以理解为一种脚本语言，像`javascript`等其它脚本语言一样，只需要一个能编写代码的文本编辑器和一个能解释执行的脚本解释器就可以。`shell脚本`的本质是：以某种语法格式将shell命令组织起来的由shell程序解析执行的脚本文本文件。由本质可知，要想掌握`shell脚本`，就需要了解并掌握下列三部分内容

- **shell命令**：即`ls/cd`等linux命令，详细可参考[shell命令](http://codetoolchains.readthedocs.io/en/latest/4-Linux/2-shellcmd/index.html)
- **shell解释器**：即`sh/bash/csh`等shell应用程序，详细可参考[shell应用程序](http://codetoolchains.readthedocs.io/en/latest/4-Linux/1-shellenv/1-shellsoft/index.html)
- **shell语法**：即`数据类型/变量/控制流语句/函数`等编程语法

Linux 的 Shell 种类众多，常见的有：

- Bourne Shell（/usr/bin/sh或/bin/sh）
- Bourne Again Shell（/bin/bash）
- C Shell（/usr/bin/csh）
- K Shell（/usr/bin/ksh）
- Shell for Root（/sbin/sh）

## 1. Shell 解释器

当系统执行一个脚本文件时，会读取脚本文件的第一行。如果这一行以 `#!` 开头，那么后面紧跟的路径就指定了要用来解释执行该脚本的解释器。

```shell
#!/bin/bash 
告诉系统在运行这个脚本时要使用 Bash 解释器来执行脚本中的命令。
```

## 2. Shell 语法

[语法链接](https://shellscript.readthedocs.io/zh-cn/latest/1-syntax/index.html)

