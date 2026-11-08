---
title: PHP 伪协议速查与详解
date: 2026-11-08 15:55:00
tags:
  - PHP
  - Web
  - CTF
  - 文件包含
categories:
  - Web
description: PHP 伪协议速查：file://、php://filter、zip://、phar:// 的用法、依赖配置与 exit 绕过技巧。
---

<!-- more -->

## 速查表

| 协议 | 核心用途 | 关键语法 | 依赖配置 | 主要安全风险 |
| :--- | :--- | :--- | :--- | :--- |
| `file://` | 访问本地文件系统 | `file:///绝对路径`<br>`file://../../相对路径` | 无依赖 | 本地文件泄露、目录遍历 |
| `http://` | 访问远程 HTTP 资源 | `http://域名/远程脚本.php`<br>`http://IP:端口/文件` | `allow_url_fopen=On`<br>`allow_url_include=On` | 远程代码执行（RFI） |
| `https://` | 访问加密远程 HTTP 资源 | `https://安全域名/脚本.php` | 同 `http://` | 同 `http://` |
| `ftp://` | 通过 FTP 访问远程文件 | `ftp://用户名:密码@服务器/文件`<br>`ftp://匿名账号@服务器/路径` | `allow_url_fopen=On` | 未授权文件访问、恶意文件上传 |
| `php://input` | 读取 HTTP 请求体原始数据 | `php://input`（无额外参数） | 无依赖 | 代码注入 |
| `php://filter` | 对文件内容过滤 / 编码转换 | `php://filter/过滤器链/resource=目标文件` | 无依赖（读取） | 源码泄露、绕过过滤 |
| `zip://` | 访问 ZIP 压缩包内文件 | `zip://压缩包绝对路径%23子文件名` | 无依赖 | 压缩包内恶意代码执行 |
| `data://` | 直接嵌入数据执行 | `data://text/plain,代码`<br>`data://text/plain;base64,编码后代码` | `allow_url_fopen=On`<br>`allow_url_include=On` | 代码注入 |
| `glob://` | 批量匹配文件路径 | `glob://目录/*.php`<br>`glob:///etc/[a-z]*` | 无依赖 | 敏感文件路径泄露 |
| `phar://` | 访问 Phar 归档 / zip 包内资源 | `phar://归档文件/子文件` | `PHP>=5.3.0` | 归档内恶意代码执行 |
| `ssh2://` | 通过 SSH 访问远程文件 | `ssh2://用户名:密码@主机/文件` | 需 SSH 扩展 | 远程文件未授权访问 |
| `rar://` | 访问 RAR 压缩包内文件 | `rar://压缩包路径#子文件名` | 需 RAR 扩展 | 同 `zip://` |
| `expect://` | 执行交互式命令 | `expect://系统命令` | 需 Expect 扩展 | 命令注入 |

---

## 详细内容

### 定义

PHP 伪协议（PHP Wrappers）是 PHP 内置的特殊协议/方案，允许通过统一的 URL 结构或数据流方式访问不同类型的资源（本地文件、远程数据、输入输出流等）。其核心作用是通过特定前缀（如 `file://`、`php://`）让 PHP 以不同方式处理数据，广泛应用于文件操作、流处理等场景。

伪协议的执行依赖 PHP 配置参数：

- **`allow_url_fopen`**：控制是否允许打开 URL 文件（`On`/`Off`）；
- **`allow_url_include`**：控制是否允许引用 URL 文件（`On`/`Off`）。

不同协议对这两个参数的依赖不同（具体见各协议说明）。

### 特殊流协议（`php://`）

- **`php://output`**：直接向 HTTP 响应输出内容，通过流控制输出（区别于 echo 的即时打印）。
- **`php://stderr`**：向标准错误流输出错误信息，用于日志记录，避免错误直接暴露在 Web 页面。
- **`php://memory`**：操作内存中的虚拟文件，数据存储在内存中，脚本结束后丢失，性能高效（内存操作快于磁盘）。
- **`php://temp`**：数据先存内存，超出限制（通常 2MB）后自动切换到磁盘，避免内存溢出，脚本结束后数据丢失。
- **`php://fd`**：直接与操作系统文件描述符交互（文件描述符是打开文件/设备的整数标识符），用于低级文件操作。

#### 过滤器流：`php://filter`

**作用**：对文件内容进行过滤、转换或解码（如编码、压缩、加密），无需修改文件本身。

**基本格式**：

```
php://filter/[read=过滤器链]/resource=目标文件
```

- `read=过滤器链`：可选，指定读取时的过滤器（如 `convert.base64-encode`）。
- `resource`：必需，指定目标文件路径。

**常见过滤器**：

| 类型 | 过滤器示例 | 作用 |
| :--- | :--- | :--- |
| 字符串过滤器 | `string.rot13` | ROT13 编码/解码（字母轮换 13 位） |
| | `string.toupper` | 内容转为大写 |
| | `string.strip_tags` | 去除 HTML/PHP 标签（防 XSS） |
| 转换过滤器 | `convert.base64-encode` | Base64 编码文件内容 |
| | `convert.base64-decode` | Base64 解码文件内容 |
| 压缩过滤器 | `zlib.deflate` | Deflate 算法压缩内容 |
| | `zlib.inflate` | 解压 Deflate 压缩内容 |

