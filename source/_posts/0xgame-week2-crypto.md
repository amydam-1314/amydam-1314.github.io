---
title: 0xGame Week2 Crypto 全解
date: 2026-10-18 18:10:00
tags:
  - 0xGame
  - CTF
  - 密码学
categories:
  - CTF
---

七道题，从古典密码一直到格攻击。按题目顺序记，每道含考点、思路和最终 flag。

<!-- more -->

## 群友群友群？
![hint](/img/0xgame-hint.jpg)

![ciphertext](/img/0xgame-ciphertext.jpg)

```
投喂完毕后，你正想离开小圈，突然发现栅栏上面被哼哼唧唧的啃上了很多凌乱的划痕，群友群友群（？雾）到底想说什么呢

最后用0xGame包裹
```

## 分析
根据题干和hint图片，第一反应是栅栏或者猪圈密码，然后在看密文，分为三列，于是在网上搜到了符合的密码表
![猪圈密码对照表](/img/0xgame-pigpen.png)

## 最终答案
```
0xGame{WELCOMETOSIGNUP}
```


---

## 老师傅的波纹

task
```
from secrets import flag
from Crypto.Util.number import getPrime
BITS = 80
a = int(flag[5:-1], 16)
p = getPrime(BITS)
while True:
    A = ZZ(randrange(1, p))
    B = ZZ(randrange(1, p))
    if (4 * A^3 + 27 * B^2) % p == 0:
        continue
    E = EllipticCurve(GF(p), [A, B])
    n = E.order()
    if is_prime(n) and n != p and n != p + 1:
        break
G = E.random_point()
Q = a * G
print(f"p = {p}")
print(f"A = {A}")
print(f"B = {B}")
print(f"n = {n}")
print(f"G = ({ZZ(G.xy()[0])}, {ZZ(G.xy()[1])})")
print(f"Q = ({ZZ(Q.xy()[0])}, {ZZ(Q.xy()[1])})")
# p = 1208925819614629174270411
# A = 183150304136760995950581
# B = 794569627338707454052582
# n = 1208925819612932406971299
# G = (181528231928396052728802, 1540136182689226613050)
# Q = (339758143822552101710854, 503250777866725616991088)
```

## 分析
一道椭圆曲线题，其中封闭了奇异曲线，smart攻击，mov攻击，Pohlig-Hellman等常见手段的情况
先在素域 `GF(p)` 上生成椭圆曲线

$$
E: y^2 = x^3 + Ax + B \pmod p
$$

将 flag 花括号内的十六进制串转换为整数 `a`，然后给出随机点 `G` 和

$$
Q = aG.
$$

因此目标就是求解椭圆曲线离散对数：

$$
a = \log_G Q.
$$

但是`p` 是接近 `2^80` 的伪梅森素数，可以用少量整数乘法和折叠完成快速模约减。结合题目名称中的“波纹”，预期解法就是对 80 位素数阶群运行 Pollard rho。其时间复杂度为 `O(sqrt(n))`，约需 `2^40` 量级的群运算，而不是枚举全部 `2^80` 个标量。
这里其实还有一种计算dlp的通用算法就是bsgs，至于为什么不用是因为bsgs的空间复杂度远大于Pollard rho且仅仅适用于小规模群，还有一个原因是Pollard rho可以并行搜索，速度其实也有保障。

sage实现Pollard rho的函数
```
sage.groups.generic.discrete_log_rho(a, base, ord=None, ...)
```
虽然曲线阶只有80位，但是单行运行还是得花几小时到一两天的时间，所有建议多行搜索。

ai写的exp就不放了又臭又长。。。
## 最终答案

```text
0xGame{c9d1bdbf5b83a0742072}
```

---

## 命运石之门

```
选择这一条时间线的椎名真由理，就会失去另外一条时间线的牧濑红莉栖，最开始的死亡直接构成了命运的无解。但现在已经知道了全部的明文密文，欺骗自己，伪造牧濑红莉栖的假死，尝试到达那个1.048596号时间线吧

EI Psy Kongroo,一切都是命运石之门的选择!
```

task

```
from Crypto.Cipher import AES
from base64 import b64decode

KEY = bytes.fromhex("185608cdb3409b141aa6812757d41cc9")
IV = bytes.fromhex("2d1aed23dff9f1f2e30836642237a752")

conditions = {
    8: 85,
    30: 108,
    41: 50,
    63: 122,
    67: 115,
    91: 75,
    104: 103,
    122: 90,
    143: 52,
}

try:
    c2 = b64decode(
        input("c2.encode('base64')=").strip(),
        validate=True
    )

    if len(c2) != 256:
        raise ValueError

    m2 = AES.new(
        KEY,
        AES.MODE_CBC,
        IV
    ).decrypt(c2)

    if not all(
        m2[pos] == value
        for pos, value in conditions.items()
    ):
        raise ValueError

    from secret import flag
    print(flag)

except Exception:
    print("失败了失败了失败了...")
```

