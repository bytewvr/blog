---
layout: post
title: GDBing the kernel
category: linux 
date: 2022-09-20
tags:
  - symbol table
  - GDB
  - kernel
  - ASLR
---
A short discussion about how to restore the symbol table of the Linux kernel `vmlinux`. To allow matching with addresses in the text segment I account for the address offset introduced by ASLR between the source and running kernel. With GDB set up like this I find struct member offsets. 
<!--more-->
# Dressing the kernel executable
The compressed kernel image `/boot/vmlinuz-<kernel version>` does not result in a kernel executable with a symbol table upon decompression. 
```
PROMPT> sudo cp /boot/vmlinuz-$(uname -r) .
PROMPT> sudo /usr/src/linux-headers-$(uname -r)/scripts/extract-vmlinux vmlinuz-$(uname -r) > decomp-vmlinuz
```
Instead of returning symbol addresses as below, we are greeted with emptiness. 
```
PROMPT> readelf -s vmlinux | head

Symbol table '.symtab' contains 158920 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: ffffffff81000000     0 SECTION LOCAL  DEFAULT    1 .text
     2: ffffffff82000000     0 SECTION LOCAL  DEFAULT    3 .rodata
     3: ffffffff825ce0c0     0 SECTION LOCAL  DEFAULT    5 .pci_fixup
     4: ffffffff825d1750     0 SECTION LOCAL  DEFAULT    7 .tracedata
     5: ffffffff825d17c8     0 SECTION LOCAL  DEFAULT    9 .printk_index
     6: ffffffff825e51a0     0 SECTION LOCAL  DEFAULT   11 __ksymtab
```
To obtain a kernel with the symbol table we can compile the kernel ourselves, check whether our distro kernel sources ship an unstripped vmlinux, and/or dress our decompressed kernel executable. 

## check sources
If you have the sources installed, you might have access to a non stripped kernel.
```
PROMPT> readelf -s /usr/src/linux/vmlinux | grep update_curr$
  14273: ffffffff810e5710   462 FUNC    LOCAL  DEFAULT    1 update_curr
```
## compile kernel
We can also just compile the kernel, following along [instructions](https://wiki.archlinux.org/title/Kernel/Traditional_compilation) shipped by our distro of choice. Nice to see yet another use of the `/proc` file system that also stores the kernel config file: `/proc/config.gz`. You can decompress and have a peek at your current kernel compilation options with `zcat /proc/config.gz > .config`. To compile the kernel I used the multiprocessor support of make by `make -j10`. After the compilation finishes, `vmlinux` will be left in the source directory. 

# Finally, getting out GDB
Having access to the symbols helps for disassembly and/or allows us to obtain offsets of members in structures used in the kernel. 
```
PROMPT> gdb vmlinux
gdb> ptype struct cfs_rq
  type = struct cfs_rq {
	struct load_weight load;
    unsigned int nr_running;
    unsigned int h_nr_running;
  ...
gdb> p (int)&((struct cfs_rq *)0)->nr_running
$1 = 16
```
However, if `/sys/kernel/btf/vmlinux` exists, structs can be more easily explored with  `pahole`. 

## Making sense of ASLR
Using any of our above kprobes to hook into `update_curr` we obtain its call address `0xffffffff92ee5710`. 
```
sleep-5755  [000]  4879.035481: uc_other:             (ffffffff92ee5710) nr_switches=1982607 curr_pid=5755 curr_tgid=5755 curr_weight=1048576 inv_weight=4194304
```
However, only the last 5 nibbles match with where we expect it to be according to our executable. And furthermore, the above address will change between reboots of the system. 
```
PROMPT> readelf -s vmlinux | grep update_curr$
 14635: ffffffff810e5690   462 FUNC    LOCAL  DEFAULT    1 update_curr
```
The reason for this randomized discrepancy is ASLR - a scheme to to protect against buffer overflow attacks. However, the offset is the same for all symbols and hence we can adjust GDB to get a match. Instead of running our kprobe we can just check for symbol position inside of `/proc/kallsyms` . `kallsyms` is created upon kernel boot and represents kernel data. As root we can obtain the same address as with the kprobe above
```
PROMPT> sudo grep update_curr$ /proc/kallsyms
  ffffffff92ee5710 t update_curr
```
Here `t` denotes that the address of the `update_curr` call is in the text section of the executable. Other symbol types are
| type  | description                                |
| ----- | ------------------------------------------ |
| A     | absolute                                   |
| B / b | unitiliazed data section (BSS)             |
| D / d | initialized data section                   |
| G / g | initialized data section for small objects |
| i     | section specific DLLs                      |
| N     | debugging symbol                           |
| p     | stack unwind section                       |
| R / r | read only data section                     |
| S / s | unitialized data section for small objects |
| T / t | text (code)                                |
| U     | undefined                                  |
| V / v | weak object                                |
| -     | stab symbols in a.out file                 |
| W / w | untagged weak object                       |
| ?     | unknown                                    |
The text section (code) itself starts at
```
PROMPT#> sudo grep -w _stext /proc/kallsyms
ffffffff92e00000 T _stext
```
We can just adjust our symbol table
```
PROMPT> gdb /usr/src/linux/vmlinux
(gdb) p &update_curr
  $1 = (<text variable, no debug info> *) 0xffffffff810e5710 <update_curr>
(gdb) add-symbol-file /usr/src/linux/vmlinux -s .text 0xffffffff92e00000
(gdb) p &update_curr
$2 = (<text variable, no debug info> *) 0xffffffff92ee5710 <update_curr>
```
The address of the `update_curr` call now matches with the value reported by the previous `kprobe` and `/proc/kallsyms` - cool. We can also use GDB to navigate a raw memory dump of the Linux kernel presented as an ELF core file at `/proc/kcore` . 
```
PROMPT> gdb -q -c /proc/kcore
(gdb) add-symbol-file vmlinux -s .text 0xffffffff92e00000
(gdb) p &update_curr
  $1 = (void (*)(struct cfs_rq *)) 0xffffffff92ee5690 <update_curr>
```
Other section addresses have to be specified separately, an example for accomplishing that for the data section can be seen [here](https://stackoverflow.com/a/69873364). 


