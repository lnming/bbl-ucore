# Lab 1

以下所有讨论只适用于Spike模拟器和QEMU上对应的`spike-board`实现，在其他RISC-V平台上未必正确。

## All About BBL

### Compilation

bbl使用了[**Autotools**](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html)作为构建系统，编译过程如下

```bash
$ mkdir build && cd build
$ ../configure --prefix=$RISCV --host=riscv32-unknown-linux-gnu --with-payload=/path/to/kernel
$ make
```

若不传入`--with-payload`选项，则默认使用`dummy_payload`，读者应当查看`bbl/payload.S`和`dummy_payload/`以初步了解bbl加载kernel的原理。

`payload.S`如下

```nasm
.section ".payload","a",@progbits
.align 3

.globl _payload_start, _payload_end
_payload_start:
.incbin BBL_PAYLOAD
_payload_end:
```

要注意`.align 3`并非3字节对齐而是$2^3$字节对齐。

### Linker Script

bbl的linker script如下

```
OUTPUT_ARCH( "riscv" )

ENTRY( reset_vector )

SECTIONS
{
  /*--------------------------------------------------------------------*/
  /* Code and read-only segment                                         */
  /*--------------------------------------------------------------------*/

  /* Begining of code and text segment */
  . = 0x80000000;
  _ftext = .;
  PROVIDE( eprol = . );

  .text :
  {
    *(.text.init)
  }

  /* text: Program code section */
  .text : 
  {
    *(.text)
    *(.text.*)
    *(.gnu.linkonce.t.*)
  }

  /* rodata: Read-only data */
  .rodata : 
  {
    *(.rdata)
    *(.rodata)
    *(.rodata.*)
    *(.gnu.linkonce.r.*)
  }

  /* End of code and read-only segment */
  PROVIDE( etext = . );
  _etext = .;

  /*--------------------------------------------------------------------*/
  /* HTIF, isolated onto separate page                                  */
  /*--------------------------------------------------------------------*/
  . = ALIGN(0x1000);
  htif :
  {
    *(htif)
  }
  . = ALIGN(0x1000);

  /*--------------------------------------------------------------------*/
  /* Initialized data segment                                           */
  /*--------------------------------------------------------------------*/

  /* Start of initialized data segment */
  . = ALIGN(16);
   _fdata = .;

  /* data: Writable data */
  .data : 
  {
    *(.data)
    *(.data.*)
    *(.srodata*)
    *(.gnu.linkonce.d.*)
    *(.comment)
  }

  /* End of initialized data segment */
  . = ALIGN(4);
  PROVIDE( edata = . );
  _edata = .;

  /*--------------------------------------------------------------------*/
  /* Uninitialized data segment                                         */
  /*--------------------------------------------------------------------*/

  /* Start of uninitialized data segment */
  . = .;
  _fbss = .;

  /* sbss: Uninitialized writeable small data section */
  . = .;

  /* bss: Uninitialized writeable data section */
  . = .;
  _bss_start = .;
  .bss : 
  {
    *(.bss)
    *(.bss.*)
    *(.sbss*)
    *(.gnu.linkonce.b.*)
    *(COMMON)
  }

  .sbi :
  {
    *(.sbi)
  }

  .payload :
  {
    *(.payload)
  }

  _end = .;
}
```

CPU加电后执行`0x00001000`处的首条指令，通过` auipc`跳转到`0x80000000`开始执行bbl的启动代码。可以看见bbl的入口为`reset_vector`，该符号位于`machine/mentry.S`中。Linker script中还需要注意的有`htif`、`.sbi`和`.payload`三个部分，它们分别位于`machine.mtrap.c`、`sbi_entry.S`和`bbl/payload.S`中。

### Loading Kernel

```c
void boot_loader()
{
  extern char _payload_start, _payload_end;
  load_kernel_elf(&_payload_start, &_payload_end - &_payload_start, &info);
  supervisor_vm_init();
#ifdef PK_ENABLE_LOGO
  print_logo();
#endif
  mb();
  elf_loaded = 1;
  enter_supervisor_mode((void *)info.entry, 0);
}
```

