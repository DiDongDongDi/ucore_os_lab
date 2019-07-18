PROJ	:= challenge
EMPTY	:=
SPACE	:= $(EMPTY) $(EMPTY)
SLASH	:= /

V       := @

# try to infer the correct GCCPREFIX
ifndef GCCPREFIX
GCCPREFIX := $(shell if i386-elf-objdump -i 2>&1 | grep '^elf32-i386$$' >/dev/null 2>&1; \
	then echo 'i386-elf-'; \
	elif objdump -i 2>&1 | grep 'elf32-i386' >/dev/null 2>&1; \
	then echo ''; \
	else echo "***" 1>&2; \
	echo "*** Error: Couldn't find an i386-elf version of GCC/binutils." 1>&2; \
	echo "*** Is the directory with i386-elf-gcc in your PATH?" 1>&2; \
	echo "*** If your i386-elf toolchain is installed with a command" 1>&2; \
	echo "*** prefix other than 'i386-elf-', set your GCCPREFIX" 1>&2; \
	echo "*** environment variable to that prefix and run 'make' again." 1>&2; \
	echo "*** To turn off this error, run 'gmake GCCPREFIX= ...'." 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif

# try to infer the correct QEMU
ifndef QEMU-x
QEMU := $(shell if which qemu-system-i386 > /dev/null; \
	then echo 'qemu-system-i386'; exit; \
	elif which i386-elf-qemu > /dev/null; \
	then echo 'i386-elf-qemu'; exit; \
	else \
	echo "***" 1>&2; \
	echo "*** Error: Couldn't find a working QEMU executable." 1>&2; \
	echo "*** Is the directory containing the qemu binary in your PATH" 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif

# eliminate default suffix rules
.SUFFIXES: .c .S .h

# delete target files if there is an error (or make is interrupted)
.DELETE_ON_ERROR:

# define compiler and flags

HOSTCC		:= gcc
HOSTCFLAGS	:= -g -Wall -O2

CC		:= $(GCCPREFIX)gcc
CFLAGS	:= -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
CTYPE	:= c S

LD      := $(GCCPREFIX)ld
LDFLAGS	:= -m $(shell $(LD) -V | grep elf_i386 2>/dev/null | head -n 1)
LDFLAGS	+= -nostdlib

OBJCOPY := $(GCCPREFIX)objcopy
OBJDUMP := $(GCCPREFIX)objdump

COPY	:= cp
MKDIR   := mkdir -p
MV		:= mv
RM		:= rm -f
AWK		:= awk
SED		:= sed
SH		:= sh
TR		:= tr
TOUCH	:= touch -c

OBJDIR	:= obj
BINDIR	:= bin

ALLOBJS	:=
ALLDEPS	:=
TARGETS	:=

include tools/function.mk

listf_cc = $(call listf,$(1),$(CTYPE))

# for cc
add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
create_target_cc = $(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))

# for hostcc
add_files_host = $(call add_files,$(1),$(HOSTCC),$(HOSTCFLAGS),$(2),$(3))
create_target_host = $(call create_target,$(1),$(2),$(3),$(HOSTCC),$(HOSTCFLAGS))

cgtype = $(patsubst %.$(2),%.$(3),$(1))
objfile = $(call toobj,$(1))
asmfile = $(call cgtype,$(call toobj,$(1)),o,asm)
outfile = $(call cgtype,$(call toobj,$(1)),o,out)
symfile = $(call cgtype,$(call toobj,$(1)),o,sym)

# for match pattern
match = $(shell echo $(2) | $(AWK) '{for(i=1;i<=NF;i++){if(match("$(1)","^"$$(i)"$$")){exit 1;}}}'; echo $$?)

# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
# include kernel/user

INCLUDE	+= libs/

CFLAGS	+= $(addprefix -I,$(INCLUDE))

LIBDIR	+= libs

$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)

# -------------------------------------------------------------------
# kernel

KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

# obj/kern/*/*.o
# 生成kern需要的.o文件, 这些.o文件生成方式和参数均类似, 举init.o为例
# obj/kern/init/init.o
# 编译需要init.c
# 实际命令为
# gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)

# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
# 生成kernel时, makefile的几条指令中有@前缀的都不必需
# 必需的命令只有
#ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
			obj/kern/init/init.o obj/kern/libs/readline.o \
			obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
			obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
			obj/kern/driver/clock.o obj/kern/driver/console.o \
			obj/kern/driver/intr.o obj/kern/driver/picirq.o \
			obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
			obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
			obj/libs/printfmt.o obj/libs/string.o
# 其中新出现的关键参数为
# -T <scriptfile>  让连接器使用指定的脚本

# 为了生成kernel, 首先需要kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o 
# kernel.ld 已存在
# -------------------------------------------------------------------

# create bootblock
# 生成bootasm.o bootmain.o 的相关makefile代码为:
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)
# 实际代码由宏批量生成

