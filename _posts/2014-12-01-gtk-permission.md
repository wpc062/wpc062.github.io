---
layout: post
title: 一个gtk下的目录权限问题
description: 本文描述了一个gtk下的目录权限问题，并分析了产生的原因。
category: gtk
tags: gtk 文件权限
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

最近，在客户的Linux GUI服务器上出现了一个文件权限相关的bug。公司的wxWidgets UI应用程序在执行某一个操作时会调用系统目录选择对话框（GTK默认提供的）。在目录选择对话框里，用户可以新建目录，但在里面创建的目录的权限和当前的umask设置不一致。

![](/assets/image/2014-12/gtk-perm-1.png)

比如： 客户默认的umask是**0022**, 但创建的目录的权限却是**0754**。由于后续操作需要检测目录权限是否和umask匹配，如果不匹配后续操作就无法进行。

客户的服务器OS版本比较老，是CentOS 4.8 （2009年的），而公司内部的测试机都是CentOS 5.X以上的。

这里把bug的分析过程写出来以供参考。

## 在线调试 ##
直接在客户的机子上用`strace`打印log，发现在执行系统调用mkdir(2)的时候，传入的权限是**0774**。这说明肯定是程序内部出错了。

## 重现问题 ##
刚好开发机的系统版本是CentOS 4.3，所以尝试在上面重现这个bug。在使用Linux服务器上的桌面程序时，通常的做法是借助于一些遵从X11协议的客户端软件，比如Xming，Exceed，VNC, NX等等。这些软件本身充当了X Server的角色。公司内存采用的是NX，关于NX的原理，可以参考一篇[cnblogs上的文章](http://www.cnblogs.com/coderzh/archive/2010/10/07/thinclient-secret-of-nomachine.html)，写的很不错。

![](/assets/image/2014-12/gtk-perm-2.png)

NX支持多组桌面协议，Unix下支持KDE、GNOME、XDM、CDM和Custom环境。其中Custom下只是运行指定的单个程序。

![](/assets/image/2014-12/gtk-perm-3.png)

写了一个小程序专门用来测试。在Custom选项下直接启动gnome-terminal，试了好久都重现不了这个问题。而在GNOME环境下很容易就重现了问题。

这里首先分析下产生bug的原因，然后分析下为什么程序在Custom和GNOME环境下执行出现差异。

## 原因分析 ##

在GNOME桌面环境下，直接使用GDB调试。在`mkdir`处设置断点，调试信息如下：

    Breakpoint 1, 0x00000034cefb86c0 in mkdir () from /lib64/tls/libc.so.6
    (gdb) bt 10
    #0  0x00000034cefb86c0 in mkdir () from /lib64/tls/libc.so.6
    #1  0x0000002a98e960bf in ?? () from /usr/lib64/gnome-vfs-2.0/modules/libfile.so
    #2  0x0000003e33745251 in gnome_vfs_make_directory () from /usr/lib64/libgnomevfs-2.so.0
    #3  0x0000002a98d8f42d in ?? () from /usr/lib64/gtk-2.0/2.4.0/filesystems/libgnome-vfs.so
    #4  0x00000034d3bd0510 in ?? () from /usr/lib64/libgtk-x11-2.0.so.0
    #5  0x00000034d18246e2 in ?? () from /usr/lib64/libgobject-2.0.so.0
    #6  0x00000034d180bfaa in g_closure_invoke () from /usr/lib64/libgobject-2.0.so.0
    #7  0x00000034d1824877 in ?? () from /usr/lib64/libgobject-2.0.so.0
    #8  0x00000034d1226606 in g_main_context_dispatch () from /usr/lib64/libglib-2.0.so.0
    #9  0x00000034d122821e in ?? () from /usr/lib64/libglib-2.0.so.0
    (More stack frames follow...)
    (gdb) info register
    rax            0xd      13
    rbx            0x1fc    508
    rcx            0x1b00352f35     115967602485
    rdx            0x128d7e0        19453920
    rsi            0x1fc    508
    rdi            0x128d7e0        19453920
    rbp            0x128d7e0        0x128d7e0
    rsp            0x7fbfffe7b8     0x7fbfffe7b8
    r8             0xfefefefefefefeff       -72340172838076673
    r9             0x2f2f2f2f2f2f2f2f       3399988123389603631
    r10            0x7fbfffe701     548682065665
    r11            0x34cefb86c0     226810889920
    r12            0x7fbfffe820     548682065952
    r13            0x0      0
    r14            0x34d18246c0     226853275328
    r15            0x0      0
    rip            0x34cefb86c0     0x34cefb86c0 <mkdir>
    eflags         0x206    [ PF IF ]
    cs             0x33     51
    ss             0x2b     43
    ds             0x0      0
    es             0x0      0
    fs             0x0      0
    gs             0x0      0
    (gdb) x /s $rdi
    0x128d7e0:      "/home/pwang/jobs/1409/test15/5"
    (gdb) p /o $rsi
    $1 = 0774

`mkdir`被调用时，传入的权限确实是**0774**。`mkdir`由函数[`gnome_vfs_make_directory`](https://developer.gnome.org/gnome-vfs/stable/gnome-vfs-2.0-gnome-vfs-directory-basic-ops.html#gnome-vfs-make-directory))触发。

这里的目录选择对话框其实就是[`GtkFileChooserDialog`](https://developer.gnome.org/gtk2/stable/GtkFileChooserDialog.html)。

通过rpm命令查询到可能的依赖库是`libgnomeui-2.8.0-1`。

    pwang@p03bc ~/p4Root/tflex_10$ rpm -qf /usr/lib64/libgnomevfs-2.so.0
    gnome-vfs2-2.8.2-8.2
    pwang@p03bc ~/p4Root/tflex_10$ rpm -qf /usr/lib64/gtk-2.0/2.4.0/filesystems/libgnome-vfs.so
    libgnomeui-2.8.0-1


通过google搜索到`libgnomeui`这个包的[源码包](http://rpm.pbone.net/index.php3/stat/4/idpl/1882136/dir/whitebox/com/libgnomeui-2.8.0-1.i386.rpm.html)。下载后搜索`gnome_vfs_make_directory`，真的找到了调用的地方在`gtkfilesystemgnomevfs.c`:706。

    static gboolean
    gtk_file_system_gnome_vfs_create_folder (GtkFileSystem *file_system,
                         const GtkFilePath *path,
                         GError   **error)
    {
      const gchar *uri = gtk_file_path_get_string (path);
      GnomeVFSResult result;
    
      gnome_authentication_manager_push_sync ();
      result = gnome_vfs_make_directory (uri,
                     GNOME_VFS_PERM_USER_ALL |
                     GNOME_VFS_PERM_GROUP_ALL |
                     GNOME_VFS_PERM_OTHER_READ);
      gnome_authentication_manager_pop_sync ();
    
      if (result != GNOME_VFS_OK)
    {
      set_vfs_error (result, uri, error);
      return FALSE;
    }
    
      return TRUE;
    }

这里gtk在调用`gnome_vfs_make_directory`的时候，直接把other的权限设置成read权限。后来找到了libgnomeui的官方文档，发现这是gtk的一个bug，而且老早就fix了（gtk版本太老好吃亏啊）。下面的注释来自它的[官方页面](https://github.com/GNOME/libgnomeui/blob/master/ChangeLog)：

    2006-02-08  Federico Mena Quintero  <federico@ximian.com>
    
        Merged from libgnomeui-2-14:
    
        * file-chooser/gtkfilesystemgnomevfs.c
        (gtk_file_system_gnome_vfs_create_folder): Add o+x permission to
        created folders; we were creating them with permissions 774
        instead of 775.
    
        * file-chooser/sucky-desktop-item.c
        (make_spawn_environment_for_sn_context),
        (make_environment_for_screen): Use g_listenv() instead of environ.
        Patch by Mark McLoughlin <mark@skynet.ie>, taken from bug #124277.
        Fixes bug #330350.
    


### Custom环境下，为何不能重现这个问题呢？ ###

首先，用GDB打印调试信息，发现调用`mkdir`时传入的权限是**0777**，而且调用栈有所不同。

    Breakpoint 1, 0x00000034cefb86c0 in mkdir () from /lib64/tls/libc.so.6
    (gdb) bt 10
    #0  0x00000034cefb86c0 in mkdir () from /lib64/tls/libc.so.6
    #1  0x00000034d3d1b38a in ?? () from /usr/lib64/libgtk-x11-2.0.so.0
    #2  0x00000034d3bd0510 in ?? () from /usr/lib64/libgtk-x11-2.0.so.0
    #3  0x00000034d18246e2 in ?? () from /usr/lib64/libgobject-2.0.so.0
    #4  0x00000034d180bfaa in g_closure_invoke () from /usr/lib64/libgobject-2.0.so.0
    #5  0x00000034d1824877 in ?? () from /usr/lib64/libgobject-2.0.so.0
    #6  0x00000034d1226606 in g_main_context_dispatch () from /usr/lib64/libglib-2.0.so.0
    #7  0x00000034d122821e in ?? () from /usr/lib64/libglib-2.0.so.0
    #8  0x00000034d122858a in g_main_loop_run () from /usr/lib64/libglib-2.0.so.0
    #9  0x00000034d3c18471 in gtk_main () from /usr/lib64/libgtk-x11-2.0.so.0
    (More stack frames follow...)
    (gdb) info register
    rax            0x132d2a0        20107936
    rbx            0x132d2a0        20107936
    rcx            0x0      0
    rdx            0x656c66742f676e61       7308328944513019489
    rsi            0x1ff    511
    rdi            0x132d2a0        20107936
    rbp            0x7fbfffead0     0x7fbfffead0
    rsp            0x7fbfffea88     0x7fbfffea88
    r8             0x34312f73626f6a78       3760839336450288248
    r9             0x31747365742f3930       3563600084735047984
    r10            0x7fbfffea01     548682066433
    r11            0x34cefb86c0     226810889920
    r12            0x0      0
    r13            0x1253550        19215696
    r14            0x107a160        17277280
    r15            0x0      0
    rip            0x34cefb86c0     0x34cefb86c0 <mkdir>
    eflags         0x206    [ PF IF ]
    cs             0x33     51
    ss             0x2b     43
    ds             0x0      0
    es             0x0      0
    fs             0x0      0
    gs             0x0      0
    (gdb) x /s $rdi
    0x132d2a0:      "/home/pwang/jobs/1409/test15/5"
    (gdb) p /o $rsi
    $1 = 0777

使用`lsof`打印，调用gnome相关库的信息。

GNOME桌面环境：

    pwang@p03bc ~$ lsof -p 13382 | grep gnome
    appCommon 13382 pwang  memREG8,236944  1613040 /usr/lib64/gtk-2.0/2.4.0/filesystems/libgnome-vfs.so
    appCommon 13382 pwang  memREG8,232280  1531529 /usr/lib64/gnome-vfs-2.0/modules/libfile.so
    appCommon 13382 pwang  memREG8,246560   965741 /usr/lib64/libgnome-keyring.so.0.0.1
    appCommon 13382 pwang  memREG8,2   200672   966294 /usr/lib64/libgnomecanvas-2.so.0.800.0
    appCommon 13382 pwang  memREG8,2   479528   966290 /usr/lib64/libgnomevfs-2.so.0.800.2
    appCommon 13382 pwang  memREG8,293016   965765 /usr/lib64/libgnome-2.so.0.800.0
    appCommon 13382 pwang  memREG8,2   662304   965740 /usr/lib64/libgnomeui-2.so.0.800.0
    appCommon 13382 pwang  memREG8,2   436648   966784 /usr/lib64/libgnomeprint-2-2.so.0.1.0
    appCommon 13382 pwang  memREG8,2   194536   965779 /usr/lib64/libgnomeprintui-2-2.so.0.1.0

Custom环境：

    pwang@p03bc ~$ lsof -p 15756 | grep gnome
    appCommon 15756 pwang  mem    REG      8,2   200672   966294 /usr/lib64/libgnomecanvas-2.so.0.800.0
    appCommon 15756 pwang  mem    REG      8,2   436648   966784 /usr/lib64/libgnomeprint-2-2.so.0.1.0
    appCommon 15756 pwang  mem    REG      8,2   194536   965779 /usr/lib64/libgnomeprintui-2-2.so.0.1.

发现`/usr/lib64/libgnomeui-2.so.0.800.0`根本就没有被调用。这说明即使同样的程序，在不同的X server环境下，运行结果也有很大差别。

### 进一步的发现 ###

通过GNOME桌面环境连接到NX server后，系统会初始化一系列的设置（窗口管理器、文件管理器。。。）。关于GNOME桌面环境的基础知识，可以参考[文档](http://www.yolinux.com/TUTORIALS/GNOME.html)。

ssh进程也直接依赖于桌面环境的`gnome-terminal` daemon进程。

    pwang     9257  0.0  0.0  91004  1844 ?        S    16:00   0:00 sshd: pwang@notty
    pwang     9258  0.0  0.0  64276  1576 ?        Ss   16:00   0:00  \_ /bin/bash /usr/bin/nxnode --slave
    pwang     9791  0.0  0.0  64276   796 ?        S    16:00   0:00      \_ /bin/bash /usr/bin/nxnode --slave
    pwang     9793  0.0  0.0  64408  1732 ?        S    16:00   0:00          \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    10044  0.0  0.0  64408  1124 ?        S    16:00   0:00              \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    10045  0.0  0.0  64408  1276 ?        S    16:00   0:00              |   \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    10047  0.0  0.1 118636 47728 ?        S    16:00   0:02              |   |   \_ /usr/lib/NX/nxagent -persistent -D -name NX - pwang@nx01bc.briontech.com:1086 - nx01bc
    pwang    10051  0.0  0.0  58936   552 ?        S    16:00   0:00              |   \_ tee /home/pwang/.nx/C-nx01bc.briontech.com-1086-486D9435A216E95E20BDDC644E7BAD44/session
    pwang    10053  0.0  0.0  64408  1292 ?        S    16:00   0:00              |   \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    10048  0.0  0.0  64544  1560 ?        S    16:00   0:00              \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    10124  0.0  0.0 242784  6192 ?        S    16:00   0:00                  \_ /usr/bin/gnome-session
    pwang    10126  0.0  0.0  53928   804 ?        Ss   16:00   0:00                      \_ /usr/bin/ssh-agent /usr/bin/dbus-launch --exit-with-session /usr/bin/gnome-session
    pwang    16942  0.0  0.0 286572 14244 ?        Sl   16:21   0:00 gnome-terminal
    pwang    16945  0.0  0.0  19128   908 ?        S    16:21   0:00  \_ gnome-pty-helper
    pwang    16946  0.0  0.0  70340  1752 pts/276  Ss   16:21   0:00  \_ bash
    pwang    16972  0.0  0.0  59232  3460 pts/276  S+   16:23   0:00      \_ ssh -X -i /home/pwang/.ssh/id_rsa_nx01 p03bc

Custom环境下，NX server只是启动了一个你指定的程序（比如：gnome-terminal）。如果没有指定，默认会运行一个Xterm程序。这里的gnome-terminal进程直接从属于当前的ssh连接。

    pwang@nx01bc ~$ ps uxf
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    pwang    15188  0.0  0.0  90212  1840 ?        S    16:10   0:00 sshd: pwang@notty
    pwang    15189  0.0  0.0  64276  1580 ?        Ss   16:10   0:00  \_ /bin/bash /usr/bin/nxnode --slave
    pwang    16406  0.0  0.0  64276   800 ?        S    16:10   0:00      \_ /bin/bash /usr/bin/nxnode --slave
    pwang    16408  0.1  0.0  64408  1732 ?        S    16:10   0:00          \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    16704  0.0  0.0  64408  1128 ?        S    16:11   0:00              \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    16705  0.0  0.0  64408  1276 ?        S    16:11   0:00              |   \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    16707  0.3  0.0 140644 13416 ?        S    16:11   0:00              |   |   \_ /usr/lib/NX/nxagent -persistent -R -name NX - pwang@nx01bc.briontech.com:1088 - nx01bc
    pwang    16712  0.0  0.0  58936   548 ?        S    16:11   0:00              |   \_ tee /home/pwang/.nx/C-nx01bc.briontech.com-1088-D0CF1BACEE30053E7BA1AE0968940F9B/session
    pwang    16714  0.2  0.0  64408  1292 ?        S    16:11   0:00              |   \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    16708  0.1  0.0  64544  1560 ?        S    16:11   0:00              \_ /bin/bash /usr/bin/nxnode --startsession
    pwang    16776  1.0  0.0 286532 14084 ?        Sl   16:11   0:00                  \_ /usr/bin/gnome-terminal
    pwang    16783  0.0  0.0  19128   908 ?        S    16:11   0:00                      \_ gnome-pty-helper
    pwang    16784  0.1  0.0  70340  1716 pts/240  Ss+  16:11   0:00                      \_ bash


在"ssh -X"登陆GUI服务器后，运行测试程序。GNOME桌面环境下，GUI服务器上产生一个临时的`gnome-vfs-daemon`后台进程。这个进程在测试程序关闭后也会退出。而在Custom环境下，`gnome-vfs-daemon` 则没有启动。

    pwang    19934  4.7  0.0 175736 20852 pts/53 Sl+  16:22   0:00 ./appCommonGui
    pwang    19937  1.2  0.0 68968 3176 ?        Ss   16:22   0:00 /usr/libexec/bonobo-activation-server --ac-activate --ior-output-fd=18
    pwang    19939  0.0  0.0 100132 4028 ?       Sl   16:22   0:00 /usr/libexec/gnome-vfs-daemon --oaf-activate-iid=OAFIID:GNOME_VFS_Daemon_Factory --oaf-ior-fd=25

可见，UI程序的运行严重依赖于X server的情况。


