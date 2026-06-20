# ブログアプリにおける設計上の工夫と実装内容の一部抜粋

## ① 記事とハッシュタグの関連付け設計とデータ取得最適化

### 該当箇所

バックエンドのデータモデル定義

```python
user = db.relationship(
    'User',
    backref=db.backref(
        'posts',
        lazy=True
    )
)

hashtags = db.relationship(
    'Hashtag',
    secondary=post_hashtags,
    lazy='selectin',
    backref=db.backref(
        'posts',
        lazy=True
    )
)
```

### 担当している機能

ブログ記事の投稿・一覧表示・タグ検索機能を支えるデータ取得処理です。

具体的には、

* 管理者と記事を紐づける機能
* 記事へ複数タグを付与する機能
* タグから関連記事を検索する機能
* 一覧画面で記事とタグを同時表示する機能

を実現しています。

例

```plaintext
記事A → hashtag1、hashtag2
記事B → hashtag3、hashtag4
記事C → hashtag5、hashtag6、hashtag7
```

このように、一つの記事に複数タグが付与できる構造です。

---

### 設計上の工夫

通常、一覧ページで記事を10件表示すると、

```plaintext
記事取得 1回
↓
記事ごとにタグ取得 10回
↓
合計11回アクセス
```

という不要なアクセスが発生する可能性があります。

そこで、

```python
lazy='selectin'
```

を採用しました。

この設定では、記事一覧取得後に必要なタグをまとめて取得します。

```plaintext
記事取得 1回
↓
タグまとめ取得 1回
↓
合計2回アクセス
```

その結果、記事数が増えても表示速度が低下しにくく、運用後の拡張性を考慮した設計を実現しました。

---

## ② トップページの記事表示制御と閲覧体験の最適化

### 該当箇所

トップページの一覧表示制御

```javascript
(function () {
    var INITIAL = 1;
    var STEP = 5;

    var cards =
        Array.from(
            document.querySelectorAll(
                '.page-wrapper > article.post-card'
            )
        );

    var total = cards.length;
    var shown = 0;

    function showCards(n) {
        var end =
            Math.min(
                shown + n,
                total
            );

        for (
            var i = shown;
            i < end;
            i++
        ) {
            cards[i].style.display = '';
        }

        shown = end;
        updateBtn();
    }

    function hideCards(n) {
        var end =
            Math.max(
                shown - n,
                INITIAL
            );

        for (
            var i = end;
            i < shown;
            i++
        ) {
            cards[i].style.display = 'none';
        }

        shown = end;
        updateBtn();
    }

    showCards(INITIAL);

})();
```

### 担当している機能

トップページの記事一覧表示です。

仕様

* 初回表示は最新記事1件のみ表示
* 「もっと見る」で5件ずつ追加表示
* 「表示を減らす」で戻せる

---

### 設計上の工夫

記事数が増えた場合、初回アクセス時に全件描画すると、

* 初期表示が遅くなる
* ユーザが情報過多になる
* モバイル端末で操作性が下がる

という課題があります。

そこで、表示制御をJavaScript側へ分離し、

```javascript
var INITIAL = 1;
var STEP = 5;
```

として段階的表示を採用しました。

結果として、

```plaintext
初回表示
↓
必要な分だけ追加表示
↓
快適な閲覧体験
```

を実現しています。

---

## この設計から得た学び

本アプリでは、機能実装だけでなく、利用者増加後の性能低下や操作性の悪化を事前に想定して設計を行いました。特に、データ取得方法や画面表示方式については、生成AIを技術的な相談相手として活用しながら複数案を比較し、運用後まで見据えた意思決定を行いました。

この経験を通じて、発生した問題へ対処するだけでなく、将来起こり得る課題を予測して設計へ反映する姿勢を身につけました。

