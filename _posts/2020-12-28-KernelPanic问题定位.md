---
layout:     post
title:      Kernel Panic
subtitle:   Kernel Panic问题定位
date:       2020-12-28
author:     Jeff
header-img: img/bg_wolf.jpg
catalog: true
tags:
    - Kernel
    - Panic
    - 
---

>Kernel Panic问题定位

# 问题描述一
<mark>严格说这个不属于panic只是一个warn ,仅以此引出问题定位方法</mark>

	[    3.204767] ------------[ cut here ]------------
	[    3.209696] WARNING: CPU: 5 PID: 1 at drivers/irqchip/irq-gic-v3.c:1031 gic_irq_domain_translate+0x150/0x158
	[    3.219572] Modules linked in:
	[    3.222702] CPU: 5 PID: 1 Comm: swapper/0 Not tainted 4.19.113 #1
	[    3.228847] Hardware name: Qualcomm Technologies, Inc. BENGAL IDP (DT)
	[    3.235432] pstate: 60400005 (nZCv daif +PAN -UAO)
	[    3.240271] pc : gic_irq_domain_translate+0x150/0x158
	[    3.245376] lr : gic_irq_domain_translate+0x14c/0x158
	[    3.250477] sp : ffffff800805b690
	[    3.253837] x29: ffffff800805b690 x28: ffffff8009fb90a0 
	[    3.259229] x27: 0000000000000000 x26: ffffff8009e721d8 
	[    3.264621] x25: 0000000000000001 x24: ffffffc0f6050800 
	[    3.270010] x23: 00000000006080c0 x22: ffffffc005ff7380 
	[    3.275399] x21: ffffff800805b6d0 x20: ffffff800805b6cc 
	[    3.280788] x19: ffffff800805b778 x18: 0000000000000034 
	[    3.286177] x17: ffffff800a805000 x16: 000000000000003a 
	[    3.291565] x15: 0000000000000000 x14: ffffff800a0c7018 
	[    3.296954] x13: ffffff800a546c48 x12: ffffff800815fd7c 
	[    3.302342] x11: 0000000000000015 x10: 000000000682aaab 
	[    3.307730] x9 : 959647ed40564800 x8 : 959647ed40564800 
	[    3.313118] x7 : 5d20657265682074 x6 : ffffff800a80748c 
	[    3.318507] x5 : 0000000000000000 x4 : 0000000000000000 
	[    3.323895] x3 : ffffff800805b308 x2 : ffffff8008097668 
	[    3.329283] x1 : ffffff800815fd7c x0 : 0000000000000000 
	[    3.334673] Call trace:
	[    3.337165] gic_irq_domain_translate+0x150/0x158
	[    3.341918] gic_irq_domain_alloc+0x58/0x260
	[    3.346243] irq_domain_alloc_irqs_parent+0x44/0x60
	[    3.351174] msm_mpm_gic_chip_alloc+0xe8/0x168
	[    3.355670] __irq_domain_alloc_irqs+0x1cc/0x398
	[    3.360339] irq_create_fwspec_mapping+0x1f4/0x358
	[    3.365180] irq_create_of_mapping+0x64/0x90
	[    3.369506] of_irq_to_resource+0xb4/0x1b0
	[    3.373653] of_irq_to_resource_table+0x3c/0x78
	[    3.378234] of_device_alloc+0x11c/0x1e0
	[    3.382207] of_platform_device_create_pdata+0x74/0x118
	[    3.387483] of_platform_bus_create+0x28c/0x500
	[    3.392063] of_platform_bus_create+0x2f0/0x500
	[    3.396643] of_platform_populate+0x94/0x120
	[    3.400966] of_platform_default_populate_init+0xac/0xc4
	[    3.406332] do_one_initcall+0x120/0x2a0
	[    3.410308] kernel_init_freeable+0x304/0x3b8
	[    3.414720] kernel_init+0x18/0x298
	[    3.418257] ret_from_fork+0x10/0x1c
	[    3.421893] ---[ end trace 1b6069d60cec4e1a ]---
	[    3.428355] ------------[ cut here ]------------

# 问题分析：

<mark>[    3.337165] gic_irq_domain_translate+0x150/0x158</mark>

	这个trace的含义是：运行到函数：gic_irq_domain_translate 的偏移 0x150 处发生warning，0x158是函数大小

# 通过符号表定位：
	
	$ grep gic_irq_domain_translate System.map
	  ffffff80085bb9d8 t gic_irq_domain_translate
	  ffffff80085bcb10 t gic_irq_domain_translate
	 ##发现搜到两个，是因为有两个地方都各自实现了一个static gic_irq_domain_translate 函数，通过源码确认用的第二个
	 ffffff80085bcb10 + 0x150 =  ffffff80085bcc60
	 因此我们找到了执行地址

# GDB定位：

	gdb vmlinux
	(gdb) b *0xffffff80085bcc60
	Breakpoint 2 at 0x85bcc60: file /media/android/kernel/msm-4.19/drivers/irqchip/irq-gic-v3.c, line 1030.
	
![](https://jeffnoteimgs.oss-cn-shanghai.aliyuncs.com/imgs20201228100056.png)

	确实在irq-gic-v3.c 的 1030 行有一个WARN_ON被触发


### Note:
<mark>System.map 和 vmlinux 都是和发生问题的版本一致</mark>
