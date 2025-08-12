---
title: "AppArmor Quickview"
date: 2025-08-11T16:29:10+08:00
description: 基础的AppArmor体验
categories: ["Linux", "Security", "AppArmor"]
layout: search
tags: ["develop"]
---

> AppArmor 是针对于程序的访问控制白名单设置 ~~(但是对普通用户和一般应用来说 并不适用)~~

AppArmor 有两套逻辑(黑名单与白名单)
- Complain: 允许所有访问 同时记录所有违规行为 方便修改配置 (allow-by-default)
- Enforce: 拒绝违规访问 (deny-by-default)

> 建议使用二进制程序 而不是脚本文件 因为脚本文件(如Python和Bash)会调用外部命令 有一层嵌套后无法追踪

## enable

可以通过 `aa-enabled` 查看是否启用了AppArmor

再通过 `sudo aa-status` 查看详细状态

另外可以考虑安装 `apparmor-utils` 会有如 `aa-complain/aa-enforce` 等辅助命令

## audit

有两种办法查看 `apparmor` 的审计日志

```bash
# 1. 通过 dmesg
sudo dmesg | grep apparmor

# 2. 通过 journalctl (systemd)
sudo journalctl -k | grep apparmor
```

## example

如下面的cpp代码 编译到 `$HOME/tmp/test_app`

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <cassert>

void read_file(const std::string& file) {
    std::ifstream fd(file);
    if (!fd.is_open()) {
        std::cerr << "Error: Could not open file " << file << std::endl;
        return;
    }
    
    std::string data((std::istreambuf_iterator<char>(fd)), 
                    std::istreambuf_iterator<char>());
    std::cout << file << ": len=" << data.length() << std::endl;
}

void write_file(const std::string& file, const std::string& content) {
    std::ofstream fd(file);
    if (!fd.is_open()) {
        std::cerr << "Error: Could not open file " << file << " for writing" << std::endl;
        return;
    }
    
    fd << content;
    std::cout << "write " << content << " into " << file << std::endl;
}

void print_usage(const std::string& program_name) {
    std::cout << "Usage:\n"
              << program_name << " read <file>\n"
              << program_name << " write <file> <content>\n";
}

int main(int argc, char* argv[]) {
    if (argc < 3) {
        print_usage(argv[0]);
        return 1;
    }

    std::string op = argv[1];
    std::string file = argv[2];

    if (op == "read") { read_file(file); } 
    else if (op == "write") {
        if (argc < 4) {
            std::cerr << "Error: Missing content for write operation\n";
            print_usage(argv[0]);
            return 1;
        }
        
        // Join remaining arguments with commas
        std::string content;
        for (int i = 3; i < argc; ++i) {
            if (i > 3) content += ",";
            content += argv[i];
        }
        
        write_file(file, content);
    } 
    else {
        print_usage(argv[0]);
        return 1;
    }

    return 0;
}
```

我们希望这个程序只能读取 `/tmp` 下的文件

于是配置如下

```apparmor
# vim:syntax=apparmor
#include <tunables/global>

