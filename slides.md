---
theme: default
title: 冗長なエラーログを削減し、スタックトレースを手に入れる
highlighter: shiki
author: Yu SERIZAWA (@upamune)
download: true
export:
  format: pdf
  withClicks: true
  withToc: false
drawings:
  persist: false
transition: slide-left
mdc: true
background: https://cdn.jsdelivr.net/gh/slidevjs/slidev-covers@main/static/Afyjbfs1rKI.webp
---

# 冗長なエラーログを削減し、
# スタックトレースを手に入れる

Goのエラーハンドリング 最新事情Lunch LT - 2024/04/23

@upamune / LayerX Inc.

---

自己紹介

# @upamune (うぱみゅん)

<br/>

<div grid="~ cols-2 gap-50" m="t-2">
  <div>
    <ul>
      <li>株式会社LayerX バクラク事業部
        <ul>
          <li>ex. 株式会社メルペイ</li>
        </ul>
      </li>
      <li>バクラク申請・経費精算 テックリード</li>
      <li>最近は趣味で始めたポッドキャストでキラキラシールを作ったんですが盛大に失敗しました</li>
      <li>🎙️ マヂカル.fm
        <ul>
          <li><a href="https://magical.fm/">https://magical.fm/</a></li>
        </ul>
      </li>
    </ul>
  </div>

  <div>
    <img src="/icon.png" class="rounded" />
  </div>
</div>



---
layout: center
---

# 話すこと

- 🏗️ バクラクではGo言語を利用してAPIサーバーを実装
- 🧑‍💻 エラーが発生したら、調査しやすいようにエラーログを出力
- 🔧 そこで発生した問題と，改善エラーハンドリングによって改善したケースのご紹介
- 🗒️ 今回の話は 👇 のブログ記事がベース

