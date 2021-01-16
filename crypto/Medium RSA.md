# 题目  
Jarvis OJ 下载Medium RSA题目附件，得到两个文件：flag.enc 和 publibc.pem。

# 思路分析  
将public.pem通过openssl工具查看其内容，  
```
kali@kali:~/crypto$ openssl rsa -pubin -in pubkey.pem -text -modulus
RSA Public-Key: (256 bit)
Modulus:
    00:c2:63:6a:e5:c3:d8:e4:3f:fb:97:ab:09:02:8f:
    1a:ac:6c:0b:f6:cd:3d:70:eb:ca:28:1b:ff:e9:7f:
    be:30:dd
Exponent: 65537 (0x10001)
Modulus=C2636AE5C3D8E43FFB97AB09028F1AAC6C0BF6CD3D70EBCA281BFFE97FBE30DD
writing RSA key
-----BEGIN PUBLIC KEY-----
MDwwDQYJKoZIhvcNAQEBBQADKwAwKAIhAMJjauXD2OQ/+5erCQKPGqxsC/bNPXDr
yigb/+l/vjDdAgMBAAE=
-----END PUBLIC KEY-----
```  
得到n和e：  
n = C2636AE5C3D8E43FFB97AB09028F1AAC6C0BF6CD3D70EBCA281BFFE97FBE30DD  ---- 需要转换成十进制  
e = 65537

将n进行分解得到p和q，已知了p/q/e之后，可以想办法生成私钥对flag.enc进行解密。得到私钥的python脚本如下：  
```
# coding=utf-8
# python2
from Crypto.PublicKey import RSA
import gmpy2

p = 275127860351348928173285174381581152299
q = 319576316814478949870590164193048041239
e = 65537
n = p * q
phi_n = (p-1) * (q-1)
d1 = gmpy2.invert(e, phi_n)
rsa_components = (n, e, int(d1), p, q)
arsa = RSA.construct(rsa_components)
private = open('private.pem', 'w')
private.write(arsa.exportKey())
private.close()
```  
得到private.pem后再用openssl进行求解：
```
kali@kali:~/crypto$ openssl
OpenSSL> rsautl -decrypt -inkey private.pem -in flag.enc -out flag
OpenSSL> quit
```
查看flag文件即得到答案。

备注：  
网上有些脚本会使用如下方式求解私钥，在运行时出现“AttributeError: can't set attribute”的错误提示，这个是因为RSA.generate的属性不能被修改：
参考：http://www.secwk.com/2020/09/11/21804/
```
from Crypto.PublicKey import RSA

keypair = RSA.generate(1024)
keypair.p =3487583947589437589237958723892346254777
keypair.q =8767867843568934765983476584376578389
keypair.e = 65537
keypair.n = keypair.p * keypair.q
……
……
```