在完成编译后，我们的kernel以二进制ELF文件的形式被打包到了生成的`bbl`中，而kernel的起始和终止地址分别为`_payload_start`和`_payload_end`，BBL会读取kernel并释放到内存中，读者可以参阅`bbl/kernel_elf.c`文件以了解详细过程；之后，BBL会利用从ELF中获得的信息为kernel建立一个基本的页表，并将SBI映射到虚拟地址空间的最后一个页上；最后，`enter_supervisor_mode`函数会将控制权转交给kernel并进入S-mode。

### Supervisor Binary Interface

之前已经提到，RISC-V利用Binary Interface实现对底层环境的抽象，从而方便了各个水平的虚拟化的实现。这个想法本身是非常优秀的，可惜直到[Privileged ISA Specification v1.9.1](https://riscv.org/specifications/privileged-isa)为止，SBI的实现思路都是错误的。为了方便说明，我们先对RISC-V ISA做进一步介绍。

#### Memory Management

对于一个32位Unix-like操作系统而言，只需要用两种内存管理管理模式

* Mbare: Physical Addresses
* Sv32: Page-Based 32-bit Virtual-Memory Systems

默认情况下使用的是Mbare模式，若想启用Sv32模式，需要向`mstatus`寄存器中的VM域写入`00100`，此时若处于S-mode，系统会自动使用页式寻址。要注意的有三点

- M-mode下使用的始终是Mbare内存管理
- `mstatus`是M-mode特有的寄存器，S-mode下的`sstatus`寄存器中无VM域，若读者对此处突然提到`sstatus`感到疑惑，建议阅读[Privileged ISA Specification v1.9.1](https://riscv.org/specifications/privileged-isa) 3.1.6小节
- 页表基址对应物理页的页号存放在`spbtr`寄存器中，该寄存器为S-mode特有寄存器，M-mode和S-mode下可写可读

如果读者还记得OOP课上学过的[single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle)，应该能意识到让M-mode的软件SEE决定是否启用页式寻址并让S-mode的软件OS管理页表是一件很糟糕的事情，而事实也确实是这样。

#### SBI Implementation

SBI呈现为一组函数，它的实现在`sbi_entry.S`中，OS只能获得头文件`sbi.h`和对应的函数地址`sbi.S`

```c
#ifndef _ASM_RISCV_SBI_H
#define _ASM_RISCV_SBI_H

typedef struct {
  unsigned long base;
  unsigned long size;
  unsigned long node_id;
} memory_block_info;

unsigned long sbi_query_memory(unsigned long id, memory_block_info *p);

unsigned long sbi_hart_id(void);
unsigned long sbi_num_harts(void);
unsigned long sbi_timebase(void);
void sbi_set_timer(unsigned long long stime_value);
void sbi_send_ipi(unsigned long hart_id);
unsigned long sbi_clear_ipi(void);
void sbi_shutdown(void);

void sbi_console_putchar(unsigned char ch);
int sbi_console_getchar(void);

void sbi_remote_sfence_vm(unsigned long hart_mask_ptr, unsigned long asid);
void sbi_remote_sfence_vm_range(unsigned long hart_mask_ptr, unsigned long asid, unsigned long start, unsigned long size);
void sbi_remote_fence_i(unsigned long hart_mask_ptr);

unsigned long sbi_mask_interrupt(unsigned long which);
unsigned long sbi_unmask_interrupt(unsigned long which);

#endif
```

```nasm
.globl sbi_hart_id; sbi_hart_id = -2048
.globl sbi_num_harts; sbi_num_harts = -2032
.globl sbi_query_memory; sbi_query_memory = -2016
.globl sbi_console_putchar; sbi_console_putchar = -2000
.globl sbi_console_getchar; sbi_console_getchar = -1984
.globl sbi_send_ipi; sbi_send_ipi = -1952
.globl sbi_clear_ipi; sbi_clear_ipi = -1936
.globl sbi_timebase; sbi_timebase = -1920
.globl sbi_shutdown; sbi_shutdown = -1904
.globl sbi_set_timer; sbi_set_timer = -1888
.globl sbi_mask_interrupt; sbi_mask_interrupt = -1872
.globl sbi_unmask_interrupt; sbi_unmask_interrupt = -1856
.globl sbi_remote_sfence_vm; sbi_remote_sfence_vm = -1840
.globl sbi_remote_sfence_vm_range; sbi_remote_sfence_vm_range = -1824
.globl sbi_remote_fence_i; sbi_remote_fence_i = -1808
```

上面`sbi.S`中的magic numbers就是各个函数所在的虚拟地址，为了将这些函数映射到这些位置上，BBL在加载kernel时做了一些额外的工作，之前在[Loading Kernel](#loading-kernel)部分也有提及，具体实现如下

```c
  // map SBI at top of vaddr space
  extern char _sbi_end;
  uintptr_t num_sbi_pages = ((uintptr_t)&_sbi_end - DRAM_BASE - 1) / RISCV_PGSIZE + 1;
  assert(num_sbi_pages <= (1 << RISCV_PGLEVEL_BITS));
  for (uintptr_t i = 0; i < num_sbi_pages; i++) {
    uintptr_t idx = (1 << RISCV_PGLEVEL_BITS) - num_sbi_pages + i;
    sbi_pt[idx] = pte_create((DRAM_BASE / RISCV_PGSIZE) + i, PTE_G | PTE_R | PTE_X);
  }
  pte_t* sbi_pte = middle_pt + ((num_middle_pts << RISCV_PGLEVEL_BITS)-1);
  assert(!*sbi_pte);
  *sbi_pte = ptd_create((uintptr_t)sbi_pt >> RISCV_PGSHIFT);
```

有兴趣的读者可以自行理解实现细节。

#### SBI Pitfall

> All problems in computer science can be solved by another level of indirection... Except for the problem of too many layers of indirection.
>
> — David Wheeler

虽然SBI的实现复杂得无以复加，但到目前为止似乎还没出什么逻辑上的问题，果真如此吗？让我们来看一个例子

```c
unsigned long sbi_query_memory(unsigned long id, memory_block_info *p);
```

这个SBI函数不可能被实现，因为它涉及到了传递地址的过程，而我们之前已经提到，M-mode永远工作在Mbare模式下，传一个32位虚拟地址给SEE毫无意义，因为SEE看到的直接就是物理地址。这样，我们发现了SBI的第一个问题

* SBI只能传值而不能传引用

第二个问题并不如第一个显然。考虑一下，既然SBI是Supervisor对SEE进行“系统”调用的过程，期间必然会发生特权级从S到M的转换，RISC-V中只有一条指令能完成这种转换——`ecall`。我们不妨来看一看`sbi_console_putchar`的实现

```nasm
# console_putchar
.align 4
li a7, MCALL_CONSOLE_PUTCHAR # MCALL_CONSOLE_PUTCHAR == 1
ecall
ret
```

所有的SBI都应该是如此实现的，但一个更合乎逻辑的Binary Interface应当是这样的——"欲使用SEE提供的console putchar功能，请将想要输出的字符放入寄存器a0，将寄存器a7置为1，并使用ecall指令"。如果上述理由不足以说服你，那么请看下面这个x86汇编程序

```nasm
section .programFlow
    global _start
    _start:
        mov edx, len
        mov ecx, msg
        mov ebx, 0x1    ;select STDOUT stream
        mov eax, 0x4    ;select SYS_WRITE call
        int 0x80        ;invoke SYS_WRITE
        mov ebx, 0x0    ;select EXIT_CODE_0
        mov eax, 0x1    ;select SYS_EXIT call
        int 0x80        ;invoke SYS_EXIT
section .programData
    msg: db "Hello World!",0xa
    len: equ $ - msg
```

我们使用了Linux操作系统提供的ABI完成了打印"Hello World!"的任务，`printf`和`putchar`等函数我们一般称之为API而非ABI。当下RISC-V中SBI的形态——一个头文件和一组函数地址——更加像是SPI而非SBI。这就是SBI存在的第二个问题

* SBI过度封装

#### SBI in BBL

```c
unsigned long sbi_query_memory(unsigned long id, memory_block_info *p);
```

前面已经说过，这个函数不可能被实现，可它确确实实在BBL中被“实现”了，读者可以参阅`machine/sbi_entry.S`和`machine/sbi_impl.c`

```nasm
# query_memory
.align 4
tail __sbi_query_memory
```

```c
uintptr_t __sbi_query_memory(uintptr_t id, memory_block_info *p)
{
  if (id == 0) {
    p->base = first_free_paddr;
    p->size = mem_size + DRAM_BASE - p->base;
    return 0;
  }

  return -1;
}
```

这个workaround似乎没有什么问题，但我们还是得仔细考量一下。`tail __sbi_query_memory`可以理解为一条jump到函数入口地址的指令，问题在于，上述代码都是在bbl中编译的，其中的地址均为物理地址，为何Supervisor能够正常调用它们呢？

原因大致有两点

* 编译器生成了[position-independent code](https://en.wikipedia.org/wiki/Position-independent_code)
* 在虚拟地址空间中，两段代码的相对位置关系和物理地址空间中的相对位置关系相同

由于上述原因，当操作系统完成对物理内存的管理后，这样的workaround也不再有效。

#### SBI in the future

SBI的众多问题有望在[Privileged ISA Specification v1.10](https://github.com/riscv/riscv-isa-manual)中得到解决，下面是我们和作者的通信

```
主　题:    
Re: I'm from Tsinghua University and have some questions about SBI in RISC-V.
发件人:    Andrew Waterman 2017-4-11 15:27:41
收件人:    张蔚
Great questions.

On Mon, Apr 10, 2017 at 8:18 PM, 张蔚 <zhangwei15@mails.tsinghua.edu.cn> wrote:
> Dear Dr. Waterman,
>
> My name is Wei Zhang and I'm an undergraduate at Tsinghua University. I'm
> working on porting our teaching operating system (ucore_os_lab) to RISC-V
> under the guidance of Prof. Chen and Prof. Xiang. And I'm confused with SBI
> in RISC-V.
>
> While investigating BBL, I realized that it's inherently difficult to pass
> reference to SBI functions since supervisor lives in virtual address space
> while SEE sees physical address space. Some SBI functions defined in
> privileged spec 1.9.1 involves passing and returning pointers, I suspect
> they can't work properly without manually doing a page walk in SEE.

Yes, this is an unfortunate complication.  We are revising the SBI for
the next version of the spec, 1.10, and have arrived at a simpler
design.  We eliminated some of the calls that pass pointers, in favor
of providing a device tree pointer upon OS boot.  It is a physical
address, but now the OS starts with address translation disabled, so
this works out fairly naturally.

The remaining calls that pass pointers (e.g. SEND_IPI) now use virtual
addresses.

>
> Another question is why SBI takes the form of a collection of virtual
> addresses. Calling a SBI function will transfer control to SEE, so there is
> supposed to be a ecall somewhere in that function. It might be more natural
> to directly tell OS-designers what they should put in each register before
> invoking ecall to get desired functionalities, so they can write a small
> library themselves to wrap things up easily. SBI entries in last page
> require extra effort for both OS-designers and SEE-writers.

Agreed.  The 1.10 design uses ECALL directly, rather than jumps to
virtual addresses.  The original approach was designed to optimize
paravirtualized guest OSes, but we decided the slight overhead in
those cases was worth the simplicity of avoiding the SBI page mapping.

>
> Could you please correct me if I have misunderstood SBI? And if above
> problems do exist, are there plans to solve them is the next privileged spec?
>
> Thank you for your help in this matter.
>
> Sincerely,
>
> Wei Zhang
>
>
>
```

### Host-Target Interface

之前介绍工具链时已经提到了Host-Target Interface (HTIF)，虽然对使用了bbl的OS开发者来说并无影响，但读者仍有必要熟悉这个重要的feature。让我们考虑一个字符是怎样被bbl输出到terminal中的。

#### Step 0: Declaring Magic Variables

首先，我们需要在源码中声明两个特殊变量`tohost`和`fromhost`，读者可以查看`machine/mtrap.c`文件

```c
volatile uint64_t tohost __attribute__((aligned(64))) __attribute__((section("htif")));
volatile uint64_t fromhost __attribute__((aligned(64))) __attribute__((section("htif")));
```

#### Step 1: Finding Magic Variables

[riscv-fesvr](https://github.com/riscv/riscv-fesvr)在加载bbl时，会在ELF文件中搜索这两个变量，并记下它们的物理地址

```cpp
std::map<std::string, uint64_t> symbols = load_elf(path.c_str(), &mem);

if (symbols.count("tohost") && symbols.count("fromhost")) {
  tohost_addr = symbols["tohost"];
  fromhost_addr = symbols["fromhost"];
} else {
  fprintf(stderr, "warning: tohost and fromhost symbols not in ELF; can't communicate with target\n");
}
```

#### Step 2: Polling

```cpp
while (!signal_exit && exitcode == 0) {
  if (auto tohost = mem.read_uint64(tohost_addr)) {
    mem.write_uint64(tohost_addr, 0);
    command_t cmd(this, tohost, fromhost_callback);
    device_list.handle_command(cmd);
  } else {
    idle();
  }

  device_list.tick();

  if (!fromhost_queue.empty() && mem.read_uint64(fromhost_addr) == 0) {
    mem.write_uint64(fromhost_addr, fromhost_queue.front());
    fromhost_queue.pop();
  }
}
```

每一个cycle，模拟器都会检测`tohost`变量的值，若不为0，说明target向host发出了某种请求，需要进一步处理。也许Wikipedia上[Polling (computer science)](https://en.wikipedia.org/wiki/Polling_(computer_science))对此过程的描述有助于理解

1. The host repeatedly reads the [busy bit](https://en.wikipedia.org/wiki/Status_register) of the controller until it becomes clear.
2. When clear, the host writes in the command [register](https://en.wikipedia.org/wiki/Hardware_register) and writes a byte into the data-out register.
3. The host sets the command-ready bit (set to 1).
4. When the controller senses command-ready bit is set, it sets busy bit.
5. The controller reads the command register and since write bit is set, it performs necessary I/O operations on the device. If the read bit is set to one instead of write bit, data from device is loaded into data-in register, which is further read by the host.
6. The controller clears the command-ready bit once everything is over, it clears error bit to show successful operation and reset busy bit (0).

#### Step 3: Writing/Reading Magic Numbers

在bbl的`machine/htif.h`头文件中，定义了一个宏来方便对`tohost`的修改和对`fromhost`的读取

```c
#if __riscv_xlen == 64
# define TOHOST_CMD(dev, cmd, payload) \
  (((uint64_t)(dev) << 56) | ((uint64_t)(cmd) << 48) | (uint64_t)(payload))
#else
# define TOHOST_CMD(dev, cmd, payload) ({ \
  if ((dev) || (cmd)) __builtin_trap(); \
  (payload); })
#endif
#define FROMHOST_DEV(fromhost_value) ((uint64_t)(fromhost_value) >> 56)
#define FROMHOST_CMD(fromhost_value) ((uint64_t)(fromhost_value) << 8 >> 56)
#define FROMHOST_DATA(fromhost_value) ((uint64_t)(fromhost_value) << 16 >> 16)
```

要注意的是，当使用32位交叉编译器时，`__riscv_xlen`的值为32，使用`TOHOST_CMD`会进入`__builtin_trap()`，根据编译器不同可能是死循环或者直接退出。`dev`、`cmd`和`payload`等参数的含义和取值，有兴趣的读者可自行研究。

### Instruction Emulation

bbl还提供了指令模拟的功能为上层的kernel提供模拟器中未实现的指令，这也是一个值得一提的feature。让我们来考虑在S-mode下尝试读取时间时会发生什么

```c
asm volatile("rdtime a0")；
```

由于`rdtime`指令未被实现，执行这一句时会引发`Illegal instruction exception`，被bbl的trap handler捕捉

```nasm
trap_table:
  .word bad_trap
  .word bad_trap
  .word illegal_insn_trap
  .word bad_trap
  .word misaligned_load_trap
  .word bad_trap
  .word misaligned_store_trap
  .word bad_trap
  .word bad_trap
  .word mcall_trap
  .word bad_trap
  .word bad_trap
#define SOFTWARE_INTERRUPT_VECTOR 12
  .word software_interrupt
#define TIMER_INTERRUPT_VECTOR 13
  .word timer_interrupt
#define TRAP_FROM_MACHINE_MODE_VECTOR 14
  .word __trap_from_machine_mode
```

注意`trap_table`中的`illegal_insn_trap`就是`illegal instruction`的处理程序

```c
void illegal_insn_trap(uintptr_t* regs, uintptr_t mcause, uintptr_t mepc)
{
  asm (".pushsection .rodata\n"
       "illegal_insn_trap_table:\n"
       "  .word truly_illegal_insn\n"
#if !defined(__riscv_flen) && defined(PK_ENABLE_FP_EMULATION)
       "  .word emulate_float_load\n"
#else
       "  .word truly_illegal_insn\n"
#endif
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
#if !defined(__riscv_flen) && defined(PK_ENABLE_FP_EMULATION)
       "  .word emulate_float_store\n"
#else
       "  .word truly_illegal_insn\n"
#endif
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
#if !defined(__riscv_muldiv)
       "  .word emulate_mul_div\n"
#else
       "  .word truly_illegal_insn\n"
#endif
       "  .word truly_illegal_insn\n"
#if !defined(__riscv_muldiv) && __riscv_xlen >= 64
       "  .word emulate_mul_div32\n"
#else
       "  .word truly_illegal_insn\n"
#endif
       "  .word truly_illegal_insn\n"
#ifdef PK_ENABLE_FP_EMULATION
       "  .word emulate_fmadd\n"
       "  .word emulate_fmadd\n"
       "  .word emulate_fmadd\n"
       "  .word emulate_fmadd\n"
       "  .word emulate_fp\n"
#else
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
#endif
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word emulate_system_opcode\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .word truly_illegal_insn\n"
       "  .popsection");

  uintptr_t mstatus;
  insn_t insn = get_insn(mepc, &mstatus);

  if (unlikely((insn & 3) != 3))
    return truly_illegal_insn(regs, mcause, mepc, mstatus, insn);

  write_csr(mepc, mepc + 4);

  extern uint32_t illegal_insn_trap_table[];
  uint32_t* pf = (void*)illegal_insn_trap_table + (insn & 0x7c);
  emulation_func f = (emulation_func)(uintptr_t)*pf;
  f(regs, mcause, mepc, mstatus, insn);
}
```

可以看到bbl甚至提供了浮点数模拟的功能。

注意`.word emulate_system_opcode`语句，在取出“非法”指令并适当判断后，bbl会将指令交给`emulate_system_opcode`函数处理。又经过各种判断后，最终来到`emulate_read_csr`函数中

```c
static inline int emulate_read_csr(int num, uintptr_t mstatus, uintptr_t* result)
{
  uintptr_t counteren =
    EXTRACT_FIELD(mstatus, MSTATUS_MPP) == PRV_U ? read_csr(mucounteren) :
                                                   read_csr(mscounteren);

  switch (num)
  {
    case CSR_TIME:
      if (!((counteren >> (CSR_TIME - CSR_CYCLE)) & 1))
        return -1;
      *result = *mtime;
      return 0;
#if __riscv_xlen == 32
    case CSR_TIMEH:
      if (!((counteren >> (CSR_TIME - CSR_CYCLE)) & 1))
        return -1;
      *result = *mtime >> 32;
      return 0;
#endif
#if !defined(__riscv_flen) && defined(PK_ENABLE_FP_EMULATION)
    case CSR_FRM:
      if ((mstatus & MSTATUS_FS) == 0) break;
      *result = GET_FRM();
      return 0;
    case CSR_FFLAGS:
      if ((mstatus & MSTATUS_FS) == 0) break;
      *result = GET_FFLAGS();
      return 0;
    case CSR_FCSR:
      if ((mstatus & MSTATUS_FS) == 0) break;
      *result = GET_FCSR();
      return 0;
#endif
  }
  return -1;
}
```

bbl从会从`mtime`中读取正确的时间然后返回，这里的`mtime`所指对象也是前面提到的HTIF的一部分。

从kernel层面看，执行指令后当前时间被正确放入了寄存器中。