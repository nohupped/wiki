tftp
服务:
	sudo apt-get install tftpd-hpa tftp-hpa
restart:
	sudo /etc/init.d/openbsd-inetd restart
	sudo in.tftpd –l /tftpboot


	打开/etc/default/tftpd-hpa它的配置文件。
	#Defaults for tftpd-hpa
	RUN_DAEMON="no"
	OPTIONS="-l -s /var/lib/tftpboot"
	修改设置如下：
	#Defaults for tftpd-hpa
	RUN_DAEMON="yes"
	OPTIONS="-l -s /home/zdreamx/tftpboot"
	其中/home/zdreamx/tftpboot是自己设定的目录，可以根据情况修改。

service tftp
{
        socket_type    = dgram
        protocol       = udp
        wait           = yes
        user           = root
        server         = /usr/sbin/in.tftpd
        server_args    = -s /var/tftpboot/
        disable        = no
        per_source     = 11
        cps            = 100 2
        flags          = IPv4
}
	sudo /etc/init.d/xinetd start


	
u－boot>setenv ipaddr 192.168.1.100 ＃设定目标板ip 
u－boot>setenv serverip 192.168.1.106 ＃主机ip 
u－boot>setenv ethaddr 00:00:00:00:ff:01 ＃设定目标板mac地址，可以其它 
u－boot>saveenv ＃保存环境变量

setenv gatewayip 192.168.1.1	#设置网关ip

mtd write vmlinux52 /dev/mtd1

52-56
=====
juson 56，52系的板子上，52的系统是烧到flash的，56的业务*.bin是在52上
下载到内存启动的。
================================================================================
Octeon Boots from Onboard Flash:
	1. Core 0 starts execution at the reset vector 0xBFC0 0000(the location
	   of the bootloader code in onboard flash).
	2. Initialize the UART
	3. Configures the DRAM controller to allow physical memory to be used.(
	   the UART provides the target console and debug console.)
	4. Relocates itself from the onboard flash to DRAM,and continues execut-
 	   ing from DRAM.
	5. Executes the default command,if present.
================================================================================
Octeon-U_Boot:	$exec,load,link,tlb,$
		$early initialize, relocate(copy) TLB Link, go on$
	Virtual Memory:		讲虚拟地址描述：
	U-Boot executes in two phases.During the first phase it executes out  of
	flash or from a RAM address chosen by the host Linux command oct-remote-
	boot.After the early initalization in U-Boot is complete it relocate it-	经过初始化(TLB..)后,重定位.
	self to a new RAM location and begins executing from the new location.		重定位后执行，利用虚拟地址.
	The standard MIPS U-Boot normally executes out of KSEG0 which consists 		标准Uboot是使用kseg0或kseg1虚拟地址
	of the address range 0x8000 0000 through 0x9fff ffff or KSEG1 which con-
	sists of the address range 0xA000 0000 through 0xBFFF FFFF.Both of these	这些都映射到低512MB物理地址.
	address rangs maps to physical address 0x0000 0000 through 0x1fff ffff.
	On the OCTEON processor,physcial address 0x1000 0000 through 0x1fff ffff	octeon中物理地址256-512留给boot bus
	are reserved for the boot bus.This effectively limits U-Boot to being l-	so: octeon uboot被限制在低256MB
	oaded in the first 256MN of RAM.						U - Boot is loaded.
	The OCTEON U-Boot is linked at address 0xC0000000 (SSEG) instead of its 	U-Boot is Linked.
	address of its address in flash.It relies on mapped memory so that 0xC00	uboot被连接到sseg段，通过映射指向uboot,
	0 0000 will always map to the physical address where U-Boot is executing	所以0xc0000000总是指向uboot开始执行的地址.
	from.By using this mapping memory the OCTEON U-Boot is not restricted to	通过映射,octeon uboot将不会被限制在
	a single location in flash nor is it restricted to the lower 256MB of ph	一个单独位置,也不会被限制大小.
	ysical memory.In fact,OCTEON U-Boot is not limited to 32-bit physcial a-
	ddress.
	TLB:
	Before any C code executes,the assembly code in arch/mips/cpu/octeon/st-	难道C才会用到虚拟地址？
	art.S clears the TLB and initializes the last entry to map two 4MB pages	重定位之后，会映射一个(4MB size)TLB表.
	at the beginning of SSEG to the physical address U-Boot is currently ex-
	ecuting from.

	Relocation and 64-bit Addressing:
	After U-Boot performs some early initialization where it typically exec-
	utes out of flash,it must copy itself to RAM.
	By default,the standard U-Boot copies itself to a fixed location near t-
	he bottom of physcial memory then updates the ELF GOT to the new address
	in KSEG0.
	The OCTEON U-Boot instead copies itself to the top of physcial memory a-	octeon把uboot复制到,物理的顶端地址,
	nd updates the TLB entry to map the new physical address to the virtual		更行TLB,映射新的物理地址到SSEG段首.
	address 0xC000 0000 in SSEG.It does not need to modify the GOT or perfo-
	rm any other relocation operations except for copying the gd and bd data
	structures.
	Since the TLB is used,the physcial address can be located anywhere and 
	are no longer restricted to the lower 256MB of RAM.This allow for more 
	of KSEG0 to be avaliable for other purposes.
	
Virtual Memory Example:
	127:Virtual = 0xffff ffff c000 0000
		page0 = 0x20f800000,C=0,D=1,V=1,G=1,RI=0,XI=0
		page1 = 0x20fc00000,C=0,D=1,V=1,G=1,RI=0,XI=0
		ASID = 0 Size = 4096KB
	In this case,the last TLB entry(127) is used and two 4MB pages are crea-
	ted.
	C. indicates that the page is cacheable and coherent.
	D. indicates that the dirty bit is set which allows the page to be writ-
	   eable.
	V. indicates that the page is valid.
	G. indicates that the page is global and not restricted to an ASID.
	RI.indicates that the page is not read inhibited.
	XI.indicates that the page is not executeable inhibited.
	means:
	0xC000 0000 -> 0x20F8 0000
	0xC040 0000 -> 0x20FC 0000
---------------------------
大体流程：
	nand 偏上执行指令，把Uboot自身拷入内存执行，download ELF，load ELF映射ELF到
	内存(octeon-linux-boot命令等执行此操作)
