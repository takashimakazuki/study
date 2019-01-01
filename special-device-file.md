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

* runme_8a10b7425cea81a043db0fd352c82a370a2d3373は64ビットの実行ファイルである。

* /dev/xorshift64 というデバイスを使っている? (予想)

* objdump コマンドが使えない

* アーキテクチャはARM,aarch64というものらしい
 objdumpコマンドはaarch64というアーキテクチャに対応していないので逆アセンブルできなかった。

* readelf -h でもアーキテクチャが確認できる


## 実行ファイルのアセンブリを読みたい

aarch64の逆アセンブルを行う方法を探したところ、
[Online Disassembler](https://onlinedisassembler.com/static/home/index.html)で逆アセンブルができるようだった。

aarch64で使われている命令を知るために[ARM公式のドキュメント](https://developer.arm.com/products/architecture/cpu-architecture/a-profile/exploration-tools)を見た。

日本語のものも読みたかったので、旧バージョンだが下のページも見ながらアセンブリの命令を確認した。

http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0801aj/BABBDDIB.html

aarch64で使われている命令たち(命令セット)のことをA64というらしい。

#### Aarch64 基本的な命令とレジスタ

|命令|動作|
|:--:|:--:|
|MOV|レジスタ->レジスタへ内容をコピー|
|STR|レジスタ->メモリへ内容をコピー|
|LDR|メモリ-> レジスタへ内容をコピー|

レジスタは64ビット幅の汎用レジスタが31個ある。

X0~X30で指定した場合は64ビット幅でアクセスする

W0\~W30で指定した場合は、レジスタX0\~X30の下位32ビットにアクセスすることができる。



#### 実行ファイルの大まかな動き

1. メモリの0x1800 ~ 0x185fによくわからん数値が1バイトずつ入っている。

2. /dev/xorshift64に何か固定値を書き込む

(確認方法がわからない)

3. decode関数でflagをデコード

(flagとrandvalとxorshift乱数のXORをしている。)

4. 文字を出力？

5. システムコールがうまく呼べていないのでそのまま実行したとしてもうまく行かなさそう。

ファイルを実行しながら解析していたり、うまく行っていない部分の命令を書き換えて動作をみているwrite upがあった。まだやり方がよくわかっていないので出来るようになりたい。。。

http://ywkw1717.hatenablog.com/entry/2018/10/28/185936


#### main関数
~~~
.text:000016a4 <main>:
.text:000016a4 ff c3 00 d1        sub	sp, sp, #0x30
.text:000016a8 f3 53 00 a9        stp	x19, x20, [sp]
.text:000016ac f5 7b 01 a9        stp	x21, x30, [sp,#16]
.text:000016b0 80 48 8f d2        mov	x0, #0x7a44                	// #31300
.text:000016b4 e0 77 b9 f2        movk	x0, #0xcbbf, lsl #16       // レジスタX0にseed値0x139408dcbbf7a44を格納
.text:000016b8 a0 11 c8 f2        movk	x0, #0x408d, lsl #32       // 
.text:000016bc 20 27 e0 f2        movk	x0, #0x139, lsl #48
.text:000016c0 f4 c3 00 91        add	x20, sp, #0x30
.text:000016c4 80 8e 1f f8        str	x0, [x20,#-8]!              // スタックにseed値0x139408dcbbf7a44を積む
.text:000016c8 13 00 00 90        adrp	x19, 0x00001000
.text:000016cc 73 a2 1d 91        add	x19, x19, #0x768            // /dev/xorshift64/の文字の先頭アドレスを代入
.text:000016d0 e0 03 13 aa        mov	x0, x19
.text:000016d4 21 00 80 52        mov	w1, #0x1                   	// #1
.text:000016d8 02 00 80 52        mov	w2, #0x0                   	// #0
.text:000016dc 54 ff ff 97        bl	0x0000142c <__open>           // おそらく/dev/xorshift64/に何か書き込んでいる(初期値か?)
.text:000016e0 f5 03 00 2a        mov	w21, w0
.text:000016e4 e1 03 14 aa        mov	x1, x20
.text:000016e8 02 01 80 52        mov	w2, #0x8                   	// #8
.text:000016ec 4d ff ff 97        bl	0x00001420 <__write>
.text:000016f0 e0 03 15 2a        mov	w0, w21
.text:000016f4 51 ff ff 97        bl	0x00001438 <__close>          //　書き込み終了？
.text:000016f8 e0 03 13 aa        mov	x0, x19
.text:000016fc 01 00 80 52        mov	w1, #0x0                   	// #0
.text:00001700 e2 03 01 2a        mov	w2, w1
.text:00001704 4a ff ff 97        bl	0x0000142c <__open>
.text:00001708 f3 03 00 2a        mov	w19, w0
.text:0000170c 01 00 00 90        adrp	x1, 0x00001000              //　#0x1800をレジスタx0に書き込み(fragの先頭)
.text:00001710 21 00 20 91        add	x1, x1, #0x800
.text:00001714 e0 03 01 aa        mov	x0, x1
.text:00001718 21 80 00 91        add	x1, x1, #0x20               //　#0x1820をレジスタx1に書き込み(randvalの先頭)
.text:0000171c e2 03 13 2a        mov	w2, w19
.text:00001720 c1 ff ff 97        bl	0x00001624 <decode>          // decode関数へ
.text:00001724 e1 03 00 aa        mov	x1, x0
.text:00001728 20 00 80 52        mov	w0, #0x1                   	// #1
.text:0000172c 8b ff ff 97        bl	0x00001558 <puts>
.text:00001730 20 00 80 52        mov	w0, #0x1                   	// #1
.text:00001734 01 00 00 90        adrp	x1, 0x00001000
.text:00001738 21 e0 1d 91        add	x1, x1, #0x778
.text:0000173c 87 ff ff 97        bl	0x00001558 <puts>
.text:00001740 e0 03 13 2a        mov	w0, w19
.text:00001744 3d ff ff 97        bl	0x00001438 <__close>
.text:00001748 00 00 80 52        mov	w0, #0x0                   	// #0
.text:0000174c 6e ff ff 97        bl	0x00001504 <exit>
~~~


#### decode関数
~~~
.text:00001624 <decode>:
.text:00001624 ff 03 01 d1        sub	sp, sp, #0x40
.text:00001628 f3 53 00 a9        stp	x19, x20, [sp]
.text:0000162c f5 5b 01 a9        stp	x21, x22, [sp,#16]
.text:00001630 f7 63 02 a9        stp	x23, x24, [sp,#32]
.text:00001634 fe 1b 00 f9        str	x30, [sp,#48]
.text:00001638 f6 03 00 aa        mov	x22, x0                     // X22に0x1800を格納（数値列flagの先頭）
.text:0000163c f7 03 01 aa        mov	x23, x1                     // X23に0x1820を格納(数値列randvalの先頭)
.text:00001640 f8 03 02 2a        mov	w24, w2
.text:00001644 00 00 40 39        ldrb	w0, [x0]
.text:00001648 00 02 00 34        cbz	w0, 0x00001688
.text:0000164c f5 03 16 aa        mov	x21, x22
.text:00001650 03 00 80 d2        mov	x3, #0x0                   	// #0
.text:00001654 f4 03 03 2a        mov	w20, w3                     //:#1658~#1684でループしている
.text:00001658 f3 6a 63 38        ldrb	w19, [x23,x3]               // アドレス#0x1820+?の中身をロード
.text:0000165c e0 03 18 2a        mov	w0, w24
.text:00001660 e8 ff ff 97        bl	0x00001600 <get_random_value>
.text:00001664 00 00 13 ca        eor	x0, x0, x19                 // xorshift64の乱数(X0)とrandval(X19)をXOR
.text:00001668 a3 02 40 39        ldrb	w3, [x21]                   // アドレス#1800の中身をロード
.text:0000166c 03 00 03 4a        eor	w3, w0, w3                  // レジスタw0とflagをXOR
.text:00001670 a3 02 00 39        strb	w3, [x21]                   // レジスタw3の内容を保存
.text:00001674 94 06 00 11        add	w20, w20, #0x1
.text:00001678 83 7e 40 93        sxtw	x3, w20
.text:0000167c d5 02 03 8b        add	x21, x22, x3
.text:00001680 c0 6a 63 38        ldrb	w0, [x22,x3]
.text:00001684 a0 fe ff 35        cbnz	w0, 0x00001658              // 0x1658番地へジャンプ
.text:00001688 e0 03 16 aa        mov	x0, x22
.text:0000168c f3 53 40 a9        ldp	x19, x20, [sp]
.text:00001690 f5 5b 41 a9        ldp	x21, x22, [sp,#16]
.text:00001694 f7 63 42 a9        ldp	x23, x24, [sp,#32]
.text:00001698 fe 1b 40 f9        ldr	x30, [sp,#48]
.text:0000169c ff 03 01 91        add	sp, sp, #0x40
.text:000016a0 c0 03 5f d6        ret
~~~

#### get_rand_value関数

~~~
.text:00001600 <get_random_value>:
.text:00001600 ff 83 00 d1                      sub	sp, sp, #0x20
.text:00001604 fe 03 00 f9                      str	x30, [sp]
.text:00001608 e1 63 00 91                      add	x1, sp, #0x18
.text:0000160c 02 01 80 52                      mov	w2, #0x8                   	// #8
.text:00001610 81 ff ff 97                      bl	0x00001414 <__read>           // /dev/xorshift64から読み出し?
.text:00001614 e0 0f 40 f9                      ldr	x0, [sp,#24]                // x0に生成した乱数を書き込み
.text:00001618 fe 03 40 f9                      ldr	x30, [sp]
.text:0000161c ff 83 00 91                      add	sp, sp, #0x20
.text:00001620 c0 03 5f d6                      ret
~~~


#### プログラムの始まり・うまく行っていない関数
~~~
0001400 <_start>:
.text:	58000800 	                               ldr	x0, 1500 <_stack_addr>      //スタックポインタを0x1500番地に設定
.text: 9100001f 	                               mov	sp, x0
.text:	940000a7 	                               bl	16a4 <main>


.text:0000140c <__exit>:
.text:0000140c 00 00 20 d4                      brk	#0x0

.text:00001410 <$x>:
.text:00001410 c0 03 5f d6                      ret

.text:00001414 <__read>:
.text:00001414 c8 00 80 d2                      mov	x8, #0x6                   	// #6

.text:00001418 <$d>:
.text:00001418 00 00 5e d4                      hlt	#0xf000

.text:0000141c <$x>:
.text:0000141c c0 03 5f d6                      ret

.text:00001420 <__write>:
.text:00001420 a8 00 80 d2                      mov	x8, #0x5                   	// #5

.text:00001424 <$d>:
.text:00001424 00 00 5e d4                      hlt	#0xf000

.text:00001428 <$x>:
.text:00001428 c0 03 5f d6                      ret

.text:0000142c <__open>:
.text:0000142c 28 00 80 d2                      mov	x8, #0x1                   	// #1

.text:00001430 <$d>:
.text:00001430 00 00 5e d4                      hlt	#0xf000

.text:00001434 <$x>:
.text:00001434 c0 03 5f d6                      ret

.text:00001438 <__close>:
.text:00001438 48 00 80 d2                      mov	x8, #0x2                   	// #2

.text:0000143c <$d>:
.text:0000143c 00 00 5e d4                      hlt	#0xf000

.text:00001440 <$x>:
.text:00001440 c0 03 5f d6                      ret
.text:00001444 1f 20 03 d5                      nop
.text:00001448 1f 20 03 d5                      nop
.text:0000144c 1f 20 03 d5                      nop
.text:00001450 1f 20 03 d5                      nop
.text:00001454 1f 20 03 d5                      nop
.text:00001458 1f 20 03 d5                      nop
.text:0000145c 1f 20 03 d5                      nop
.text:00001460 1f 20 03 d5                      nop
.text:00001464 1f 20 03 d5                      nop
.text:00001468 1f 20 03 d5                      nop
.text:0000146c 1f 20 03 d5                      nop
.text:00001470 1f 20 03 d5                      nop
.text:00001474 1f 20 03 d5                      nop
.text:00001478 1f 20 03 d5                      nop
.text:0000147c 1f 20 03 d5                      nop
.text:00001480 1f 20 03 d5                      nop
.text:00001484 1f 20 03 d5                      nop
.text:00001488 1f 20 03 d5                      nop
.text:0000148c 1f 20 03 d5                      nop
.text:00001490 1f 20 03 d5                      nop
.text:00001494 1f 20 03 d5                      nop
.text:00001498 1f 20 03 d5                      nop
.text:0000149c 1f 20 03 d5                      nop
.text:000014a0 1f 20 03 d5                      nop
.text:000014a4 1f 20 03 d5                      nop
.text:000014a8 1f 20 03 d5                      nop
.text:000014ac 1f 20 03 d5                      nop
.text:000014b0 1f 20 03 d5                      nop
.text:000014b4 1f 20 03 d5                      nop
.text:000014b8 1f 20 03 d5                      nop
.text:000014bc 1f 20 03 d5                      nop
.text:000014c0 1f 20 03 d5                      nop
.text:000014c4 1f 20 03 d5                      nop
.text:000014c8 1f 20 03 d5                      nop
.text:000014cc 1f 20 03 d5                      nop
.text:000014d0 1f 20 03 d5                      nop
.text:000014d4 1f 20 03 d5                      nop
.text:000014d8 1f 20 03 d5                      nop
.text:000014dc 1f 20 03 d5                      nop
.text:000014e0 1f 20 03 d5                      nop
.text:000014e4 1f 20 03 d5                      nop
.text:000014e8 1f 20 03 d5                      nop
.text:000014ec 1f 20 03 d5                      nop
.text:000014f0 1f 20 03 d5                      nop
.text:000014f4 1f 20 03 d5                      nop
.text:000014f8 1f 20 03 d5                      nop
.text:000014fc 1f 20 03 d5                      nop

.text:00001500 <$d>:　　　　　　　　　　　　　　　　　// ここがスタックの１番下の部分だよ
.text:00001500 60 1c 00 00                      .inst	0x00001c60 ; undefined

.text:00001504 <exit>:
.text:00001504 ff 43 00 d1                      sub	sp, sp, #0x10
.text:00001508 fe 03 00 f9                      str	x30, [sp]
.text:0000150c c0 ff ff 97                      bl	0x0000140c <__exit>

.text:00001510 <write1>:
.text:00001510 ff 83 00 d1                      sub	sp, sp, #0x20
.text:00001514 fe 03 00 f9                      str	x30, [sp]
.text:00001518 e2 83 00 91                      add	x2, sp, #0x20
.text:0000151c 41 fc 1f 38                      strb	w1, [x2,#-1]!
.text:00001520 e1 03 02 aa                      mov	x1, x2
.text:00001524 22 00 80 52                      mov	w2, #0x1                   	// #1
.text:00001528 be ff ff 97                      bl	0x00001420 <__write>
.text:0000152c fe 03 40 f9                      ldr	x30, [sp]
.text:00001530 ff 83 00 91                      add	sp, sp, #0x20
.text:00001534 c0 03 5f d6                      ret

.text:00001538 <putchar>:
.text:00001538 ff 43 00 d1                      sub	sp, sp, #0x10
.text:0000153c f3 7b 00 a9                      stp	x19, x30, [sp]
.text:00001540 f3 03 01 2a                      mov	w19, w1
.text:00001544 f3 ff ff 97                      bl	0x00001510 <write1>
.text:00001548 e0 03 13 2a                      mov	w0, w19
.text:0000154c f3 7b 40 a9                      ldp	x19, x30, [sp]
.text:00001550 ff 43 00 91                      add	sp, sp, #0x10
.text:00001554 c0 03 5f d6                      ret

.text:00001558 <puts>:
.text:00001558 ff 83 00 d1                      sub	sp, sp, #0x20
.text:0000155c f3 53 00 a9                      stp	x19, x20, [sp]
.text:00001560 fe 0b 00 f9                      str	x30, [sp,#16]
.text:00001564 f4 03 00 2a                      mov	w20, w0
.text:00001568 f3 03 01 aa                      mov	x19, x1
.text:0000156c 21 00 40 39                      ldrb	w1, [x1]
.text:00001570 a1 00 00 34                      cbz	w1, 0x00001584
.text:00001574 e0 03 14 2a                      mov	w0, w20
.text:00001578 f0 ff ff 97                      bl	0x00001538 <putchar>
.text:0000157c 61 1e 40 38                      ldrb	w1, [x19,#1]!
.text:00001580 a1 ff ff 35                      cbnz	w1, 0x00001574
.text:00001584 00 00 80 52                      mov	w0, #0x0                   	// #0
.text:00001588 f3 53 40 a9                      ldp	x19, x20, [sp]
.text:0000158c fe 0b 40 f9                      ldr	x30, [sp,#16]
.text:00001590 ff 83 00 91                      add	sp, sp, #0x20
 .text:00001594 c0 03 5f d6                      ret

.text:00001598 <putxval>:
.text:00001598 ff c3 00 d1                      sub	sp, sp, #0x30
.text:0000159c fe 03 00 f9                      str	x30, [sp]
.text:000015a0 ff a3 00 39                      strb	wzr, [sp,#40]
.text:000015a4 81 00 00 b5                      cbnz	x1, 0x000015b4
.text:000015a8 5f 00 1f 6b                      cmp	w2, wzr
.text:000015ac e3 17 9f 1a                      cset	w3, eq
.text:000015b0 42 00 03 0b                      add	w2, w2, w3
.text:000015b4 e4 9f 00 91                      add	x4, sp, #0x27
.text:000015b8 06 00 00 90                      adrp	x6, 0x00001000
.text:000015bc c6 40 1d 91                      add	x6, x6, #0x750
.text:000015c0 06 00 00 14                      b	0x000015d8
.text:000015c4 25 0c 40 92                      and	x5, x1, #0xf
.text:000015c8 c5 68 65 38                      ldrb	w5, [x6,x5]
.text:000015cc 85 f4 1f 38                      strb	w5, [x4],#-1
.text:000015d0 21 fc 44 d3                      lsr	x1, x1, #4
.text:000015d4 42 00 03 4b                      sub	w2, w2, w3
.text:000015d8 5f 00 1f 6b                      cmp	w2, wzr
.text:000015dc e3 07 9f 1a                      cset	w3, ne
.text:000015e0 23 ff ff 35                      cbnz	w3, 0x000015c4
.text:000015e4 01 ff ff b5                      cbnz	x1, 0x000015c4
.text:000015e8 81 04 00 91                      add	x1, x4, #0x1
.text:000015ec db ff ff 97                      bl	0x00001558 <puts>
.text:000015f0 00 00 80 52                      mov	w0, #0x0                   	// #0
.text:000015f4 fe 03 40 f9                      ldr	x30, [sp]
.text:000015f8 ff c3 00 91                      add	sp, sp, #0x30
.text:000015fc c0 03 5f d6                      ret
~~~




## 実行ファイルで行っていることを同じようにpythonで書いてみる


#### xorshift64(乱数を生成)

~~~ python
seed = 0x139408dcbbf7a44
mask = 0xffffffffffffffff
randval_xorshift64 =[]

def xorshift64():
    global seed
    seed = seed ^ (seed << 13) & mask
    seed = seed ^ (seed >> 7) & mask
    seed = seed ^ (seed << 17) & mask
    return seed

for i in range(32):
    lastbyte = xorshift64() & 0xff
    randval_xorshoft64.append(hex(lastbyte))
    
~~~

/dev/xorshiftで行われているであろう操作を名前から推測して、xorshiftという方法で乱数を生成しているのではないかと仮定して考える。
なのでxorshiftを実装して32個の乱数を作っていく。この乱数は後でメモリの0x1800\~に入っている32個の数値と0x1820\~に入っている32個
の数値とXORされることによって答えの文字列が得られる。

pythonでは任意の桁数の演算ができてしまうので左シフトを繰り返していいると桁がだんだん大きくなっていく。しかし、実際の64ビットの
計算機のなかで計算が行われるときには64ビットを超えた値は捨てられてしまう。このことを再現するために上のpythonのコードではシフト
するたびに1が64個並んだ２進数とAND演算をすることによって下位64ビットだけを取り出している。

xorshiftのシフト回数はここでは(左13,右7,左17)のようになっている。この組み合わせによって良い乱数になるかどうかが決まってくるらしい。

シフト回数は実装によるが[wikipedia](https://ja.wikipedia.org/wiki/Xorshift)のxorshift64を真似した。



#### decode関数内

~~~python
flag = "fe 75 88 a9 5a aa 10 52 9c 6a 67 f4 82 be 21 56 59 0b 97 32 21 46 93 ae 40 0d 2e 1f 83 43 40".split(" ")
randval = "1d ab 1b 0f a7 d9 1a b0 61 7e b6 48 a4 56 cf 7e 49 05 fd 05 9c f9 54 45 fa 24 c6 1d 68 f2 46 ce".split(" ")
result = ""


for i in range(len(flag)):
    result += chr((int(randval_xorshoft64[i], 16) ^ int(randval[i], 16)) ^ int(flag[i], 16))

print(result)

~~~

* 0x1800~0x181fに入っている1バイトの数値32個 -> flag

* 0x1820~0x183fに入っている1バイトの数値32個 -> randval

　としている。この２つの数値と先にxorshiftで生成した乱数をXORして出来上がった新しい数値を文字に変換すると正解のflagが得られる。

### 参考にしたWrite Up
* https://garasubo.github.io/hexo/2018/10/31/seccon.html
* http://ywkw1717.hatenablog.com/entry/2018/10/28/185936


