# 题目
相信你已经做出了medium RSA，这题的pubkey在medium RSA的基础上我做了点手脚，继续挑战吧。  
Hint1: 1.不需要爆破。2.用你的数学知识解决此题。3.难道大家都不会开根号吗？  

# 思路分析
与`Medium RSA`题目一样，对pubkey.pem分析后发现e有点不同，e=2，这里可以参考ctf-wiki的[Rabin算法](https://ctf-wiki.org/crypto/asymmetric/rsa/rsa_e_attack/#rsa-rabin)求解：  
代入代码：
```
#!/usr/bin/python
# coding=utf-8
import gmpy2
import string
from Crypto.PublicKey import RSA

# 读取公钥参数
with open('pubkey.pem', 'r') as f:
    key = RSA.importKey(f)
    N = key.n
    e = key.e
with open('flag.enc', 'r') as f:
    cipher = f.read().encode('hex')
    cipher = string.atoi(cipher, base=16)
    # print cipher

p = 275127860351348928173285174381581152299
q = 319576316814478949870590164193048041239
# 计算yp和yq
inv_p = gmpy2.invert(p, q)
inv_q = gmpy2.invert(q, p)

# 计算mp和mq
mp = pow(cipher, (p + 1) / 4, p)
mq = pow(cipher, (q + 1) / 4, q)

# 计算a,b,c,d
a = (inv_p * p * mq + inv_q * q * mp) % N
b = N - int(a)
c = (inv_p * p * mq - inv_q * q * mp) % N
d = N - int(c)

for i in (a, b, c, d):
    s = '%x' % i
    if len(s) % 2 != 0:
        s = '0' + s
    print s.decode('hex')
```
