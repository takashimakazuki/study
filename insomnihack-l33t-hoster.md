# l33t-hoster [web]

## 問題の概要
* 問題文 You can host your l33t pictures [here](http://35.246.234.136/) 

hereのリンクで移動すると、画像を投稿が出来るサイトへ行くことになる。

## 問題サイトを調べてみる
ファイルを選択するボタンと送信するボタンがあるので適当なファイルを選んで送信してみる

## ソースコードを見たい
問題サイトのhtmlを見てみると、フォームのhtmlが書かれているだけで特に変わったことは書かれていない。
と、思いきや最後の行の<!-- /?source -->は手がかりになっていて、getパラメータのsourceがセットされているとソースコードが手に入ることを表しているらしい。

```html
<h1>Upload your pics!</h1>
<form method="POST" action="?" enctype="multipart/form-data">
    <input type="file" name="image">
    <input type="submit" name=upload>
</form>
<!-- /?source -->
```
urlに?source=aaaaaや、?sourceなどを加えてみると、たしかにソースコードが得られた。

## ソースコード（コメントを各所に追加）
```php
<?php
// ここのコードによってGETパラメータにsourceと入れるとファイルの内容が見られる。
if (isset($_GET["source"]))
    die(highlight_file(__FILE__));

session_start();

// 新しいセッションの場合にはユーザがアップロードする画像を保存するためのディレクトリを作ってくれる。
// ユーザ用のディレクトリの例：　images/4f6c8e238dfc6a82d0bd2642a6b6593c52b0cea4/
if (!isset($_SESSION["home"])) {
    $_SESSION["home"] = bin2hex(random_bytes(20));
}
$userdir = "images/{$_SESSION["home"]}/";
if (!file_exists($userdir)) {
    mkdir($userdir);
}

$disallowed_ext = array(
    "php",
    "php3",
    "php4",
    "php5",
    "php7",
    "pht",
    "phtm",
    "phtml",
    "phar",
    "phps",
);


if (isset($_POST["upload"])) {
    if ($_FILES['image']['error'] !== UPLOAD_ERR_OK) {
        die("yuuuge fail");
    }

    $tmp_name = $_FILES["image"]["tmp_name"];
    $name = $_FILES["image"]["name"];
    $parts = explode(".", $name);
    $ext = array_pop($parts);

    if (empty($parts[0])) {
        array_shift($parts);
    }

    if (count($parts) === 0) {
        die("lol filename is empty");
    }
    
    // アップロードしたファイルの拡張子が.phpとかだと弾かれる。
    if (in_array($ext, $disallowed_ext, TRUE)) {
        die("lol nice try, but im not stupid dude...");
    }

    // ファイル内に"<?"の文字があると弾かれる。 
    $image = file_get_contents($tmp_name);
    if (mb_strpos($image, "<?") !== FALSE) {
        die("why would you need php in a pic.....");
    }
    
    // 
    if (!exif_imagetype($tmp_name)) {
        die("not an image.");
    }

    $image_size = getimagesize($tmp_name);
    if ($image_size[0] !== 1337 || $image_size[1] !== 1337) {
        die("lol noob, your pic is not l33t enough");
    }

    $name = implode(".", $parts);
    move_uploaded_file($tmp_name, $userdir . $name . "." . $ext);
}

echo "<h3>Your <a href=$userdir>files</a>:</h3><ul>";
foreach(glob($userdir . "*") as $file) {
    echo "<li><a href='$file'>$file</a></li>";
}
echo "</ul>";

?>

<h1>Upload your pics!</h1>
<form method="POST" action="?" enctype="multipart/form-data">
    <input type="file" name="image">
    <input type="submit" name=upload>
</form>
<!-- /?source -->
```
if文が書かれており、条件に引っかかってしまうと`die()`でメッセージを残してプログラムが終了してしまう。
アップロードしたファイルがサーバに保存されるためにはif文を抜けたあとに書かれている

```php
move_uploaded_file($tmp_name, $userdir . $name . "." . $ext);
```

これが実行されなければいけない。

このコードにたどり着くためにファイルが満たすべき条件をまとめる
 * ファイルには名前が必要(`.abcd`などの隠しファイルはアップロードできない)
 * PHP拡張子はだめ（`.php`, `.php3`, `pht`）
 * ファイルの中に`<?`が含まれてはいけない
 * 画像のサイズが1337×1337である

### 目標
 1. おそらくサーバの中のどこかにflagが隠されているので、サーバでphpのスクリプトを実行して内部のディレクトリをのぞく。
 2. if文をすべて通過し、保存されるようなファイルを作る。
 3. php拡張子を用いずにphpを実行できるようにする。

3を達成するためには`.htaccess`ファイルをアップロードしてサーバの設定を書き変えると良い。ただし、2のことも同時に満たしている必要がある。

### ファイルを作成する
アップロードされたファイルが画像かどうか調べている関数は、`exif_imagetype()`であるので、これが何をしているのかphp公式マニュアルから調べてみる。

> exif_imagetype() reads the first bytes of an image and checks its signature.

画像ファイルは、ファイルの種類を特定するために最初の数バイトに決まった数字が入っている。この関数では最初の数バイトを調べてその画像ファイル
の種類やサイズなどの情報を読んでいる。つまり、はじめの数バイトを書き換えていれば画像ファイルとして偽装し、
この関数を騙すことが出来る。`exif_imagetype`が識別できる画像ファイルの種類は[php公式ドキュメント](http://php.net/manual/en/function.exif-imagetype.php#refsect1-function.exif-imagetype-constants)から確認できる。

ここでは.htaccessファイルを`.wbmp`ファイル（画像ファイル）にみせかける。理由は以下。

* wbmpファイルのヘッダーは `0000`から始まりその後の4バイトは画像サイズを表している。
* .htaccess内では`\x00`で始まるラインはコメントアウトされる。

つまり、この問題の場合、
```
0000 8a39 8a39 ...
```
というバイトから始めていれば、phpにはwbmpファイルだと判断され、サーバ内で設定ファイルとして働くときにはエラーを起こさずに動いてくれる！


次に`.htaccess`の中にどのような設定を書くかについて考える。

htaccessファイルとはapacheサーバで使われる設定ファイル。リダイレクト、アクセス制限、MIMEタイプの設定などの設定の変更、
追加がディレクトリ単位で出来る。
ここでは`MIMEタイプ`設定で`.wani`拡張子(何でも良い)をphpファイルだと認識させる。そのため、htaccessファイルにに次の一行を加える。

[htaccessについて](https://xn--web-oi9du9bc8tgu2a.com/how-to-use-php-in-html-files/)

```
AddType application/x-httpd-php .wani
```

またファイルの中に`<?`の文字は使えないのであらかじめファイルの内容をbase64でエンコードしてアップロードし、
サーバで実行されるときにデコードすることを考える。





## 参考にしたwriteup
https://corb3nik.github.io/blog/insomnihack-teaser-2019/l33t-hoster
