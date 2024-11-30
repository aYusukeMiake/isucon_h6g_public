# 当日のTODO

## 事前準備

- チーム用のプライベートリポジトリをあらかじめ作成しておく
- ISUCONポータルはあらかじめ開いておく(始まってから開くと重い)

## 開始直後にやること

- インスタンスの立ち上げ
- SSH接続
- パプリックIPの共有
- 動画を撮りながらアプリケーションの動作確認
- ベンチマークの実行
- レギュレーションの確認
- メインで編集することになるフォルダ(例年ならば`webapp`)の中身を確認
- 言語をRubyに切り替える

## Gitの設定

- (`開始時に行う設定.md`の「Gitの設定」を参照しながら設定)
- メインで編集することになるフォルダで`git init`をして事前準備で用意したリポジトリにpush
  - `git add .`と`git commit -m "Initial commit"`
  - `git remote add origin [リポジトリのURL]`
  - `git push -u origin master`
- ミドルウェア設定ファイルをGit管理下におく
  - リンクを作成して、Git管理下に置く
  - 手順については`開始時に行う設定.md`の「設定をGit管理下に置く」を参照
  - 同様の手順でOS設定ファイル(/etc/sysctl.conf)やアプリの起動ファイルも必要に応じてGit管理下に置く

## 他のサーバーも同様の状況にする

例年であれば、サーバーが3台あるので、他のサーバーも同様の状況にする。
なお、各アクションごとにベンチマークをそちらで実行する。

- ベンチマークの対象を、それらのサーバーに変更して実行する
- 元々あったディレクトリ(`webapp`)を`webapp-old`にリネーム
- `git clone`
- 設定ファイルのリンクを作成
- 実際にそちらでベンチマークを実行する

## 事前に用意した設定を反映

- 事前に用意した設定を反映する
  - `mysqld.cnf`
  - `nginx.conf`
  - `sysctl.conf`
  - `Makefile`
- 手順については`開始時に行う設定.md`の「事前に用意した設定を反映」を参照
- Makefileの冒頭の環境変数を設定して、各インストールコマンドを実行
  - install-tools
  - install-alp
  - install-query-digest
- 事前に用意した.vscodeを`/home/isucon`において、拡張機能を入れる

## 各ログを確認

`Makefile`のログを取るためのコマンドを順に実行して動作確認する。

- 動作確認したいものは下記のとおり
  - アクセスログの解析: alpを使う
    - `make check-access-log`が実行できること
  - CPU使用率の解析: dstatを使う
    - `make check-top-cpu`が実行できること
  - MySQLのクエリ解析: mysqldumpslow、query-digesterを使う
    - `make check-slow-log`が実行できること
    - `make check-query-digest`が実行できること
- 実行前に環境変数が使われているかを確認し、必要に応じて`Makefile`の冒頭の環境変数のパスを修正
- alp_config.ymlの設定方法は`isucon_share/docs/ログの取り方.md`を参照

## プロファイラの導入(後回しでも可)

- 改善方法が思いつかず、詰まったときのためにEstackprofを導入しておく。
- 詳しくは`開始時に行う設定.md`の「Rubyプロファイラ(Estackprof)の導入と使い方」を参照

## 終了前の確認

- 再起動して問題なく動作するかの確認
- 再起動せずに2回連続でベンチマークを実行しても動作するかの確認
- ログを切る
  - nginxのアクセスログ
  - MySQLのslow query log
  - estackprofのログ