**用法示例**：

- 读取 Base64 编码的 `flag.php`：

```
?file=php://filter/convert.base64-encode/resource=flag.php
```

- ROT13 编码读取：

```
?file=php://filter/read=string.rot13/resource=flag.php
```

### 其他常用伪协议

#### `zip://` 协议

**作用**：处理 ZIP 压缩文件中的子文件，支持任意后缀名（如将 `test.zip` 改为 `test.jpg` 仍可访问）。

**基本格式**：

```
zip://绝对路径/压缩文件%23子文件名
```

（`%23` 是 `#` 的 URL 编码）

**用法示例**：压缩 `shell.php` 为 `test.zip`，改名为 `test.xxx`，通过以下方式执行代码：

```
?file=zip:///var/www/test.xxx%23shell.php
```

**配置依赖**：不受 `allow_url_fopen` 和 `allow_url_include` 影响。

#### `data://` 协议

**作用**：通过 Data URI 方案嵌入数据，无需文件系统即可处理数据，支持直接执行 PHP 代码。

**基本格式**：

```
data://text/plain,[数据]
data://text/plain;base64,[Base64编码数据]
```

**用法示例**：

- 直接执行代码：

```
?file=data://text/plain,<?php phpinfo();?>
```

- Base64 编码执行：

```
?file=data://text/plain;base64,PD9waHAgcGhwaW5mbygpPz4=
```

**配置依赖**：需 `allow_url_fopen=On` 和 `allow_url_include=On`。

#### `glob://` 协议

**作用**：查找匹配的文件路径模式（如批量匹配 `*.php` 文件）。

**用法示例**：

```
glob:///var/www/*.php
```

（列出目录下所有 PHP 文件）

#### `phar://` 协议

**作用**：访问 PHP 归档文件（Phar，类似 ZIP/TAR）中的资源，支持 ZIP 格式压缩包。

**基本格式**：

```
phar://绝对路径/压缩包/子文件
```

**用法示例**：

```
?file=phar://test.zip/shell.php
```

（读取压缩包内的 `shell.php`）

**配置依赖**：`PHP>=5.3.0`，不受 `allow_url_fopen` 和 `allow_url_include` 影响。

#### 其他边缘协议

- **`ssh2://`**：通过 SSH2 协议访问远程资源。
- **`rar://`**：处理 RAR 压缩文件中的子文件。
- **`ogg://`**：访问音频流资源。
- **`expect://`**：处理交互式流（如命令交互）。

---

## 安全风险与防御措施

### 主要安全风险

1. **代码注入**：通过 `php://input`、`data://` 等协议注入恶意代码（如 `<?php system('rm -rf /');?>`）。
2. **信息泄露**：未过滤的 `file://`、`php://filter` 可能泄露服务器敏感文件（如 `/etc/passwd`、源码）。
3. **远程文件包含（RFI）**：`http://`、`ftp://` 等协议可执行远程恶意脚本。
4. **目录遍历**：结合 `../` 与 `file://` 访问未授权路径。

### 防御措施

1. **输入验证与过滤**：严格校验用户输入的文件路径，限制仅访问合法文件。
2. **禁用危险配置**：在 `php.ini` 中设置 `allow_url_fopen=Off` 和 `allow_url_include=Off`，禁用远程资源访问。
3. **限制文件访问范围**：通过 `open_basedir` 配置 PHP 可访问的基目录，禁止跨目录访问。
4. **禁用危险函数**：限制 `include`、`file_get_contents` 等函数的使用，或通过 `disable_functions` 禁用风险函数。
5. **错误处理优化**：配置错误日志记录，避免敏感信息（如文件路径）直接显示在 Web 页面。

---

## 特殊场景：`exit` 死亡绕过技巧

当代码中存在 `file_put_contents($filename, "<?php exit();" . $content);` 时，`exit()` 会阻止后续代码执行，可通过以下方法绕过。

### 1. Base64 编码绕过

**原理**：利用 `php://filter` 的 `convert.base64-decode` 过滤器，将恶意代码 Base64 编码后写入，解码时覆盖 `exit()`。

**示例**：（详细 payload 可参考相关 CTF 题解）

### 2. ROT13 编码绕过

**原理**：用 `string.rot13` 过滤器对恶意代码 ROT13 编码，解码后还原代码，规避 `exit()`。

**示例**：（详细 payload 可参考相关 CTF 题解）

### 3. .htaccess 预包含利用

**原理**：用 `string.strip_tags` 过滤器去除 `.htaccess` 中的 PHP 标签，设置 `auto_prepend_file` 预包含恶意文件。

**示例**：（详细 payload 可参考相关 CTF 题解）

**限制**：仅 PHP5 支持 `string.strip_tags` 写入过滤。

### 4. 过滤器组合绕过

**原理**：通过压缩-转小写-解压过滤器链破坏 `exit()` 代码结构。

**示例**：（详细 payload 可参考相关 CTF 题解）

**限制**：PHP7.3.0 以上版本可能报错，低版本有效。

---

## 参考来源

内容整理自 zsm's blog。
