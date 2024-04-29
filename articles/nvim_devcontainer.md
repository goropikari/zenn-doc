---
title: "Neovim & devcontainer cli での運用方法を模索する"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [neovim, devcontainer]
published: true
---


先に結論

- `devcontainer up` のオプションで sshd の devcontainer feature を追加しコンテナへは ssh で接続する
- ssh したあとに Neovim はコンテナの中で起動する

とするのが VSCode user との環境差分を最小限に抑えられ、今後のメンテも楽という結論に至った。

![demo](https://github.com/goropikari/local-devcontainer.nvim/blob/main/docs/demo.gif?raw=true)

このでも動画では Neovim から操作して devcontainer を立ち上げ、ターミナル(wezterm)に新規タブをつくってそちらでコンテナの中に ssh で入るまでを行っている。内部的にはシェルスクリプトを呼び出しているだけなので Neovim から操作する必然性は正直ない。

https://github.com/goropikari/local-devcontainer.nvim


# はじめに

VSCode で使うことを想定した devcontainer を VSCode を使わず devcontainer cli を使う方法で環境構築し直していく方法を模索した。
自分の都合の良いように `devcontainer.json` を編集できる場合は自由に何でもできるものの、チーム開発で使っているものだと特定の個人用の設定を入れるのは憚られるので既存の devcontainer 周りのファイルを編集することなく devcontainer cli を使う方法に移行する術の確立を目指した。
また最終的にエディタは Neovim を使うので Neovim の起動方法についても考えた。

注意: GitHub へは ssh で接続することを前提としている。また docker daemon はホストマシン上で動いているものとする。

# 動作確認環境

Host OS は ArchLinux を使用。その他のソフトウェアは以下の通り。
```
~ $ node --version
v21.7.3

~ $ devcontainer --version
0.58.0

~ $ docker version
Client:
 Version:           26.0.1
 API version:       1.45
 Go version:        go1.22.2
 Git commit:        d260a54c81
 Built:             Fri Apr 12 06:20:40 2024
 OS/Arch:           linux/amd64
 Context:           default

Server:
 Engine:
  Version:          26.0.1
  API version:      1.45 (minimum version 1.24)
  Go version:       go1.22.2
  Git commit:       60b9add796
  Built:            Fri Apr 12 06:20:40 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.7.15
  GitCommit:        926c9586fe4a6236699318391cd44976a98e31f1.m
 runc:
  Version:          1.1.12
  GitCommit:
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

~ $ docker compose version
Docker Compose version 2.26.1
```

# devcontainer の起動方法について

公式の [devcontainers/cli](https://github.com/devcontainers/cli) を使うか [open-devcontainer](https://gitlab.com/smoores/open-devcontainer) や Neovim であれば [nvim-remote-containers](https://github.com/jamestthompson3/nvim-remote-containers) といったサードパーティ製のものを使う方法があるが今回は公式の devcontainers/cli を使うことにした。


VSCode を使わずに devcontainer を使う人口はそもそも少ないので一番情報が溜まりそう、かつ今後も新機能がついたときの追従がされることを期待すると公式のものを使うのが無難と考えた。

open-devcontainer は公式の方ではまだ対応していない ssh agent forward に対応しているのが魅力的であるが後述の代替方法で我慢することにした。


# 設定を上書きする


`devcontainer up` 時に設定の編集ができるオプションとして主に以下が使える。

```
--override-config
--mount
--remote-env
--additional-features
```

`--override-config` は既存の `.devcontainer/devcontainer.json` の特定の項目を上書きしたり追加したりするのではなく、完全に置き換わっているような挙動に見える。
サンプルがないため想定動作の正解がわからないが、`.devcontainer/devcontainer.json` に features を書き、上書き用の json に features を書かなかったら features で指定したものが入ってこなかった。
`--config` だとファイル名を `devcontainer.json` 以外にするとエラーが出るが、`--override-config` の場合はどんなファイル名でも受け入れてくれて `--config` の完全な上位互換のようにも思える。

一方で他のオプションは既存の置き換えではなく追加で設定される。

devcontainer cli で使う用のファイルを用意するとそのファイルをどこで管理するかという問題が出てくるので、 `--mount`, `--remote-env`, `--additional-features` で上書きできない項目がある場合のみ `--override-config` を使うようにするのが良さそう。


既存の Dockerfile を編集せずに image build した時点で自分にとって必要なソフトウェアをインストールするための工夫として私の場合は自前で devcontainer features も作っている。
dotfiles を入れる段階でソフトウェアインストールする方法もあるが、コンテナの生成・破壊を何度も繰り返すような場合には image に初めから入れ込める devcontainer features が便利だと思う。

https://github.com/goropikari/devcontainer-feature


## repository が `Clone Repository in Container Volume...` を使う前提だった場合

現在の devcontainer cli は `Clone Repository in Container Volume...` に対応していない。
このとき、devcontainer.json が Dockerfile や image を直指定しているものであった場合は devcontainer cli で立ち上げるとローカルのファイルが bind mount されるものの、docker-compose.yml を使っている場合は mount されないので、`--mount` オプションで明示的に mount させる必要がある。


実行例
```bash
devcontainer up --workspace-folder=. \
  --mount "type=bind,source=$(pwd),target=/workspaces/$(basename $(pwd))"
```


# コンテナの中での ssh をどうするか

VSCode の場合、[Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) でも [Docker extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker) でもどちらでも良いが、コンテナを VSCode に attach するとホストの ssh agent がコンテナに転送される。
そのため、特に設定せずともコンテナ内で GitHub へ ssh で接続できる。

container attach 時の log から抜粋
```
[1170 ms] Start: Launching Dev Containers helper.
[1170 ms] ssh-agent: SSH_AUTH_SOCK in container (/tmp/vscode-ssh-auth-d2334073-1c3e-4cd2-b909-beaee358ee5d.sock) forwarded to local host (/tmp/ssh-XXXXXXhIVllL/agent.7391).
```

一方で devcontainer cli だと ssh agent forward されないのでコンテナ内で ssh で接続できない。

アプリケーションビルド時に社内の private repository からライブラリを引っ張ってきたいこともあるため、コンテナ内で ssh が使えないのは不便極まりない。
そのため ssh が使えるようにどうにかする必要がある。

今回は ssh を使えるようにするために
- sshd
- socket を bind mount
- socat でリレーする

の3つを試した。


## コンテナの中で sshd を起動して接続する

```bash
# features で sshd を入れる
containerID=$(devcontainer up --workspace-folder=. --additional-features='{"ghcr.io/devcontainers/features/sshd:1": {}}' | tail -n1 | jq -r .containerId)

# コンテナ内に公開鍵を置く
docker exec -u vscode $containerID bash -c 'mkdir -p /home/vscode/.ssh'
docker cp ~/.ssh/id_rsa.pub $containerID:/home/vscode/.ssh/authorized_keys
devcontainer exec --workspace-folder=. bash -c 'chmod 644 /home/vscode/.ssh/authorized_keys'
devcontainer exec --workspace-folder=. bash -c 'chmod 700 /home/vscode/.ssh'

# ssh でコンテナに接続する
# docker-compose.yml を使っていると devcontainer cli では port forward を増やせないのでコンテナの ip address を docker inspect で調べて直接つなぎに行く
containerHostname=$(docker inspect $containerID --format='{{.Config.Hostname}}')
ipAddress=$(docker inspect $containerHostname --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')

ssh -t -i ~/.ssh/id_rsa -o NoHostAuthenticationForLocalhost=yes -o UserKnownHostsFile=/dev/null -o GlobalKnownHostsFile=/dev/null -p 2222 vscode@$ipAddress

# コンテナ内で GitHub に接続できるか確認
ssh -T git@github.com
```

この方法だと `postCreateCommand` などで private repository から clone してくる処理が入っていた場合にコンテナ起動でエラーが出てしまう。だが `postCreateCommand` で実行しているコマンドはコンテナ内に入ってあとから人力で実行もできると思われるので、自動的に `postCreateCommand` が実行されることに相当こだわりがある場合を除き運用でカバーできると思われる。
運用でカバーが許容できる場合は `--skip-post-create` オプションをつけて `devcontainer up` をすると良い。



## SSH_AUTH_SOCK を bind mount
### devcontainer cli `--mount` オプション使う

とりあえずこんな感じで動くことには動く
```bash
id=$(devcontainer up --workspace-folder=. --mount "type=bind,source=$SSH_AUTH_SOCK,target=/tmp/ssh_auth_sock" | tail -n1 | jq -r .containerId)
docker exec -it -e SSH_AUTH_SOCK=/tmp/ssh_auth_sock $id bash

# コンテナ内で GitHub に接続できるか確認
$ ssh -T git@github.com
Hi goropikari! You've successfully authenticated, but GitHub does not provide shell access.
```


一方で、ホスト側の SSH_AUTH_SOCK は `/tmp` 配下にファイル名がランダムで作られるので(ssh agent よって変わる) PC を再起動すると以前の socket file がなくなって bind mount できなくなり、stop していたコンテナを start させる段階でファイルがないため起動エラーになる。

```bash
$ echo $SSH_AUTH_SOCK
/tmp/ssh-XXXXXXhIVllL/agent.7391
```

コンテナ起動前に名前が固定になるように事前に link を張っておけばとりあえずエラーは回避できる。

```bash
AUTH_SOCK=/tmp/ssh_auth_sock
ln -f $SSH_AUTH_SOCK $AUTH_SOCK
id=$(devcontainer up --workspace-folder=. --mount "type=bind,source=$AUTH_SOCK,target=$AUTH_SOCK" | tail -n1 | jq -r .containerId)
docker exec -it -e SSH_AUTH_SOCK=$AUTH_SOCK $id bash
```

`postCreateCommand` で private repository を clone してくる処理がある場合はこの方法が選択肢に入ってくると思う。


### 人力

ファイルがなくて mount できずにエラーになるのは docker の機能を使って mount していることが原因なので人力で mount してエラーを回避するという手もあることにはある。
ここでは Docker の storage driver として Overlay2 を使っているとする。

```bash
id=$(devcontainer up --workspace-folder=. --skip-post-create | tail -n1 | jq -r .containerId)
devcontainer exec --workspace-folder=. touch /tmp/ssh_auth_sock
sudo mount --bind $SSH_AUTH_SOCK $(docker inspect $id --format={{.GraphDriver.Data.MergedDir}})/tmp/ssh_auth_sock
docker exec -it -e SSH_AUTH_SOCK=/tmp/ssh_auth_sock $id bash
```

一応この方法でもコンテナ内で ssh できることにはできるが公式 Doc には `/var/lib/docker` 配下のファイルを直接編集するなと書かれているのでお行儀の良い方法ではない。

> Don't directly manipulate any files or directories within /var/lib/docker/. These files and directories are managed by Docker.

https://docs.docker.com/storage/storagedriver/overlayfs-driver/

またこの方法はコンテナが出来上がったあとに mount しているので ` postCreateCommand` で private repository は触れない。

## socat でつなぐ

socat を使って通信の転送を繰り返しても今回のやりたいことは達成できる。
ただし、ホスト・コンテナの両方で socat が使える必要がある。

```bash
# コンテナ内の /tmp/test_ssh.sock に来たリクエストをホスト側の port 60000 へ転送
id=$(devcontainer up --workspace-folder=. --additional-features='{"ghcr.io/goropikari/devcontainer-feature/socat:1": {}}' | tail -n1 | jq -r .containerId)
gateway=$(docker inspect $id --format='{{range .NetworkSettings.Networks}}{{.Gateway}}{{end}}')
docker exec $id bash -c "socat unix-listen:/tmp/test_ssh.sock,fork tcp-connect:$gateway:60000 &"

# port 60000 に来たリクエストを SSH_AUTH_SOCK に転送
socat ${SSH_AUTH_SOCK} tcp-listen:60000,fork &

# コンテナ内で ssh で接続できることを確認
docker exec -it -e SSH_AUTH_SOCK=/tmp/test_ssh.sock $id bash
ssh -T git@github.com
```

上記の例では devcontainer を使っているが普通の起動中のコンテナであっても socat さえ使えればコンテナ内で ssh 接続できる環境を構築することができるため汎用性は一番高そうである。
昔の devcontainer の document にはホスト側に socat を入れるように指示があったため似たようなことをしていたのかもしれない。


# Neovim
## 起動方法


container の中で Neovim を使う方法として考えたのは以下の3つ
- docker exec でしてから Neovim 起動
- Neovim server をコンテナ内で立ててホスト側からつなぎに行く
- ssh でコンテナに接続して Neovim 起動


### docker exec
ssh agent forward されないので前出のいずれかの方法でコンテナの中で ssh 接続できる環境を整える必要がある。

### [neovim remote](https://github.com/neovim/neovim/blob/v0.9.5/runtime/doc/remote.txt)
コンテナの中で nvim server を立てて、client 側で `--remote-ui` オプションでつなげた場合、client 側で `:q` するともれなくコンテナの中の Neovim も落ちてしまう。
terminal を落とすと server は落ちないものの手癖で `:q` してしまうのでその場合 server を立て直す手間がかかるのが難点。
Neovim server も結局 docker exec で起動するだろうからそれであれば最初から server/client の形態取らず直接編集すれば良いだろうと気付いた。
ssh の問題も工夫する必要がある。

Vim では使えて Neovim では使えない [`remote_foreground`](https://vim-jp.org/vimdoc-ja/remote.html) が使えるようになれば気軽にコンテナ内の Neovim とホスト側の Neovim の編集をシームレスに切り替えられるようにできそうな気がするのでこれが使えるようになったら Neovim Remote 運用も考えたい。



### ssh でコンテナに接続
公開鍵を登録する必要があるが、自然に ssh agent forward できるので一番素直な方法だと思う。

すでに起動中のコンテナで `sshd` を立ち上げるのは面倒なのでその場合は他の方法を取ったほうが楽そうだけれども、devcontainer の場合は立ち上げ時に [sshd の devcontainer feature](https://github.com/devcontainers/features/tree/bb7b7ea29f84ea31262c1290a2e14e9984286295/src/sshd) を指定すれば自動的に sshd の設定が終わるので楽ができる。


上記3つの方法を実際に試してみてコンテナに ssh で入って Neovim を起動する方法が一番運用管理コストが低い方法だと思った。

## ホストマシンとの clipboard 共有

https://github.com/ojroques/nvim-osc52 を使えば OSC52 に対応したターミナルエミュレータと組み合わせればコンテナ内に専用のサーバーを立てずともコンテナ内でコピーしたものをホスト側に持ってくることができる。

Neovim 0.10 からはプラグインを入れずとも公式で OSC52 に対応するらしい。

# おわり

いろいろ試した結果一番工夫のない方法に落ち着いた気がするが、devcontainer cli は枯れたツールではないので今後の機能追従のことを考えると一番工夫しない方法がメンテナンスコストをかけずに運用するにはよさそうという結論に至った。


# 参考
- https://blog.bridgey.dev/2023/02/19/neovim-devcontainer/
- https://zenn.dev/mai/articles/3fc3418871c85d
- https://github.com/devcontainers/cli/tree/c1c8b08263c6dca7cd79c97a2d0bc581fcef4f6c/example-usage/tool-vim-via-ssh
