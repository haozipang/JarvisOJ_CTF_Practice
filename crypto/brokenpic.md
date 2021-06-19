参考题解链接 https://github.com/xalanq/jarvisoj-solutions/blob/master/crypto/BrokenPic.md

# 描述
这里有个图片，可是好像打不开？

Hint1: 图片大小是1366*768

brokenpic.bmp.d82d5d25027ff5e9e3ae90022ec386f9

# 题解
strings、binwalk 均得不到信息，根据提示，图片大小是 1366×768，那么 bmp 图片的照片大小理应（假设 24-bit）为 1366×768×24/8 = 3147264 byte，和给出的图片 3148800 byte 差不多，可是是丢了头吧。

打开 http://www.ece.ualberta.ca/~elliott/ee552/studentAppNotes/2003_w/misc/bmp_file_format/bmp_file_format.htm 查看 bmp 文件的存储格式，发现和给的文件完全对不上，header 全部没有，所以猜测应该是直接加一串 header 到前面。

但是这是愚蠢的办法，在网上我搜到了 666 的方法，新建一张空的 1366×768，然后把数据复制进去即可。。。。。。nb

加上头后，图片能显示了，出现了一个二维码和一串 key

K3Y: PHRACK-BROKENPIC

然后发现自己并不会做了，只能看题解

题解说是图片的数据特别有规律？所以应该是块加密，用 AES 尝试一下解密原来那张图片的数据。需要添加一个AES的模式，用ECB试试看

```python
from Crypto.Cipher import AES
from icecream import ic

import sys

def decrypt(encrypted, passphrase):
  aes = AES.new(passphrase, AES.MODE_ECB)
  return aes.decrypt(encrypted)

raw = open(r"D:\01 CTF practise\06 crypto\Jarvis OJ\brokenpic.bmp", 'rb').read()
ic(raw)
data = decrypt(raw, "PHRACK-BROKENPIC")
ic(data)
open(r'C:\Users\xuhao\Desktop\brokenpic_test111.bmp', 'wb').write(data)
```
