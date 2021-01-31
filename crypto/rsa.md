# 题目
下载附件后得到几个文件（alice_public_key.py/bob_public_key.py/communication.py/keygen.sage/send.bak）

# 分析思路
1，查看communication.py文件知道是alice发送消息给bob，中间会有加密签名；然后解密校验。  
其中一句关键内容`m = urandom(126-len(m)) + '\x00' + m`是在明文前面添加padding，这个与我们常见的这个攻击（Coppersmith' short-pad attack）略有不同。

2，查找一些资料后，是与`Dual RSA`相关，利用别人的脚本sage调试可以得出结果，具体的原理暂时还没有弄的很清楚。就把sage内容放在下面：
环境信息：windows上运行sage.shell,内置是python3的版本。提前把密文c求出后，把n/e赋值进去计算

读取密文脚本：
```
import binascii
from Crypto.Util.number import *

with open('send.bak', 'rb') as f:
    c = f.read()
    print(bytes_to_long(c))
```

sage脚本：
```
from sage.all import *
import math
import itertools
import binascii

# display matrix picture with 0 and X
# references: https://github.com/mimoo/RSA-and-LLL-attacks/blob/master/boneh_durfee.sage
def matrix_overview(BB):
    for ii in range(BB.dimensions()[0]):
        a = ('%02d ' % ii)
        for jj in range(BB.dimensions()[1]):
            a += ' ' if BB[ii,jj] == 0 else 'X'
            if BB.dimensions()[0] < 60:
                a += ' '
        print( a)

def dual_rsa_liqiang_et_al(e, n1, n2, delta, mm, tt):
    '''
    Attack to Dual RSA: Liqiang et al.'s attack implementation

    References:
        [1] Liqiang Peng, Lei Hu, Yao Lu, Jun Xu and Zhangjie Huang. 2016. "Cryptanalysis of Dual RSA"
    '''
    N = (n1+n2)/2
    A = ZZ(floor(N^0.5))

    _XX = ZZ(floor(N^delta))
    _YY = ZZ(floor(N^0.5))
    _ZZ = ZZ(floor(N^(delta - 1./4)))
    _UU = _XX * _YY + 1

    # Find a "good" basis satisfying d = a1 * l'11 + a2 * l'21
    M = Matrix(ZZ, [[A, e], [0, n1]])
    B = M.LLL()
    l11, l12 = B[0]
    l21, l22 = B[1]
    l_11 = ZZ(l11 / A)
    l_21 = ZZ(l21 / A)

    modulo = e * l_21
    F = Zmod(modulo)

    PR = PolynomialRing(F, 'u, x, y, z')
    u, x, y, z = PR.gens()

    PK = PolynomialRing(ZZ, 'uk, xk, yk, zk')
    uk, xk, yk, zk = PK.gens()

    # For transform xy to u-1 (unravelled linearlization)
    PQ = PK.quo(xk*yk+1-uk)

    f = PK(x*(n2 + y) - e*l_11*z + 1)

    fbar = PQ(f).lift()

    # Polynomial construction
    gijk = {}
    for k in range(0, mm + 1):
        for i in range(0, mm-k + 1):
            for j in range(0, mm-k-i + 1):
                gijk[i, j, k] = PQ(xk^i * zk^j * PK(fbar) ^ k * modulo^(mm-k)).lift()

    hjkl = {}
    for j in range(1, tt + 1):
        for k in range(floor(mm / tt) * j, mm + 1):
            for l in range(0, k + 1):
                hjkl[j, k, l] = PQ(yk^j * zk^(k-l) * PK(fbar) ^ l * modulo^(mm-l)).lift()

    monomials = []
    for k in gijk.keys():
        monomials += gijk[k].monomials()
    for k in hjkl.keys():
        monomials += hjkl[k].monomials()

    monomials = sorted(set(monomials))[::-1]
    assert len(monomials) == len(gijk) + len(hjkl) # square matrix?
    dim = len(monomials)

    # Create lattice from polynmial g_{ijk} and h_{jkl}
    M = Matrix(ZZ, dim)
    row = 0
    for k in gijk.keys():
        for i, monomial in enumerate(monomials):
            M[row, i] = gijk[k].monomial_coefficient(monomial) * monomial.subs(uk=_UU, xk=_XX, yk=_YY, zk=_ZZ)
        row += 1
    for k in hjkl.keys():
        for i, monomial in enumerate(monomials):
            M[row, i] = hjkl[k].monomial_coefficient(monomial) * monomial.subs(uk=_UU, xk=_XX, yk=_YY, zk=_ZZ)
        row += 1

    matrix_overview(M)
    print ('=' * 128)

    # LLL
    B = M.LLL()

    matrix_overview(B)

    # Construct polynomials from reduced lattices
    H = [(i, 0) for i in range(dim)]
    H = dict(H)
    for j in range(dim):
        for i in range(dim):
            H[i] += PK((monomials[j] * B[i, j]) / monomials[j].subs(uk=_UU, xk=_XX, yk=_YY, zk=_ZZ))
    H = list(H.values())

    PQ = PolynomialRing(QQ, 'uq, xq, yq, zq')
    uq, xq, yq, zq = PQ.gens()

    # Inversion of unravelled linearlization
    for i in range(dim):
        H[i] = PQ(H[i].subs(uk=xk*yk+1))

    # Calculate Groebner basis for solve system of equations
    '''
    Actually, These polynomials selection (H[1:20]) is heuristic selection.
    Because they are "short" vectors. We need a short vector less than
    Howgrave-Graham bound. So we trying test parameter(at [1]) and decided it.
    '''
    I = Ideal(*H[1:20])
    g = I.groebner_basis('giac')[::-1]
    mon = list(map(lambda t: t.monomials(), g))

    PX = PolynomialRing(ZZ, 'xs')
    xs = PX.gen()

    x_pol = y_pol = z_pol = None

    for i in range(len(g)):
        if mon[i] == [xq, 1]:
            print (g[i] / g[i].lc())
            x_pol = g[i] / g[i].lc()
        elif mon[i] == [yq, 1]:
            print (g[i] / g[i].lc())
            y_pol = g[i] / g[i].lc()
        elif mon[i] == [zq, 1]:
            print (g[i] / g[i].lc())
            z_pol = g[i] / g[i].lc()

    if x_pol is None or y_pol is None or z_pol is None:
        print ('[-] Failed: we cannot get a solution...')
        return

    x0 = x_pol.subs(xq=xs).roots()[0][0]
    y0 = y_pol.subs(yq=xs).roots()[0][0]
    z0 = z_pol.subs(zq=xs).roots()[0][0]

    # solution check
    assert f(x0*y0+1, x0, y0, z0) % modulo == 0

    a0 = z0
    a1 = (x0 * (n2 + y0) + 1 - e*l_11*z0) / (e*l_21)

    d = a0 * l_11 + a1 * l_21
    return d

if __name__ == '__main__':
    delta = 0.334
    mm = 4
    tt = 2

    an1 = 73871864906066571406532457145607595847175670495017175565388132342574183187410380661917315431292987479576202710800243301821110154265031908769855345111418669574603508401169110155260587673272035301147668990301239964950207744512780378527326467318883681760821710529273075846801071296284930852211925448126134456371
    an2 = 10548244241379959973012698997313565838321599691374783075397881482031959048170291643267402594849260277539547268187806692681844616430159988384126260317181183835757828333838579492655650376150221839149317599035426574857154820265992983356076281713571392012617269312376565128009638293040584673854656713040776519371
    ae = 16140311304784625807971244520043756160102111440186500282911137675250199677022940503982292569233803572106227632954920805936151799049981849612210588147943318190599476331582153430089688143361226994947530603424082905816220548156412649246782363716825121201733242060234874574393149811078415952220812029904096645141

    d1 = dual_rsa_liqiang_et_al(ae, an1, an2, delta, mm, tt)
    print ('[+] d for alice = %d' % d1)

    bn1 = 68253958478963934124300215757460723273691925280596309221863375905836387105252457670797482776100691909463871547306272253629697109054703333176926409544379734932863965587299902358818026560280727724874176607409038276642286734734838152097800253007709025880304435661500771500046972983240470823245360867996258239121
    bn2 = 58831168144192031952989136031836716698976227434725683505500936300434624499242689272670040374754049533739858144785202063762391111213777476380071324869525517503277889732429358219404717664025176836710995641523459333571168229803403205064106761733957455221168205091994936332421038092333687260644067187030525515969
    be = 57019382772166837488750427040015464490863175936396575556290232679668259683514801918298546850349718538222635977304361506235546587039055174992606638351023434322917005735910505716857963030159773221874370814472973319422853023944637626372539074831161749650232791256091117301145054914651557523038888181457290682237

    d2 = dual_rsa_liqiang_et_al(be, bn1, bn2, delta, mm, tt)
    print ('[+] d for bob = %d' % d2)

    d = 30188087781431663910278476447291254596857891395494139124262164625255234021264319375632495236432448013412895064964007396287217600534301744900532989602468245511065073457458795318105641417330210058198108486298751761073666821130984342724649986836971657675474869345788972834637141803437794987136914926817947807825
    m = ZZ(Mod(ZZ(Mod(d, bn1)^d2), an2)^ae)
    m_ = hex(m)[2:]
    if len(m_) % 2 == 1:
        m_ = '0' + m_
    print (repr(binascii.a2b_hex(m_)))
 ```
 
 得到结果：
 // Groebner basis computation time 6.2e-05 Memory 238.456M
zq - 143145407796820419691921116
yq + 15352514463512545429409965865844475808601686641370002128014114724769375278708730306689584590428850472250709065231080455974049923847598927444538942404357569
xq - 6046141695019859939882486410408884140504805414384153622531566562183840881867683963179563847287957629047
[+] d for bob = 6238257262527822874210763706228720299138117835182790315423266965504012841647775705014056496108579279573
b"\x7f\xc6\x0b/\x89\xd5\xf4u\x8d\xcf\nH&\x04V\xff\xbd:\xe8\x02\x9c\xd5\xca\xe8\x9bFo'?\x1d\x195Ci\xa3\xb8\xa0w\x02W\x9d*@$\xc5\xaa=\xba\x00Well, OK... Here is what you what: `flag{eNj0y_th3_CrYpt4na1ysi5_0F_Dual_RSA~~}`"