```
####*####*####*####Game Start####*####*####*####
# '#' for conditions
# '@' for questions
####@####
# c1=aes_cbc_encode(password,iv,m1)
# c1.encode('base64')=78k8daGvSR3MgT/oq6f/ysaYJTIPJGwgHmPkyiUmZbe+fekHTMi0iw8lEC8jOtBVI+nNiJtIXgOi
xXibyYXBNJys8Cx350Be7noECiSlvzx5ScG7smEQ+Lu2k2ehSo0xnN3nsyXY0gxQOXiTzjkFdD2Z
soKbxGLTHCaRyeTtm2Sl2W0xeslLN/6Ro7ANlL0eUciX9zVp21ps6cIjjEZvU6cl0nByaOr2Wo+Y
uwtgVlGfNfYs7HURVjoxU2W0DKHzqQd6FdplZrRFRaRCA6sp2TYizKWCs97Mv+dWmJ90XgF6BRgy
kV49XT6FyZEvDDLuZEKPtjrVKqrgfulLx7rFsQ==
# m1.encode('base64')=eUZRY2Q4Zjhzc31Mbi1LQmZLMzZ5V3hHdjh3SjNBMWw1OWVoZEFXa194Si1rZUlCZmtSVVlJa29i
ak9Ba0NtZGl9TH1SSEtuU2ZwcjJPQ0xrcE9Bbn1YfVNJWE05bFI5SklyYzN2R3JuQzFLa0RrR2lR
azN3Z19pUER2ekgzbEdxWS10czI1ejJjQnFRNnBnSG1WaUswRjBMSkN5eUV3Z3p7R0lmWHY4VV9p
LWhrRGxWTEJ5SHNGYTY3cmhua3dpUFZTTHI3YnhFenJBS29ELXVZX31pUkpCOUw3M2R4SmgzVzhS
dG5RaGs2UFpwMGszVWNHb09qejJ5RFhqUVNCWQ==
# m2=aes_cbc_decode(password,iv,c2)
# m2[8]=U
# m2[30]=l
# m2[41]=2
# m2[63]=z
# m2[67]=s
# m2[91]=K
# m2[104]=g
# m2[122]=Z
# m2[143]=4
```
## 分析
源码已经提供给我们AES.CBC的KEY与向量IV，同时有一个字典condition，给出flag需满足输入的密文c2为256位长的base64编码，且c2经过AES.CBC解密后对应索引的值的ASCII需满足condition字典的分布。
至此我们的攻击链就十分明确了，因为AES加解密不会影响位数，所以可以构造一个256位长的明文，其中对应condition索引的值替换成condition的value值，再通过题目给出的KEY和IV获得加密后的c2即可与服务器交互。

## EXP
```
from Crypto.Util.number import *
from pwn import *
from itertools import product
from hashlib import *
from math import *
from gmpy2 import *
from tqdm import tqdm
from sympy import *
import ast
import os
from gf2bv import *
import json
import numpy as np
import binascii
from Crypto.Cipher import AES

from base64 import b64encode

KEY = bytes.fromhex("185608cdb3409b141aa6812757d41cc9")
IV = bytes.fromhex("2d1aed23dff9f1f2e30836642237a752")

conditions = {
    8: 85, 30: 108, 41: 50, 63: 122, 67: 115,
    91: 75, 104: 103, 122: 90, 143: 52,
}

# 构造 256 字节明文并加密
m2 = bytearray(b'A' * 256)
for pos, val in conditions.items():
    m2[pos] = val

c2 = AES.new(KEY, AES.MODE_CBC, IV).encrypt(bytes(m2))
assert len(c2) == 256
payload = b64encode(c2).decode()

# 连接服务器
io = remote("nc1.ctfplus.cn", 45047)
io.recvuntil(b"c2.encode('base64')=")
io.sendline(payload.encode())
print(io.recvall(timeout=5).decode(errors="ignore"))
io.close()
```

## 最终答案
```
0xGame{th15 15 my v1ct0ry}
```

---

## 夏天仍未结束

task
```
夏天？（国庆）仍未结束
从测量记录和加密音频容器中恢复原始音频。
附件：
text
simon_measurements.json  测量记录
quantum_echo.qflac       自定义 QFLAC2 容器

容器以 `QFLAC2` 开头，随后是小端序长度、JSON manifest、FLAC 元数据和加密数据。manifest 包含公开的解析信息。
测量记录来自 Simon 型周期测量，每条测量向量与对应周期在 GF(2) 上正交。每条字符串从左到右对应一个周期向量的连续位，实例顺序和位宽由 JSON 给出；恢复出的位流按记录顺序连续拼接，并采用 LSB-first 打包。
manifest 中的 `auxiliary` 字段描述了第一条载荷的公开派生方式。它使用 SHA-256 计数器扩展，输入由 `label`、打包后的测量材料和大端序 32 位计数器组成。第二条载荷是目标数据。两条记录属于同一个加密会话。也许夏天走得太急，忘了更换自己乐碟加密留下的随机数？
2个qflac正常拼接后才可以开始解密
恢复出的结果应当是一个可以播放的 FLAC 文件，surprise?
```

