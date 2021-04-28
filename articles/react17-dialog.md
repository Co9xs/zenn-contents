---
title: "【React17対応】コンポーネントの外側をクリックしたら閉じるダイアログを実装する"
emoji: "💬"
type: "tech"
topics: ["React", "JavaScript", "TypeScript", "フロントエンド", "tips"]
published: false
---

React17での破壊的変更によってReact16で動いていたダイアログのコードが動かなくなったため、調査と期待通り動作するように修正した経緯をまとめておきます。
使う技術は以下の通りです。
- react 17.0.2 (16.14.0←比較のために使用)
- reat-dom 17.0.2 (同上)
- typescript

# まずは仕様を確認する
まずは、今回作るDialogコンポーネントの仕様を確認していきます。

- toggleボタンがクリックされると、ダイアログの表示・非表示を切り替えられる
- 開いている状態でダイアログの範囲外(toggleボタンを含む)をクリックすると、ダイアログは閉じる
- ダイアログの範囲内をクリックしても、ダイアログは閉じない
- ダイアログの中にcloseボタンを設置する

以下に実際に動作するコードを載せておきます。
※React17で正常に動作します。（React16では正常に動作しません。理由は後述します）

# React16までの実装
冗長にはなりますが比較のため、
「React16で動くコードを実装→React17に移植→正常に動作するように変更を加えていく」
という手順で解説していきます。
「結論だけ知りたい！」という方はこちらから最終的なコードが見れます。

## ボタンを押したらDialogが表示されるようにする
とりあえずボタンを押したら、Dialogが表示されるようにします。
dialogを開くためのボタンはDialogコンポーネントの外側に置きたいので、Dialogが開いているかどうかというstateはコンポーネント本体には持たせず、外側（使用側）で管理することにします。そのため、propsでstateとstateを更新する関数、Dialogの内部に表示したいコンテンツを受け取ります。
また、Dialogが開いているときのスタイルを当てるためにカスタムdata属性のdata-dialog-openを付与しておきます。
```tsx: Dialog.tsx
import React, { Dispatch, SetStateAction } from 'react';
import './Dialog.scss'

type Props = {
  isOpen: boolean,
  setIsOpen: Dispatch<SetStateAction<boolean>>,
  children: React.ReactNode
}

const Dialog: React.VFC<Props> = (props) => {
  const { isOpen, setIsOpen, children } = props

  const handleCloseButtonClick = () => {
    setIsOpen(false)
  }

  return (
    <div className="dialog" data-dialog-open={isOpen}>
      <div className="dialog__content">
        <div>{ children }</div>
        <button onClick={handleCloseButtonClick}>close</button>
      </div>
    </div> 
  )
}
```
表示されるDialogにそれっぽいスタイルを当てていきます。

```scss: Dialog.scss
.dialog {
  width: 400px;
  padding: .25rem .5rem .5rem;
  background: #FFF;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translateY(-50%) translateX(-50%);
  display: none;
  &[data-dialog-open="true"] {
    display:block;
  }
}
 ```
次にDialogコンポーネントを使用する側の処理を書いていきます。
前述したとおり、Dialogが開いているかどうかというstateは使用側であるApp.tsxで管理します。
```tsx: App.tsx
import React, { useState } from 'react'
import { Dialog } from './Dialog'

const App: React.VFC<Props> = () => {
  const [isOpen, setIsOpen] = useState<boolean>(false)

  const handleOpenButtonClick = () => {
    setIsOpen(true)
  }

  return (
    <>
      <button onClick={handleOpenButtonClick}>toggle</button>
      <Dialog
        isOpen={isOpen}
        setIsOpen={setIsOpen}
      >
      ダイアログの中身です
      </Dialog>
    </>
  )
}
```
一旦ここまでで、openボタンをクリックしたらDialogが表示、Dialog内のcloseボタンをクリックしたら非表示されるようになりました🎉

## どこをクリックしても閉じる処理を追加する
ここからこのDialogの肝である、範囲外クリックで閉じる処理を実装していきます。
段階的に実装していくため、まずはどこをクリックしても閉じるという処理を書いていきましょう。
handleOutsideClickHandleという関数を用意し、今は単にDialogを非表示にする処理だけ書いておきます。
このイベントハンドラをブラウザ全体に登録することで、どこをクリックしても閉じる機能が実装できそうです。

登録する処理はuseEffect関数内で行います。
isOpenの値が変わるごとにuseEffectの処理が実行され、Dialogが開いている場合は、document.bodyに対してイベントリスナを登録、クリーンアップ関数でそのイベントリスナを削除していることに注意してください。

