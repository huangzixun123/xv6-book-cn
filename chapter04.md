#

## 4.3 Code:调用系统调用

  第二章中的initcode.S的结尾，调用了exec系统调用(user.initcode.S)。让我们看一下exec系统调用在内核的实现。

  initcode.S将exec的参数放在寄存器a0和a1，并且将系统调用代表的值放在a7。系统调用的值会匹配syscalls数组(一个函数指针表)的值。
通过ecall指令陷入到内核中，然后调用uservec,usertrap，然后syscall执行。  

  syscall(kernel/syscall.c)从trapframe中的寄存器a7取出系统调用的值，索引到syscalls。对于第一个系统调用，a7里的值就是
SYS_exec(kernel/syscall.h)，通过该值，调用sys_exec函数。

  当sys_exec返回了，syscall记录他的返回值到p->trapframe->a0。该值会返回给原始调用exec的用户空间。如果出错，系统调用会返回负
数，零或正数表示成功。如系统调用值是无效的，syscall出打印出错误并返回-1。

## 4.4 Code:系统调用参数

  在内核代码中实现系统调用，需要拿到用户传过来的参数。因为用户代码将系统调用包装函数，参数按照RISC-V C调用惯例放在寄存器中。内核
trap代码保存用户的寄存器到内核代码可以访问到的进程trap frame。内核函数argint，argaddr，argfd从trap frame获取对应第n个调用
参数，作为整数、指针或文件描述符。他们调用argraw获取合适的用户保护寄存器。(saved user register, kernel/syscall/c:35)

  一些系统调用会传一些指针作为参数，然后内核拿这些指针来读或写用户内存。例如，exec向内核传一个指向用户空间字符串的指针数组。这些
指针遇到两个挑战。第一，用户程序可能有bug或者是恶意性的，可能会传给内核一些无效的指针或欺骗内核访问内核空间而不是用户空间的指针。
第二，xv6内核页表映射跟用户页表映射不一样，所以内核不能用来自用户地址所提供的普通指令来读或存。

  内核实现了用户提供的地址中安全的传输数据的功能。fetchstr就是一个例子(kernel/syscall.c:25)。文件系统来调用例如exec使用
fetchstr来获取用户空间传过来的文件名参数。fetchstr调用copyinstr来做最重要的工作。

  copyinstr(kernel/vm.c:398)从用户页表pagetable的虚拟地址srcva最多复制max字节数据到dst中。因为pagetable不是当前的页表，
copyinstr用walkaddr(walk)查找页表中的srcva，产生(yielding)物理地址pa0。内核会映射RAM物理地址到对应的内核虚拟地址，所以
copyinstr可以从pa0直接复制字符串到dst。walkaddr(kernel/vm.c:104)会检查用户提供的虚拟地址在不在用户进程的地址空间中，所以
程序无法欺骗内核读或写内存。一个相似的函数，copyout，从内核复制数据到用户提供的地址中。

