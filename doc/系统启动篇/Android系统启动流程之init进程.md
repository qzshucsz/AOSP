## 前言
上一篇中讲到，Linux系统执行完初始化操作最后会执行根目录下的init文件，init是一个可执行程序，它的源码在platform/system/core/init/init.cpp，本篇主要讲init进程相关的一些东西

本文主要讲解以下内容：

- idle进程启动
- kthreadd进程启动
- init进程启动

本文涉及到的文件
```
platform/system/core/init/init.cpp
```

#### 1.main
```
/*
 * 1.C++中主函数有两个参数，第一个参数argc表示参数个数，第二个参数是参数列表，也就是具体的参数
 * 2.init的main函数有两个其它入口，一是参数中有ueventd，进入ueventd_main,二是参数中有watchdogd，进入watchdogd_main
 */
int main(int argc, char** argv) {

    /*
     * 1.strcmp是String的一个函数，比较字符串，相等返回0
     * 2.C++中0也可以表示false
     * 3.basename是C库中的一个函数，得到特定的路径中的最后一个'/',后面的内容，
     * 比如/sdcard/miui_recovery/backup，得到的结果是backup
     */
    if (!strcmp(basename(argv[0]), "ueventd")) { //当argv[0]的内容为ueventd时，strcmp的值为0,！strcmp为1
    //1表示true，也就执行ueventd_main,ueventd主要是负责设备节点的创建、权限设定等一些列工作
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {//watchdogd俗称看门狗， "看门狗"本身是一个定时器电路，
    //内部会不断的进行计时（或计数）操作。计算机系统和"看门狗"有两个引脚相连接，
    //正常运行时每隔一段时间就会通过其中一个引脚向"看门狗"发送信号，"看门狗"接收到信号后会将计时器清零并重新开始计时。
    //而一旦系统出现问题，进入死循环或任何阻塞状态，不能及时发送信号让"看门狗"的计时器清零，
    //当计时结束时，"看门狗"就会通过另一个引脚向系统发送“复位信号”，让系统重启

        return watchdogd_main(argc, argv);
    }

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        install_reboot_signal_handlers(); //初始化重启系统的处理信号，内部通过sigaction 注册信号，当监听到该信号时重启系统
    }

    add_environment("PATH", _PATH_DEFPATH);//注册环境变量PATH
    //#define	_PATH_DEFPATH	"/sbin:/system/sbin:/system/bin:/system/xbin:/odm/bin:/vendor/bin:/vendor/xbin"
       

    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);//查看是否有环境变量INIT_SECOND_STAGE



    /*
     * 1.init的main方法会执行两次，由is_first_stage控制,first_stage就是第一阶段要做的事
     * 2.第一阶段主要做一些文件挂载，新建，修改权限的操作，还有就是启动SELinux policy
     * 3.mount是挂载文件，mkdir是新建文件，chmod修改文件权限，mknod创建字符设备
     */
    if (is_first_stage) {//只执行一次，因为在方法体中有设置INIT_SECOND_STAGE
        boot_clock::time_point start_time = boot_clock::now();

        // Clear the umask.
        umask(0); //清空文件权限

        // Get the basic filesystem setup we need put together in the initramdisk
        // on / and then we'll let the rc file figure out the rest.
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        // Don't expose the raw commandline to unprivileged processes.
        chmod("/proc/cmdline", 0440);
        gid_t groups[] = { AID_READPROC };
        setgroups(arraysize(groups), groups);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));
        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));

        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
        // talk to the outside world...
        InitKernelLogging(argv);

        LOG(INFO) << "init first stage started!";

        if (!DoFirstStageMount()) {
            LOG(ERROR) << "Failed to mount required partitions early ...";
            panic();//重启系统
        }

        SetInitAvbVersionInRecovery();

        // Set up SELinux, loading the SELinux policy.
        selinux_initialize(true);//加载SELinux policy，也就是安全策略，
        //SELinux是「Security-Enhanced Linux」的简称，是美国国家安全局「NSA=The National Security Agency」
        //和SCC（Secure Computing Corporation）开发的 Linux的一个扩张强制访问控制安全模块。
        //在这种访问控制体系的限制下，进程只能访问那些在他的任务中所需要文件

        // We're in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.

        /*
         * 1.这句英文大概意思是，我们执行第一遍时是在kernel domain，所以要重新执行init文件，切换到init domain，
         * 这样SELinux policy才已经加载进来了
         * 2.后面的security_failure函数会调用panic重启系统
         */
        if (restorecon("/init") == -1) { //restorecon命令用来恢复SELinux文件属性即恢复文件的安全上下文
            PLOG(ERROR) << "restorecon failed";
            security_failure();
        }

        setenv("INIT_SECOND_STAGE", "true", 1);

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
        setenv("INIT_STARTED_AT", StringPrintf("%" PRIu64, start_ms).c_str(), 1);//记录第二阶段开始时间戳

        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args); //重新执行main方法，进入第二阶段

        // execv() only returns if an error happened, in which case we
        // panic and never fall through this conditional.
        PLOG(ERROR) << "execv(\"" << path << "\") failed";
        security_failure();
    }

    // At this point we're in the second stage of init.
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";

    // Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 1); //初始化进程会话密钥

    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));//关闭/dev/.booting文件的相关权限

    property_init();//初始化属性，接下来的一系列操作都是从各个文件读取一些属性，然后通过property_set设置系统属性

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    /*
     * 1.这句英文的大概意思是，如果参数同时从命令行和DT传过来，DT的优先级总是大于命令行的
     * 2.DT即device-tree，中文意思是设备树，这里面记录自己的硬件配置和系统运行参数，参考http://www.wowotech.net/linux_kenrel/why-dt.html
     */
    process_kernel_dt();//处理DT属性
    process_kernel_cmdline();//处理命令行属性

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();//处理其他的一些属性

    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);

    // Clean up our environment.
    unsetenv("INIT_SECOND_STAGE"); //清空这些环境变量，因为都已经存入到系统属性中去了
    unsetenv("INIT_STARTED_AT");
    unsetenv("INIT_SELINUX_TOOK");
    unsetenv("INIT_AVB_VERSION");

    // Now set up SELinux for second stage.
    selinux_initialize(false); //第二阶段初始化SELinux policy
    selinux_restore_context();

    epoll_fd = epoll_create1(EPOLL_CLOEXEC);//创建epoll实例，并返回epoll的文件描述符
    //EPOLL类似于POLL，是Linux特有的一种IO多路复用的机制，对于大量的描述符处理，EPOLL更有优势
    //epoll_create1是epoll_create的升级版，可以动态调整epoll实例中文件描述符的个数
    //EPOLL_CLOEXEC这个参数是为文件描述符添加O_CLOEXEC属性，参考http://blog.csdn.net/gqtcgq/article/details/48767691

    if (epoll_fd == -1) {
        PLOG(ERROR) << "epoll_create1 failed";
        exit(1);
    }

    signal_handler_init();//主要是创建handler处理子进程终止信号，创建一个匿名socket并注册到epoll进行监听

    property_load_boot_defaults();//从文件中加载一些属性，读取usb配置
    export_oem_lock_status();//设置ro.boot.flash.locked 属性
    start_property_service();//开启一个socket监听系统属性的设置
    set_usb_controller();//设置sys.usb.controlle r 属性

    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) parser.DumpState();

    ActionManager& am = ActionManager::GetInstance();

    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(set_kptr_restrict_action, "set_kptr_restrict");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms = -1;

        if (!(waiting_for_prop || ServiceManager::GetInstance().IsWaitingForExec())) {
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || ServiceManager::GetInstance().IsWaitingForExec())) {
            restart_processes();

            // If there's a process that needs restarting, wake up in time for that.
            if (process_needs_restart_at != 0) {
                epoll_timeout_ms = (process_needs_restart_at - time(nullptr)) * 1000;
                if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
            }

            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}

 
```