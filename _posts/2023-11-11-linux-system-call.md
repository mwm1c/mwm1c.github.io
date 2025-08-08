---
layout: post
title:  "在Linux中添加新的系统调用"
date:   2023-11-11 10:00:00 +0800
categories: blog
tags: [Linux, 内核开发]
---


## 系统调用
需要更新多个文件以添加新的系统调用：

+ new/path/to/implementation - 系统调用函数实现（如果实现位置在一个新的文件或者目录，还需要进行一些Makefile更改）
+ include/linux/syscalls.h - 函数原型  
通用和体系结构特定的系统调用表： 
    1. include/uapi/asm-generic/unistd.h - 用户空间API中的通用系统调用表
    2. arch/x86/entry/syscalls/syscall_64.tbl - 体系结构特定的系统调用表

## 函数实现(**Function Implementation**)
首先，创建一个新目录，其中包含自己的Makefile和hello_world.c文件：

```bash
# from the root of the linux source tree
mkdir hello_world
touch hello_world/{Makefile,hello_world.c}
```

将以下内容添加到Makefile中：

```bash
obj-y := hello_world.o
```

在 hello_world/hello_world.c 中，使用SYSCALL_DEFINE<N>宏定义新的系统调用实现，其中N是系统调用接受的参数数量。

```c
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/syscalls.h>

SYSCALL_DEFINE2(hello_world, const char*, message, unsigned long, len) {
  char *kmessage;
  int res;

  if (len > 0x1000) {
    pr_err("User message length was unreasonably long\n");
    return -EINVAL;
  }

  kmessage = kzalloc(len + 1, GFP_KERNEL);
  if (!kmessage) {
    pr_err("Failed to allocate %08lux bytes\n", len + 1);
    return -ENOMEM;
  }

  res = copy_from_user(kmessage, message, len);
  if (res) {
    return -EFAULT;
  }

  pr_info("IN HELLO WORLD! Message: %s\n", kmessage);

  return 0;
}
```

关于这段代码需要注意以下几点：

1. 绝对不要直接访问用户内存！请使用copy_from_user/copy_to_user/get_user/put_user函数。
2. 在Linux中可以通过多种调用来分配内存。这里Linux内核文档中的[官方内存分配指南](https:_www.kernel.org_doc_html_latest_core-api_memory-allocation)值得阅读一下。
3. 返回值是错误码负值，或者为0表示成功。
4. pr_<LOG_LEVEL>(...)函数是包装了printk(KERN_<LOG_LEVEL> ...)函数的宏。有关详细信息，请参阅 [Message logging with printk](https:_www.kernel.org_doc_html_latest_core-api_printk-basics)。

## 函数声明(**Function Prototype**)
函数声明本身位于include/linux/syscalls.h中：

```c
asmlinkage long sys_hello_world(const char* message, unsigned long len);
```

这是新系统调用的主入口点，一般这些系统调用函数声明是以sys_为前缀。

## 系统调用表(**System Call Table**)
通用系统调用表（include/uapi/asm-generic/unistd.h）和特定架构的系统调用表（arch/x86/entry/syscalls/syscall_64.tbl）都存在。该系统调用应添加到通用表和所有支持它的架构中。

对于通用系统调用表的更改（不要忘记增加__NR_syscalls！）：

```plain
diff --git a/include/uapi/asm-generic/unistd.h b/include/uapi/asm-generic/unistd.h
index 1c48b0ae3ba3..395bd563d6f2 100644
--- a/include/uapi/asm-generic/unistd.h
+++ b/include/uapi/asm-generic/unistd.h
@@ -886,8 +886,11 @@ __SYSCALL(__NR_futex_waitv, sys_futex_waitv)
 #define __NR_set_mempolicy_home_node 450
 __SYSCALL(__NR_set_mempolicy_home_node, sys_set_mempolicy_home_node)
 
+#define __NR_hello_world 451
+__SYSCALL(__NR_hello_world, sys_hello_world)
+
 #undef __NR_syscalls
-#define __NR_syscalls 451
+#define __NR_syscalls 452
```

对x86系统调用表的更改。

```plain
diff --git a/arch/x86/entry/syscalls/syscall_64.tbl b/arch/x86/entry/syscalls/syscall_64.tbl
index c84d12608cd2..e44d6d33e7ce 100644
--- a/arch/x86/entry/syscalls/syscall_64.tbl
+++ b/arch/x86/entry/syscalls/syscall_64.tbl
@@ -372,6 +372,7 @@
 448    common  process_mrelease        sys_process_mrelease
 449    common  futex_waitv             sys_futex_waitv
 450    common  set_mempolicy_home_node sys_set_mempolicy_home_node
+451    common  hello_world     sys_hello_world
```

## 重新构建内核
在进行更改后，重新构建内核。如果想首先清除所有内容（包括配置文件），需要运行 make mrproper，然后重新配置。

## 测试系统调用
在之前关于构建内核的帖子基础上，我们将使用qemu来调试系统调用。  
首先，我们需要在用户空间中实际进行系统调用。我们将使用syscall(2)函数直接调用我们的新系统调用：

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char **argv) {
  const char* message = "Hello, world!";
  int res;

  printf("Going to call hello_world, syscall 451\n");
  res = syscall(451, message, strlen(message));
  printf("Tried to call it! Result: %d\n", res);
}
```

编译其为 hello_world_test

```bash
gcc hello_world_test.c -o hello_world_test
```

使用qemu-system-x86_64运行Linux内核，并运行我们的hello_world_test二进制文件。

```bash
qemu-system-x86_64 \
    -kernel arch/x86_64/boot/bzImage \
    -nographic \
    -append "console=ttyS0" \
    -m 1024 \
    -initrd initfs \
    --enable-kvm \
    -cpu host \
    -s -S \
    -fsdev local,path=$(pwd),security_model=none,id=test_dev \
    -device virtio-9p,fsdev=test_dev,mount_tag=test_mount
```

通过远程调试会话（在另一个终端中）附加gdb。

```bash
$> gdb vmlinux
(gdb) target remote :1234
(gdb) c
```

回到qemu shell，在我们运行的虚拟机中将 [9p](https:_www.kernel.org_doc_html_latest_filesystems_9p) 共享目录挂载到/shared目录。这样，Linux源代码树的根目录（或者当你运行qemu-system-x86时所在的位置）就可以在虚拟机内访问了。

```bash
(initramfs) mkdir /shared
(initramfs) mount -t 9p -o trans=virtio test_mount /shared/ -oversion=9p2000.L,posixacl,msize=512000,cache=loose
```

在qemu shell中，现在应该能够从/shared目录下运行已构建的hello_world_test二进制文件。

```bash
(initramfs) /shared/hello_world_test
Going to call hello_world, syscall 451
[   38.305249] hello_world: IN HELLO WORLD! Message: Hello, world!
Tried to call it! Result: 0
[   38.308184] hello_world_tes (186) used greatest stack depth: 27392 bytes left
```
