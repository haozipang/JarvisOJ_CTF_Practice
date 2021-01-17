# 题目
下载附件后查看py内容，flag长度为32位，将其转为16进制后进行bbencode运算，每次会移动一位
```
flag = open("flag", "r").read().strip()
assert len(flag) == 32
def str2num(s):
    return int(s.encode('hex'), 16)
def bbencode(n):
    a = 0
    for i in bin(n)[2:]:
        a = a << 1
        if (int(i)):
            a = a ^ n
        if a >> 256:
            a = a ^ 0x10000000000000000000000000000000000000000000000000000000000000223L
    return a

print bbencode(str2num(flag))

#result:61406787709715709430385495960238216763226399960658358000016620560764164045692
```

# 思路分析
查找资料大概了解是与有限域相关
https://www.pythonf.cn/read/154184

参考其中的方式写了一个脚本试运行，可以得出结果：
```
def bbencode(n):
    a = 0
    for i in bin(n)[2:]:
        a = a << 1
        if (int(i)):
            a = a ^ n
        if a >> 256:
            a = a ^ 0x10000000000000000000000000000000000000000000000000000000000000223
    return a

a = 61406787709715709430385495960238216763226399960658358000016620560764164045692

for i in range(10000):
    try:
        h = hex(a)[2:]
        s = bytes.fromhex(h)
        if b'flag' in s:
            print(s)
    except:
        pass
    a = bbencode(a)
```