```
{
  "instance_count": 8,
  "secret_bits": 10,
  "samples": {
    "0": [
      "1101101000",
      "0010010111",
      "1011001010",
      "0010111010",
      "1000001111",
      "1010001101",
      "1101010011",
      "1011010011",
      "0101110111",
      "1111010111",
      "1000001001",
      "1010001110",
      "0001101100",
      "0100100101",
      "0110100100",
      "0101011010",
      "1110101000",
      "1010010001",
      "0010000001",
      "1101000101",
      "0100100110",
      "1101101110",
      "1000010101",
      "1001111111"
    ],
    "1": [
      "1110100111",
      "1110111100",
      "0001100010",
      "1001101110",
      "0001110100",
      "0110100011",
      "0110100100",
      "1010101000",
      "1000100100",
      "0001010110",
      "0001011001",
      "0101011011",
      "1110011001",
      "0011000100",
      "0111101100",
      "0010110010",
      "1001011111",
      "1010110001",
      "0110011000",
      "1110011100",
      "0101111100",
      "0010100100",
      "0111111000",
      "1011101111"
    ],
    "2": [
      "1100100001",
      "1001101000",
      "1111010111",
      "1000101010",
      "0100000001",
      "1010110101",
      "0111001011",
      "0101000011",
      "1100000010",
      "0010011110",
      "0101110111",
      "1011001010",
      "0000110110",
      "1101011101",
      "0110100000",
      "1100101000",
      "0100000011",
      "1001011111",
      "1011100000",
      "1100011100",
      "0100000010",
      "0110001010",
      "0100010111",
      "0101111111"
    ],
    "3": [
      "1011101010",
      "0010111001",
      "1010111100",
      "1001011010",
      "0000100100",
      "0111011101",
      "0000110011",
      "0111001001",
      "0010011001",
      "0111000100",
      "1101010101",
      "1101110110",
      "1001001110",
      "0111011010",
      "1100101001",
      "1011001010",
      "1011011010",
      "0011110010",
      "1011000111",
      "0011000010",
      "0010110100",
      "0000001101",
      "0100001000",
      "1011010100"
    ],
    "4": [
      "0111001011",
      "0000111100",
      "0101000001",
      "0001000101",
      "0000000000",
      "1001110000",
      "1000001000",
      "0101011000",
      "0101001100",
      "1110101010",
      "0100001001",
      "0010000110",
      "1010000111",
      "1110110111",
      "1111101110",
      "1010000011",
      "1000110000",
      "0000111001",
      "0111110111",
      "0010000111",
      "1011000110",
      "1001011000",
      "1111000011",
      "1111010110"
    ],
    "5": [
      "0110011011",
      "0010110111",
      "1000110101",
      "0101011001",
      "1000011000",
      "1000101001",
      "0101000000",
      "1011110000",
      "1101101000",
      "0110110000",
      "0001110000",
      "1011000100",
      "0000000111",
      "0000110100",
      "1100000000",
      "1100101010",
      "1110110001",
      "0001000100",
      "1010011101",
      "0001110001",
      "0001110110",
      "0110110010",
      "1101011000",
      "1100101001"
    ],
    "6": [
      "1010100001",
      "1100111110",
      "1101101010",
      "0001001011",
      "0111011010",
      "1010010001",
      "1100011101",
      "1010010000",
      "1011101010",
      "0000101100",
      "0110011000",
      "1011011010",
      "0110110011",
      "0001101001",
      "1101010111",
      "1100001101",
      "0111110110",
      "1010110100",
      "1101100101",
      "1100010011",
      "0110111111",
      "1011100100",
      "0000010110",
      "1101010000"
    ],
    "7": [
      "0110011111",
      "1001111001",
      "1001111010",
      "1111100010",
      "0011111000",
      "1001101001",
      "0110100000",
      "0000101111",
      "0110001110",
      "1001000110",
      "1001101011",
      "0000000010",
      "0000010010",
      "1101000111",
      "0001100111",
      "1110000110",
      "1011001111",
      "1110010100",
      "0111010110",
      "1001000101",
      "0100111111",
      "0011101010",
      "1101111100",
      "1100001000"
    ]
  }
}
```

## 分析

### Simon 测量恢复周期

 记忆的宝藏给出 8 个实例，每个实例 24 条 10-bit 字符串。题目明确说这是 Simon 型周期测量，每条测量向量 y 与隐藏周期 s 在 GF(2) 上正交：

 $y \cdot s = 0 \pmod 2$

 对每个实例构造 24 × 10 的 GF(2) 矩阵，求零空间：

 ```python
   B = nullspace(rows, 10)
   assert len(B) == 1
 ```

 8 个实例的 nullity 全部为 1，得到唯一周期：

 ```text
   0111101111  1010010101  0000010100  1101001011
   0010000010  0000111000  1110000000  1011101000
 ```

 按实例顺序拼接 80 bit，再按 LSB-first 打包：

 ```text
   de970acad204c1714017
 ```

###  拼接 QFLAC2 容器

 两个 qflac 各 33651081 字节，直接拼接：

 ```text
   QFLAC2(6) + flags(2) + manifest_len(4 LE) + manifest(JSON)
   + meta_len(4 LE) + FLAC metadata + C1 + T1 + C2 + T2
 ```

 由 manifest 的 payload_lengths 和 tag_len 定位：

 ```text
   meta_len     = 164034
   payload1     @ 164508
   payload2     @ 33733335
 ```

 这里容易漏掉 manifest 后面的 4 字节小端 FLAC 元数据长度。

