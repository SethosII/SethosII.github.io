---
layout: post
title:  "Memory accounting in SLURM with cgroups"
date:   2018-09-01 18:29:00 +0100
author: Paul Jähne
---

Ever wondered how memory is counted for in [SLURM](https://slurm.schedmd.com/)? I was specifically courious whether allocated but unused memory counts against the memory limit and how SLURM kills jobs that exceed their memory limit.

I configured CPU and memory as consumable resources in SLURM with the help of cgroups and the relevant parameters in `slurm.conf` are as follows (also don't forget to add `cgroup_enable=memory swapaccount=1` to the `GRUB_CMDLINE_LINUX` line in the file `/etc/default/grub` on Debian/Ubuntu):

```
SelectType=select/cons_res
SelectTypeParameters=CR_CPU_Memory
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup
```

To test this I wrote two small programs. They both allocate a given amount of memory and wait for a given amount of time but `mem_use.c` also initialises the memory:

```
$ diff -y mem_do_nothing.c mem_use.c
#include <stdint.h>                                             #include <stdint.h>
#include <stdio.h>                                              #include <stdio.h>
#include <stdlib.h>                                             #include <stdlib.h>
                                                              > #include <string.h>
#include <unistd.h>                                             #include <unistd.h>

int main(int argc, char *argv[]) {                              int main(int argc, char *argv[]) {
    uint32_t mb = atoi(argv[1]);                                    uint32_t mb = atoi(argv[1]);
    uint32_t sec = atoi(argv[2]);                                   uint32_t sec = atoi(argv[2]);
    char *mem;                                                      char *mem;

    mem = (char*)malloc(mb * 1024 * 1024 * sizeof(char));           mem = (char*)malloc(mb * 1024 * 1024 * sizeof(char));
                                                              >     memset(mem, '#', mb * 1024 * 1024);
    sleep(sec);                                                     sleep(sec);
    free(mem);                                                      free(mem);
}                                                               }
```

The source files are compiled into an executable binary and assembler:

```
$ gcc -S -o mem_do_nothing.s mem_do_nothing.c
$ gcc -o mem_do_nothing mem_do_nothing.c
$ gcc -S -o mem_use.s mem_use.c
$ gcc -o mem_use mem_use.c
```

The assembly only differs in the call to the function `memset`. The `malloc` in `mem_do_nothing.s` is kept although the memory is not used because no compiler optimisation was chosen:

```
$ diff -y mem_do_nothing.s mem_use.s
        .file   "mem_do_nothing.c"                            |         .file   "mem_use.c"
        .text                                                           .text
        .globl  main                                                    .globl  main
        .type   main, @function                                         .type   main, @function
main:                                                           main:
.LFB2:                                                          .LFB2:
        .cfi_startproc                                                  .cfi_startproc
        pushq   %rbp                                                    pushq   %rbp
        .cfi_def_cfa_offset 16                                          .cfi_def_cfa_offset 16
        .cfi_offset 6, -16                                              .cfi_offset 6, -16
        movq    %rsp, %rbp                                              movq    %rsp, %rbp
        .cfi_def_cfa_register 6                                         .cfi_def_cfa_register 6
        subq    $32, %rsp                                               subq    $32, %rsp
        movl    %edi, -20(%rbp)                                         movl    %edi, -20(%rbp)
        movq    %rsi, -32(%rbp)                                         movq    %rsi, -32(%rbp)
        movq    -32(%rbp), %rax                                         movq    -32(%rbp), %rax
        addq    $8, %rax                                                addq    $8, %rax
        movq    (%rax), %rax                                            movq    (%rax), %rax
        movq    %rax, %rdi                                              movq    %rax, %rdi
        call    atoi                                                    call    atoi
        movl    %eax, -16(%rbp)                                         movl    %eax, -16(%rbp)
        movq    -32(%rbp), %rax                                         movq    -32(%rbp), %rax
        addq    $16, %rax                                               addq    $16, %rax
        movq    (%rax), %rax                                            movq    (%rax), %rax
        movq    %rax, %rdi                                              movq    %rax, %rdi
        call    atoi                                                    call    atoi
        movl    %eax, -12(%rbp)                                         movl    %eax, -12(%rbp)
        movl    -16(%rbp), %eax                                         movl    -16(%rbp), %eax
        sall    $20, %eax                                               sall    $20, %eax
        movl    %eax, %eax                                              movl    %eax, %eax
        movq    %rax, %rdi                                              movq    %rax, %rdi
        call    malloc                                                  call    malloc
        movq    %rax, -8(%rbp)                                          movq    %rax, -8(%rbp)
                                                              >         movl    -16(%rbp), %eax
                                                              >         sall    $20, %eax
                                                              >         movl    %eax, %edx
                                                              >         movq    -8(%rbp), %rax
                                                              >         movl    $35, %esi
                                                              >         movq    %rax, %rdi
                                                              >         call    memset
        movl    -12(%rbp), %eax                                         movl    -12(%rbp), %eax
        movl    %eax, %edi                                              movl    %eax, %edi
        call    sleep                                                   call    sleep
        movq    -8(%rbp), %rax                                          movq    -8(%rbp), %rax
        movq    %rax, %rdi                                              movq    %rax, %rdi
        call    free                                                    call    free
        movl    $0, %eax                                                movl    $0, %eax
        leave                                                           leave
        .cfi_def_cfa 7, 8                                               .cfi_def_cfa 7, 8
        ret                                                             ret
        .cfi_endproc                                                    .cfi_endproc
.LFE2:                                                          .LFE2:
        .size   main, .-main                                            .size   main, .-main
        .ident  "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0            .ident  "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0
        .section        .note.GNU-stack,"",@progbits                    .section        .note.GNU-stack,"",@progbits
```

The binaries are now executed with different parameters via SLURM. The submit scripts also include an `echo` command to check whether something happens afterwards.

Using more memory than requested and staying below 30 seconds works:

```
#SBATCH --mem=42
~/mem_use 1337 10
echo "done"
---
done
```

Using more memory then requested and running for more than 30 seconds fails:

```
#SBATCH --mem=42
~/mem_use 1337 40
echo "done"
---
slurmstepd: Job 175463 exceeded memory limit (1373120 > 43008), being killed
slurmstepd: Exceeded job memory limit
slurmstepd: *** JOB 175463 ON node017 CANCELLED AT 2018-08-31T13:24:43 ***
```

Allocating more memory than requested and running for more than 30 seconds works:

```
#SBATCH --mem=42
~/mem_do_nothing 1337 40
echo "done"
---
done
```

For completeness sake I also provide the accounting data. There you can see that the job which allocates and uses more memory is killed after 30 seconds. I also tested other timings and the job always got killed at half or full minutes:

```
$ sacct -P -o JobID,Elapsed,REQMEM,MaxRSS,ExitCode -j 175462,175463,175464 | column -s '|' -t
JobID         Elapsed   ReqMem  MaxRSS    ExitCode
175462        00:00:11  42Mn    0:0
175462.batch  00:00:11  42Mn    1140K     0:0
175463        00:00:30  42Mn    0:0
175463.batch  00:00:32  42Mn    1373120K  0:15
175464        00:00:40  42Mn    0:0
175464.batch  00:00:40  42Mn    3624K     0:0
```

TLDR: You can allocate as much memory as you want to, only really used memory counts for SLURM/cgroups. Also you can use as much memory as you want to until the next 30 second interval is reached.