```tsx: Dialog.tsx
import './Dialog.scss'
import { Dispatch, SetStateAction, useEffect } from 'react';

type Props = {
  isOpen: boolean,
  setIsOpen: Dispatch<SetStateAction<boolean>>,
  children: React.ReactNode
}

const Dialog: React.VFC<Props> = (props) => {
  const { isOpen, setIsOpen, children } = props

  const handleOutSideClick = () => {
    setIsOpen(false)
  }

  const handleCloseButtonClick = () => {
    setIsOpen(false)
  }

  useEffect(() => {
    if (isOpen) {
      document.body.addEventListener('click', handleOutsideClick)
    }
    return () => {
      document.body.removeEventListener('click', handleOutsideClick)
    };
  },[isOpen]);

  return (
    <div className="dialog" data-dialog-open={isOpen}>
      <div className="dialog__content">
        <div>{ children }</div>
        <button onClick={handleCloseButtonClick}>close</button>
      </div>
    </div> 
  )
}
```
ここまででtoggleボタンで開いて、その状態でどこかをクリックするとDialogが閉じるようになりました。

一見うまく言ってるように見えますが、このコードではDialogが開いている状態で再度toggleボタンをクリックしてもDialogが閉じてくれないという不具合があります。
実際に以下のCodePenで挙動を確かめることができます。

```CodePen```

何が起こっているのでしょうか？

これを紐解くにはReactDOMにおけるイベントの委譲(event delegation)という仕組みを知る必要があります。
簡単に言うと、ReactDOM上で設定したイベントハンドラはそれに紐付いた実DOMに直接追加されているのではなく、documentオブジェクトに一括で登録されるという仕組みです。

例えば、ReactDOM上で以下のようにクリックイベントハンドラを登録したとします。
```tsx: App.tsx
<button onClick={handleOpenButtonClick}>toggle</button>
```
この場合ReactDOMは裏側で
`button.addEventListener('click', handleOpenButtonClick)`
のようにReactDOMに紐付いた実DOMに直接イベントハンドラを設定しているのではなく、
`document.addEventListener('click', handleOpenButtonClick)`
のようにすべてのイベントハンドラをdocumentに登録しているということです。

実際にブラウザで確認することもできます。
ChromeのDevToolのElementsタブを開いて, body要素を選択してみてください。
body要素を選択するとさらにタブが表示されていると思うので、そこのEvent Listenersというタブを見てみましょう。
clickに対するイベントリスナーがdocumentオブジェクトに設定されているのが分かると思います。
さらに表示しているReactコンポーネント内にピュアなinputタグを設置してみます。
すると、input要素に対するイベントリスナがdocumentに追加されたのが分かります。

このようにしてdocumentに設定されたイベントハンドラは、それが実際に呼び出されると、ReactDOMがどの要素から呼び出されたのかを判断するという仕組みになっているようです。

このevent delegationの仕組みを頭に入れて、「toggleボタンを押してDialogを開いた状態で、もう一度toggleボタンを押してもDialogを閉じることができない」という事象はどのようにして起こっているのか確認してみます。

1. toggleボタンを押す
2. toggleボタンのイベントは`document`オブジェクトに委譲されている
3. 2で委譲された`document`オブジェクトのイベントによってDialogを表示し`document.body`に`handleOutsideClick`のイベントリスナを設定
4. 3で追加された`document.body`のイベントは`document`よりも階層が深いのでこの時点ではバブリングによって`document.body`に設定したイベントは起こることはない
5. 再度openボタンを押すと`document > document.body`の親子関係より、バブリングによって`document.body`の処理→`document`の処理(toggleボタンから委譲された処理)の順番で発火
6. まず `document.body`のイベントによってDialogを閉じて、useEffectで`document.body`から`handleOutsideClick`のイベントリスナを削除
7. 最後に`document.body`からのバブリングによって発火した `document`のイベント(toggleボタンから委譲されたイベント)によってDialogを表示して、useEffectで`handleOutsideClick`のイベントリスナを設定

つまり、document.bodyにDialogを閉じるための処理を設定すると、再度toggleボタンを押した時に、document.bodyに登録されたDialogを閉じる処理→toggleボタンからdocumentに委譲されたDialogを開く処理の順番で発火するので結果として開いたままになってしまうということですね。

この問題を解決するためにはdocument.bodyではなく、documentオブジェクトに閉じるための処理を設定する必要があります。