# include简单理解为必要的宏
# 这里的flag表示模式
/tmp/test flags=(complain) {
  #include <abstractions/base>

  # 允许读与执行 (ix 表示子进程权限一样)
  /tmp/test r,
  /tmp/test ix,

  # r: 读 w: 写 k: 锁
  /tmp/** rwk,
  /usr/lib/** r,
  /usr/lib64/** r,

  # 下面是显示禁止读
  deny /etc/** r,
  deny @{PROC}/** r,
  deny /sys/** r,
}
```

于是我们可以得到如下结果 `/tmp/test` 只能在 `/tmp` 目录下读写

```bash
% /tmp/test read /etc/apt/sources.list
Error: Could not open file /etc/apt/sources.list

% /tmp/test write /tmp/test.log hello apparmor
write hello,apparmor into /tmp/test.log

% /tmp/test read /tmp/test.log                
/tmp/test.log: len=14

% /tmp/test read ~/tmp/tmp.log 
Error: Could not open file ./tmp.log
```

查看日志可以发现程序写入失败 同时发现原因为需要写权限 但是没有 `requested_mask="wc" denied_mask="wc"` 如果允许该行为 我们就可以修改配置文件

```bash
% sudo dmesg | grep "/tmp/test"
apparmor="DENIED" operation="open" profile="/tmp/test" name="<HOME>/tmp/tmp.log" pid=2380818 comm="test" requested_mask="wc" denied_mask="wc" fsuid=1000 ouid=1000
```

当我们使用 `enforce` 模式 会发现没显式允许的访问被禁止

```bash
% /tmp/test write ~/tmp/tmp.log hello apparmor
Error: Could not open file tmp.log for writing
```

## 挑战QQ

对于整个 AppImage QQ 而言 需要先开complain进行漫长的debug 最后发现
- 要给mount和子程序设置权限才能正常启动
- 要给`/run/user/@{uid}` 和 `/etc/machine_id` 才能使用输入法

具体的配置如下

```
# vim:syntax=apparmor
#include <tunables/global>

/home/hugh/.local/app/qq {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/X>
  #include <abstractions/fonts>
  #include <abstractions/audio>
  #include <abstractions/ssl_certs>

  # AppImage executable
  @{HOME}/.local/app/qq mr,
  @{HOME}/.local/app/qq ix,
  @{HOME}/.cache/mesa_shader_cache/ rw,
  @{HOME}/.cache/mesa_shader_cache/** rw,
  @{HOME}/.icons/ r,
  @{HOME}/.icons/** r,
  @{HOME}/.pki/** rwk,
  @{HOME}/.config/user-dirs.dirs r,
  @{HOME}/.config/dconf/user r,

  # ibus IME
  /etc/machine-id r,
  @{HOME}/.config/ibus/ r,
  @{HOME}/.config/ibus/** r,
  @{HOME}/.cache/ibus/ rw,
  @{HOME}/.cache/ibus/** rwk,
  @{HOME}/.local/share/vulkan/** rw,
  /usr/share/ibus/** r,
  /usr/lib/*/ibus/** r,
  /usr/libexec/ibus-* ix,
  
  # ibus socket
  /tmp/ibus-*/ rw,
  /tmp/ibus-*/** rw,

  /run/user/@{uid}/** rwm,
  /var/tmp/ r,
  /var/tmp/** r,


  # AppImage inner executable
  /tmp/.mount_*/AppRun ix,
  /tmp/.mount_*/qq ix,
  /tmp/.mount_*/chrome_crashpad_handler ix,

  # lib and bin
  /usr/bin/** r,
  /usr/bin/** ix,
  /usr/lib/ r,
  /usr/lib/** mr,
  /usr/lib64/ r,
  /usr/lib64/** mr,
  /usr/share/ r,
  /usr/share/** r,

  # 系统配置文件 - AppImage启动必需
  /etc/ld.so.cache r,
  /etc/ld.so.conf r,
  /etc/ld.so.conf.d/ r,
  /etc/ld.so.conf.d/** r,
  /etc/nsswitch.conf r,
  /etc/passwd r,
  /etc/group r,
  /etc/hosts r,
  /etc/resolv.conf r,
  /etc/localtime r,
  /etc/timezone r,
  /etc/fuse.conf r,
  /etc/ssl/certs/** r,
  /etc/ca-certificates/** r,
  /etc/vulkan/** r,
  
  # AppImage
  /tmp/ rw,
  /tmp/** rwkm,
  /dev/shm/ rw,
  /dev/shm/** rwm,
  /dev/fuse rw,
  /dev/tty rw,
  /dev/null rw,
  /dev/zero r,
  /dev/random r,
  /dev/urandom r,
  /dev/shm/.org.chromium.** rwkm,
  
  # Capability
  capability mknod,
  capability sys_ptrace,
  capability sys_admin,
  capability sys_chroot,
  capability dac_read_search,
  capability dac_override,

  # AppImage mount points
  mount fstype="fuse*" -> /tmp/.mount_*/,
  umount /tmp/.mount_*/,

  # Proc filesystem
  @{PROC}/ r,
  @{PROC}/version r,
  @{PROC}/meminfo r,
  @{PROC}/cpuinfo r,
  @{PROC}/devices r,
  @{PROC}/sys/** r,
  @{PROC}/@{pid}/ r,
  @{PROC}/@{pid}/** rw,
  
  # Sys filesystem
  /sys/devices/system/cpu/ r,
  /sys/devices/system/cpu/** r,
  /sys/class/net/ r,
  /sys/devices/** r,
  /sys/bus/** r,
  /sys/fs/** r,


  # QQ usage
  owner @{HOME}/.config/QQ/ rw,
  owner @{HOME}/.config/QQ/** rwk,
  owner @{HOME}/Downloads/ rw,
  owner @{HOME}/Downloads/** rwk,

  # network
  network inet stream,
  network inet6 stream,
  network inet dgram,
  network inet6 dgram,
  network netlink raw,

  deny @{HOME}/.ssh/ r,
  deny @{HOME}/.ssh/** rw,
  deny @{HOME}/Documents/ r,
  deny @{HOME}/Documents/** rw,
  deny @{HOME}/Pictures/ r,
  deny @{HOME}/Pictures/** rw,

  deny /etc/profile r,
  deny /dev/disk/by-id/ r,
  deny /dev/disk/by-uuid/ r,
  deny /proc/uptime r,

  # deny @{HOME}/ r,
  # deny @{HOME}/.* r,
  # deny @{HOME}/* r,
}
```

## conclusion

AppArmor 对于一般用户和一般应用来说 并不太友好 甚至不太建议尝试

这个东西的主要作用还是在于服务器对于特定软件的权限控制 如nginx 但这些软件一般不需要自己写 厂商会提供的