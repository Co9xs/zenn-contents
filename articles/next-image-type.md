---
title: "next/image の width, height 指定を型レベルで強制する"
emoji: "✋"
type: "tech"
topics: ["React", "Next.js", "TypeScript", "フロントエンド", "tips"]
published: true
published_at: 2022-06-09 19:30
---

next/image をラップしたコンポーネントを作る機会があり、せっかくなら width, height の指定を型レベルで強制したいと感じたのでメモ。

# next/image のおさらい

next/image は Next.js 10 から正式に使えるようになったコンポーネントで、画像のレスポンシブ対応（サイズやレイアウト）、遅延ロード、webp 変換など様々な画像表示に関する最適化をよしなにやってくれます。

https://nextjs.org/docs/api-reference/next/image

その中でも今回フォーカスを当てるのは、画像の縦横比やレイアウトを指定する際に使う `layout` プロパティです。

# layout, width, height プロパティについて

next/image で画像のサイズに関して指定する `layout` プロパティは以下のようになっており、これらを指定することによって、viewport が変わった時の挙動などを制御することができます。

https://nextjs.org/docs/api-reference/next/image#layout

以下公式ドキュメントより抜粋 & 簡単に要約。

| layout props | 挙動                                                                                                                                                            |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| intrinsic    | デフォルト値。width が viewport 幅よりも小さい場合は viewport 幅に合わせて小さくなりますが、画像の幅が viewport 幅よりも大きい場合は width の値に設定されます。 |
| fixed        | viewport の幅によらず、設定された width, height の画像を表示します。                                                                                            |
| responsive   | viewport 幅に依存して画像幅が変化します。layout='intrinsic'の場合と異なり、画像の幅が viewport 幅よりも大きい場合は viewport 幅に合わせて画像幅が変化します。   |
| fill         | 親の DOM 要素の height, width に合わせて画像の幅と高さが設定されます。                                                                                          |

これらの `layout` プロパティは `layout="fill"` を指定したとき以外は、`width`, `height` も合わせて指定する必要があります。

しかしデフォルトでは `layout="fill"` 以外を指定して `width`, `height` の指定をしなかったとしても型エラーにはならず、ランタイムエラーになります。

```jsx
<Image src={"./path/to/image.png"} layout="responsive" />
```

![layout="fill" 以外を指定して width, height の指定を忘れても runtime error になる](/images/next-image-type/next-image-runtime-error.png)
_layout="fill" 以外を指定して width, height の指定を忘れても runtime error になる_

この width, height 指定のエラーを型レベルで気付けるようにしたいというのが今回の趣旨です。

# next/image の width, height 指定を型レベルで強制する

いきなりですが、最終的なコードはこんな感じです。

```tsx
import NextImage, { ImageProps } from "next/image";

// NOTE: 指定したプロパティだけを必須にする utility type 的な型
type RequiredOnly<T, U extends keyof T> = T & Required<Pick<T, U>>;

type WithLayout<T extends ImageProps["layout"]> = ImageProps & {
  layout?: T;
};

// NOTE: layout="fill"以外の時は props の width と height を必須にする
type Props<T extends ImageProps["layout"]> = T extends "fill"
  ? WithLayout<T>
  : RequiredOnly<WithLayout<T>, "width" | "height">;

export const Image = <Layout extends ImageProps["layout"]>(
  props: Props<Layout>
) => {
  const { layout, ...restProps } = props;
  return (
    <div>
      <NextImage {...restProps} layout={layout} />
    </div>
  );
};
```

順番に何をやってるかみていきましょう。

## 特定のプロパティだけを必須に
まずは、今回の要件では `width`, `height` だけを必須にしたいので特定のプロパティだけを必須にするような utility type を作ります。

```tsx
type RequiredOnly<T, U extends keyof T> = T & Required<Pick<T, U>>;
```

## ジェネリクスを利用して layout プロパティの型を制限する
次に、ジェネリクスを利用して、`layout` プロパティの型をより厳しく制限します。

これによって `layout` プロパティの型は `"fill" | "fixed" | "intrinsic" | "responsive"` からジェネリクスで1つ指定されたものだけを受け付けるようになります。
```tsx
type WithLayout<T extends ImageProps["layout"]> = ImageProps & {
  layout?: T;
};
```

## layout プロパティの値によって型を出し分ける
最後に `layout` プロパティが `fill` であれば通常の型、そうでなければ、`width`, `height` を必須にした型を出し分けてあげれば OK です。
```tsx
type Props<T extends ImageProps["layout"]> = T extends "fill"
  ? WithLayout<T>
  : RequiredOnly<WithLayout<T>, "width" | "height">;
```

最終的に、width, height の指定が必須の場合に指定がないと型エラーになるようになりました 🎉

![layout="fill" 以外を指定して width, height の指定を忘れると type error になる](/images/next-image-type/next-image-type-error.png)
_layout="fill" 以外を指定して width, height の指定を忘れると type error になる_

# 参考URL
- https://nextjs.org/docs/api-reference/next/image#layout
- https://www.forcia.com/blog/001561.html