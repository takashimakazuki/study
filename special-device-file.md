# special device file

## 問題

* runme_8a10b7425cea81a043db0fd352c82a370a2d3373というファイルが与えられる。

* cross-gcc494-v1.0.zipというファイルも与えられている。様々なアーキテクチャのelfファイルを逆アセンブルできるgdbが
入っているらしい。



## 解き方

1.とりあえずファイルの種類を知るためにfileコマンドを打ってみる。

~~~
$ file runme_8a10b7425cea81a043db0fd352c82a370a2d3373
runme_8a10b7425cea81a043db0fd352c82a370a2d3373: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, not stripped
~~~

2.stringsコマンドを打ってみる

~~~
$ strings runme_8a10b7425cea81a043db0fd352c82a370a2d3373
/dev/xorshift64
GCC: (GNU) 4.9.4
/tmp/ccZkzjIB.o
_stack_addr
aarch64-elf.c
decode
~~~

3.逆アセンブルもやってみるが、できなかった。

~~~
objdump -d runme_8a10b7425cea81a043db0fd352c82a370a2d3373

runme_8a10b7425cea81a043db0fd352c82a370a2d3373:     ファイル形式 elf64-little

objdump: アーキテクチャ UNKNOWN! 用に逆アセンブルできません

~~~

## ここまでで分かったこと

* /dev/xorshift64 というデバイスを使っている

* objdump コマンドが使えない

* アーキテクチャはARM,aarch64というものらしい
 objdumpコマンドはaarch64というアーキテクチャに対応していないので逆アセンブルできなかった。

* readelf -h でもアーキテクチャが確認できる


## 実行ファイルのアセンブリを読みたい

aarch64の逆アセンブルを行う方法を探したところ、
[Online Disassembler](https://onlinedisassembler.com/static/home/index.html)で逆アセンブルができる

aarch64で使われている命令を知るために[ARM公式のドキュメント](https://developer.arm.com/products/architecture/cpu-architecture/a-profile/exploration-tools)を見た。

日本語のものも読みたかったので、旧バージョンだが下のページも見ながらアセンブリの命令を確認した。

http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0801aj/BABBDDIB.html

aarch64で使われている命令たち(命令セット)のことをA64というらしい。


### 実行ファイルの大まかな動き

1. メモリの0x1800 ~ 0x185fになんかよくわからん数値が1バイトずつ入っている。

2. /dev/xorshift64に何か固定値を書き込む(確認方法がわからない)

3. decode関数でflagをデコード

(flagとrandvalとxorshift乱数のxorをしている。)

4. 文字を出力？


### xorshift64(乱数を生成)
~~~ python
seed = 0x139408dcbbf7a44
mask = 0xffffffffffffffff
randval_xor =[]

def xorshift64():
    global seed
    seed = seed ^ (seed << 13) & mask
    seed = seed ^ (seed >> 7) & mask
    seed = seed ^ (seed << 17) & mask
    return seed

for i in range(32):
    lastbyte = xorshift64() & 0xff
    randval_xor.append(hex(lastbyte))
    
 print(randval_xor)
~~~

xorshiftのシフト回数は実装によるが[wikipedia](https://ja.wikipedia.org/wiki/Xorshift)を真似した。(seed値もwikipediaのものを使った。)

##### decode関数内

~~~python
flag = "fe 75 88 a9 5a aa 10 52 9c 6a 67 f4 82 be 21 56 59 0b 97 32 21 46 93 ae 40 0d 2e 1f 83 43 40".split(" ")
randval = "1d ab 1b 0f a7 d9 1a b0 61 7e b6 48 a4 56 cf 7e 49 05 fd 05 9c f9 54 45 fa 24 c6 1d 68 f2 46 ce".split(" ")
result = ""


for i in range(len(flag)):
    result += chr((int(randval_xor[i], 16) ^ int(randval[i], 16)) ^ int(flag[i], 16))

print(result)

~~~


