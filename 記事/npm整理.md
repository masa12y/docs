# npm を基礎から理解し直す — package.json・lock・install の仕組みを整理する

`npm install` は、フロントエンド開発をしている中で何度も使っています。
それなのに、`package-lock.json` を消したら動いた、CI でだけインストールが失敗する、気づいたらバージョンが変わっていた、といった場面で、何が起きているかを説明できないことに気づきました。

使えているからいいだろう、と思いながらもどこかモヤモヤしていたので、実際に使う中で大切だと感じたポイントを中心に、改めて自分なりに整理してみました。

同じようになんとなくで使ってきた方の参考になれば嬉しいです。

---

## npm とは

npm（Node Package Manager）は、JavaScript のパッケージ管理ツールです。

npm で依存関係を管理する上では、以下の 3 つが重要な役割を持っています。

- レジストリ（npmjs.com）: 世界中の開発者が公開したパッケージが登録されているデータベース。`npm install` を実行すると、ここからパッケージをダウンロードしてきます。
- CLI（コマンドラインツール）: ターミナルから `npm install`・`npm run`・`npm publish` などのコマンドを実行するためのツール。日常的な作業のほとんどはこれを通じて行います。
- package.json: プロジェクトのメタ情報と使うパッケージの一覧を管理するファイル。npm の動作はすべてこのファイルを起点にしています。

```bash
# Node.js に同梱されているため、インストール後すぐに使える
$ npm --version
10.9.0
```

npm の基本的な使い方は、`npm install <パッケージ名>` でパッケージをインストールし、`node_modules/` 以下に展開するだけです。しかし「なぜそのバージョンがインストールされたのか」「なぜ環境によって動作が違うのか」を理解するには、もう少し深いところまで知る必要があります。

---

## 1. package.json と package-lock.json

この 2 つのファイルは役割がまったく異なります。

package.json は「意図」を書くファイルです。人間が手で編集し、どのパッケージのどのバージョン範囲を使いたいかを宣言します。たとえば以下のように記述します。

```json
{
  "dependencies": {
    "react": "^18.0.0",
    "axios": "~1.6.0"
  }
}
```

ここで重要なのは、バージョンが範囲で書かれている点です。`^18.0.0` は「18.x.x の最新を使う」という意味で、`18.2.0` でも `18.3.1` でも条件を満たします。

package-lock.json は「結果」を記録するファイルです。npm が自動生成し、実際にインストールされた node_modules ツリー全体を正確に記録します。バージョンは完全に固定されており、`"version": "18.2.0"` のように特定の 1 バージョンだけが記録されます。推移的依存（自分が依存しているパッケージが、さらに依存しているパッケージ）も含む、すべてのパッケージのバージョン・取得元 URL・整合性ハッシュが記録されます。

一言でまとめると「package.json は設計図、package-lock.json はスナップショット」です。

### よくあるミス: package-lock.json を .gitignore に追加してしまう

package-lock.json は自動生成されるからとコミットしない、あるいは `.gitignore` に追加してしまうケースがあります。しかしこれをやると、チームメンバーや CI 環境でインストールするたびに異なるバージョンが解決されてしまいます。「自分の環境では動くのに CI では動かない」の原因の多くはここにあります。

package-lock.json は必ずコミットしてください。

---

## 2. セマンティックバージョニング — `^` と `~` の意味を正確に知る

npm のバージョン指定は `MAJOR.MINOR.PATCH` の 3 つの数字で構成されています。

- MAJOR — 後方互換性のない変更。`1.x.x` → `2.0.0` のように上がったら、API や動作が変わっている可能性があります。
- MINOR — 後方互換のある機能追加。`1.2.x` → `1.3.0` のように新機能が増えたが、既存のコードは壊れません。
- PATCH — 後方互換のあるバグ修正。`1.2.3` → `1.2.4` のように小さな修正です。

この定義をもとに、範囲指定の演算子を理解すると一気にわかりやすくなります。

`^`（キャレット）は MAJOR を固定し、MINOR と PATCH の更新を許可します。`npm install` でパッケージを追加したときにデフォルトで付く記号です。

```
^1.2.3  →  >=1.2.3 <2.0.0
```

`~`（チルダ）は MAJOR と MINOR を固定し、PATCH の更新のみ許可します。

```
~1.2.3  →  >=1.2.3 <1.3.0
```

バージョン番号のみ（ピン留め）は完全固定で、そのバージョンだけを使います。

```
1.2.3  →  1.2.3 のみ
```

ピン留めしたい場合は `--save-exact`（または `-E`）フラグを使います。

```bash
npm install react --save-exact
# package.json に "react": "18.2.0" と記録される
```

### よくあるミス: `^` がついているから同じバージョンになると思っていた

`"react": "^18.0.0"` と書いてあっても、package-lock.json がなければインストールするたびに `18.0.0`、`18.1.0`、`18.2.0` など最新の MINOR/PATCH バージョンが選ばれます。「半年前にセットアップした環境と今の環境で微妙に動作が違う」という場合、これが原因のことがあります。

---

## 3. npm install の内部で何が起きているか

`npm install` は単純に「パッケージを落としてくる」だけではありません。内部では package.json と package-lock.json の整合性チェックが走っています。

### 整合性の判定基準

整合性があるとは「lock に記録されたバージョンが、package.json のバージョン範囲を満たしているかどうか」です。

