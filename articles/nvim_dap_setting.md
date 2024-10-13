---
title: "nvim-dap の設定方法"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [neovim]
published: true
---

nvim-dap の設定方法を覚えては忘れを繰り返しているので文章として残しておく。
この記事は [nvim-dap][7] v0.8.0 に基づいている

そもそも Debug Adapter Protocol (DAP) とは何かという詳細な説明は公式ページを参照
https://microsoft.github.io/debug-adapter-protocol/overview

# 基本

言語問わず知っておいたほうがいいのは doc のこの3箇所

- adapters テーブルの key
  - https://github.com/mfussenegger/nvim-dap/blob/0.8.0/doc/dap.txt#L51-L54
- configurations テーブルの key
  - https://github.com/mfussenegger/nvim-dap/blob/0.8.0/doc/dap.txt#L242-L244
- configurations での adapter 指定
  - https://github.com/mfussenegger/nvim-dap/blob/0.8.0/doc/dap.txt#L232

- `configurations` テーブルのキーはファイルタイプと紐づく
- `adapters` テーブルのキーの名前は任意
  - `configuration` に書いた `type` が adapters テーブルの key 名と対応する

例えばこのような設定を書いたとする

```lua
local dap = require 'dap'

dap.configurations.foo = {
  {
    name = "debug sample 1",
    type = "foo",
    request = "attach",
  },
  {
    name = "debug sample 2",
    type = "bar",
    request = "attach",
  },
}

dap.adapters.foo = {
  type = "server",
  host = "127.0.0.1",
  port = 4040,
}

dap.adapters.bar = {
  ...
}
```

このとき filetype `foo` を開いているときに nvim-dap を使うと

- `dap.configurations.foo` の設定が読み出される
- ユーザーが好みの設定を選ぶ
- 例として1番目の設定 (`name = "debug sample 1"`) を選んだとするとの `type` が `foo` なので `dap.adapters.foo` が使われる

という流れになる。同様に2番目の configurations を選択すると `type` が `bar` のため `dap.adapters.bar` が使用される。


# nvim-dap-xxx は何をしているのか？

[nvim-dap-python][10] や [nvim-dap-go][8]といった nvim-dap の extension があるがこれらは一体何をしてくれるものなのだろうか？
始め私はこれらが各言語用の debug adapter なのだと思っていたがそうではなかった。あくまで `dap.configuration.xxx` や `dap.adapters.xxx` を各言語用に設定してくれるものであって debug adapter そのものではないようだ。
そのため自力で書けるのであれば nvim-dap-xxx plugin を無理に使う必要はない。
configuration に独自のものを入れ始めると、今度は adapter に手を入れたくなってくるので次第に自分で extension を書き始めるという状況になりそうな気はする。

# vscode-xxx を使っているものとそうでないものがあるのは何故か？

[nvim-dap の各言語の adapter の設定方法][3]を見ると debug adapter を VSCode 用の extension から拝借してくるように指示されたものとそうでないものがある。ものによっては両方ある。
debug adapter は vscode-xxx という名前がついていたとしても VSCode に特化したものでない限り VSCode でなくても使用することができる。Neovim なのに vscode-xxx を使えるのはそのため。
一方で Go の debugger である [ `delve` ][9] は adapter を介さず直接 DAP に対応している。このような場合、delve につなぐ設定を書くだけで nvim-dap との連携ができる。delve をインストールさえしておけば次のように書くだけで Go のプログラムを debug することができる。

```lua
local dap = require 'dap'

dap.configurations.go = {
  {
    {
      type = 'delve',
      name = 'Debug',
      request = 'launch',
      mode = 'debug',
      program = '${file}',
    },
  }
}

dap.adapters.delve = function(cb, cfg, parent)
  local uds = os.tmpname() .. '.sock'
  cb({
    type = 'pipe',
    pipe = uds,
    executable = {
      command = 'dlv',
      args = { 'dap', '-l', 'unix:' .. uds },
    },
  })
end
```

delve は DAP に対応しているし、[rdbg][2] も [GDB][4] も DAP に対応しているので最近は個別の debug adapter を介さず debugger が DAP に歩みよるケースも珍しくないようである。

