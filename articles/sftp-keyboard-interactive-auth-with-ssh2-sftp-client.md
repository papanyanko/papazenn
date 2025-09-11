---
title: "ssh2-sftp-clientでkeyboard-interactive認証を使ってSFTPログインする方法"
emoji: "🔑"
type: "tech"
topics: ["sftp"]
published: true
---

Node.jsでSFTP接続をする際によく利用されるライブラリとしてssh2-sftp-clientがあります。
password認証でログインできると思いきや、サーバー側がkeyboard-interactive認証のみを受け付けているケースがありました。
ssh2-sftp-clientを使ってkeyboard-interactive認証する方法に関する日本語の具体的な情報がほとんどなかったので、そのうちどこかの誰かの役に立つのではと思い、その対応方法を紹介します。

使用した環境およびライブラリのバージョンは以下の通りです。

- Node.js v22 on AWS Lambda
- ssh2-sftp-client: ^12.0.1
- ssh2: 1.17.0（型参照用にdevDependenciesとして使用）
- @types/ssh2-sftp-client: 9.0.5

## トラブルの発端

Lambdaで取ってきたデータをフォーマットしてCSVファイルとして取引先のSFTPサーバに格納しようとしていました。
IDとパスワードを渡されて「これで接続してください」と言われていたので `sftp.connect()` にhost/username/passwordを指定して接続を試みたところ、以下のように失敗しました。

```sh
Inbound: Received USERAUTH_FAILURE (publickey,keyboard-interactive)
```

これはサーバーがpublickeyまたはkeyboard-interactive認証しか受け付けていないことを意味しています。
IDとパスワードが合っていても、認証方式が違うと失敗してしまうわけです。

## デバッグ方法

`sftp -vvv` のような詳細ログを出すには、`debug` オプションを使います。

```ts
debug: (message: string) => console.log(message)
```

これを設定しておくと、どの認証方式が試され、なぜ失敗しているかが分かります。

## keyboard-interactive認証を使う

ssh2-sftp-clientは内部でssh2を利用しています。ssh2にはkeyboard-interactive認証にも対応できるAPIがあります。
keyboard-interactive認証は、サーバーからの質問（プロンプト）にクライアントが回答する方式で、単純にパスワードを入力させる場合もあれば、ワンタイムコードや追加の認証情報を求められる場合もあります。
keyboard-interactive認証を使うには、以下の2つを設定する必要があります。

- `tryKeyboard: true` を設定する
  - 通常はプログラムからSFTP接続する状況でキーボード入力を求められることを想定しないので、ssh2はデフォルトでkeyboard-interactive認証を試みない
- keyboard-interactiveイベントのハンドラを登録する

参考になったIssueはこちら:
👉 https://github.com/mscdex/ssh2/issues/604#issuecomment-1144193157
"bullet proofed"と言っているだけあり、幅広い認証方法に対応できるものとなっています。

## 実装例

上記Issueはssh2のものなので、ssh2-sftp-clientを使う実装例を示しておきます。
（公開鍵認証には対応しておらずbullet-proofでないですが、今回の主題ではないのでご容赦ください）

```ts
import type { KeyboardInteractiveAuthMethod } from 'ssh2';
import SFTPClient from 'ssh2-sftp-client';

const sftp = new SFTPClient();

type SFTPCredentials = {
  host: string;
  port?: number;
  username: string;
  password: string;
  distPath: string;
};

const sendFileToSFTP = async (
  credentials: SFTPCredentials,
  fileContent: Buffer,
  fileName: string,
) => {
  try {
    const { host, port = 22, username, password, distPath } = credentials;

    // keyboard-interactiveイベントのハンドラを登録
    sftp.on('keyboard-interactive', keyboardHandler(password));

    await sftp.connect({
      host,
      port,
      username,
      password,
      tryKeyboard: true, // キーボード認証を有効化
      readyTimeout: 20000,
    });

    await sftp.put(fileContent, `${distPath}/${fileName}`);
    console.log('successfully uploaded: ', fileName)

  } catch (error) {
    console.error(error)
    throw error;
  } finally {
    await sftp.end();
  }
};

// パスワードを受け取ってkeyboard-interactiveイベントのハンドラを返す
type KeyboardHandler = (password: string) => KeyboardInteractiveAuthMethod['prompt'];
const keyboardHandler: KeyboardHandler =
  (password: string) => (name, instructions, lang, prompts, finish) => {
    finish([password]);
  };
```

注意点として、`sftp.on('keyboard-interactive', ...)` は `sftp.connect()` の前に登録しておく必要があります。

### 補足

`KeyboardInteractiveAuthMethod` がssh2においてkeyboard-interactive認証を表すクラスであり、 `prompt` メソッドがイベントハンドラです。
`prompt` メソッドの引数で重要なのが `prompts` と `finish` です。
`prompts` は `{ prompt: 'Prompt text', echo: true }` という形式のオブジェクトの配列で、たとえばパスワードを求められる場合は `{ prompt: 'Password:', echo: false }` となります。 `echo` はレスポンスが表示されるかどうかを表します。パスワードの場合は `false` なので入力したパスワードが表示されないということです。
プロンプトに対するクライアントのレスポンスは `finish` に配列として渡します。今回は `password` を渡しました。
詳細は[ssh2のREADME](https://github.com/mscdex/ssh2#readme)を参照してください。

## まとめ

IDとパスワードを渡されていたのでpassword認証かと思いきや、publickeyとkeyboard-interactive認証しか受け付けていないというパターンでした。
冷静になってこの設定の意図を想像すると、ファイル送信をシステムで行う場合はpublickey認証、手動で行う場合はkeyboard-interactive認証をさせるつもりだったのでは...？と思われます。
取引先に公開鍵を設定してもらったりpassword認証を許可してもらったりする時間がなかったので、今回は実装でどうにかしてしまいました。
しかし、セキュリティ的にもpublickey認証の方が望ましいので、IDとパスワードを渡された時点で公開鍵ではだめか確認しておくべきでした。
