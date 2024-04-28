---
title: "React 19 の新機能を理解する 〜 use 編"
emoji: "🕌"
type: "tech"
topics: ["react"]
published: true
---

[React 19 の新機能](https://react.dev/blog/2024/04/25/react-19) シリーズ第 2 弾です
第 1 弾はこちら：
https://zenn.dev/papanyanko/articles/react19-beta-release-blog-1

今回は use を見ていきます。

## use

Promise の結果や context のデータを読み込むことができます。もともとは Promise をサポートするために考案されました。

### 基本的な使い方

`use(resource)` という形で resource に Promise や context を渡します。

```js
import { use } from 'react';

function MessageComponent({ messagePromise }) {
  const message = use(messagePromise);
  const theme = use(ThemeContext);
  // ...
```

通常の Hooks との共通点はコンポーネントまたはカスタムフックの中でのみ呼び出せることです。しかし、 use は必ずしもトップレベルで呼び出される必要はなく、分岐やループの中、あるいは early return の後でも使えるところが違います。
context の読み込みでこの共通点と相違点を具体を見たあと、 Promise と組み合わせたデータのストリーミングを見てみたいと思います。

### context の読み込み

useContext(ThemeContext) と use(ThemeContext) はどちらも ThemeContext の値を読み取ります。
useContext は Hooks なので if 文の中で呼び出せませんが、 use はそれができます。

```
function HorizontalRule({ show }) {
  if (show) {
    const theme = use(ThemeContext);
    return <hr className={theme} />;
  }
  return false;
}
```

より柔軟な使い方ができるため、useContext よりも use の方が推奨されています。

### Promise によるサーバーからクライアントへのストリーミング

こちらが use のもともとの主目的です。
サーバーコンポーネントからクライアントコンポーネントへ Promise を props として渡すことにより非同期的にデータを受け渡すことができます。
以下の例で App はサーバーコンポーネント、 Message はクライアントコンポーネントです。

```js
import { fetchMessage } from './lib.js';
import { MessageContainer } from './message.js';

export default function Page() {
  const messagePromise = fetchMessage();
  // When `use` suspends in Message,
  // this Suspense boundary will be shown.
  return (
    <Suspense fallback={<div>waiting for message...</div>}>
      <MessageContainer messagePromise={messagePromise} />
    </Suspense>
  )
}

// message.js
'use client'
import { use, Suspense } from 'react';

function Message({ messagePromise }) {
  // `use` will suspend until the promise resolves.
  const messages = use(messagePromise);
  return messages.map(message => <p key={message.id}>{message.content}</p>);
}

export function MessageContainer({ messagePromise }) {
  return (
    <Suspense fallback={<p>⌛Downloading message...</p>}>
      <Message messagePromise={messagePromise} />
    </Suspense>
  );
}
```

Message が Suspense で囲まれており、 Promise が解決するまではフォールバックが表示されます。
Promise が解決すると、その値が use によって読み取られてフォールバックは Message で置き換わります。
このように、 use(Promise) を使うことによってサーバーコンポーネントで生成した Promise をクライアントコンポーネントで解決することができます。

一方、 use を使わずにサーバーコンポーネントで Promise を解決して、データをクライアントコンポーネントに渡すこともできます。

```js
export default async function Page() {
  const messages = await fetchMessage();
  return <Message messages={messages} />
}
```

Promise をサーバー側で解決すべきかクライアント側で解決すべきかについては [React Server Components](https://react.dev/reference/rsc/server-components) のページに書いてありました。

> The note content is important data for the page to render, so we await it on the server. The comments are below the fold and lower-priority, so we start the promise on the server, and wait for it on the client with the use API. This will Suspend on the client, without blocking the note content from rendering.

ブログの本文などページにとって重要なデータであればサーバーで await し、コメントはページの下方にあり優先度が低いので use を使います。それによって本文のレンダリングを妨げることなくコメントを Suspend することができます。

また、 use はカスタムフックの中でも使えます。

## まとめ

Promise を扱いやすくなるのは嬉しいです。
Server Component の async / await についてもあわせて掘り下げてみたいです。