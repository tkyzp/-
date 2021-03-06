#### 2021/1/16

##### 本机强制修改ubuntu密码的办法

1. 重启并按住`Shift`进入grub菜单，进入ubuntu高级选项

2. 选中 *recovery mode* ，按 ***e*** 编辑

3. 往下翻， 找到`ro recovery nomodeset`，将这里及其后面的删除（仅这一行），最后两行保留。在删除的地方添加

   ```bash
   rw single init=/bin/bash
   ```

   

4. 按`Ctrl+x`或`F10`进行引导，进入单用户模式

5. 修改密码后，重启即可

> 引导结束后可以按两下回车0.0



#### 2021/3/10

> 冷知识：
>
> windows 下， 真正执行python命令的是 pythonXX.dll（XX为版本号）,
>
> 而Linux下， 执行python命令的是python



#### 2020/12/17  

##### 又一种获取`link_map`的方法  

```c
//<link.h>
struct r_debug

 {

  int r_version;  /* Version number for this protocol. */

  struct link_map *r_map; /* Head of the chain of loaded objects. */
    

  /* This is the address of a function internal to the run-time linker,

​    that will always be called when the linker begins to map in a

​    library or unmap it, and again when the mapping change is complete.

​    The debugger can set a breakpoint at this address if it wants to

​    notice shared object mapping changes. */

  ElfW(Addr) r_brk;

  enum

   {

 /* This state value describes the mapping change taking place when

   the `r_brk' address is called. */

 RT_CONSISTENT,  /* Mapping change is complete. */

 RT_ADD,   /* Beginning to add a new object. */

 RT_DELETE  /* Beginning to remove an object mapping. */

   } r_state;

  ElfW(Addr) r_ldbase; /* Base address the linker is loaded at. */

 };

/* This is the instance of that structure used by the dynamic linker. */

extern struct r_debug _r_debug;
```

*<link.h>* 中导出了`_r_debug`全局变量，这个变量的`r_map`成员即为`link_map`链表头地址。  

##### 关于link_map链表的排列  

一般来说link_map链表按以下方式排列:    

（图片位置：./link_map.jpg）  

![](./link_map.jpg)

> 参考资料：[https://paper.seebug.org/papers/Archive/refs/elf/ELF-berlinsides-0x3.pdf](https://paper.seebug.org/papers/Archive/refs/elf/ELF-berlinsides-0x3.pdf)  
>
> 根据实际测试，程序中使用dlopen函数加载的库的link_map会依次被链接在这个链表的最后  



#### 2020/12/10  

##### 关于`dlopen`的返回值：  

[man dlopen linux手册]: https://linux.die.net/man/3/dlope

根据实际测试，`dlopen`返回值为对应库的`link_map`结构体指针：  

```c
//test.c
#include<stdlib.h>

#include<stdio.h>

#include<dlfcn.h>

#include<elf.h>

#include<link.h>

int main(int argc, char const *argv[])

{

  void *hd = dlopen("./system_hook.so", RTLD_NOW);

  printf("%s\n", ((struct link_map*)hd)->l_name);

  return 0;

}
```

输出：  

```bash
xxx@ubuntu:~/$ ./test
dynamic:7fce371b8e08
./system_hook.so
```

其中第一行输出的`dynamic:7fce371b8e08`为库中的初始化输出，第二行对main函数中的输出，由此可以证明，dlopen函数的返回值确实是对应库的`link_map`结构体指针。  



#### 2020/12/9

Liunx下对函数hook时如何识别自身：  

Linux下采用修改got表方式进行hook，并且是将新的函数通过库加载到目标进程空间时，难免会遇到这样的问题：如果修改了自身的got表，那么调用被hook的函数就会进入死循环导致程序出错，如何识别自身就非常重要。  

##### 动态库识别自身镜像的方法

1,前面讲了获取*link_map*的方法，可以通过比对`link_map->l_name`来判断。

[^注]: 这个办法容易被绕过，并且判断依据是硬编码的，是个比较笨的办法



```c
struct link_map

 {

  /* These first few members are part of the protocol with the debugger.

    This is the same format used in SVR4. */

  ElfW(Addr) l_addr;  /* Difference between the address in the ELF

      file and the addresses in memory. */

  char *l_name;  /* Absolute file name object was found in. */

  ElfW(Dyn) *l_ld;  /* Dynamic section of the shared object. */

  struct link_map *l_next, *l_prev; /* Chain of loaded objects. */

 };
```

2,前面提到 *<link.h>*中导出了`_DYNAMIC`变量,这个变量标识的是当前镜像的动态段地址（当前是针对代码运行的位置而言的），可以做一个小小的测试。  

我们分别在一个动态库和程序主函数中输出`_DYNAMIC`变量，查看结果，就能验证上面所说的。  

有了上面的结论，只需要将动态库空间的这一变量保存下来，在hook的时候对比`link_map->l_ld`就能识别是否自身。  

#### 2020/12/7

##### Linux下获取动态段地址的方法：

1，通过*getauxval(AT_PHDR)*获取到phdr地址，再解析段表，获取到.dynamic段地址  
需要头文件  *#include <sys/auxv.h>*  
此方式的原理是读取 */proc/[pid]/auxv* 文件，更多详情：  

> man getauxval : [https://man7.org/linux/man-pages/man3/getauxval.3.html]( https://man7.org/linux/man-pages/man3/getauxval.3.html)  
> man proc : [https://linux.die.net/man/5/proc](https://linux.die.net/man/5/proc)

按照此原理，我们也能获取到其他程序的相关结构，前提是要知道pid

2,通过`_DYNAMIC`全局变量直接获取  
需要头文件  *#include<link.h>*  
此方式的原理为<link.h>中导出了此变量，和函数符号一样，此变量保存在got表中作为一个指针，在链接和重定位时，此变量被设置为对应镜像的动态段地址，因此实际上`_DYNAMIC`变量保存的是对应镜像的动态段地址。
> ps:通过动态段的内容还能获取到link_map等其他重要的程序动态链接相关数据结构
>