先ほど確認したようにReactDOMに設定したイベントハンドラは内部でdocumentオブジェクトに委譲されます。
documentに対してあとから自分でリスナーを追加することで、ReactDOMのイベントハンドラよりもあとに任意の処理が実行されるようにできます。

詳しく見てみましょう。

```tsx: Dialog.tsx
import './Dialog.scss'
import React, { Dispatch, SetStateAction, useEffect } from 'react';

type Props = {
  isOpen: boolean,
  setIsOpen: Dispatch<SetStateAction<boolean>>,
  children: React.ReactNode
}

const Dialog: React.VFC<Props> = (props) => {
  const { isOpen, setIsOpen, children } = props

  const handleOutSideClick = () => {
    setIsOpen(false)
  }

  const handleCloseButtonClick = () => {
    setIsOpen(false)
  }

  useEffect(() => {
    if (isOpen) {
      document.addEventListener('click', handleOutsideClick)
    }
    return () => {
      document.removeEventListener('click', handleOutsideClick)
    };
  },[isOpen]);

  return (
    <div className="dialog" data-dialog-open={isOpen}>
      <div className="dialog__content">
        <div>{ children }</div>
        <button onClick={handleCloseButtonClick}>close</button>
      </div>
    </div> 
  )
}
```

閉じるためのイベントリスナの設定先をdocumentオブジェクトに変更しただけです。
結果としてこれはうまく動きます。
どんな挙動なのか詳しく見てみます。

1. toggleボタンを押す
2. toggleボタンのイベントは`document`オブジェクトに委譲されている
3. 2で委譲された`document`オブジェクトのイベントによってDialogを表示し`document`に`handleOutsideClick`のイベントリスナを設定
4. 3で追加されたイベントは`docuemnt`に対して、ReactDOMによって委譲されたイベントも`document`であり、同階層なのでこの時点でバブリングは起こらない
5. 再度toggleボタンを押すとdocumentに追加されている2つのイベントハンドラが、toggleボタンからdocumentに委譲されたイベントハンドラ→docuemntにあとから追加した`handleOutsideClick`の順で発火する


## 範囲外クリックで閉じるように変更
ここまでくればあとは簡単です。

useRefを使うことで、`dialogRef.current`でDialogのDOMにアクセスできます。
`Node.contains`というメソッドを使うことでクリックされた要素がDialogの範囲内かどうかを判定しています。

ついでに関数にuseCallbackを適用して、無駄な関数インスタンスの再生成を防ぎます。

```tsx: Dialog.tsx
import './Dialog.scss'
import React, { Dispatch, SetStateAction, useCallback, useEffect, useRef } from 'react';

type Props = {
  isOpen: boolean,
  setIsOpen: Dispatch<SetStateAction<boolean>>,
  children: React.ReactNode
}

const Dialog: React.VFC<Props> = (props) => {
  const { isOpen, setIsOpen, children } = props
  const dialogRef = useRef<HTMLDivElement | null>(null)

  const handleOutSideClick = useCallback((e: MouseEvent) => {
    if (dialogRef.current?.contains(e.target as Node)) return
    setIsOpen(false)
  }, [dialogRef])

  const handleCloseButtonClick = useCallback(() => {
    setIsOpen(false)
  }, [])

  useEffect(() => {
    if (isOpen) {
      document.addEventListener('click', handleOutsideClick)
    }
    return () => {
      document.removeEventListener('click', handleOutsideClick)
    };
  },[isOpen]);

  return (
    <div className="dialog" data-dialog-open={isOpen} ref={dialogRef}>
      <div className="dialog__content">
        <div>{ children }</div>
        <button onClick={handleCloseButtonClick}>close</button>
      </div>
    </div> 
  )
}
```
これで最初に定義した機能を全て満たすDialogが完成しました🎉
# React17で動くように変更を加えていく
お疲れだと思いますがここからが本題です。
まずはReact16で作ったコードをコピペしてReact17ではどんな挙動になるか見てみましょう。


# 参考記事
Qiita - 【React】コンポーネントの外がクリックされたら閉じるポップアップメニューを実装しよう
(特にこちらの記事はとても参考になりました。ありがとうございます🙇‍♂️）
https://qiita.com/G4RDSjp/items/58364a6655d4968a90d9

React公式ブログ
https://ja.reactjs.org/blog/2020/08/10/react-v17-rc.html

Zenn - React17におけるuseEffectの破壊的変更を理解する
https://zenn.dev/uhyo/articles/react17-useeffect