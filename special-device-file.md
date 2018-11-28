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

このコマンドでAArch64のobjdumpができる
aarch64-linux-gnu-objdump


