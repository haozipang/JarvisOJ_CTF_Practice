题目链接：https://www.jarvisoj.com/challenges

![image](https://github.com/haozipang/firstRep/blob/master/%E5%9B%BE%E7%89%87/xbase64-2020-12-27-163748.png)

下载附件查看内容是一个base64的加密函数encode_b64：大致原理是将字符转为二进制，按8位右对齐，3个字符24位拼接后按6 bit一组，转化为4组二进制，每组转为10进制后在base64_table中查找字符进行encode拼接，最终输出加密后字符串result“mZOemISXmpOTkKCHkp6Rgv==”。要得到flag，就是需要反过来求解出原文。

由于正向思路简单来说是 8/8/8 ----> 6/6/6/6，那么反向就是从result每四个字符经过运算得到原文的三个字符。代码如下：

```
# /usr/bin/python
# encoding: utf-8
base64_table = ['=','A', 'B', 'C', 'D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R','S','T','U','V','W','X','Y','Z',
                'a', 'b', 'c', 'd','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z',
                '0', '1', '2', '3','4','5','6','7','8','9',
                '+', '/'][::-1]

result = "mZOemISXmpOTkKCHkp6Rgv=="

i = 0
l = len(base64_table)
s = ''
while i < len(result):
    ri1 = base64_table.index(result[i])
    i += 1
    ri2 = base64_table.index(result[i])
    i += 1
    ri3 = base64_table.index(result[i])
    i += 1
    ri4 = base64_table.index(result[i])

    if ri3 == (l-1):
        ri3 = 0
    if ri4 == (l-1):
        ri4 = 0

    rb1 = bin(ri1)[2:]
    rb1 = rb1.rjust(6, '0')
    rb2 = bin(ri2)[2:]
    rb2 = rb2.rjust(6, '0')
    rb3 = bin(ri3)[2:]
    rb3 = rb3.rjust(6, '0')
    rb4 = bin(ri4)[2:]
    rb4 = rb4.rjust(6, '0')

    # 将四字节转换为三字节
    cb = rb1 + rb2 + rb3 + rb4
    cb1 = cb[:8]
    cb2 = cb[8:16]
    cb3 = cb[16:]

    s += chr(int(cb1, 2)) + chr(int(cb2, 2)) + chr(int(cb3, 2))
    i += 1
print(s)
```

执行输出得到： flag{hello_xman}  