# 为了生成bootblock, 首先需要生成bootasm.o bootmain.o sign
#
# 生成bootasm.o需要bootasm.S
# 实际命令为
# gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
# 其中关键的参数为:
# -ggdb 生成可供gdb使用的调试信息. 这样才能用qemu+gdb来调试bootloader or ucore.
# -m32 生成适用于32位环境的代码. 我们用的模拟硬件是32bit的80386, 所以ucore也要是32位的软件.
# -gstabs 生成stabs格式的调试信息. 这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息.
# -nostdinc 不使用标准库. 标准库是给应用程序用的, 我们是编译ucore内核, os内核是提供服务的, 所以所有的服务要自给自足.
# -fno-stack-protector 不生成用于检测缓冲区溢出的代码. 这是for应用程序的, 我们是编译内核, ucore内核好像还用不到此功能.
# -Os 为减小代码大小而进行优化. 根据硬件spec, 主引导扇区只有512字节, 我们写的简单bootloader的最终大小不能大于510字节.
# -I<dir> 添加搜索头文件的路径.
# -fno-builtin 除非用__builtin_前缀, 否则不进行builtin函数的优化.
#
# 生成bootmain.o需要bootmain.c
# 实际命令为
# gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
#
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	# 首先生成bootblock.o
	# ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
	# 其中关键的参数为:
	# -m <emulation> 模拟为i386上的连接器
	# -nostdlib 不使用标准库
	# -N 设置代码段和数据段均可读写
	# -e <entry> 指定入口
	# -Ttext 制定代码段开始位置
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	# 拷贝二进制代码bootblock.o到bootblock.out
	# objcopy -S -O binary obj/bootlblock.o obj/bootblock.out
	# 其中关键的参数为:
	# -S 移除所有符号和重定位信息
	# -O <bfdname> 指定输出格式
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	# 使用sign工具处理bootblock.out, 生成bootblock
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)

# -------------------------------------------------------------------

# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
# 实际命令为
# gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
# gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign

# -------------------------------------------------------------------

# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

# 为了生成ucore.img, 首先需要生成bootblock kernel
$(UCOREIMG): $(kernel) $(bootblock)
	# 生成一个有10000个块的文件, 每个块默认512字节, 用0填充
	$(V)dd if=/dev/zero of=$@ count=10000
	# 把bootblock中的内容写到第一个块
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	#从第二个块开始写kernel中的内容
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

$(call finish_all)

IGNORE_ALLDEPS	= clean \
				  dist-clean \
				  grade \
				  touch \
				  print-.+ \
				  handin

ifeq ($(call match,$(MAKECMDGOALS),$(IGNORE_ALLDEPS)),0)
-include $(ALLDEPS)
endif

# files for grade script

TARGETS: $(TARGETS)
all: $(TARGETS)
.DEFAULT_GOAL := TARGETS

.PHONY: qemu qemu-nox debug debug-nox
lab1-mon: $(UCOREIMG)
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/lab1init"
debug-mon: $(UCOREIMG)
#	$(V)$(QEMU) -S -s -monitor stdio -hda $< -serial null &
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -monitor stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/moninit"
qemu-mon: $(UCOREIMG)
	$(V)$(QEMU) -monitor stdio -hda $< -serial null
qemu: $(UCOREIMG)
	$(V)$(QEMU) -parallel stdio -hda $< -serial null

qemu-nox: $(UCOREIMG)
	$(V)$(QEMU) -serial mon:stdio -hda $< -nographic
TERMINAL        :=gnome-terminal
gdb: $(UCOREIMG)
	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null
debug: $(UCOREIMG)
	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
	$(V)sleep 2
	$(V)$(TERMINAL)  -e "cgdb -q -x tools/gdbinit"
	
debug-nox: $(UCOREIMG)
	$(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"

.PHONY: grade touch

GRADE_GDB_IN	:= .gdb.in
GRADE_QEMU_OUT	:= .qemu.out
HANDIN			:= proj$(PROJ)-handin.tar.gz

TOUCH_FILES		:= kern/trap/trap.c

MAKEOPTS		:= --quiet --no-print-directory

grade:
	$(V)$(MAKE) $(MAKEOPTS) clean
	$(V)$(SH) tools/grade.sh

touch:
	$(V)$(foreach f,$(TOUCH_FILES),$(TOUCH) $(f))

print-%:
	@echo $($(shell echo $(patsubst print-%,%,$@) | $(TR) [a-z] [A-Z]))

.PHONY: clean dist-clean handin packall
clean:
	$(V)$(RM) $(GRADE_GDB_IN) $(GRADE_QEMU_OUT)
	-$(RM) -r $(OBJDIR) $(BINDIR)

dist-clean: clean
	-$(RM) $(HANDIN)

handin: packall
	@echo Please visit http://learn.tsinghua.edu.cn and upload $(HANDIN). Thanks!

packall: clean
	@$(RM) -f $(HANDIN)
	@tar -czf $(HANDIN) `find . -type f -o -type d | grep -v '^\.*$$' | grep -vF '$(HANDIN)'`