###  auxiliary 载荷的公开派生

 manifest 的 auxiliary 字段：

 ```text
   scheme      = sha256-counter
   label       = QFLAC-NONCE-REUSE-PROBE
   source      = packed_measurements
   counter     = uint32-be
   input_order = [label, source, counter]
 ```

 即：
 
```
 P₁ = SHA256 (label ‖ source ‖ counter_be)
```

 逐块拼接，共 1,049,026 个 SHA-256 块，得到 auxiliary 明文 P₁。

###  AES-256-GCM 复用 nonce

 两段载荷使用同一 key、同一 nonce：

 ```text
   493f129a43aa177e65baca25
 ```

 GCM 复用 nonce 时密钥流相同，因此

```
 P₂ = C₁ ⊕ C₂ ⊕ P₁.
```

 恢复出的 P₂ 以 ff f8 开头，这是 FLAC 帧同步码。说明得到的是目标 FLAC 的音频帧，不是带 fLaC 头的完整文件。

 ### 5. 还原 FLAC 并读元数据

 把容器里的公开 FLAC 元数据与 P₂ 拼回去：

 ```python
   flac = flac_meta + P2   # flac_meta 以 fLaC 开头
 ```

 解析 Vorbis comment：

 ```text
   Title=打上花火
   Album=打上花火 (通常盤)
   Artist=Daoko (ダヲコ)/米津玄師 (よねづ けんし)
 ```

最后得到的歌名即答案。

## 最终答案

```
 0xGame{打上花火}
```

---

## 契约之吻

task

```
from random import randrange
from secret import f_coeffs, s_coeffs, error

d = 64
scale = 2^44
n = 24
p = 2147483647

secret = vector(ZZ, [f_coeffs[i] if i < len(f_coeffs) else (1 if i == n else 0) for i in range(d)])
mt = matrix(ZZ, [[randrange(0, scale) for _ in range(d)] for _ in range(d)])
mt[0] = [i for i in secret]

new_mt = matrix(ZZ, d)
for i in range(d):
    row = random_vector(ZZ, d, x=257)
    for j in range(i):
        row[j] = 0
    row[i] = 1
    new_mt[i] = row * mt

H = new_mt

R.<x> = PolynomialRing(ZZ)
F = x^n + sum(f_coeffs[i] * x^i for i in range(len(f_coeffs)))
A = vector(ZZ, [1624513444, 7887634, 1850144002, 1333320193, 1148549154, 28619938, 1601341874, 353058494, 224962944, 1491387826, 612016532, 1078815215, 1270428426, 748164318, 1219377558, 1238933391, 1824650003, 469831872, 712295407, 559376837, 1804640269, 1147167506, 818193445, 1043390130])
a = sum(A[i] * x^i for i in range(n))
M_A = matrix(ZZ, n, n)
for j in range(n):
    col = (a * x^j) % F
    for i in range(n):
        M_A[i, j] = col[i]

B = (M_A * vector(ZZ, s_coeffs) + vector(ZZ, error)).apply_map(lambda v: v % p)
```

```
output=.....
```

输出太长了这里就不放了。
## 分析

### 先构造 H

`mt[0] = secret`，其余行是 (0, 2^44) 的随机数；`new_mt` 的每一行都是对 mt 做单位上三角随机变换得到的，即

$$
H = U \cdot mt, \qquad
U = \begin{pmatrix}
1 & * & \cdots & * \\
0 & 1 & \cdots & * \\
\vdots & & \ddots & \vdots \\
0 & 0 & \cdots & 1
\end{pmatrix}.
$$

$U$ 的行列式为 1，所以 $H$ 与 $mt$ 生成同一个格。`secret` 是格中的短向量（`f_coeffs` 很小，且第 24 位是 1），$H$ 只是它的一组“打乱基”。直接对 $H$ 做 LLL 就能把短向量找回来。

LLL 得到

```text
[-4, -2, 0, -2, 0, ..., -1 (index 24), ...]
```

取反，让 $x^{24}$ 的系数为 +1：

$$
F(x) = x^{24} + 2x^3 + 2x + 4.
$$

### 构造 LWE

有了 $F$，按题面计算

$$
M_A[:,j] = (a \cdot x^j) \bmod F, \qquad a = \sum A_i x^i.
$$

于是有

$$
B = M_A s + e \pmod p, \qquad |e_i| \le 3,
$$

这是一个维度 24、误差界 3 的 LWE 实例。

### Kannan embedding 构造格

$$
L = \begin{pmatrix}
pI_n & 0 & 0 \\
M_A^T & I_n & 0 \\
-B^T & 0 & c
\end{pmatrix}.
$$

其中短向量形如 $(-e, s, c)$。对 $L$ 做 LLL + BKZ(20)，取第三坐标为 $c$ 的行，归一化后验证

$$
M_A s + e \equiv B \pmod p, \qquad |e_i| \le 3.
$$

解得

