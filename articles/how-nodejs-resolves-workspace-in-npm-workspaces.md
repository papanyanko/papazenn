---
title: "Node.jsのモジュール解決とnpm workspacesにおけるモノレポ参照の仕組み"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "nodejs", "typescript", "javascript"]
published: true
---

## はじめに

モノレポを利用しはじめるにあたって、npm workspacesで他パッケージのモジュールを参照する仕組みを調べたのでまとめました。

## 1. Node.js の基本モジュール解決ルール

Node.jsでは、`require('foo')` のようにモジュール名が渡されたとき、次の順で探索します。

1. **コアモジュールの確認**
`http` や `fs` など、Node.jsに組み込みのモジュールがあればそれを即座に返す。

2. **相対／絶対パス指定の確認**  
`./foo` や `/path/to/foo` のようにパスで指定された場合は、そのパスを直接解決。

3. **名前（識別子）による探索**  
現在のファイルのディレクトリから順に上位へ辿り、`node_modules` を探す。
例： `/project/src/index.js` から `require('foo')` を実行した場合

```
/project/src/node_modules/foo
/project/node_modules/foo
/node_modules/foo
```

> **ポイント：** Node.jsはディレクトリ構造を「現在地から上に向かって」探していく

## 2. npm workspacesの動作

npm workspacesは、モノレポ構成で複数パッケージを一括管理する仕組みです。

- ルートの `package.json` に `"workspaces"` フィールドを設定
- ルートで `npm install` を実行すると
  - 共通依存を **ルートの `node_modules`** にhoist（集約）
  - 各ワークスペースパッケージを **ルートの `node_modules` にシンボリックリンク**として配置（こちらが重要）

ディレクトリ例：

```
/project
├── node_modules
│   ├── package-a -> ../packages/package-a (symlink)
│   └── package-b -> ../packages/package-b (symlink)
└── packages
    ├── package-a
    │   └── package.json
    └── package-b
        └── package.json
```

> **ポイント：** 各ワークスペースパッケージをルートの `node_modules` にシンボリックリンクとして配置

## 3. モノレポで他パッケージを参照できる理由

例として `/project/packages/package-b/index.js` で

```js
require('package-a');
```

を実行した場合のNode.jsの探索順はこうなります。

1. /project/packages/package-b/node_modules/package-a → 存在しない
2. /project/packages/node_modules/package-a → 存在しない
3. /project/node_modules/package-a → symlinkがあるのでヒット
4. 以降の探索は不要

つまり、Node.js自体は通常通り「下から上へ順に」探索していますが、npm workspacesが各ワークスペースパッケージをルートの `node_modules` にシンボリックリンクとして配置しているため、他のワークスペースを参照できます。
