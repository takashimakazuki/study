# Special Instructions

* runme_f3abe874e1d795ffb6a3eed7898ddcbcd929b7be
* いろいろなアーキテクチャのクロスコンパイラ

objdumpする
```
objdump: アーキテクチャ UNKNOWN! 用に逆アセンブルできません
```

fileコマンドを叩くと...

```
runme_~: ELF 32-bit MSB executable, *unknown arch 0xdf* version 1 (SYSV), statically linked, not stripped
```
「アーキテクチャがわかりません。」だそうです。

stringsコマンドを叩くと...

```JavaScript
This program uses special instructions.
SETRSEED: (Opcode:0x16)
    RegA -> SEED
GETRAND: (Opcode:0x17)
    xorshift32(SEED) -> SEED
    SEED -> RegA
GCC: (GNU) 4.9.4
moxie-elf.c
```

* とりあえずアーキテクチャはmoxieということがわかる。

* xorshiftとは何ぞや？

乱数を作るアルゴリズム,ビットシフトとxorのみでシンプル
