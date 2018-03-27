# linux内核启动调试过程报告

>cpu:i7-6820hk
>
>ram:32GB
>
>system:ubuntu-mate 17.10

1. 我们从[此处](http://www.kernel.org/pub/linux)下载了linux4.9的内核，解压

2. 我们对内核进行编译，首先打开编译菜单

   ```shell
   make menuconfig
   ```

   将其中的以下功能开启：

   Kernel debugging

   > 内核的基本调试开关

   Debug low-level entry code

   > 在内核的标准错误缓冲区还未载入时提供早期的错误输出

   Complie the kernal with debug info

   >编译内核时携带调试信息

3. 保存设置为.config文件，运行

   ```shell
   make
   ```

   进行内核编译，提示缺少文件

   ```shell
   curses.h：No such file or directory
   ```

   于是安装依赖项

   ```shell
   sudo apt-get install libncurses5-dev libncursesw5-dev
   ```

   再次make，提示

   ```shell
   include/linux/compiler-gcc.h:106:30: fatal error: linux/compiler-gcc7.h：No such file or directory
   ```

   检查知./include/linux下存在的compiler-gcc*.h文件决定了可用来编译的gcc版本，\*只有3/4/5，但本机内置的gcc版本为7.2.0，因此我们需要安装gcc-5，并临时设置高于gcc7的优先级

   ```shell
   sudo apt install gcc-5
   sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 50
   sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 30
   ```

   再次make，等待大约1小时编译完成

4. 我们首先建立一个有效的虚拟机镜像文件OS.raw

   ```shell
   qemu-img create -f raw OS.img 10G
   ```

   尝试从qemu启动虚拟机

   ```shell
   qemu-system-x86_64 -kernel linux-4.9/arch/x86/boot/bzImage -hda OS.img -m 2048M
   ```

   内核基本正常启动，但在启动过程中卡在：

   ```shell
   end kernal panic - not syncing: VFS: Unable to mount root fs on unknown-block(0, 0)
   ```

   认为是没有指定root根文件系统，启动命令修改为

   ```shell
   qemu-system-x86_64 -kernel linux-4.9/arch/x86/boot/bzImage -hda OS.img -enable-kvm -m 2048M -append "root=/dev/sda" -cpu host -smp cores=2
   ```

   错误信息变为：

   ```shell
   end kernal panic - not syncing: No working init found
   ```

   那么，我们需要加入init启动点，于是，我们编译安装busybox作为init程序

   编译过程，报错：

   ```shell
   loginutils/passwd.c: In function ‘passwd_main’:
   loginutils/passwd.c:104:16: error: storage size of ‘rlimit_fsize’ isn’t known
   loginutils/passwd.c:188:2: warning: implicit declaration of function ‘setrlimit’ [-Wimplicit-function-declaration]
   loginutils/passwd.c:188:12: error: ‘RLIMIT_FSIZE’ undeclared (first use in this function)
   loginutils/passwd.c:188:12: note: each undeclared identifier is reported>for each function it appears in
   loginutils/passwd.c:104:16: warning: unused variable ‘rlimit_fsize’ [-Wunused-variable]
   ```

   查询相关资料后，修改busybox源代码./include/libbbb.h，增加一行

   ```c
   #include<sys/resource.h>
   ```

   即可编译成功

5. 挂载，将内核写入镜像

   ```shell
   mkdir img
   sudo mount -o loop ./OS.img ./img
   sudo make modules_install INSTALL_MOD_PATH='/home/star/Downloads/img'
   ```

   将busybox写入镜像

   ```shell
   make CONFIG_PREFIX='/home/star/Downloads/img' install
   ```

   用以下指令启动：

   ```shell
   qemu-system-x86_64 -kernel ./linux-4.9/arch/x86/boot/bzImage -hda OS.img -enable-kvm -m 2048M -append "root=/dev/sda init=/linuxrc" -cpu host -smp cores=2
   ```

   顺利挂载文件系统，但是出现以下错误：

   ```shell
   can't open /dev/tty2: no such file or directory
   can't open /dev/tty3: no such file or directory
   can't open /dev/tty4: no such file or directory
   ```

   查询相关资料后，新建/etc，/etc/inittab，/etc/init.d，/etc/init.d/rcS

   inittab内容：

   ```
   ::sysinit:/etc/init.d/rcS
   ::askfirst:/bin/ash
   ::ctrlaltdel:/sbin/reboot
   ::shutdown:/sbin/swapoff -a
   ::shutdown:/bin/umount -a -r
   ::restart:/sbin/init
   ```

   rcS内容：

   ```shell
   #!/bin/sh
   mount -t proc proc /proc
   mount -t sysfs sysfs /sys
   ```

   再次用原命令启动后正常进入sh

6. 使用gdb调试内核

   将启动命令修改为

   ```shell
   qemu-system-x86_64 -kernel ./linux-4.9/arch/x86/boot/bzImage -hda OS.img -enable-kvm -m 2048M -append "root=/dev/sda init=/linuxrc" -cpu host -smp cores=2 -s -S
   ```

   > -s 打开qemu内置的gdb server并设置端口为1234
   >
   > -S冻结虚拟机直至收到gdb调试信号

   另开一个shell，输入

   ```shell
   gdb << EOF
   file ./vmlinux
   target remote :1234
   b kernel_init
   b rest_init
   c
   ```

   使用list命令得到rest_init函数内容

   ```c
   static noinline void __ref rest_init(void)
   {
   	int pid;

   	rcu_scheduler_starting();
   	/*
   	 * We need to spawn init first so that it obtains pid 1, however
   	 * the init task will end up wanting to create kthreads, which, if
   	 * we schedule it before we create kthreadd, will OOPS.
   	 */
   	kernel_thread(kernel_init, NULL, CLONE_FS);
   	numa_default_policy();
   	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
   	rcu_read_lock();
   	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
   	rcu_read_unlock();
   	complete(&kthreadd_done);

   	/*
   	 * The boot idle thread must execute schedule()
   	 * at least once to get things moving:
   	 */
   	init_idle_bootup_task(current);
   	schedule_preempt_disabled();
   	/* Call into cpu_idle with preempt disabled */
   	cpu_startup_entry(CPUHP_ONLINE);
   }
   ```

   kernel_thread首先生成了一个pid=1的进程，在kthreadd中，该线程被设定为当cpu空闲时一直运行，而当有其他线程时让出cpu。所有其他的进程都由这个进程生成。而init_idle_bootup_task则生成了一个空闲线程，并在schedule_preempt_disabled中将其设置为不可调度，用其作为cpu是否空闲的依据。

   使用list命令得到kernel_init函数内容

   ```c
   static int __ref kernel_init(void *unused)
   {
   	int ret;

   	kernel_init_freeable();
   	/* need to finish all async __init code before freeing the memory */
   	async_synchronize_full();
   	free_initmem();
   	mark_readonly();
   	system_state = SYSTEM_RUNNING;
   	numa_default_policy();

   	flush_delayed_fput();

   	rcu_end_inkernel_boot();

   	if (ramdisk_execute_command) {
   		ret = run_init_process(ramdisk_execute_command);
   		if (!ret)
   			return 0;
   		pr_err("Failed to execute %s (error %d)\n",
   		       ramdisk_execute_command, ret);
   	}

   	/*
   	 * We try each of these until one succeeds.
   	 *
   	 * The Bourne shell can be used instead of init if we are
   	 * trying to recover a really broken machine.
   	 */
   	if (execute_command) {
   		ret = run_init_process(execute_command);
   		if (!ret)
   			return 0;
   		panic("Requested init %s failed (error %d).",
   		      execute_command, ret);
   	}
   	if (!try_to_run_init_process("/sbin/init") ||
   	    !try_to_run_init_process("/etc/init") ||
   	    !try_to_run_init_process("/bin/init") ||
   	    !try_to_run_init_process("/bin/sh"))
   		return 0;

   	panic("No working init found.  Try passing init= option to kernel. "
   	      "See Linux Documentation/init.txt for guidance.");
   }
   ```

   我们注意到了之前发生过的错误的来源，显然内核试图从"/sbin/init"，"/etc/init"，"/bin/init"，"/bin/sh"和用户指定的参数五个位置启动init进程，如果不成功会抛出错误。