```
# 整合性あり
package.json:      "react": "^18.0.0"
package-lock.json: "version": "18.2.0"
→ 18.2.0 は >=18.0.0 <19.0.0 を満たす

# 整合性なし
package.json:      "react": "^19.0.0"  （手動で変更した）
package-lock.json: "version": "18.2.0"
→ 18.2.0 は >=19.0.0 <20.0.0 を満たさない
```

### ① 整合性が取れている場合（lock が package.json の範囲を満たす）

lock ファイルが「正解」として扱われ、次の順序で処理されます。

1. package-lock.json を参照: lock に記録された正確なバージョンを使用します（レジストリへの問い合わせを最小化）
2. node_modules との差分チェック: 未インストールまたは壊れた依存を検出します
3. 不足分のみ追加インストール: 既存パッケージはそのまま保持されます
4. package-lock.json は変更しない: これにより環境間での再現性が保たれます

つまり、lock がある状態で `npm install` を実行すると、バージョンは lock に従って固定されます。チームメンバーが同じ lock ファイルを使えば、全員が同じバージョンのパッケージをインストールできます。

### ② 整合性が取れていない場合（lock が package.json の範囲を満たさない）

package.json が「真実の源泉（source of truth）」として扱われ、lock が更新されます。

1. 不一致を検出: package.json の react を ^18 から ^19 に手動変更した場合など
2. 新バージョンをレジストリで解決: package.json の範囲を満たす最新バージョンを取得します
3. 新バージョンをインストール: node_modules に配置・更新します
4. package-lock.json を上書き更新: 新しく解決されたバージョンで lock を再生成します

### 推移的依存のバージョンはどう決まるか

自分が直接 `npm install` したパッケージだけでなく、そのパッケージが依存しているパッケージ（推移的依存）のバージョンも、package-lock.json に完全に記録・固定されます。

lock ファイルがある状態では推移的依存も固定されるため、どの環境でも同一のツリーを再現できます。逆に lock ファイルがない初回インストール時は、npm がレジストリから semver 範囲を解決して推移的依存のバージョンを決定するため、実行タイミングによって異なるバージョンが選ばれる可能性があります。

---

## 4. npm install vs npm ci — 使い分けを正しく理解する

`npm ci` は CI/CD 環境やチームへの初回セットアップのために設計されたコマンドです。`npm install` とは動作が根本的に異なります。

npm ci の特徴をひとつひとつ見ていきます。

package-lock.json が必須です。lock ファイルが存在しない場合、即座にエラーで終了します。

```bash
$ npm ci
npm error The `npm ci` command can only install with an existing package-lock.json or
npm error npm-shrinkwrap.json with lockfileVersion >= 1.
```

整合性が取れていない場合もエラーで終了します。`npm install` は不一致を検出したら lock を更新して進みますが、`npm ci` は更新せずにエラーを出します。これは「意図していない変更が混入しないようにする」ための設計です。

```bash
$ npm ci
npm error `npm ci` can only install packages when your package.json and
npm error package-lock.json or npm-shrinkwrap.json are in sync.
```

node_modules をまるごと削除してから再構築します。差分更新ではなくクリーンインストールを行うため、環境の汚染が起きません。

lock ファイルに絶対に書き込みません。インストール結果が lock と必ず一致するため、再現性が完全に保証されます。

使い分けのシンプルな基準はこうです。

- ローカル開発で依存を追加・更新するとき → `npm install`
- CI/CD パイプライン、本番ビルド、初回セットアップ → `npm ci`

### よくあるミス: CI でも npm install を使ってしまう

`npm install` を CI で実行すると、package.json と lock の不一致があった場合に lock が更新されてしまいます。また node_modules を削除しないため、以前のビルドキャッシュが残って環境が汚染されるリスクがあります。CI では `npm ci` を使うことで、常にクリーンで再現性のあるビルドが保証されます。

---

## まとめ

日々の開発で npm を扱う際は、以下のポイントを押さえておくとトラブルを大きく減らせます。

- package.json は「意図（範囲）」、package-lock.json は「結果（スナップショット）」として役割が違う
- 環境による予期せぬバージョン差異を防ぐため、package-lock.json は必ず git 管理する
- バージョンを厳密に固定したい場合は --save-exact を使う
- package-lock.json があれば npm install は lock のバージョンを優先して守ってくれる
- CI や本番ビルドなど、再現性とクリーンさを保証したい場面では npm install ではなく npm ci を使う

「なんとなく動いているから大丈夫」ではなく、裏側の仕組みを知っておくだけで、インストール周りのエラーやバージョントラブルにも落ち着いて対応できるようになると思います。

---

## 参考

- [npmの概要（構成要素・役割） | npm Docs](https://docs.npmjs.com/about-npm)
- [package-lock.jsonの仕様と役割 | npm Docs](https://docs.npmjs.com/configuring-npm/package-lock-json)
- [依存関係のロックと再現性の仕組み | npm Docs](https://docs.npmjs.com/configuring-npm/package-locks)
- [npm install コマンドの動作仕様 | npm Docs](https://docs.npmjs.com/cli/v11/commands/npm-install)
- [npm ci コマンドの動作仕様 | npm Docs](https://docs.npmjs.com/cli/v11/commands/npm-ci)
- [セマンティックバージョニング（SemVer）の基礎 | npm Docs](https://docs.npmjs.com/about-semantic-versioning)
- [npmにおけるバージョン範囲（^ や ~ など）の指定ルール | npm Docs](https://docs.npmjs.com/cli/v6/using-npm/semver)
