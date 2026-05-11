# SecNote

安全笔记。构建我个人的网络安全知识框架。不断扩充中...

菜鸡在路上...

> **更新说明**
>
> 本文档最新更新内容由 AI 模型 **MiniMax-M2.7** 辅助编写。
>
> 文档仅用于**归档知识、形成知识树**，内容不保证正确性，请勿直接用于实际安全测试。所有技术操作请在合法授权范围内进行。

---

## 0x00 躲避检测

### 1 隐藏

[1] [渗透测试中的身份隐藏](https://github.com/aplyc1a/blogs/blob/master/躲避检测/渗透测试中的身份隐藏.md)

### 2 绕过

### 3 免杀

[1] [基本二进制免杀](https://github.com/aplyc1a/blogs/blob/master/躲避检测/二进制免杀技术研究.md)

## 0x01 信息收集

### 1 人

\[1] [自然人信息社工](https://github.com/aplyc1a/blogs/blob/master/信息收集/自然人信息社工.md)

### 2 企业

\[1] [企业资产信息收集](https://github.com/aplyc1a/blogs/blob/master/信息收集/企业目标资产信息收集.md)

## 0x02 入口突破

### 1 web服务

\[1] [Web服务入口突破](https://github.com/aplyc1a/blogs/blob/master/入口突破/Web服务入口突破.md)

### 2 钓鱼邮件

\[1] [钓鱼邮件攻击](https://github.com/aplyc1a/blogs/blob/master/入口突破/钓鱼邮件攻击.md)

### 3 字典

\[1] [社工字典生成器RainCode](https://github.com/aplyc1a/blogs/blob/master/入口突破/社工字典生成器RainCode.md)

\[2] [口令模型分析](https://github.com/aplyc1a/blogs/blob/master/入口突破/自然人口令常见模式.md)

## 0x03 权限提升

### 1 Linux提权

#### 1.1 配置不当提权

\[1] [suid提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/suid.md)

\[2] [sudo提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/sudo.md)

\[3] [shell脚本定时任务提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/shell脚本定时任务提权.md)

\[4] [shell脚本调用权限继承提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/shell脚本调用权限继承提权.md)

\[5] [sudo脚本篡改提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/sudo脚本篡改提权.md)

\[6] [sudo脚本参数提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/sudo脚本参数提权.md)

\[7] [环境变量劫持提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/环境变量劫持提权.md)

\[8] [软链接提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/配置不当提权/软链接提权.md)

#### 1.2 漏洞提权

\[1] [Linux漏洞提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Linux提权/漏洞提权/Linux漏洞提权.md)

### 2 Windows提权

\[1] [Windows提权](https://github.com/aplyc1a/blogs/blob/master/权限提升/Windows提权.md)

## 0x04 内网与后渗透

### 1 信息与数据搜集

\[1] [getshell后的基本信息收集](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/getshell后的基本信息收集.md)

\[2] [敏感数据搜集](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/敏感数据搜集.md)

\[3] [Windows常用命令行操作](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/常用命令行操作.md)

### 2 通道构建

\[1] [通道构建](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/通道构建.md)

### 3 扫描探测

\[1] [扫描探测](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/扫描探测.md)

### 4 权限提升

### 5 横向移动

\[1] [横向移动](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/横向移动.md)

### 6 数据回传

### 7 接管域控

\[1] [接管域控](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/接管域控.md)

### 8 系统破坏

## 0x05 持久控制

### 1 Linux

\[1] [门罗挖矿技术研究](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/monero挖矿研究.md)

#### 1.2 后门

\[1] [Linux $PATH劫持命令后门](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-PATH环境变量抢占后门.md)

\[2] [Linux 后门账户添加](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-后门账户.md)

\[3] [Linux SSHWrapper(过时)](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-SSHWrapper后门.md)

\[4] [Linux (x)inetd后门(过时)](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-(x)inetd后门.md)

\[5] [Linux $PROMPT_COMMAND后门(过时)](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-PROMPT_COMMAND后门.md)

\[6] [Linux 计划任务后门族（新）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-计划任务后门.md)

\[7] [Linux SSH软链接后门（新）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-SSH软链接后门)

\[8] [Linux 别名后门（新）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-各种别名后门.md)

\[9] [Linux OpenSSH后门（新）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/定制化OpenSSH后门.md)

\[10] [Linux PAM后门（参考）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-PAM后门制作.md)

\[11] [Linux systemd后门](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-systemd服务后门.md)

\[12] [Linux-fake命令偷密码（新）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-fake命令偷密码.md)

\[12] [Linux-内存执行ELF技术总结（新）](https://github.com/aplyc1a/blogs/blob/master/持久控制/Linux/Linux-内存执行ELF.md)

#### 1.3 勒索

#### 1.4 隐蔽通信

\[1] [ICMP隐蔽shell-p1ngp0ng](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/p1ngp0ng.md)

\[2] [DNS隐蔽shell-DNShell](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/DNShell.md)

\[3] [NTP隐蔽shell-NTPShell](https://github.com/aplyc1a/blogs/blob/master/内网安全与后渗透/NTPShell.md)

## 0x06 取证溯源

\[1] [Linux 入侵痕迹取证-1](https://github.com/aplyc1a/blogs/blob/master/取证溯源/Linux取证-(1).md)

\[2] [Linux 入侵痕迹取证-2](https://github.com/aplyc1a/blogs/blob/master/取证溯源/Linux取证-(2).md)

\[3] [Linux 入侵痕迹取证-3](https://github.com/aplyc1a/blogs/blob/master/取证溯源/Linux取证-(3).md)

\[4] [Windows 入侵痕迹取证](https://github.com/aplyc1a/blogs/blob/master/取证溯源/Windows取证.md)

\[5]  [攻击溯源下的信息收集](https://github.com/aplyc1a/blogs/blob/master/取证溯源/攻击溯源下的信息收集.md)

## 0x07 审查对抗

### 1 反审查

\[1]  [匿名与反审查技术](https://github.com/aplyc1a/blogs/blob/master/审查对抗/反审查技术/反审查技术.md)

\[2]  [隐写术](https://github.com/aplyc1a/blogs/blob/master/审查对抗/反审查技术/隐写术.md)

### 2 司法审查

\[1] [司法审查](https://github.com/aplyc1a/blogs/blob/master/审查对抗/司法审查.md)

## 0x08 持久控制 - Windows

\[1] [Windows后门](https://github.com/aplyc1a/blogs/blob/master/持久控制/Windows后门.md)

\[2] [勒索软件](https://github.com/aplyc1a/blogs/blob/master/持久控制/勒索软件.md)