nvim-dap-xxx と vscode-xxx のどちらを使うかは個人の好みで決めて良いと思うが、すでに `.vscode/launch.json` がある場合はそのまま使える(可能性が高い) VSCode 用のものを使うと考えることが少なくて良いかもしれない。

# 設定方法
## `dap.adapters`

`dap.adapters` テーブルの要素は debug adapter そのものではなく、adapter の起動方法や adapter につなぐための設定が書かれたテーブルまたは関数である。
渡された configuration の項目をもとに adapter の起動方法を変える場合など、複雑なことをしたい場合はテーブルより関数で定義すると良い。
前述のとおり `dap.adapters` の key 名は何でも良いので一つの言語を扱うときでも configuration の type を変えれば使用する adapter を変えることができる。

### adapter の設定方法

テーブルの場合 `type` として `executable`, `server`, `pipe` のいずれかを指定する。debug adapter によってそれぞれ選択すべきものが異なるので適したものを選ぶ必要がある。
debug adapter との通信方法が stdio 経由なら executable、TCP 経由なら server、unix domain socket 経由なら pipe を選択する。

関数の場合は、`function (callback, config, parent)` という interface を満たした関数を定義する。`callback` にはテーブルだけで設定した場合と同じものを渡す。
例えば、以下の2つの書き方はどちらも同じ意味になる。

```lua
dap.adapters.foo = {
  type = 'server',
  host = '127.0.0.1',
  port = 8080,
}

dap.adapters.foo = function(callback, config, parent)
  callback({
    type = 'server',
    host = '127.0.0.1',
    port = 8080,
  })
end
```

`config` にはユーザーが選択した configuration が渡ってくる。adapter の設定は一つにして config の要素によって callback に渡す設定を分岐するといったこともできる。
実際に debug adapter に config を渡す前に config を編集したい場合は後述の enrich_config を使うことで実現できる。
`parent` は親の session が渡ってくるらしいのだが私は使ったことがないので実際のユースケースはよくわかっていない。

ref: https://github.com/mfussenegger/nvim-dap/blob/0.8.0/doc/dap.txt#L180-L183

### enrich_config

どの adapter type でも使えるもので `enrich_config` という debug adapter にリクエストを投げる前に config を編集する機能がある。
全 configuration で共通して設定したいものなどは元の configuration の方に書くのではなく、`enrich_config` の時点で要素を追加・編集することで冗長性を減らすことができる。
コンパイルなどもこの `enrich_config` 関数のなかで実行することができるので debug に実行ファイルが必要な場合でも事前にコンパイルする手間を省くことができる。

```lua
dap.adapters.foo = {
  type = 'server',
  host = '127.0.0.1',
  port = 8080,
  enrich_config = function(config, on_config)
    local final_config = vim.deepcopy(config) -- 元の configuration の内容に副作用がないように deepcopy
    final_config.args = {'some', 'extra', 'args'}
    on_config(final_config)
  end
}
```

## `dap.configurations`

configuration の必須項目は `type`, `request`, `name` でそれ以外の項目は基本的には各 debug adapter 依存である。
debug adapter が対応していない独自項目も追加可能でそれらは前途の adapter の処理の中で参照して独自動作の制御に使用することができる。

### configuration の設定方法

`request` には `launch` か `attach` のいずれかを指定する。
configuration を選択した際に debuggee (debug 対象のプログラム)を起動する際は `launch` を選択し、すでに起動している場合は `attach` を選択する。
ただし、configuration の厄介なところは、request に間違ったものを選択しても普通に debug できてしまうことがあることである。
debug ができるけれども引数が渡らないなどのときは request の種別が間違っていることを疑うと良い。

