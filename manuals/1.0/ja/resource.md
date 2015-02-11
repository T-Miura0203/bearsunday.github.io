---
layout: docs-ja
title: リソース
category: Manual
permalink: /manuals/1.0/ja/resource.html
---

BEAR.Sundayアプリケーションではアプリケーションはリソースの集合です。RESTfulアプリケーションを[REST](http://ja.wikipedia.org/wiki/REST)（Representational State Transfer）のスタイルで作成します。

## サービスとしてのオブジェクト

リソースクラスはHTTPのメソッドをPHPのメソッドにマップしてPHPのクラスをサービスとして扱います。

{% highlight php %}
<?php
class Index extends ResourceObject
{
    public function onGet($a, $b)
    {
        $this->code = 200; // 省略可
        $this['result'] = $a + $b;

        return $this;
    }
}
{% endhighlight %}

{% highlight php %}
<?php
class Todo extends ResourceObject
{
    public function onPut($id, $todo)
    {
        $this->code = 201; // Created

        return $this;
    }
}
{% endhighlight %}

PHPのリソースクラスはWebのURIと同じような`app://self/blog/posts/?id=3`, `page://self/index`などのURIを持ち、HTTPのメソッドに準じた`onGet`, `onPost`, `onPut`, `onPatch`, `onDelete`インターフェイスを持ちます。
リクエストはステートレスで行われ、リソースは`@Link`アノテーションのハイパーリンクで相互に接続することができます。

`koriym/todo`アプリケーションの場合、URIとクラスはこのように対応します。

| URI | Class |
|-----+-------|
| page://self/index | Koriym\Todo\Resource\Page\Index |
| app://self/blog/posts | Koriym\Todo\App\Blog\Posts |


## クライント

リソースのリクエストにはリソースクライアントを使用します。

{% highlight php %}
<?php

use BEAR\Sunday\Inject\ResourceInject;

class Index extends ResourceObject
{
    use ResourceInject;

    public function onGet($a, $b)
    {
        $this['post'] = $this
            ->resource
            ->get
            ->uri('app://self/blog/posts')
            ->withQuery(['id' => 1])
            ->eager
            ->request();
    }
}
{% endhighlight %}

このリクエストは`app://self/blog/posts`リソースに`?id=1`というクエリーでリクエストをすぐ`eager`に行います。

リソースのリクエストはlazyとeagerがあります。リクエストにlazyがついてないものがeagerリクエストです。

{% highlight php %}
<?php
$posts = $this->resource->get->uri('app://self/posts')->request(); //lazy
$posts = $this->resource->get->uri('app://self/posts')->eager->request(); // eager
{% endhighlight %}

lazy `request()`で帰って来るオブジェクトは実行可能なリクエストオブジェクトです。`$posts()`で実行することができます。
このリクエストをテンプレートやリソースに埋め込むと、その要素が使用されるときに評価されます。

## リンクリクエスト

クラインアントはハイパーリンクで接続されているリソースをリンクすることができます。

{% highlight php %}
<?php
$blog = $this
    ->resource
    ->get
    ->uri('app://self/User')
    ->withQuery(['id' => 1])
    ->linkSelf("blog")
    ->eager
    ->request()->body;
{% endhighlight %}

リンクは３種類あります。`$rel`をキーにして元のリソースの`body`リンク先のリソースが埋め込まれます。

 * `linkSelf($rel)` リンク先と入れ替わります。
 * `linkNew($rel)` リンク先のリソースがリンク元のリソースに追加されます
 * `linkCrawl($rel)` リンクをクロールして"リソースツリー"を作成します。



## アノテーション

### @Link
{% highlight php %}
<?php
    /**
     * @Link(rel="profile", href="/profile{?id}")
     */
    public function onGet($id)
{% endhighlight %}

リンクを`rel`と`href`で指定します。`hal`コンテキストではHALのリンクフォーマットとして扱われます。BEARのリソースリクエストのときには`linkSelf()`, `linkNew`, `linkCrawl`の時にリソースリンクとして使われます。

{% highlight php %}
<?php
/**
 * @Link(crawl="post-tree", rel="post", href="app://self/post?author_id={id}")
 */
public function onGet($id = null)
{% endhighlight %}

`linkCrawl`は`crawl`の付いたリンクを[クロール](https://github.com/koriym/BEAR.Resource#crawl)してリソースを集めます。

### @Embed
{% highlight php %}
<?php
    /**
     * @Embed(rel="website", src="/website{?id}")
     */
    public function onGet($id)
{% endhighlight %}

リソースの中に`src`でリンクしたリソースを埋め込みます。HTMLページの中に別のURLの画像リソースを埋め込む`<img src="...">`タグをイメージしてみてください。HALレンダラーでは`__embed`として扱われます。

### @ResourceRepository

{% highlight php %}
<?php
/**
 * @QueryRepository
 * @Etag
 */
class User extends ResourceObject
{% endhighlight %}

`@QueryRepository`とアノテートすると`get`リクエストは読み込み用のレポジトリ`QueryRepository`が使わ時間無制限のキャッシュとして機能します。
`get`以外のリクエストがあると該当する`QueryRepository`のリソースが更新されます。

`@QueryRepository`から読まれるリソースオブジェクトはHTTPに準じた`Last-Modified`と`ETag`ヘッダーが付加されます。

同一クラスの`onGet`以外のリクエストメソッドがリクエストされ引数を見てリソースが変更されたと判断すると`QueryRepository`の内容も更新されます。

{% highlight php %}
<?php
class Todo
{
    public function onGet($id)
    {
        // read
    }

    public function onPost($id, $name)
    {
        // update
    }
}
{% endhighlight %}

例えばこのクラスでは`->post(10, 'shopping')`というリクエストがあると`id=10`の`QueryRepository`の内容が更新されます。

### @Purge @Reload

もう１つの方法は`@Purge`アノテーションや、`@Reload`アノテーションで更新対象のURIを指定することです。

 
{% highlight php %}
<?php
/**
 * @Purge(uri="app://self/user/friend?user_id={id}")
 * @Reload(uri="app://self/user/profile?user_id={id}")
 */
public function onPut($id, $name, $age)
{% endhighlight %}

別のクラスのリソースや関連する複数のリソースの`QueryRepository`の内容を更新することができます。`@Purge`はリソースを消去するだけで、`@Reload`は生成まで行います。

### @Etag

クラスにアノテートされていてHTTPリクエストに`Etag`が含まれていれば、コンテンツを照合し変更がなければ`304 Not Modified`を返します。