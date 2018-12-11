
### 関数呼び出し部分
~~~
PUSH 2          (数値２　第２引数)
PUSH 1          (数値１　第１引数)
CALL xxxxxxxxxx (関数本体の先頭へ)
ADD ESP, 8      (不要な引数のサイズ分を破棄してスタックを修正）
~~~

### 関数本体
~~~
PUSH EBP        (EBPレジスタを新しいフレームポインタとして使うため、もとの値を退避)
MOV EBP, ESP    (スタックポインタの値　-> ベースポインタ)
SUB ESP, 8      (ローカル変数の場所を確保)
PUSH EDI        (EDIレジスタを使いたいのでもとの値を退避)
PUSH ESI        (ESIレジスタを使いたいのでもとの値を退避)
~関数の処理

スタックフレームの内容
DWORD PTR [EBP-10]　ESIレジスタのもとの値
DWORD PTR [EBP-0C]　EDIレジスタのもとの値
DWORD PTR [EBP-08]　ローカル変数２
DWORD PTR [EBP-04]　ローカル変数１
DWORD PTR [EBP]　　　EBPレジスタのもとの値（前のフレームレジスタ）
DWORD PTR [EBP+04]　関数のリターンアドレス
DWORD PTR [EBP+08]　数値１（第一引数）
DWORD PTR [EBP+0C]　数値２（第二引数）

POP ESI         (ESIレジスタのもとの値を復帰)
POP EDI         (EDIレジスタのもとの値を復帰)
MOV ESP, EBP    (スタックポインタの復帰)
POP EBP         (フレームポインタの復帰)
RET             (関数の呼び出し元に戻る)
~~~

### 補足
- RET 10

リターン後にADD ESP, 10を行うのと同じ
