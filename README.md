# CHERI exercise

This is a small exercise to get started with CHERI (on RISC-V, QEMU emulation). 

## Recommended setup

While QEMU with CHERI-RISC-V should run on most Linux/Unix/Mac platforms, we recommend using Ubuntu 18.04 - if needed you can do this using a VM (for example from https://www.osboxes.org/ubuntu/#ubuntu-1804-vbox). Note that the tools take a while to build (several hours depending on CPU etc), so plan in some time to wait for the compilation to finish.

## Resources

The following resources by the CHERI team from Cambridge are useful:

 * The getting started guide, including installation instructions for the emulator etc: https://ctsrd-cheri.github.io/cheri-exercises/cover/index.html 
 * The `cheribuild` tool: https://github.com/CTSRD-CHERI/cheribuild.git
 * How to copy files in/out of QEMU to the host is documented here: https://github.com/CTSRD-CHERI/cheri-exercises/pull/26
   Essentially, you can simply use `mount_smbfs -I 10.0.2.4 -N //10.0.2.4/source_root /mnt` in the QEMU guest, which will mount the CHERI base directory of the host on `/mnt`.

Also, if you ever need to exit QEMU: press `Ctrl-a` then release and press `x`   

## Task

We will use a (slightly modified) exercise from https://github.com/CTSRD-CHERI/cheri-exercises/

 * Fork this repository here (*not* the CHERI exercise one) - we expect you to add your solutions in this README.md where it says *INSERT SOLUTION HERE*. Please make sure you do reasonable commits and commit messages. You can also use other features of Github e.g. issues.
 
 * Assuming that you have installed CHERI-RISC-V in `~/cheri`, make sure your forked repo is cloned to `~/cheri/riscv-exercise`
 
 ### Task One
 
 * Compile `buffer-overflow.c` to a RISC-V binary `buffer-overflow-hybrid` in hybrid capability mode (`riscv64-hybrid`). You can use the `ccc` script from `task/tools` (see the exercise docs for details) for that. What is the full commandline for compilation? 

We can invoke the the `ccc` script with the below command:

 ```bash
 sh ~/cheri/riscv-exercise/task/tools/ccc riscv64-hybrid ~/cheri/riscv-exercise/task/buffer-overflow.c -o buffer-overflow-hybrid
 ```

The full commandline for actual compilation is: 

```bash
/home/osboxes/cheri/output/sdk/bin/clang -target riscv64-unknown-freebsd -march=rv64gcxcheri -mabi=lp64d -mno-relax --sysroot=/home/osboxes/cheri/output/sdk/sysroot-riscv64-hybrid/ -g -O2 -fuse-ld=lld -Wall -Wcheri buffer-overflow.c -o buffer-overflow-hybrid
```

If we take the recommended route of using aliases:

```bash
SYSROOT=~/cheri/output/sdk/sysroot-riscv64-hybrid/; export SYSROOT
CLANG=~/cheri/output/sdk/bin/clang; export CLANG
```

We could manually compile the code without using the provided `ccc` script as so: 

```bash
$CLANG -g -O2 -target riscv64-unknown-freebsd --sysroot="$SYSROOT" -fuse-ld=lld -mno-relax -march=rv64gcxcheri -mabi=l64pc128d -Wall -Wcheri buffer-overflow.c -o buffer-overflow-hybrid
```

*N.b. what is the importance between the ccc scipt using `-mabi=lp64d` vs the recommended CheriABI flag of `-mabi=l64pc128d`?  
It seems the `lp64d` is specifically for conventional RISC-V, I assume this to allow for hybrid capability rather than purecap.*

---

### Task Two

* There is a security flaw in `buffer-overflow.c`. Briefly explain what the flaw is: 

As the filename aptly suggests, the flaw is a buffer overflow contained within the code where the `strcpy` function fails to bounds-check the the input argument prior to copying it into the buffer held for `Arg`.  
  
Due to the structure of the code whereby the buffer is initialised prior to the char `c` variable, the compiled code would place the initialised `c` next to the buffer in memory. Subsequently, any overflowing bytes would overwrite `c`.  
  
Specifically, the buffer holds 16 bytes which allows for 15 characters and the string's null termination found on line 65, however, as the `strcpy` is on the line preceeding this string termination, C language will allow it to fill the subsequent memory.  
  
In 64-bit architecture, memory is addressed in 8-byte increments and with the buffer being 16-bytes, and allowing for an 8-byte increment, this means the `c` variable will be 24-bytes from the start of the buffer, therefore by inputting a 24-byte string as an argument this will overwrite the `c` variable. 

---

### Task Three

* Start CHERI-RISC-V in QEMU, copy `buffer-overflow-hybrid` to the QEMU guest, and run it with a commandline argument that triggers the mentioned security flaw to overwrite the variable `c` with an attacker-controlled value. Give all the commands you have to run (assuming CHERI is in `~/cheri` and cheribuild in `~/cheribuild`):

```bash
python3 cheribuild.py run-riscv-hybrid #enter the cheribsd-hybrid system
mount_smbfs -I 10.0.2.4 -N //10.0.2.4/source_root /mnt #mount the shared /cheri folder from the host to /mnt
/mnt/riscv-exercise/task/buffer-overflow-hybrid 012345678901234567890123 # 24-character input
c = c #prior to the strcpy function
Arg = 012345678901234 #cut to 15-bytes to allow the null-termination on the 16th
c = 3 #the 24th byte '3' has overwritten c
```

---

### Task Four

* Now, compile the same program in pure capability mode (`riscv64-purecap`) to `buffer-overflow-purecap`. What happens when you run this program in QEMU with the same input that triggered the flaw in `buffer-overflow-hybrid`? Explain why this happens!

Firstly, to assert that the program works under generally expected input:  

```bash
python3 cheribuild.py run-riscv-purecap #enter the cheribsd-hybrid system
mount_smbfs -I 10.0.2.4 -N //10.0.2.4/source_root /mnt #mount the shared /cheri folder from the host to /mnt
/mnt/riscv-exercise/task/buffer-overflow-purecap 012345678901234 # 15-character input
c = c #prior to the strcpy function
Arg = 012345678901234 # 15-bytes to allow the null-termination on the 16th
c = c #the 24th byte '3' has overwritten c
```

Secondly, to assess whether the purecap version works with a greater than 15-byte input (fifteen characters plus the obliged null byte):  

```bash
~/work/riscv-exercise/task/buffer-overflow-purecap 0123456789012345
c = c
In-address space security exception (core dumped)
```

And for posterity, a 24-byte input:  

```bash
~/work/riscv-exercise/task/buffer-overflow-purecap 012345678901234567890123
c = c
In-address space security exception (core dumped)
```

Using the diagnostic message tool `dmesg` provides us with the following messages:  

`
pid 751 (buffer-overflow-pur), jid 0, uid 0: exited on signal 34
  
pid 753 (buffer-overflow-pur), jid 0, uid 0: exited on signal 34`

From the host, we are able to determine that to which this signal refers with the following command:  

```bash
cat /home/osboxes/cheri/output/sdk/sysroot-riscv64/usr/include/sys/signal.h | grep 34
#define SIGPROT   34  /* in-address space security exception. */
```

In order to identify and confirm the exact point of failure, we can process the coredump file with `gdb` using the following command: 

```bash
/mnt/build/gdb-riscv64-hybrid-build/gdb/gdb buffer-overflow-purecap -c buffer-overflow-pur.core
```

The output: 

```bash
Core was generated by `./buffer-overflow-purecap 012345678901234567890123'.
Program terminated with signal SIGPROT, CHERI protection violation
Capability bounds fault caused by register ca2.
#0  0x000000004033baf6 in strcpy (
    to=0x3fffdfff3f [rwRW,0x3fffdfff2f-0x3fffdfff3f] "c\200\367\337\277?", 
    from=0x3fbfdffd3a [rwRW,0x3fbfdffd2a-0x3fbfdffd43] "67890123")
    at /home/osboxes/cheri/cheribsd/lib/libc/string/strcpy.c:54
```

It is clear that the offending function was `strcpy.c:54`, that is `strcpy` on line 54 of the sourcecode. Further, `gdb` has identified the offending characters as `67890123` (bytes 17-24, i.e. > 16-byte buffer) from the input argument due to a `Capability bounds fault`.   

This is a result of pure Cheri, where all pointers are wrapped within the Cheri capability architecture including a bound-checking function to allow for spacial protection.


#### Permissions

**Important**: The steps involved with task four must not completed within the `/mnt` folder as we will not have the necessary permissions to actually write the coredump, and it will not be produced.  

Should you wish to access the coredump, copy the compiled binary to one where you have necessary permissions, i.e. `~/`
