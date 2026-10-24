---
title: VS Code をローカル環境(Dev Container)でビルドする
tags:
  - VSCode
  - Docker
private: false
updated_at: '2026-10-25T03:36:30+09:00'
id: e7bba0ec4f9ff037f149
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
## 概要

[VS Codeのソース](https://github.com/microsoft/vscode)を、Dev Container 上でビルドして起動するまでの記録。
検証日: 2026-10-24

1. リポジトリを clone し、`release/1.142` ブランチに切り替える
2. VS Code で開き、通知から「ボリュームでクローン」を選んで Dev Container を起動する
3. コンテナ内で `npm install` が失敗することを確認し、原因を調べる
4. コンテナ内でコンパイラを clang に切り替えて、`npm install` が通ることを確認する
5. `.devcontainer/post-create.sh` に clang の導入を書き足し、コンテナを Rebuild して、`./scripts/code.sh` で VS Code の起動を確認する

結果は、末尾の「結論」と「注意点」にまとめている。

## 環境

| 名前 | ID | バージョン |
|-|-|-|
| ホストOS | - | Windows 11 Pro 26H2(26300.9711) |
| PowerShell | Microsoft.PowerShell  | 7.6.8.0 |
| Git | Git.Git | 2.55.0.2 |
| Docker Desktop | Docker.DockerDesktop | 4.96.0 |
| Docker への割り当て | - | WSL2 バックエンド / CPU: 16 / メモリ: 14.52 GB |
| Microsoft Visual Studio Code (User) | Microsoft.VisualStudioCode | 1.143.1 |
| Dev Containers 拡張 | - | ms-vscode-remote.remote-containers@0.469.0 |

ログ上、ホストのカーネルは WSL2(`6.18.54.1-microsoft-standard-WSL2`)だった。

コンテナ側(調査で確認した値):

| 項目 | 値 |
|-|-|
| OS | Debian 12 (bookworm) |
| gcc / g++ | 12.2.0 |
| Python | 3.11.2 |
| Node | 24.21.0(`.nvmrc` は 24.18.0 で、ずれていた。今回の失敗の原因とは見ていない) |
| ビルド対象ランタイム | Electron 43.7.7(`.npmrc` の `runtime="electron"` / `target="43.7.7"`) |

## 手順

以下のホスト側の作業は `PowerShell` 上で実施。

```powershell
git clone https://github.com/microsoft/vscode.git
cd vscode
```

簡易な手順で再現性を高めるため、リリース済みの VS Code 1.143.1 のひとつ前にあたる `release/1.142` ブランチに切り替える。
`main` は日々変わるが、`release/1.142` は先日の確認時から変わっていなかった。
なお、`main` のままでも、後述のとおり同じエラーで失敗する。

```powershell
git switch release/1.142
git rev-parse HEAD
```

release ブランチも後から更新されうるため、`git rev-parse HEAD` の結果を、下表の検証したコミットと照合する。
一致しない場合は、ブランチが更新されている。
「ボリュームでクローン」ではリモートから新たにクローンされるので、コンテナ起動後に、コンテナ内のターミナルでも同じ確認をすると確実。

検証したコミット:

| ブランチ | コミット | 初期状態(clang 導入前)の `npm i` |
|-|-|-|
| `main` | `b3ce41974e82428b77417f6654976737e4d0b2a7` | 失敗(同じエラー) |
| `release/1.142` | `6d62b1d5dace93a33b3a6d3eca1317bd5e15bcb1` | 失敗 |

VS Code を起動する。

```powershell
code .
```

起動後、しばらく待つと、「コンテナーで開発するフォルダーをもう一度開きます。」という通知が表示される。
選択肢はブランチの upstream の有無で変わる。

- (a) ブランチの upstream が設定されている場合: 「コンテナーで再度開く」と「ボリュームでクローン」の2択

  ![upstream あり: 2択の通知](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/176367/2fb4d483-c4e2-4226-b9f6-2c4867094609.png)

- (b) ブランチの upstream が設定されていない場合: 「コンテナーで再度開く」のみ

  ![upstream なし: 1択の通知](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/176367/49769df2-46a0-4a63-b0c4-274c141f46e7.png)

**「ボリュームでクローン」は (a) のときだけ選べる。**
ここでは「ボリュームでクローン」を選択する。

### (b) になった場合

- ブランチの upstream を設定し直して、VS Code を開き直す。

  ```powershell
  git branch --set-upstream-to=origin/release/1.142
  ```

- または、コマンドパレットから **Dev Containers: Clone Repository in Container Volume...** を直接実行し、リポジトリのURLを入力する。
ブランチはプルダウンで選択できる。
**タグは選択できない**。
タグで作業したい場合は、ブランチでクローンしたあと、コンテナ内で `git switch -c <ブランチ名> <タグ名>` を実行する。

### コンテナ起動後

開発コンテナで VS Code が再起動すると、`.devcontainer/post-create.sh` が実行される。元の内容は次のとおり。

```sh
#!/bin/sh

npm i
npm run electron
```

ここで `npm i` (`npm install`) が異常終了する。

決定的なエラー行:

```text
v8config.h:854:20: error: expected identifier before ‘__attribute__’
```

<details>
<summary>npm i のエラー(全文)</summary>

```log
npm error code 1
npm error path /workspaces/vscode/node_modules/native-keymap
npm error command failed
npm error command sh -c node-gyp rebuild
npm error make: Entering directory '/workspaces/vscode/node_modules/native-keymap/build'
npm error   CXX(target) Release/obj.target/keymapping/src/string_conversion.o
npm error   CXX(target) Release/obj.target/keymapping/src/keymapping.o
npm error make: Leaving directory '/workspaces/vscode/node_modules/native-keymap/build'
npm error gyp info it worked if it ends with ok
npm error gyp info using node-gyp@12.4.0
npm error gyp info using node@24.21.0 | linux | x64
npm error gyp info find Python using Python version 3.11.2 found at "/usr/bin/python3"
npm error gyp info spawn /usr/bin/python3
npm error gyp info spawn args [
npm error gyp info spawn args '/usr/local/lib/node_modules/npm/node_modules/node-gyp/gyp/gyp_main.py',
npm error gyp info spawn args 'binding.gyp',
npm error gyp info spawn args '-f',
npm error gyp info spawn args 'make',
npm error gyp info spawn args '-I',
npm error gyp info spawn args '/workspaces/vscode/node_modules/native-keymap/build/config.gypi',
npm error gyp info spawn args '-I',
npm error gyp info spawn args '/usr/local/lib/node_modules/npm/node_modules/node-gyp/addon.gypi',
npm error gyp info spawn args '-I',
npm error gyp info spawn args '/home/node/.cache/node-gyp/43.7.7/include/node/common.gypi',
npm error gyp info spawn args '-Dlibrary=shared_library',
npm error gyp info spawn args '-Dvisibility=default',
npm error gyp info spawn args '-Dnode_root_dir=/home/node/.cache/node-gyp/43.7.7',
npm error gyp info spawn args '-Dnode_gyp_dir=/usr/local/lib/node_modules/npm/node_modules/node-gyp',
npm error gyp info spawn args '-Dnode_lib_file=/home/node/.cache/node-gyp/43.7.7/<(target_arch)/node.lib',
npm error gyp info spawn args '-Dmodule_root_dir=/workspaces/vscode/node_modules/native-keymap',
npm error gyp info spawn args '-Dnode_engine=v8',
npm error gyp info spawn args '--depth=.',
npm error gyp info spawn args '--no-parallel',
npm error gyp info spawn args '--generator-output',
npm error gyp info spawn args 'build',
npm error gyp info spawn args '-Goutput_dir=.'
npm error gyp info spawn args ]
npm error gyp info spawn make
npm error gyp info spawn args [ 'BUILDTYPE=Release', '-C', 'build' ]
npm error In file included from /home/node/.cache/node-gyp/43.7.7/include/node/v8-handle-base.h:8,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/v8-local-handle.h:13,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/v8-array-buffer.h:12,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/v8.h:24,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/node.h:80,
npm error                  from ../src/keymapping.cc:7:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-internal.h:468:1: warning: multi-line comment [-Wcomment]
npm error   468 | //         / \
npm error       | ^
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-internal.h:470:1: warning: multi-line comment [-Wcomment]
npm error   470 | //       / \
npm error       | ^
npm error In file included from /home/node/.cache/node-gyp/43.7.7/include/node/cppgc/common.h:8,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/v8.h:23:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8config.h:854:20: error: expected identifier before ‘__attribute__’
npm error   854 | # define V8_EXPORT __attribute__((visibility("default")))
npm error       |                    ^~~~~~~~~~~~~
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:657:32: note: in expansion of macro ‘V8_EXPORT’
npm error   657 |       "buffer if you cannot.") V8_EXPORT Value {
npm error       |                                ^~~~~~~~~
npm error In file included from /home/node/.cache/node-gyp/43.7.7/include/node/v8-object.h:14,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/v8-array-buffer.h:14:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:675:3: warning: ‘visibility’ attribute ignored [-Wattributes]
npm error   675 |   };
npm error       |   ^
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:658:4: error: expected primary-expression before ‘public’
npm error   658 |    public:
npm error       |    ^~~~~~
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:658:4: error: expected ‘}’ before ‘public’
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:657:48: note: to match this ‘{’
npm error   657 |       "buffer if you cannot.") V8_EXPORT Value {
npm error       |                                                ^
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:658:4: error: cannot convert ‘<brace-enclosed initializer list>’ to ‘int’ in initialization
npm error   658 |    public:
npm error       |    ^~~~~~
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-primitive.h:658:4: error: expected ‘;’ before ‘public’
npm error In file included from /home/node/.cache/node-gyp/43.7.7/include/node/v8-function.h:15,
npm error                  from /home/node/.cache/node-gyp/43.7.7/include/node/v8.h:33:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h: In static member function ‘static v8::Intercepted (* v8::NamedPropertyHandlerConfiguration::ConvertSetter(v8::NamedPropertySetterCallback))(v8::Local<v8::Name>, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<v8::Boolean>&)’:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h:762:12: warning: cast between incompatible function types from ‘v8::NamedPropertySetterCallback’ {aka ‘v8::Intercepted (*)(v8::Local<v8::Name>, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<void>&)’} to ‘v8::NamedPropertySetterCallbackV2’ {aka ‘v8::Intercepted (*)(v8::Local<v8::Name>, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<v8::Boolean>&)’} [-Wcast-function-type]
npm error   762 |     return NamedPropertySetterCallbackV2(value);
npm error       |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h: In static member function ‘static v8::Intercepted (* v8::NamedPropertyHandlerConfiguration::ConvertDefiner(v8::NamedPropertyDefinerCallback))(v8::Local<v8::Name>, const v8::PropertyDescriptor&, const v8::PropertyCallbackInfo<v8::Boolean>&)’:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h:775:12: warning: cast between incompatible function types from ‘v8::NamedPropertyDefinerCallback’ {aka ‘v8::Intercepted (*)(v8::Local<v8::Name>, const v8::PropertyDescriptor&, const v8::PropertyCallbackInfo<void>&)’} to ‘v8::NamedPropertyDefinerCallbackV2’ {aka ‘v8::Intercepted (*)(v8::Local<v8::Name>, const v8::PropertyDescriptor&, const v8::PropertyCallbackInfo<v8::Boolean>&)’} [-Wcast-function-type]
npm error   775 |     return NamedPropertyDefinerCallbackV2(value);
npm error       |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h: In static member function ‘static v8::Intercepted (* v8::IndexedPropertyHandlerConfiguration::ConvertSetter(v8::IndexedPropertySetterCallbackV2))(uint32_t, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<v8::Boolean>&)’:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h:893:12: warning: cast between incompatible function types from ‘v8::IndexedPropertySetterCallbackV2’ {aka ‘v8::Intercepted (*)(unsigned int, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<void>&)’} to ‘v8::IndexedPropertySetterCallback’ {aka ‘v8::Intercepted (*)(unsigned int, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<v8::Boolean>&)’} [-Wcast-function-type]
npm error   893 |     return IndexedPropertySetterCallback(value);
npm error       |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h: In static member function ‘static v8::Intercepted (* v8::IndexedPropertyHandlerConfiguration::ConvertDefiner(v8::IndexedPropertyDefinerCallbackV2))(uint32_t, const v8::PropertyDescriptor&, const v8::PropertyCallbackInfo<v8::Boolean>&)’:
npm error /home/node/.cache/node-gyp/43.7.7/include/node/v8-template.h:906:12: warning: cast between incompatible function types from ‘v8::IndexedPropertyDefinerCallbackV2’ {aka ‘v8::Intercepted (*)(unsigned int, const v8::PropertyDescriptor&, const v8::PropertyCallbackInfo<void>&)’} to ‘v8::IndexedPropertyDefinerCallback’ {aka ‘v8::Intercepted (*)(unsigned int, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<v8::Boolean>&)’} [-Wcast-function-type]
npm error   906 |     return IndexedPropertyDefinerCallback(value);
npm error       |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
npm error make: *** [keymapping.target.mk:131: Release/obj.target/keymapping/src/keymapping.o] Error 1
npm error gyp ERR! build error 
npm error gyp ERR! stack Error: `make` failed with exit code: 2
npm error gyp ERR! stack at ChildProcess.<anonymous> (/usr/local/lib/node_modules/npm/node_modules/node-gyp/lib/build.js:219:23)
npm error gyp ERR! System Linux 6.18.54.1-microsoft-standard-WSL2
npm error gyp ERR! command "/usr/local/bin/node" "/usr/local/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js" "rebuild"
npm error gyp ERR! cwd /workspaces/vscode/node_modules/native-keymap
npm error gyp ERR! $npm_package_name native-keymap
npm error gyp ERR! $npm_package_version 3.3.9
npm error gyp ERR! not ok
npm error A complete log of this run can be found in: /home/node/.npm/_logs/2026-10-23T20_26_30_666Z-debug-0.log
```

</details>

## 調査

コンテナ内のターミナルで、環境を調べた。

```bash
gcc --version | head -1
g++ --version | head -1
cat /etc/os-release | head -3
clang --version 2>/dev/null | head -1
cat .nvmrc
grep -iE "target|runtime|disturl" .npmrc
```

結果:

```text
gcc (Debian 12.2.0-14+deb12u1) 12.2.0
g++ (Debian 12.2.0-14+deb12u1) 12.2.0
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
24.18.0
disturl="https://electronjs.org/headers"
target="43.7.7"
runtime="electron"
```

インストール可能な clang を確認した。

```bash
sudo apt update
apt-cache search '^clang-[0-9]+$'
```

結果:

```text
clang-13 - C, C++ and Objective-C compiler
clang-14 - C, C++ and Objective-C compiler
clang-15 - C, C++ and Objective-C compiler
clang-16 - C, C++ and Objective-C compiler
clang-19 - C, C++ and Objective-C compiler
clang-22 - C, C++ and Objective-C compiler
```

### コンパイラごとの結果

| コンパイラ | `npm install` | 備考 |
|-|-|-|
| GCC 12.2 | 失敗 | 初期状態 |
| clang-13 | 未検証 | |
| clang-14 | 失敗 | |
| clang-15 | 失敗 | |
| clang-16 | 成功 | VS Code 起動まで確認 |
| clang-19 | 成功 | VS Code 起動まで確認 |
| clang-22 | 成功 | VS Code 起動まで確認 |

検証した範囲では、失敗と成功の境目は clang-15 と clang-16 の間にある。原因が clang の新しさそのものなのかは、今回の検証だけでは断定できない。

## 対処

### 1. コンテナ内で手動で切り替える

コンテナ内のターミナルで実施する。clang-16 以上であればどれでもよい(以下は clang-22 の例)。

```bash
sudo apt-get install -y clang-22
export CC=clang-22 CXX=clang++-22
# 追跡されていないファイルをすべて削除する。未コミットの新規ファイルがある場合は、先に退避すること。
git clean -xfd
rm -rf ~/.cache/node-gyp
# 再度 npm i を実施
npm install
# .devcontainer/post-create.sh の残りを実施
npm run electron
```

最後に、ビルドされた VS Code が起動することを確認した。

```bash
./scripts/code.sh
```

### 2. post-create.sh に書き足して Rebuild する

手動で入れた clang は、コンテナを Rebuild すると消える。そこで、`.devcontainer/post-create.sh` に同じ内容を書き足す。

```sh
#!/bin/sh
set -e

sudo apt-get update
sudo apt-get install -y clang-22
export CC=clang-22 CXX=clang++-22

npm i
npm run electron
```

- `set -e` は、途中のコマンドが失敗したときにそこで止めるための指定。

書き換えたら、コマンドパレットから **Dev Containers: Rebuild Container** を実行する。通常の Rebuild で `post-create.sh` が再実行され、そのあと `./scripts/code.sh` で VS Code を起動できた。
**Rebuild Container Without Cache** は使わなかった。`post-create.sh` はコンテナの作成後に実行されるため、イメージのキャッシュを捨てる必要はない。

## 結論

- 公式の Dev Container でそのままビルドすると、`.devcontainer/post-create.sh` 内の `npm i` が **GCC 12.2 では失敗する**。`main` と `release/1.142` のどちらでも、`native-keymap` のビルドで V8 ヘッダのエラーになり、同じ結果だった。
- 検証した範囲では、clang-14 / 15 は失敗し、**clang-16 以上**(16 / 19 / 22)に切り替えると `npm install` が通り、`./scripts/code.sh` で VS Code(Code - OSS)を起動できた。
- `.devcontainer/post-create.sh` に clang の導入と `export CC=... CXX=...` を書き足し、**Dev Containers: Rebuild Container** を実行すると、`./scripts/code.sh` で VS Code を起動できた。
- 失敗の原因は、Electron のV8ヘッダ(`native-keymap` のネイティブビルド時に参照される)をコンパイルできないことだと推測している。公式に確認した原因ではない。

## 注意点

- 手動で `export CC=... CXX=...` を実行した場合、そのターミナルセッションだけで有効。新しいターミナルを開くと再設定が必要。`post-create.sh` に書いた `export` は、そのスクリプト内のコマンドにだけ効く。
- 手動で入れた clang はコンテナ内の変更なので、コンテナを Rebuild すると消える。`post-create.sh` に導入を書き足しておけば、Rebuild のたびに自動で入る。書き足していない場合は、Rebuild 後に再び `npm i` が失敗する。
- `.devcontainer/post-create.sh` はリポジトリの追跡対象ファイルなので、書き換えると `git status` に差分が出る。コミットしないこと。
- 「ボリュームでクローン」する流れでは、最初のコンテナ作成の時点で `post-create.sh` を書き換えられないため、初回は `npm i` が必ず失敗する。初回から成功させる方法として、ホスト側にクローンして、コンテナを開く前に `post-create.sh` を書き換えてから「コンテナーで再度開く」を選択する。
- ボリュームにクローンした場合、Docker側でボリュームを削除すると、push していない変更は失われる。
- GUI が表示できない環境では、`./scripts/code-server.sh --launch` でブラウザから確認する方法もある(今回は `code.sh` でウィンドウが表示された)。
