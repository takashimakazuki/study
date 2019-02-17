# l33t-hoster [web]

### 問題の概要
* 問題文 You can host your l33t pictures [here](http://35.246.234.136/) 

hereのリンクで移動すると、画像を投稿してね！というサイトへ行くことになる。

### 問題サイトを調べてみる
ファイルを選択するボタンと送信するボタンがあるので適当なファイルを選んで送信してみる

### ソースコードを見る
問題サイトのhtmlを見てみると、フォームのhtmlが書かれているだけで特に変わったことは書かれていない。
と、思いきや最後の行の<!-- /?source -->は手がかりになっていて、getパラメータのsourceに値を入れるとソースコードが手に入ることを表しているらしい。

```html
<h1>Upload your pics!</h1>
<form method="POST" action="?" enctype="multipart/form-data">
    <input type="file" name="image">
    <input type="submit" name=upload>
</form>
<!-- /?source -->
```
urlに?source=aaaaaや、?sourceなどを加えてみると、たしかにソースコードが得られた。

### ソースコード php
```php
$disallowed_ext = array( "php", "php3", "php4", "php5", "php7", "pht", "phtm", "phtml", "phar", "phps");


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

    if (in_array($ext, $disallowed_ext, TRUE)) {
        die("lol nice try, but im not stupid dude...");
    }

    $image = file_get_contents($tmp_name);
    if (mb_strpos($image, "<?") !== FALSE) {
        die("why would you need php in a pic.....");
    }

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
```

### phpの関数
* 
*