```text
e = [1, 1, 3, -3, -1, 1, 3, -1, -3, -2, 3, 0,
     1, -2, -2, 2, -2, 0, 0, -3, -2, -3, -1, -3]
s = [48, 120, 71, 97, 109, 101, 123, 107, 105, 115, 115, 32,
	116, 104, 101, 110, 32, 107, 105, 108, 108, 125, 0, 0]
```

再将 `s` 按 ASCII 解码。

## exp

```
import sys
OUT = sys.argv[1] if len(sys.argv) > 1 else "/mnt/d/2026/output (5).txt"
exec(open(OUT).read())

# ---------- 1. 从 H 恢复 F ----------
Hr = H.LLL()
secret = None
for row in Hr.rows():
    if abs(row[24]) == 1 and max(abs(x) for x in row) < 1000:
        secret = [-x for x in row]   # 令 x^n 系数为 +1
        break
assert secret is not None, "未找到 secret 短向量"
f = [int(secret[i]) for i in range(n)]
print("f_coeffs =", f)

R.<x> = PolynomialRing(ZZ)
F = x^n + sum(f[i] * x^i for i in range(n))
print("F =", F)

a = sum(A[i] * x^i for i in range(n))
M = matrix(ZZ, n, n)
for j in range(n):
    col = (a * x^j) % F
    for i in range(n):
        M[i, j] = col[i]

# ---------- 2. Kannan embedding 解 LWE ----------
def solve_lwe(c=1, block=20):
    dim = 2 * n + 1
    L = matrix(ZZ, dim)
    for i in range(n):
        L[i, i] = p
    for i in range(n):
        for j in range(n):
            L[n + i, j] = M[j, i]      # M^T
        L[n + i, n + i] = 1
    for j in range(n):
        L[2 * n, j] = -B[j]
    L[2 * n, 2 * n] = c

    Lr = L.LLL()
    try:
        from fpylll import IntegerMatrix, BKZ
        A2 = IntegerMatrix.from_matrix(L)
        BKZ.reduction(A2, BKZ.Param(block_size=block))
        Lr = matrix(ZZ, [[A2[i, j] for j in range(dim)] for i in range(dim)])
    except Exception as e:
        print(" fpylll BKZ 不可用，仅用 LLL:", e)

    for row in Lr.rows():
        w = row[2 * n]
        if w == 0 or c % w:
            continue
        mul = c // w
        e = [-row[i] * mul for i in range(n)]
        s = [row[n + i] * mul for i in range(n)]
        if all(abs(v) <= error_bound for v in e):
            if all((sum(M[i, j] * s[j] for j in range(n)) + e[i] - B[i]) % p == 0
                   for i in range(n)):
                return s, e
    return None

res = None
for c in [1, 3, 10, 100]:
    res = solve_lwe(c)
    if res:
        break
assert res is not None, "LWE 未解出"
s, e = res
print("e =", e)
print("s =", s)

flag = bytes([v & 0xFF for v in s]).rstrip(b"\x00").decode("utf-8", "replace")
print("FLAG =", flag)
```

## 最终答案

```text
0xGame{kiss then kill}
```

---

## charlotte

```
掠夺了世界的人终究遗忘了世界

踽踽独行的少年挣扎着站立，但是珍视之物却脱手而出

不行，唯独那个不行，等到剥夺了最后者的勇气，就可以回去...什么了

单词本的残页躺在力竭的救世者的脸旁，揭示着勇者穿越时间的理由掠夺了世界的人终究遗忘了世界

踽踽独行的少年挣扎着站立，但是珍视之物却脱手而出

不行，唯独那个不行，等到剥夺了最后者的勇气，就可以回去...什么了

单词本的残页躺在力竭的救世者的脸旁，揭示着勇者穿越时间的理由
```

task
```
-----BEGIN RSA PRIVATE KEY-----
MIICawIBAAKBgQDRkl2wWYm9CI7ZYxeC8hq4NTnAf6z8yt18npO30RTsvzcFv8Mv
eX5d0188HxfwJVEwhsco9PFRJ7ge/NLWzZ2OADzDruWmUO+wrkLbOVtB+ZRDJswa
RIU2OrU+4GYmIQK1GamEuH/jHPzcg699CJfgcungDWD/zapCz/sVHRhXBwKBgCmJ
M+mKr8yk6OaePOmC6xAKWVW2NlCExuUvbggibmmLk5W/e5ryeTh03J0TRwy8MdUR
qSEQwkyn4hiCEQYlmAUrU+NxsrtMM9sKEsaDWq0/fD+7e/DVAq/ANxnuTfci4g==
-----END RSA PRIVATE KEY-----

-----BEGIN RSA PRIVATE KEY-----
MIICawIBAAKBgQDRkl2wWYm9CI7ZYxeC8hq4NTnAf6z8yt18npO30RTsvzcFv8Mv
eX5d0188HxfwJVEwhsco9PFRJ7ge/NLWzZ2OADzDruWmUO+wrkLbOVtB+ZRDJswa
RIU2OrU+4GYmIQK1GamEuH/jHPzcg699CJfgcungDWD/zapCz/sVHRhXBwKBgDk/
eMS772edGG2N8B60m6Edivr0I/8l/pDryqRp08/OJI7pCMknMDGUe6QsuIQhBv6r
CpXntVm/56t/QRCee9EYc/NsHqvGy+lcKnNhnlR3kIzsAiQr2mVZD17tzkMmoQ==
-----END RSA PRIVATE KEY-----

-----BEGIN RSA PRIVATE KEY-----
MIICawIBAAKBgQDRkl2wWYm9CI7ZYxeC8hq4NTnAf6z8yt18npO30RTsvzcFv8Mv
eX5d0188HxfwJVEwhsco9PFRJ7ge/NLWzZ2OADzDruWmUO+wrkLbOVtB+ZRDJswa
RIU2OrU+4GYmIQK1GamEuH/jHPzcg699CJfgcungDWD/zapCz/sVHRhXBwKBgDSV
PPAI5Nmat0rSpTVuWDz5o+riInx1gqBCRtkrLbyYtwuGDo6rVEZkmj2+50syn0UC
uc6HJ4cuNqtrHNQdU539tbgNq/tPHJUd00a7lJAmUYTukuGCbPHVKWMqBGXuWXJG
GDonMSQpaIMakVOAhkfooGFUQDOCmr+3HRVYjwzJ
-----END RSA PRIVATE KEY-----
```


