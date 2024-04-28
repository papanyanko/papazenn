---
title: "React 19 の新機能を理解する 〜 <form> Actions, useActionState, useFormStatus 編"
emoji: "🕌"
type: "tech"
topics: ["react"]
published: true
---

4 月 25 日、ついに React 19 Beta がリリースされました！
https://react.dev/blog/2024/04/25/react-19
このリリースはライブラリ開発者向けですが、内容を把握してReact 19に備えておきたいと思います。

本記事では公式 BLOG の内容のうち、 <form> Actions と useActionState, useFormStatus の部分を紹介します。

## そも Actions とは

React 19 では、"Actions" と呼ばれる概念が導入されました。これにより、データの更新に伴う状態を自分で管理する必要がなくなりました。
具体的には、従来は name, error, pending などを state を使って管理するコードを書かなければいけませんでしたが、 以下のように useTransition フックが返す startTransition を使うことによりペンディング状態の管理を自動化できます。

```diff js
import { updateName } from './lib.js'

function UpdateName() {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
-  const [isPending, setIsPending] = useState(false);
+  const [isPending, startTransition] = useTransition();

  const handleSubmit = async () => {
-   setIsPending(true);
+   startTransition(async () => {
    const { error } = await updateName(name);
-   setIsPending(false);
    if (error) {
      setError(error);
      return;
    } 
    redirect("/path");
+   })
  };

  return (
    <div>
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        Update
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

useTransition フックは以前からありましたが、今までは startTransition に渡す関数は同期的でなければならなかったことに注意してください。
非同期的な [transition](https://ja.react.dev/blog/2022/03/29/react-v18#new-feature-transitions) を使用する関数を総じて "Action" と呼ぶようです。
"Action" にはペンディング状態、楽観的な更新、エラーハンドリングのほか、以下で説明する <form> Action といったものが該当し、データの更新にまつわる状態管理を自動化してくれます。

上記のサンプルコードは <form> Actions と useActionState を使って次のようにさらに単純化できます。

```js
import { updateName } from './lib.js'

async function changeNameAction(previousState, formData) {
  const { error } = await updateName(formData.get("name"));
  if (error) {
    return error;
  }
  redirect("/path");
}

function ChangeName() {
  const [error, submitAction, isPending] = useActionState(changeNameAction, 'John Doe');

  return (
    <form action={submitAction}>
      <input type="text" name="name" />
      <button type="submit" disabled={isPending}>Update</button>
      {state.error && <p>{state.error}</p>}
    </form>
  );
}
```

pending だけでなく error を管理するコードがなくなりました。

それでは、 <form> Action と useActionState について順番に見てみましょう。

### <form> Actions

React DOM の新機能で、 `<form action={submitAction}>` の部分が <form> Action です。
submit すると submitAction が実行されます。

```js
function ChangeName() {
  /**
   * calls useActionState
   */

  return (
    <form action={submitAction}>
      <input type="text" name="name" />
      <button type="submit" disabled={isPending}>Update</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

<form> に限らず <button> と <input> にも formAction という props で action を渡すことができます。

### useActionState

Actions の一般的な処理を共通化するため、 useActionState フックが用意されました。
useActionState は関数 action を必須の引数に取ります。この関数がまさに上述の "Action" です。

```js
import { updateName } from './lib.js'

async function changeNameAction(previousState, formData) {
  const { error } = await updateName(formData.get("name"));
  if (error) {
    return error;
  }
  redirect("/path");
}

function ChangeName() {
  const [formState, submitAction, isPending] = useActionState(changeNameAction, 'John Doe');

  /**
   * returns the form
   */
}
```

上記の例では error だけを取り出して返していますが、useActionState のType 定義を見ると
`useActionState(action, ?initialState, permalink?): [state, formAction, isPending]`
となっており、 error に限らず成功した場合もstateとして Action の結果を返すことができます。

**useActionState はカナリアリリースでは React DOM の一部で useFormState と呼ばれていましたが、 useFormState は deprecated になっている** ので注意が必要です。

:::details useActionState と useFormState について

「 useFormState は本来、フォームの状態ではなく action の状態を管理するもの」という点がキーになります。
したがって、 useFormState はその名前にもかかわらず、 <form> と一緒に使われなければならない必然性はまったくありませんでした。
例えば <button> の handleSubmit の中で useFormState の返す action を使うこともできました。
```js
import { useActionState, useRef } from "react";

function Form({ someAction }) {
  const ref = useRef(null);
  const [state, action] = useFormState(someAction);

  async function handleSubmit() {
    await action({ email: ref.current.value });
  }

  return (
    <div>
      <input ref={ref} type="email" name="email" />
      <button onClick={handleSubmit} >
        Submit
      </button>
      {state.errorMessage && <p>{state.errorMessage}</p>}
    </div>
  );
}
```

これは <form> 専用の useFormStatus との違いです（そもそも名前が似すぎててまぎらわしい）。

また、 useFormState は <form> 専用ではないため React DOM に限らず React Native などでも利用可能なはずのものでした。

そこで action の状態を管理するものという意味を明確にするため、名前を useActionState と改め、返値に pending が加えられました。

:::

### useFormStatus

React DOM の新機能で、ボタンなど <form> の部品コンポーネントの内部から、親となる <form> の状態を参照することができるようになります。
これにより、例えば以下のように <form> が pending のときはボタンを押せなくするといったことが props を渡すことなく実装できます。

```js
import { useFormStatus } from 'react-dom';

function DesignButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending} />
}
```

## まとめ

フォームの実装が便利になりそうです。
他の新機能についても続けて見ていきたいと思います。