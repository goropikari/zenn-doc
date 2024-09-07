---
title: "sqlc plugin を書いてみる"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sqlc"]
published: true
---

# 環境

| software | version | url                              |
|----------|---------|----------------------------------|
| go       | 1.23.0  | https://go.dev/                  |
| sqlc     | v1.27.0 | https://github.com/sqlc-dev/sqlc |
| buf      | 1.40.1  | https://buf.build/               |

# sqlc plugin

[sqlc の documentation](https://github.com/sqlc-dev/sqlc/blob/v1.27.0/docs/guides/plugins.md) には詳細な plugin の書き方が書かれていなかったので軽く調べてみた。
挙げられていたサンプルを見る限り process plugin, WASM plugin ともに [`GenerateRequest`](https://github.com/sqlc-dev/sqlc/blob/v1.27.0/protos/plugin/codegen.proto#L121) を標準入力から受け取って [`GenerateResponse`](https://github.com/sqlc-dev/sqlc/blob/v1.27.0/protos/plugin/codegen.proto#L130) を標準出力に出すようなプログラムを書けばよいだけのよう。

実際に作った sample program はこちら
https://github.com/goropikari/sqlc-plugin-sample


設定ファイルの `queries`, `schema` で指定した SQL の内容は `GenerateRequest.Queries`, `GenerateRequest.Catalog` の中にそれぞれ入っており、plugin option で指定したものは `GenerateRequest.PluginOptions` に json 形式で byte 列として入っているので unmarshal して情報を取ってくることができる。

ファイルへの書き出しは `GenerateResponse` の内容を解釈して sqlc 側がやってくれるので自力でファイルへ書き出す必要はない。

以下はオプション、テーブル名、カラム名を出力するだけのサンプルプログラム
```go:main.go
func Generate(req *plugin.GenerateRequest) (*plugin.GenerateResponse, error) {
	w := new(bytes.Buffer)

	opts := struct {
		Filename string
		Foo      string
		Bar      string
	}{}

	if err := json.Unmarshal(req.GetPluginOptions(), &opts); err != nil {
		return nil, err
	}

	w.WriteString(opts.Foo + "\n")
	w.WriteString(opts.Bar + "\n")
	for _, schema := range req.GetCatalog().GetSchemas() {
		for _, tb := range schema.GetTables() {
			w.WriteString(tb.GetRel().GetName() + "\n")
			for _, col := range tb.GetColumns() {
				w.WriteString("\t" + col.GetName() + "\n")
			}
		}
	}

	return &plugin.GenerateResponse{
		Files: []*plugin.File{
			{
				Name:     opts.Filename,
				Contents: w.Bytes(),
			},
		},
	}, nil
}
```


build 方法は以下のよう。
```bash
# process plugin
go build -o sample_plugin main.go

# wasm plugin
GOOS=wasip1 GOARCH=wasm go build -o sample_plugin.wasm main.go
sha256sum sample_plugin.wasm  # sha256 を調べて sqlc 設定ファイルに書く
```

`sqlc.yaml` の書き方はこんな感じ
```yaml:sqlc.yaml
version: "2"
plugins:
- name: sample_plugin_process
  process:
    cmd: ./sample_plugin
- name: sample_plugin_wasm
  wasm:
    url: file:///src/sample_plugin.wasm # inside sqlc container path
    sha256: 39927f037b03d406a50e1a2e000d14bd37757db334d67c0c5e6cae50d33381ba # sha256sum sample_plugin.wasm
sql:
- engine: "mysql"
  queries: "query.sql"
  schema: "schema.sql"
  codegen:
    - out: tutorialprocess_output
      plugin: sample_plugin_process
      options:
        filename: "process_output"
        foo: 'plugin options 1'
        bar: 'plugin options 2'
    - out: tutorial
      plugin: sample_plugin_wasm
      options:
        filename: "wasm_output"
        foo: 'plugin options 3'
        bar: 'plugin options 4'
```

あとは `sqlc generate` を実行すればファイルが生成される。

# plugin の debug 方法

plugin を書き始めてまずはじめに思うことは `GenerateRequest` の具体的な中身がよくわからないので処理を書こうにも書きづらいということだ。
構造は proto を見ればわかるものの、実際のところどんな値がどこに入ってくるのかまではさすがにわからないので実物の内容を見たくなる。一方 plugin のプログラムは sqlc から外部プロセスとして実行されるので debugger で変数確認というのもしづらい。

そんなわけで、`GenerateRequest` の中身をそのまま書き出す plugin を最初に書いてその出力されたバイナリデータを読み込むことで debug 作業をすることにした。
plugin を書くためにまず plugin を書くという不思議なことをしてしまっているが、debug するならこの方法が一番手軽な気がする。


プログラムとしては以下のような感じ
```go:main.go
func main() {
	if err := run(); err != nil {
		fmt.Fprint(os.Stderr, "error generating dump")
		os.Exit(2)
	}
}

func run() error {
	reqBlob, err := io.ReadAll(os.Stdin)
	if err != nil {
		return err
	}
	w := bufio.NewWriter(os.Stdout)
	resp := &plugin.GenerateResponse{
		Files: []*plugin.File{
			{
				Name:     "dump_proto",
				Contents: reqBlob,
			},
		},
	}
	respBlob, err := proto.Marshal(resp)
	if err != nil {
		return err
	}
	if _, err := w.Write(respBlob); err != nil {
		return err
	}
	if err := w.Flush(); err != nil {
		return err
	}
	return nil
}
```


# おわりに

当初の私のモチベーションとしてはテスト書く際に DB にデータ保存する処理が毎度必要だから `INSERT` する処理を自動生成できればと思っていたのだが、`GenerateRequest` の中身は SQL を parse した情報しかなく、Go 用のコードを生成するプログラムは sqlc の `internal` pkg に配置されていたため外部ライブラリとしても参照できずで sqlc plugin として作るのはハードルが高そうという結論に至った。(sqlc plugin じゃない方法で INSERT 文を書き出した SQL を用意してそれを sqlc に処理させたほうが楽)
