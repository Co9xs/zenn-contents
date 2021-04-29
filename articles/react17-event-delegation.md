---
title: "React17のevent delegationの破壊的変更を理解する"
emoji: "👍"
type: "tech"
topics: ["React", "JavaScript", "TypeScript", "フロントエンド", "tips"]
published: false
---

React17が出てからしばらく経ちましたが、React17の破壊的変更で既存コードが動かないということがあり、調査と修正を行いました。
そこで調査の過程で得られたことを、自分自身の理解の整理も兼ねてまとめておきます。

本記事ではReact17の破壊的変更のうち、event delegationにおけるイベントの委譲先の変更について取り上げます。
この変更については[公式ブログ](https://ja.reactjs.org/blog/2020/08/10/react-v17-rc.html)での説明がとても分かりやすかったですが、実際にどんなユースケースで問題になるのかという点を詳しく解説できたらと思います。
# event delegation（イベントの委譲）とは？
いきなり聞き慣れない言葉なので、まずはevent delegationとは何かという部分から確認していきましょう。
通常Reactでイベントハンドラを登録する場合、以下のようにインラインで記述します。
```tsx
<button id="myButton" onClick={handleClick}>ボタン</button>
```
このコードは実際のDOMでは以下とほぼ同等です。
```js
document.getElementById('myButton').addEventListenrer('click', handleClick)
```
しかし、実際Reactは裏側でこのような処理を行っているわけではありません。
ReactはDOMノードにイベントハンドラを直接追加することはせず、イベントタイプごとにハンドラを1つだけ`document`オブジェクトに付与します。
この挙動は実際にブラウザで確認することができるので試してみましょう。

先ほど例で挙げたような、ボタンにイベントハンドラを設定したReactコンポーネントをブラウザの検証ツールで見てみました。
すると`document`オブジェクトに対してclickイベントのハンドラが1つだけ登録されていることが分かります。
逆に`button`からonClickイベントを削除すれば`document`からも消えます。
![React16系のイベントデリゲーション](https://storage.googleapis.com/zenn-user-upload/tciup6b0jpyc60hzwzymr0v6l47l)
*documentのclickに対してイベントハンドラが設定されている*

このように、ほぼすべてのイベントハンドラを内部的に`document`オブジェクトに対して設定するというのがReact16までのevent delegationという仕組みです。
# React17でevent delegationはどう変わったのか？
では、React17ではevent delegationの挙動がどのように変わったのでしょうか？
一言でいうと、イベントハンドラの設定先が`document`から`Reactアプリがマウントされるルート要素`に変わりました。
つまり、React17では裏側で、`document.addEventListener()` ではなく `rootNode.addEventListener()` という呼び出しを行うようになります。
この変更に関しては公式ブログの画像が非常にわかりやすいので引用します。

![イベントの委譲のイメージ図](https://ja.reactjs.org/static/bb4b10114882a50090b8ff61b3c4d0fd/31868/react_17_delegation.png)
*公式ブログから引用*

この変更も同じくブラウザの検証ツールで確認することができます。
reactを17系にして、先ほどと同じボタンのコンポーネントをブラウザで確認してみましょう。
すると`document`オブジェクトではなく、`div#app`に対してイベントハンドラが設定されていることが分かりますね。
![React17系のイベントデリゲーション](https://storage.googleapis.com/zenn-user-upload/s00bizro0cdhk60nvrtln1yqsfso)
*div#appのclickに対してイベントハンドラが設定されている*

ここまでのポイントは以下の2点です。
- React16までは`document`に対してイベントが委譲される。
- React17からは`rootNode`にイベントが委譲される。

それでは、この2点を踏まえてどんなケースでこの変更が問題になるのかを実際に見ていきます。
# 問題となるケース
さて、ここまではevent delegationの概要とReact17でどんな変更があったかを見てきました。
ここからはこの破壊的変更がどんなケースで問題になったのかを、実例を交えて紹介していきます。

## ポップアップメニューを考える
event delegationの破壊的変更がどんな影響を与えるのかを知るために、以下のような仕様をもつPopupMenuコンポーネントを考えます。
- openボタンをクリックすると、メニューが開く
- 開いている状態でメニューの範囲外(openボタン含む)をクリックすると、メニューが閉じる
- メニュー内のcloseボタンをクリックすると、メニューが閉じる

実際の動きと一緒に確認してみてください。非常によくあるUIですね。

--- gif ---

いきなりですが、このPopupMenuコンポーネントをReact16で動作するように実装したコードの全体像を載せておきます。

```tsx: PopupMenu.tsx
import React, { useCallback, useRef, useState } from 'react';
import './PopupMenu.scss'

type Props = {}

export const PopupMenu: React.VFC<Props> = (props) => {
  const [isOpen, setIsOpen] = useState<boolean>(false)
  const popupRef = useRef<HTMLDivElement | null>(null);

  const outsideClickHandler = useCallback((e: MouseEvent) => {
    if (popupRef.current?.contains(e.target as Node)) return
    setIsOpen(false)
    removeOutsideClickHandler()
  }, [popupRef])

  const openButtonClickHandler = useCallback(() => {
    setIsOpen(true)
    addOutsideClickHandler()
  }, [])

  const closeButtonClickHandler = useCallback(() => {
    setIsOpen(false)
    removeOutsideClickHandler()
  }, [])

  const addOutsideClickHandler = useCallback(() => {
    document.addEventListener('click', outsideClickHandler)
  }, [])

  const removeOutsideClickHandler = useCallback(() => {
    document.removeEventListener('click', outsideClickHandler)
  }, [])

  return (
    <>
      <button onClick={openButtonClickHandler}>open</button>
      <div className='popup' data-popup-active={isOpen} ref={popupRef}>
        <button onClick={closeButtonClickHandler}>close</button>
      </div>
    </>
  )
};
```
長いですが、少しずつ分割して確認していきましょう。

まず`useState`でメニューの表示・非表示の状態を管理しています。
本記事ではスタイルは省略しますが、表示・非表示のスタイルを当てるため、`data-popup-active`というカスタムdata属性を付与しています。
```tsx: PopupMenu.tsx
const [isOpen, setIsOpen] = useState<boolean>(false)
...
return (
  <>
    <button onClick={openButtonClickHandler}>open</button>
    <div className='popup' data-popup-active={isOpen} ref={popupRef}>
      <button onClick={closeButtonClickHandler}>close</button>
    </div>
  </>
)
```

次に`outsideClickHandler`という関数を見てみましょう。

ここでは`useRef`を使って開閉されるメニューのDOMへの参照を保存しています。
`popupRef.current`で取得できる`Element`のスーパークラスである`Node`には `Node.contains()`というメソッドがあります。
これは引数に指定したノードがこのノードの子孫ノード（自分自身を含む）であるかどうかを判定する関数なので、これを利用して範囲外クリックかどうかを判定しています。

つまり、メニュー範囲外がクリックされたらメニューを閉じて自分自身を`document`のイベントリスナから削除するという処理を行っています。

```tsx: PopupMenu.tsx
...
const popupRef = useRef<HTMLDivElement | null>(null);
const outsideClickHandler = useCallback((e: MouseEvent) => {
  if (popupRef.current?.contains(e.target as Node)) return
  setIsOpen(false)
  removeOutsideClickHandler()
}, [popupRef])
...
```

最後に残りの部分です。

- openボタンがクリックされたらメニューを表示して、`document`のclickイベントに対して`outsideClickHandler`を追加。
- 逆にcloseボタンがクリックされたらメニューを非表示にして、`document`のclickイベントから`outsideClickHandler`を削除。
という一連の処理を行うための関数です。

```tsx: PopupMenu.tsx
...
const openButtonClickHandler = useCallback(() => {
  setIsOpen(true)
  addOutsideClickHandler()
}, [])

const closeButtonClickHandler = useCallback(() => {
  setIsOpen(false)
  removeOutsideClickHandler()
}, [])

const addOutsideClickHandler = useCallback(() => {
  document.addEventListener('click', outsideClickHandler)
}, [])

const removeOutsideClickHandler = useCallback(() => {
  document.removeEventListener('click', outsideClickHandler)
}, [])
...
```

前述したとおり、このコードはReact16では問題なく動作します。

しかし、ここではなぜ「`document`に対して`outsideClickHandler`を追加している」のでしょうか？
単に範囲外クリックを検知するためのイベントリスナを登録するだけなら、`document.body`などの要素でも問題ないように思えます。

その答えは、React16までのevent delegationでは`document`に対してイベントが委譲されるという点と関係しています。
仮に`document.body`などの`document`よりも下の階層にこのイベントリスナを登録した場合どんな挙動をするか確認してみます。

--- gif or CodePen ---

openボタンでメニューを開き、closeボタンか範囲外クリックでメニューを閉じるということは問題なくできています。
しかし、メニューが開いている状態で、再度openボタンをクリックしてみると、閉じることができません。

何が起きているのかを以下にまとめてみました。

### `document.body`に範囲外クリックのイベントリスナを設定した場合の処理の流れ（悪い例）
1. 最初にopenボタンを押す
2. openボタンから`document`に委譲されたイベントハンドラによって、メニューを表示し`document.body`に`outsideClickHandler`を追加
3. 再度openボタンを押すと`document` > `document.body`の階層関係であるので、バブリングによって`document.body`の処理→`document`の処理の順番で発火
4. `document.body`のイベントによってメニューを閉じて、`document.body`から`outsideClickHandler`を削除
5. openボタンから`document`に委譲されたイベントによってメニューを表示し、`document.body`に`outsideClickHandler`を追加

結果としてメニューが開いた状態で、openボタンを押しても閉じれないということになります。
対して`document`に対してoutsideClickHandlerを設定した場合を見てみます。

:::message
`outsideClickHandler`に`e.stopPropagation()`を加えることでも3のバブリングを防げるのでうまく動きますが、今回はevent delegationの挙動を理解する目的のため、`document`に設定する方法で対処していきます。
:::
### `document`に範囲外クリックのイベントリスナを設定した場合の処理の流れ（良い例）
1. 最初にopenボタンを押す
2. openボタンから`document`に委譲されたイベントハンドラによって、メニューを表示し`document`に`outsideClickHandler`を追加
3. 再度openボタンを押すとどちらも`document`に対するイベントハンドラが設定されているので、追加された順番で発火
（openボタンから`document`に委譲された処理→`document`にあとから追加した`outsideClickHandler`の処理の順番で発火)
4. openボタンから`document`に委譲されたイベントハンドラによって、メニューを表示し`document`に`outsideClickHandler`を追加
5. `document`にあとから追加した`outsideClickHandler`の処理によって、メニューを閉じて`document`から`outsideClickHandler`を削除

このようにevent delegationによって委譲される先と同じ対象にあとからイベントリスナを追加してあげることで、ReactDOMの処理→あとから追加した処理という順番で処理してくれるようになるので期待通りに動作するというわけですね。
## このポップアップメニューをReact17で動作確認する
ここまで確認してきた以下のコードはそのままに、react, react-domのバージョンを17系にあげてみます。

:::details PopupMenuのコード(再掲)
```tsx: PopupMenu.tsx
import React, { useCallback, useRef, useState } from 'react';
import './PopupMenu.scss'

type Props = {}

export const PopupMenu: React.VFC<Props> = (props) => {
  const [isOpen, setIsOpen] = useState<boolean>(false)
  const popupRef = useRef<HTMLDivElement | null>(null);

  const outsideClickHandler = useCallback((e: MouseEvent) => {
    if (popupRef.current?.contains(e.target as Node)) return
    setIsOpen(false)
    removeOutsideClickHandler()
  }, [popupRef])

  const openButtonClickHandler = useCallback(() => {
    setIsOpen(true)
    addOutsideClickHandler()
  }, [])

  const closeButtonClickHandler = useCallback(() => {
    setIsOpen(false)
    removeOutsideClickHandler()
  }, [])

  const addOutsideClickHandler = useCallback(() => {
    document.addEventListener('click', outsideClickHandler)
  }, [])

  const removeOutsideClickHandler = useCallback(() => {
    document.removeEventListener('click', outsideClickHandler)
  }, [])

  return (
    <>
      <button onClick={openButtonClickHandler}>open</button>
      <div className='popup' data-popup-active={isOpen} ref={popupRef}>
        <button onClick={closeButtonClickHandler}>close</button>
      </div>
    </>
  )
};
```
:::

これを動作確認すると、そもそもopenボタンを押しても開くことができません。

-- gif or CodePen ---

React17ではイベントの委譲先がrootNodeになったということを思い出して、どんな処理が起きているのかを考えてみましょう。

1. openボタンを押す
2. openボタンから`div#app`に委譲されたイベントハンドラによって、メニューを表示し`document`に`outsideClickHandler`を追加
3. 2の終了時で`document` > `div#app`の階層関係なのでバブリングによって`document`の処理も即座に発火する
4. `document`の処理でメニューを消して、`document`から`outsideClickHandler`を削除

つまり、openボタンを押してメニューを表示した瞬間に、`document`に設定した`outsideClickHandler`でメニューが非表示にされてしまうということです。

これを動くように修正していきます。
## コードを修正する
それではReact17でも期待通りに動くように修正していきます。
修正といっても、`outsideClickHandler`の登録先をReact17のイベントの委譲先と同じ,`div#app`にしてあげるだけです。

```diff tsx: PopupMenu.tsx
...
  const addOutsideClickHandler = useCallback(() => {
-   document.addEventListener('click', outsideClickHandler)
+   document.getElementById('app').addEventListener('click', outsideClickHandler)
  }, [])

  const removeOutsideClickHandler = useCallback(() => {
-   document.removeEventListener('click', outsideClickHandler)
+   document.getElementById('app').removeEventListener('click', outsideClickHandler)
  }, [])
...
```

この修正によって処理は以下のように変わります。

1. openボタンを押す
2. openボタンから`div#app`に委譲されたイベントハンドラによって、メニューを表示し`div#app`に`outsideClickHandler`を追加
3. 2の終了時で`outsideClickHandler`は`div#app`というReactのイベント委譲先と同じ階層に設定されただけなので、バブリングによって`outsideClickHandler`の処理が即座に発火することはない

これで期待通りに動作するようになりました🎉
お疲れさまでした!
## 補足
今回はevent delegationの挙動を理解する目的のため、イベントの委譲先と同じオブジェクトに自分で追加したいイベントハンドラを追加するという方針で修正しましたが、他にも色々やり方があると思います。
例えば、
- メニューが開いている時に、画面いっぱいの背景要素を表示して、それに対してクリックイベントを追加する
- useEffectを使ってイベントリスナの追加・削除を管理する

などです。
むしろこっちのほうが簡単かもしれませんが、今回は割愛します。

# まとめ
本記事ではReact17におけるevent delegationの破壊的変更をまとめました。
今回紹介したPopupMenuのようにdocumentに対してイベントリスナを自分で設定するようなケースは少ないかもしれませんが、少しでも理解の助けとなれば幸いです。
# 参考記事
https://ja.reactjs.org/blog/2020/08/10/react-v17-rc.html
https://qiita.com/G4RDSjp/items/58364a6655d4968a90d9