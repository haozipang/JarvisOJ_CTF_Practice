# 题目描述
Jarvisoj的Crypto的xfz附件下载后得到两个文件：fez.py 和 fez.log  
查看py脚本内容了解大致内容：
```
import os
def xor(a,b):
    assert len(a)==len(b)
    c=""
    for i in range(len(a)):
        c+=chr(ord(a[i])^ord(b[i]))
    return c
def round(M,K):
    L=M[0:27]
    R=M[27:54]
    new_l=R
    new_r=xor(xor(R,L),K)
    return new_l+new_r
def fez(m,K):
    for i in K:
        m=round(m,i)
    return m

K=[]
for i in range(7):
    K.append(os.urandom(27))
m=open("flag","rb").read()
assert len(m)<54
m+=os.urandom(54-len(m))

test=os.urandom(54)
print test.encode("hex")
print fez(test,K).encode("hex")
print fez(m,K).encode("hex")
```
K是一个7元素列表，每个元素是一个27位的随机比特字符串，记作[[K1]，[K2]，[K3]，[K4]，[K5]，[K6]，[K7]]  
m是我们需要求解得到的flag内容，它不足以54位，所以后面会随机补充以得到是一个54位字符串  
test是已知条件，具体值在log的第一行得知。

# 思路分析
加密的方式是将54位分成两段，与K列表元素进行7次异或运算
|           | 左27位          |右27位       |
|-----------|--------------- |-------------|
|test       |L               |R            |
|test1      |R               |R^L^K1       |
|test2      |R^L^K1          |R^R^L^K1^K2 = L^K1^K2  |
|test3      |L^K1^K2         |R^L^K1^L^K1^K2^K3 = R^K2^K3   |
|test4      |R^K2^K3         |L^K1^K2^R^K2^K3^K4 = L^K1^R^K3^K4 |
|test5      |L^K1^R^K3^K4    |R^K2^K3^L^K1^R^K3^K4^K5 = L^K1^K2^K4^K5   |
|test6      |L^K1^K2^K4^K5   |L^K1^R^K3^K4^L^K1^K2^K4^K5^K6 = R^K2^K3^K5^K6  |
|test7      |R^K2^K3^K5^K6   |L^K1^K2^K4^K5^R^K2^K3^K5^K6^k7 = L^R^k1^k3^k4^K6^K7    |  
最后左边27位是 R^K2^K3^K5^K6， 右边27位是 L^R^k1^k3^k4^K6^K7

后面就简单了，与log中的对应数值进行求解即可，详细脚本：
```
# /usr/bin/python
# -*- coding: utf-8 -*-

test = '50543fc0bca1bb4f21300f0074990f846a8009febded0b2198324c1b31d2e2563c908dcabbc461f194e70527e03a807e9a478f9a56f7'
test_k = '66bbd551d9847c1a10755987b43f8b214ee9c6ec2949eef01321b0bc42cffce6bdbd604924e5cbd99b7c56cf461561186921087fa1e9'
m_k = '44fc6f82bdd0dff9aca3e0e82cbb9d6683516524c245494b89c272a83d2b88452ec0bfa0a73ffb42e304fe3748896111b9bdf4171903'


def xor(a, b):
    assert len(a) == len(b)
    c = ""
    for i in range(len(a)):
        c += chr(ord(a[i]) ^ ord(b[i]))
    return c


test_byte = test.decode('hex')
test_byte_L = test_byte[0:27]
test_byte_R = test_byte[27:54]

test_k_byte = test_k.decode('hex')
test_k_byte_L = test_k_byte[0:27]
test_k_byte_R = test_k_byte[27:54]

m_k_byte = m_k.decode('hex')
m_k_byte_L = m_k_byte[0:27]
m_k_byte_R = m_k_byte[27:54]

k2k3k5K6 = xor(test_k_byte_L, test_byte_R)
print("k2k3k5:", k2k3k5K6)
LR = xor(test_byte_L, test_byte_R)
print("LR:", LR)
K1K3K4K6K7 = xor(test_k_byte_R, LR)
print("k1k3k4:", K1K3K4K6K7)

m_R = xor(m_k_byte_L, k2k3k5K6)
print("m_R:", m_R)
mLR = xor(m_k_byte_R, K1K3K4K6K7)
m_L = xor(mLR, m_R)
print("m_L:", m_L)
```

最后拼接起来就是flag：  
('m_R:', '9vh12h3nvm}\x0ei\x10\xf1B\xeaX\x99H\x95\x96\xe04\x00\xb55')  
('m_L:', 'flag{festel_weak_666_10fjid')