```
fingerprint_1 = 117621280447918956283798935355235757929776352599339931163286329028682288672928741296937650952747752164593075018079028467934679452946461204759555416978276173572349600240326984045009804232377138973495099451743567280873976580203792736765439561418352263559096298617341483219124316753620841386349210989125279314842
fingerprint_2 = 39329072771123843841472391585293227954224412823311391442958322610173321736884077961544300334756676692058523958855671814128654378389736884837669229612690790592451391611463044331025427743215976335544181395997619074049060029702945823017674337670737757740219953460965325772553356128080668320929608609852645125536

missing_bytes = 32
private_exponent_bits = 384

c = 32087036959114728161878680850058726355415338046892509653810739925909118278199023226482176143218312195930020611529961801744464444373329743005046371825348619733884590604543726940740692929644171610583904558144464026665557404873812798180784825671288988482757869607517107787003813976979858269972652526464857516123

```

```
三本损坏的单词本静静地躺在黄沙之上
它们披着旧式 RSA PRIVATE KEY 的外壳，但末页像是被某种能力掠夺，常规程序无法读入。
友利的摄像机留下了两段能力观测记录，规则为 fingerprint = e^3 mod n。
三本单词本来自同一个 n，也就是同一个 RSA 模数。
恢复以 E = e0 * e1 * e2 为指数加密的最后记忆。
```python
from math import prod
from Crypto.Util.number import bytes_to_long

n = p * q
phi = (p - 1) * (q - 1)

d0, d1, d2 = private_exponents


