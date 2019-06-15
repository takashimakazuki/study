# bashの設定 .bash_profile　.bashrc .profile

## ログインシェルとは

> bash をプログラムとして起動するとき、オプション - または --login が付加された際にスタートするシェル。
なんじゃそりゃ、ってかんじ。でも実際にこれを実行するとログインシェルとしてbashが起動しているため、logoutが出来る。
```bash
~$ bash --login
~$ logout
~$
```

ログインシェルでない場合(コンピュータを起動したあとはじめに操作できるシェルはログインシェルでない)にはlogoutができない。

```bash
~$ logout
bash: logout: ログインシェルではありません: `exit' を使用してください
```

## ログインシェルを起動したときに読み込まれる順序
1. /etc/profile
1. ~/.bash_profile
1. ~/.bash_login
1. ~/.profile

まず1が実行され、その後に2,3,4の順番でファイルが存在するかどうかを確認する。この順番で一番はじめに見つかったものを
実行して後の２つのファイルは読み込まれない。

[このサイト](https://qiita.com/tatesuke/items/88629e9550b813109964)の図が非常にわかりやすい。
