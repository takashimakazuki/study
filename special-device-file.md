# special device file

strings コマンド
~~~
/dev/xorshift64
GCC: (GNU) 4.9.4
/tmp/ccZkzjIB.o
_stack_addr
aarch64-elf.c
decode
~~~
xorshift64が出てくる（乱数生成のアルゴリズム）


* objdump コマンドが使えない
* readelf -h でアーキテクチャが確認できる
  * runme~ファイルはAArch64というアーキテクチャ用

下のコマンドでAArch64のobjdumpができる
~~~
aarch64-linux-gnu-objdump
~~~

または[このページ](https://onlinedisassembler.com/static/home/index.html)で逆アセンブルができる

#### 実行ファイルの大まかな動き

1. /dev/xorshift64に何か固定値を書き込む(確認方法がわからない)
2. decode関数でflagをデコード
(flagとrandvalとxorshift乱数のxorをしている。)
3. 文字を出力？



###### xorshift64(乱数を生成)
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

###### decode関数内
~~~python
# flag is written in #1800~
# randval is written in #1820~
flag = "fe 75 88 a9 5a aa 10 52 9c 6a 67 f4 82 be 21 56 59 0b 97 32 21 46 93 ae 40 0d 2e 1f 83 43 40".split(" ")
randval = "1d ab 1b 0f a7 d9 1a b0 61 7e b6 48 a4 56 cf 7e 49 05 fd 05 9c f9 54 45 fa 24 c6 1d 68 f2 46 ce".split(" ")
result = ""


for i in range(len(flag)):
    result += chr((int(randval_xor[i], 16) ^ int(randval[i], 16)) ^ int(flag[i], 16))

print(result)

~~~