項目には関数も書くことができ [nvim-dap-go][8] では[動的に args を取る](https://github.com/leoluz/nvim-dap-go/blob/5511788255c92bdd845f8d9690f88e2e0f0ff9f2/lua/dap-go.lua#L34-L42)ためにこれが使用されている。
doc には [`thread` を返す関数が書ける](https://github.com/mfussenegger/nvim-dap/blob/0.8.0/doc/dap.txt#L270-L275) と書いてあるのだが、[nvim-dap-python では vim.split の結果を返却する](https://github.com/mfussenegger/nvim-dap-python/blob/03fe9592409236b9121c03b66a682dfca15a5cac/lua/dap-python.lua#L256-L259)関数が定義されているため、thread を返す関数に現状は限定はされていないように見える。

debug adapter 依存の項目に関して何が設定できるのかわからないときは VSCode での設定方法を参照すると良い。
JSON を lua table に直せばだいたい動く。

### `.vscode/launch.json`


`load_launchjs` 関数を使うことで VSCode の `.vscode/launch.json` を取り込むことができる。

```
require('dap.ext.vscode').load_launchjs(path, type_to_filetypes}))
```
第一引数は `launch.json` のへの path でデフォルト値は `.vscode/launch.json` である。
第二引数は `launch.json` に書かれている configuration をどの言語のものとして取り込むかの対応テーブルである。
第二引数を渡さなかった場合は、configuration に `type` を filetype とみなして取り込まれる。

```json
{
    "type": "delve",
    "name": "Debug",
    "request": "launch",
    "mode": "debug",
    "program": "${file}"
}
```
上記のような設定が `launch.json` に書かれていた場合、delve は Go の debugger なので `dap.configurations.go` に設定が取り込まれてほしいが、第二引数を渡さないと `dap.configurations.delve` に取り込まれてしまう。
このようなことを回避するには以下のように `type` と filetype の紐付けを行う

```
require('dap.ext.vscode').load_launchjs(nil, { delve = {'go'}))
```

v0.8.0 の時点では人力で `load_launchjs` を使って読み込む必要があるが、現在の main branch だとファイルタイプ関係なく自動的に読み込まれるようになっているので次のリリースのタイミングではこの関数はもう使わなくて良くなると思われる。

# 各言語の設定例
## Go
### delve に直接接続する場合

```lua
-- remote debug でない debug の場合は dap-go に設定されたものを素直に使う
local dap = require 'dap'

require('dap-go').setup({
  dap_configurations = {
  },
  delve = {
    path = "dlv",
    initialize_timeout_sec = 20,
    port = "${port}", -- '${port}' と書いておくと nvim-dap 側が自動的に空いている port を使ってくれる
    args = {},
    build_flags = "",
  }
})

-- remote debug 用の configuration
table.insert(dap.configurations.go, {
  name = "Remote Debug",
  type = "remote_delve",
  request = "attach",
  mode = "remote",
  substitutePath = {
    {
      from = "/workspaces/nvim-dap-sample/go", -- host 側の path
      to = "/app"			                   -- remote 側の path
    },
  },
})

-- remote debug 用 adapter
dap.adapters.remote_delve = {
  type = "server",
  host = "127.0.0.1",
  port = 4040,
}
```

dap-go の `dap_configurations` に追加したい configuration を書くと `dap.configurations.go` に追加してくれるが、 [`type = "go"` 以外のものは追加してくれない](https://github.com/leoluz/nvim-dap-go/blob/36abe1d320cb61bfdf094d4e0fe815ef58f2302a/lua/dap-go.lua#L119-L123)ので remote debug 用の設定が欲しい場合は直接 `table.insert` で入れる必要がある。ちなみに今後も[任意の type 対応はされない雰囲気を感じる](https://github.com/leoluz/nvim-dap-go/issues/83#issuecomment-2081708865)。

### [ vscode-go ][5] を経由する場合

go の場合 vscode-go にこだわる必要はないが一応手順を残しておく。

vscode-go は [mason.nvim](https://github.com/williamboman/mason.nvim) を使っても入れられるが、mason が何やっているのかを知っておくという意味で今回は手動で入れていく。

```bash
cd ~/.local/share/${NVIM_APPNAME:-nvim}
VSCODE_GO_VERSION=0.42.1
wget https://github.com/golang/vscode-go/releases/download/v${VSCODE_GO_VERSION}/go-${VSCODE_GO_VERSION}.vsix
unzip go-${VSCODE_GO_VERSION}.vsix -d vscode-go
```

vscode-go を使った場合は adapter を remote debug 用とそうでないもので分ける必要はない。


```lua
local dap = require 'dap'

dap.adapters.vscode_go = function(callback, config)
  config.dlvToolPath = vim.fn.exepath('dlv')
  callback {
    type = "executable",
    command = "node",
    args = { vim.fn.stdpath('data') .. '/vscode-go/extension/dist/debugAdapter.js' }
  }
end

dap.configurations.go = dap.configurations.go or {}


vim.list_extend(dap.configurations.go, {
  {
    name = "vscode go",
    type = "vscode_go",
    request = "launch",
    program = "${file}",
  },
  {
    name = "remote debug vscode go",
    type = "vscode_go",
    request = "attach",
    mode = "remote",
    port = 4040,
    host = '127.0.0.1',
    apiVersion = 1,
    -- (remotePath, cmd) を設定するか substitute-path を設定する
    -- remotePath = "/app",
    -- cwd = "/workspaces/nvim-dap-sample/go",
    substitutePath = {
      {
        from = "/workspaces/nvim-dap-sample/go", -- ホスト側の path
        to = "/app"                              -- リモート側(docker とか) の path
      },
    },
  },
})
```

## Ruby

nvim-dap の wiki でも [nvim-dap-ruby][1] でも rdbg と TCP で通信する設定になっているが rdbg は Unix Domain Socket (UDS) 経由にも対応しているので UDS を使う方式で書いてみるとこんな感じ

```lua
local dap = require 'dap'

dap.adapters.ruby = function(callback, config)
  callback {
    type = "pipe",
    pipe = "${pipe}", -- '${pipe}' とすると nvim-dap が自動的に pipe (socket) を作ってくれる
    executable = {
      command = "rdbg",
      args = { "--sock-path", "${pipe}", "--open", "--command", "--", config.command, config.script },
    }
  }
end

dap.configurations.ruby = {
  {
    type = "ruby",
    name = "debug current file",
    request = "launch",
    command = "ruby",
    script = "${file}",
  },
}
```

単一の Ruby ファイルであれば adapter の設定は executable で rdbg を起動するだけで普通に debug できると思うが、Rails だとおそらく失敗する。

rdbg 使って rails を起動すると以下のように debugger のログが2つ出る
```
$ rdbg --open --nonstop --command -- rails server
DEBUGGER: Debugger can attach via UNIX domain socket (/tmp/rdbg-1000/rdbg-30519)
DEBUGGER: Debugger can attach via UNIX domain socket (/tmp/rdbg-1000/rdbg-30519)
=> Booting Puma
=> Rails 7.1.4 application starting in development 
=> Run `bin/rails server --help` for more startup options
Puma starting in single mode...
* Puma version: 6.4.3 (ruby 3.3.5-p100) ("The Eagle of Durango")
*  Min threads: 5
*  Max threads: 5
*  Environment: development
*          PID: 30519
* Listening on http://127.0.0.1:3000
* Listening on http://[::1]:3000
Use Ctrl-C to stop
```

`--nonstop` オプションを外して実験してみると実際に2回 dap server が立ち上がっているらしいことがわかる。このうち後半の方に接続したいのだが nvim-dap は1回目の方を掴んでしまって失敗しているのではないかと想像している。

![rdbg_rails](/images/nvim_dap_setting/rdbg_rails.gif)


最近だと、rails new したときの初期設定は `debugger.break` などに到達したときに debugger が有効になるという設定になっているらしいのだが、それだと editor 側で設定した nvim-dap の breakpoint に反応してくれないので nvim-dap との相性は悪くなった気がする。
https://github.com/rails/rails/pull/51692

`require: debug/prelude` を外すとあとは debug adapter に接続しに行くタイミングをずらす adapter の設定を書けば debug できるものの、人力で立ち上げたあとに attach したほうが悩むこと少なくて楽そうではある。
非同期処理の練習がてら接続タイミングをずらした処理を書いてみたものの、書き方が正しいのかはよくわからない。ChatGPT が教えてくれたコードをつないでいったら動くことには動いた。
https://github.com/goropikari/nvim-dap-rdbg/blob/09a2d8abc5b91d3fb44211387c770e980b81cf6a/lua/dap-rdbg.lua#L96-L139

VSCode だと `require: debug/prelude` を外さずとも debugger が起動してきたが私は Ruby に精通していないのでどういう仕組みなのかはよくわかっていない。

## C++

### [vscode-cpptools][6] を使う場合

mason で入れられる cpptools は古いので人力で download してくる

```bash
cd ~/.local/share/nvim/
publisher=ms-vscode
extension_name=cpptools
version=1.22.6
targetPlatform=linux-x64
curl -L http://${publisher}.gallery.vsassets.io/_apis/public/gallery/publisher/${publisher}/extension/${extension_name}/${version}/assetbyname/Microsoft.VisualStudio.Services.VSIXPackage?targetPlatform=${targetPlatform} -o cpptools.vsix
unzip -o -d cpptools cpptools.vsix
chmod +x ./cpptools/extension/debugAdapters/bin/OpenDebugAD7
```


```lua
local dap = require 'dap'

dap.adapters.cppdbg = {
  id = 'cppdbg',
  type = 'executable',
  command = vim.fn.stdpath('data') .. '/cpptools/extension/debugAdapters/bin/OpenDebugAD7',
  enrich_config = function(config, on_config)
    local final_config = vim.deepcopy(config)
    vim.fn.system({ 'g++', '-g', '-O0', vim.fn.expand('%'), '-o', vim.fn.expand('%:r') })
    on_config(final_config)
  end,
}

dap.configurations.cpp = {
  {
    name = 'Build and debug active file',
    type = 'cppdbg',
    request = 'launch',
    program = '${fileDirname}/${fileBasenameNoExtension}',
    cwd = '${fileDirname}',
  },
}
```

cpptools を使う際は事前にプログラムをコンパイルしておく必要があるので `enrich_config` を使って debug が始まる前にコンパイルを実行している。
ここでは簡単のために固定のコンパイル方法にしているが、configuration にコマンドを書いておき、それを `enrich_config` の中でとりだして使うということもできる。

事前にコンパイルする方法として下記のように configuration の項目に関数定義をすることでも実現できる。
ただし、`.vscode/launch.json` を流用したい場合はこの方法が通用しないので `enrich_config` にまとめると良いと思う。

```lua
{
  name = 'Build and debug active file',
  type = 'cppdbg',
  request = 'launch',
  program = '${fileDirname}/${fileBasenameNoExtension}',
  cwd = '${fileDirname}',
  prehook = function()
    vim.fn.system({ 'g++', '-g', '-O0', vim.fn.expand('%'), '-o', vim.fn.expand('%:r') })
  end,
},
```


### GDB を直接使う場合

```lua
local dap = require 'dap'

dap.adapters.gdbdbg = {
  type = 'executable',
  command = 'gdb',
  args = { '-i', 'dap' },
  enrich_config = function(config, on_config)
    local final_config = vim.deepcopy(config)
    vim.fn.system({ 'g++', '-g', '-O0', vim.fn.expand('%'), '-o', vim.fn.expand('%:r') })
    on_config(final_config)
  end,
}

dap.configurations.cpp = {
  {
    name = 'Build and debug active file',
    type = 'cppdbg',
    request = 'launch',
    program = '${fileDirname}/${fileBasenameNoExtension}',
  },
}
```

[1]: https://github.com/suketa/nvim-dap-ruby/tree/4176405d186a93ebec38a6344df124b1689cfcfd
[2]: https://github.com/ruby/debug
[3]: https://github.com/mfussenegger/nvim-dap/wiki/Debug-Adapter-installation
[4]: https://sourceware.org/gdb/current/onlinedocs/gdb.html/Debugger-Adapter-Protocol.html
[5]: https://github.com/golang/vscode-go
[6]: https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools
[7]: https://github.com/mfussenegger/nvim-dap
[8]: https://github.com/leoluz/nvim-dap-go
[9]: https://github.com/go-delve/delve
[10]: https://github.com/mfussenegger/nvim-dap-python