e0 = pow(d0, -1, phi)
e1 = pow(d1, -1, phi)
e2 = pow(d2, -1, phi)
assert len(e0.to_bytes((e0.bit_length() + 7) // 8, "big")) == len(e1.to_bytes((e1.bit_length() + 7) // 8, "big")) == len(e2.to_bytes((e2.bit_length() + 7) // 8, "big"))


fingerprint_1 = pow(e1, 3, n)
fingerprint_2 = pow(e2, 3, n)

E = e0 * e1 * e2
c = pow(bytes_to_long(flag), E, n)
```


## 分析
首先从三个残缺的pem中可以提取出共同的n，以及2个残缺的e，其中两个e的丢失位数都是256bit
```
n  = 147166282029656457998360872951214656289139820777849266859228612656170358779790747150621966432249221091595835491298815807063687497957286824190809200951028795124638720548213390425164324111639894914301123391572929156854952741317337553853362883576444555679603687465622636290838121167626233018536879058328305030919   # 1024 bit
e0 = 36925010997027579107982931161379201712051423963604223715827313390309707389941302877553627013224095798587551115252710218246305055598067725170796349902720150003295988555702543188816519877069380915208155252458414807066478575108304051313328848400802948378454052554632982000277499489002763367545399874756407135433   # 残片3，1022 bit

e1_hi = 251895747354813682810330746102591421665896727175152048030297352122655688232916287692914172561613882098059303667670621915907348468557778513025196865372082542449628123147118995264672011942707360381558712440184080273856340246413124322   # 96 字节，残片1
e2_hi = 347181473481211852104366332246003086153313173146974682558863651231107972947388234566177083419595973413436201883370502988056523544977130554162948158304768508286237048224153746049979432821902249121891516551575579856604311195490657953   # 96 字节，残片2
```

那么e1，e2的完整状态可以这样表示出来
```
 e1 = e1_hi * 2**256 + x1,  0 <= x1 < 2**256
 e2 = e2_hi * 2**256 + x2,  0 <= x2 < 2**256
```

x1，x2约为n位数的四分之一，这种情况可以用copperSmith来恢复x1，x2,恢复出的结果可以用
```
fingerprint_1 = pow(e1, 3, n)
fingerprint_2 = pow(e2, 3, n)
```
来校验

结果
```
15039102176350504488514583955622444631556561408311901856025703807210399894779
110584866455075200407382092199859021148299383162214530170832155435499502589289
```

然后就是恢复pq，因为三把公钥共用 n，私钥 d0/d1/d2 均为 384 bit；且有
```
d ≈ n^0.375  >  n^0.292（Boneh-Durfee 界）> n^0.25（Wiener 界）
```
所以这里我们可以采用Howgrave-Graham–Seifert,根据论文，存在两个共享模数N的密钥对时，且`d < n^0.357`时，存在高效恢复私钥的方法，随着样本增多，d的界最多可以达到`n^0.5`

解出pq后就是常规rsa了。

### EXP
```
from itertools import combinations  
from math import prod, isqrt  
  
n = 147166282029656457998360872951214656289139820777849266859228612656170358779790747150621966432249221091595835491298815807063687497957286824190809200951028795124638720548213390425164324111639894914301123391572929156854952741317337553853362883576444555679603687465622636290838121167626233018536879058328305030919  
e0 = 36925010997027579107982931161379201712051423963604223715827313390309707389941302877553627013224095798587551115252710218246305055598067725170796349902720150003295988555702543188816519877069380915208155252458414807066478575108304051313328848400802948378454052554632982000277499489002763367545399874756407135433  
e1_hi = 251895747354813682810330746102591421665896727175152048030297352122655688232916287692914172561613882098059303667670621915907348468557778513025196865372082542449628123147118995264672011942707360381558712440184080273856340246413124322  
e2_hi = 347181473481211852104366332246003086153313173146974682558863651231107972947388234566177083419595973413436201883370502988056523544977130554162948158304768508286237048224153746049979432821902249121891516551575579856604311195490657953  
fp1 = 117621280447918956283798935355235757929776352599339931163286329028682288672928741296937650952747752164593075018079028467934679452946461204759555416978276173572349600240326984045009804232377138973495099451743567280873976580203792736765439561418352263559096298617341483219124316753620841386349210989125279314842  
fp2 = 39329072771123843841472391585293227954224412823311391442958322610173321736884077961544300334756676692058523958855671814128654378389736884837669229612690790592451391611463044331025427743215976335544181395997619074049060029702945823017674337670737757740219953460965325772553356128080668320929608609852645125536  
c = 32087036959114728161878680850058726355415338046892509653810739925909118278199023226482176143218312195930020611529961801744464444373329743005046371825348619733884590604543726940740692929644171610583904558144464026665557404873812798180784825671288988482757869607517107787003813976979858269972652526464857516123  
  
B = 2^256  
  
# ---------- Step 1: Coppersmith 恢复 e1, e2 ----------  
R.<x> = PolynomialRing(Zmod(n))  
def recover(hi, fp, name):  
    f = ((hi * B + x)^3 - fp).monic()  
    roots = f.small_roots(X=B, beta=1.0, epsilon=0.05)  
    assert roots, f"{name}: small_roots failed"  
    e = hi * B + int(roots[0])  
    assert pow(e, 3, n) == fp, f"{name}: verify failed"  
    print(f"{name} = {e}  (bits={e.nbits()}, verify OK)")  
    return e  
  
e1 = recover(e1_hi, fp1, "e1")  
e2 = recover(e2_hi, fp2, "e2")  
  
# ---------- Step 2: HGS 多公钥小私钥攻击分解 n ----------  
def construct_relation(N, monomial, x):  
    vars = monomial.variables()  
    l = [x[i] for i in range(x.index(vars[-1]) + 1)]  
    R = 1; u = 0; v = 0; i = len(vars)  
    for var in vars:  
        if var != x[0] and var < l[0] and len(l) >= 2 * i:  
            R *= l[0] - var; l.pop(0); v += 1  
        else:  
            R *= var - N; u += 1  
        l.remove(var); i -= 1  
    return R, u, v  
  
def hgs_factor(N, e, alpha):  
    k = len(e)  
    pr = ZZ[",".join(f"x{i}" for i in range(k))]  
    xs = pr.gens()  
    monomials = [1]  
    for i, xi in enumerate(xs):  
        monomials.append(xi)  
        for j in range(i):  
            for comb in combinations(xs[:i], j + 1):  
                monomials.append(prod(comb) * xi)  
    L = matrix(ZZ, len(monomials))  
    exp_a = [k]; exp_b = [0]  
    for col, monomial in enumerate(monomials):  
        if col == 0:  
            L[0, 0] = 1; continue  
        R, u, v = construct_relation(N, monomial, xs)  
        for row, mon2 in enumerate(monomials):  
            if row == 0:  
                L[0, col] = R.constant_coefficient()  
            else:  
                L[row, col] = R.monomial_coefficient(mon2) * mon2(*e)  
        exp_a.append(k - v); exp_b.append(u / 2)  
    max_a = max(exp_a); max_b = max(exp_b)  
    D = matrix(ZZ, len(monomials))  
    for i, (a, b) in enumerate(zip(exp_a, exp_b)):  
        D[i, i] = int(RR(N) ** ((max_a - a) * alpha + (max_b - b)))  
    L = L * D  
    L_ = L.LLL()  
    for row in L_.rows():  
        if row.is_zero():  
            continue  
        try:  
            b = L.solve_left(row)  
        except Exception:  
            continue  
        if b[0] == 0:  
            continue  
        phi = round(b[1] / b[0] * e[0])  
        if phi <= 0 or phi >= N:  
            continue  
        s = N + 1 - phi  
        disc = s * s - 4 * N  
        if disc < 0:  
            continue  
        r = isqrt(int(disc))  
        if r * r != disc:  
            continue  
        p = (s - r) // 2; q = (s + r) // 2  
        if p * q == N and p > 1:  
            return int(p), int(q)  
    return None  
  
p = q = None  
for alpha in [0.30, 0.32, 0.34, 0.35, 0.36, 0.37, 0.375, 0.38, 0.40, 0.45]:  
    r = hgs_factor(n, [e0, e1, e2], alpha)  
    if r:  
        p, q = r  
        break  
assert p and q and p * q == n  
print(f"p = {p}")  
print(f"q = {q}")  
  
# ---------- Step 3: 解密 ----------  
phi = (p - 1) * (q - 1)  
E = e0 * e1 * e2  
d = inverse_mod(E, phi)  
m = pow(c, d, n)  
flag = int(m).to_bytes((int(m).bit_length() + 7) // 8, "big")  
print("d0 bits:", int(inverse_mod(e0, phi)).bit_length())  
print("d1 bits:", int(inverse_mod(e1, phi)).bit_length())  
print("d2 bits:", int(inverse_mod(e2, phi)).bit_length())  
print("FLAG =", flag.decode())  
```

## 最终答案
```
0xGame{Welcome home}
```

---

## None game None life zero!


## 分析

服务每次 GET 返回 312 行十六进制数

每行是下面两个值之一，各 50% 概率：
MT19937 的下一个输出；
LFSR 的下一个输出。

目标是恢复 MT19937 的 128 位 static_seed。

其中四个附件的作用如下：

| 文件 | 作用 |
|------|------|
| `mt19937.py` | 标准 MT19937，128 位种子走 init_by_array |
| `lfsr.py` | 128 位 LFSR，输出经过 256 项 SBox |
| `ot.py` | RSA 1-out-of-2 OT，负责"二选一" |
| `protocol.py` | 每批 312 行，起点随机 |

可以观察到以下三点：

**1.OT 是干扰项**
服务端只回显"二选一"的结果，不发送 RSA 公钥，也不发送密文。所以 OT 对解题没有帮助，它只是一个 50% 的随机采样器。

**2.LFSR 只有 226 种输出**
LFSR 的 SBox 虽然有 256 项，但只有 226 个不同的值。
MT 输出是 32 位随机数，落进这 226 个值的概率极低。所以：

- 反复出现的值 → LFSR 输出；
- 基本只出现一次的值 → MT 输出。

这样就能把每行分类。

**3.MT 输出可以还原**
MT19937 的输出经过 temper，但 temper 是可逆的。把 MT 行逆 temper，就得到该位置的内部状态词。

**首先分类**

统计所有批次里每个值出现的次数。出现次数特别多的 226 个值就是 LFSR 的值域。

- 值在这个集合里 → LFSR 行，丢弃；
- 值不在 → MT 行，保留。

**再还原MT状态**

对每个 MT 行做逆 temper：

```
y ^= y >> 18
y ^= (y << 15) & 0xefc60000
y ^= (y << 7)  & 0x9d2c5680
y ^= y >> 11
```

得到状态词 `s[p]`。

然后把不同批次拼起来
每批起点是随机的。如果两批起点相差小于 312，它们就有重叠。重叠区内，同一个绝对位置的 MT 输出是相同的。利用这些相同值，可以算出两批的起点差，从而把所有批次拼成一个个"簇"。

然后解出 MT 状态
MT 的递推关系是：

```
s[p] = s[p-227] ⊕ ((s[p-624]₃₁ ‖ s[p-623]_(low31)) ≫ 1) ⊕ (A · (s[p-623] & 1))
```

也就是说，第 p 个词完全由前面 227、623、624 个词决定。

这里我们取 624 个词作为未知数（共 19968 bit），把每个观测词展开成这些未知数的线性方程，用 GF(2) 消元求解。解出来后会缺 31 个 bit（恰好是第一个词的低 31 位），可以用已知区域反推补上。

最后倒推回种子上面的递推可以反着用：
从已知状态出发，一步步往前推； 一直推到种子刚生成时的初始状态 S_0；
初始状态的第一词有固定值 `0x80000000`（代码里写死的），正好补上缺失的 bit。

初始状态的绝对位置还不知道，需要枚举（范围 `[0, 2^20)`）。对每个候选位置，逆 `init_by_array` 得到 4 个 key 字，真正的种子会让 4 个 key 槽完全一致； 错误的位置只会得到随机噪声。枚举完就能唯一确定种子。

这题主要还是采集样本花费时间。

## exp

ai写的脚本太长了，这里也不放了。。。

### 最终答案

 ```text
0xGame{C6A4F19B7D2E8F0315D9A6E4B80C3F27}（最后答案是大写有点忘记是十进制大写还是16进制大写了）
 ```



## 总结

质量还行，希望下次不要再惨misc进来了。