[Go言語のAPIサーバーの冗長なエラーログを40%削減した話](https://link.serizawa.me/err40) 🔗 [link.serizawa.me/err40](https://link.serizawa.me/err40)

---
layout: fact
---

# スタックトレースが欲しい!!

---
layout: center
---

# スタックトレースが欲しい!!
- ログから簡単にどこから発生したエラーなのかを追いたい
- スタックトレースが無いと，同じエラーメッセージを返している部分があったら，どっちが返したエラーなのか特定が難しい

<br />

`ErrInternal` が `doSomething1` と `doSomething2` のどちらから返ってきたのか特定が難しい

```go
func handler() error {
  if err := doSomething1(); err != nil {
    return ErrInternal
  }

  if err := doSomething2(); err != nil {
    return ErrInternal
  }

  return nil
}
```

---
layout: center
---


# 今までの実現方法

- [rs/zerolog](https://github.com/rs/zerolog)をログライブラリとして利用
- `Caller` オプションを利用することでログが出力される時に `file:line` も出力されるように
- エラーハンドリングしているところで，エラーログを出力するように

```go
func handler() error {
  if err := doSomething1(); err != nil {
    logger.Error().Caller().Err(err).Send() // {"level":"error","error":"err1","caller":"handler.go:10"}
    return ErrInternal
  }

  if err := doSomething2(); err != nil {
    logger.Error().Caller().Err(err).Send() // {"level":"error","error":"err2","caller":"handler.go:15"}
    return ErrInternal
  }

  return nil
}
```

⚠️ エラーはWrapせずに，ただ `return` するだけのコードベースだった

---
layout: fact
---


## どこから発生したエラーなのかは分かるようになった

---
layout: center
---

## `rs/zerolog` の `caller` で代用した時の問題点


- 🕵️ 複数のエラーログを見ないとスタックトレースを見ることができないのでエラーの原因を追いづらい

- 🤑 冗長なエラーログにより、エラーログの件数が増加しログ全体の検索コストと金銭的コストが増加する
  - 1件のエラーにつき `caller` フィールドしか違いがないログが4件出たりする

- 🤔 エラーログの件数を元にアラートを設定している場合、アラートのしきい値を決めるのが難しい

---
layout: center
---

## 📑ここまでの課題整理

<br/>

- エラー調査のためにスタックトレースが欲しい
- エラーログを複数出力することによって，どこから呼ばれているかは分かったが以下の点で課題アリ
  - エラー調査の煩雑さ
  - ログのコスト
  - ログのアラートの閾値調整の難しさ

---
layout: center
---

# 冗長なエラーログを削減し、
# スタックトレースを手に入れるぞ!!

---
layout: center
---

## 🗺️ 冗長なエラーログを削減し、
## スタックトレースを手に入れるまでの道程

<br />

1. エラーハンドリングライブラリの選定
1. Wrapされたエラーのスタックトレースを1箇所でログ出力できるように設定
1. Wrapされたエラーに対応できてない部分を見つけて(`errors.Is,As`)修正
1. エラーをWrapするように修正
1. 冗長なエラーログを削除
1. 今後Wrapされていないエラーが増えないように/冗長なエラーログを記述されないように対策

---
layout: center
---

### エラーハンドリングライブラリの選定

- バクラクでは[morikuni/failure](https://github.com/morikuni/failure)を採用
  - `lxerror` というパッケージを作って `failure` を薄くラップして使っている
  ```go
  func Wrap(err error, wrappers ...failure.Wrapper) error {
	  return failure.Custom(failure.Custom(err, failure.WithFormatter(), WithCallstack(1)), wrappers...)
  }
  ```
- Why `morikuni/failure` ?
  - エラーそのものに情報をもたせるのではなく，関数内で情報を付与してラップしていく
  - 拡張性と汎用性が高くて、様々なエラーユースケースに対応できる🫰
  - ex. エラーコードを埋め込んだり、多言語メッセージを埋め込んだり、**スタック情報**を付与したり...
- 👇 詳細は次の morikuni さんの発表で!!

<br />

> ![](https://i.gyazo.com/96189d06adb73ae8167567f5333de82c.png)

---
layout: center
---

###  Wrapされたエラーのスタックトレースを1箇所でログ出力できるように設定

- `morikuni/failure` でWrapしてスタックトレースを保持しているエラーログで出力できるように
- `rs/zerolog` では `Stack()` を呼ぶとエラーからスタックトレース取り出してログに付与できる
  - `Stack()` を付けたエラーログをミドルウェアの1箇所で出力するように
  - どうやってエラーからスタックトレースを取り出すかは `zerolog.ErrorStackMarshaler` によって決まるので実装してあげる

```go
func ZerologErrorMarshalStack(err error) interface{} {
	// ...
	stack, ok := lxerror.CallStackOf(err)
	if !ok {
		return nil
	}
	out := &bytes.Buffer{}
	for i, f := range stack.Frames() {
		p := f.Path()
		// ...
		fmt.Fprintf(out, "%s:%d [%s.%s]\n", p, f.Line(), f.Pkg(), f.Func())
	}
	return out.String()
}
```

---
layout: fact
---

## Wrapされているエラーはスタックトレースが出るようになった

---
layout: center
---

### Wrapされたエラーに対応できてない部分を見つけて修正

`errors.{Is,As}` を利用するようにして，Wrapされてても正しく動作するように修正する

````md magic-move

```go
// エラーの比較: エラーがWrapされていない時
if err == ErrNotFound {
  return nil
}
```

```go
// エラーの比較: エラーがWrapされている時
if errors.Is(err, ErrNotFound) {
  return nil
}
```

```go
// エラーから情報を取り出す: エラーがWrapされていない時
if perr, ok := err.(*fs.PathError); ok {
	fmt.Println(perr.Path)
}
```

```go
// エラーから情報を取り出す: エラーがWrapされている時
var perr *fs.PathError
if errors.As(err, &perr) {
	fmt.Println(perr.Path)
}
```
````

---

### Wrapされたエラーに対応できてない部分を見つける... 大変 🙄

[polyfloyd/go-errorlint](https://github.com/polyfloyd/go-errorlint)にお世話になりました🙏

<br/>

<div grid="~ cols-2 gap-2" m="t-2">

<div>
  <code>errors.Is</code> を使ってない部分の検出

  ```go
  // bad
  err == ErrFoo

  // bad
  switch err {
  case ErrFoo:
  }

  // good
  errors.Is(err, ErrFoo)
  ```
</div>

<div>
  <code>errors.As</code> を使ってない部分の検出

  ```go
  // bad
  myErr, ok := err.(*MyError)

  // bad
  switch err.(type) {
  case *MyError:
  }

  // good
  var me MyError
  ok := errors.As(err, &me)
  ```
  </div>
</div>

---
layout: fact
---

## エラーをWrapするように修正

---
layout: center
---

### エラーをWrapするように修正

<br>

- かなりマッチョな方法でIDEの置換でゴリゴリやっていった 💪
  - [tomarrell/wrapcheck](https://github.com/tomarrell/wrapcheck)で漏れている部分を検知しつつ...
- ex. 
  - `return nil, err` => `return nil, lxerror.Wrap(err)`
  - `return "", err` => `return "", lxerror.Wrap(err)`
  - `return nil, nil, err` => `return nil, nil, lxerror.Wrap(err)`

意外と綺麗に置換できたが... もっと良い方法があったかも? 🤔


---
layout: fact
---

## 冗長なエラーログを削除 🗑️

---
layout: center
---

## 冗長なエラーログを削除 🗑️

<br/>

- これもIDEの一括置換で対応
- `logger.Error().Caller().Err(err).Send()` を一括で削除
- 有用なログはメッセージを付けていたり，他のフィールドが付与されているので，完全に一致するコードだけを削除対応

---
layout: center
---

## エラーのWrap漏れ，必要ないエラーログの検知 🕵️

<br/>

- エラーのWrap漏れ検知
  - [tomarrell/wrapcheck](https://github.com/tomarrell/wrapcheck)を[golangci-lint](https://golangci-lint.run/)で利用
- 必要なくなったエラーログの検知
  - [semgrep](https://semgrep.dev/)でルールを書いてCIで検知

  ```yaml
  rules:
  - id: meaningless-error-log
    patterns:
      - pattern-either:
          - pattern: $X.Error().Err($ERR).Send()
    message: "冗長なエラーログを出力しないでください"
    languages: [ go ]
    severity: ERROR
  ```
  - 👇 以下のような，フィールドが設定されていたりメッセージが設定されているログには引っかからない
    - `logger.Error().Err(err).Str("key", "value").Msg("blah blah blah")`


---
layout: center
---

# まとめ 🎯

<br/>

- `rs/zerolog` がエラーからスタックトレースをエラーから正しく抽出できるように実装
- [polyfloyd/go-errorlint](https://github.com/polyfloyd/go-errorlint)を利用してエラーがWrapされても問題ないように対応
- エラーを `morikuni/failure` でWrapするようにしてスタックトレースを保持
- エラーがWrapされていなかったり，不要になったログが出力されないように [tomarrell/wrapcheck](https://github.com/tomarrell/wrapcheck)，[semgrep](https://semgrep.dev/)で検知

---
layout: center
---

# おわり 🔚

ご清聴ありがとうございました


---
layout: image
image: /pr.svg
---

