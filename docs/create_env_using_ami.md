# AMIを用いた環境構築メモ

## 概要

[https://github.com/matsuu/aws-isucon](https://github.com/matsuu/aws-isucon)

- 上記のリポジトリに掲載されているAMIを用いて、環境構築した際のメモ
- ISUCON5からISUCON12の環境が再現

## 環境構築の説明

### カスタムAMIからEC2インスタンスを作成する

下記のページのAMIのリンクを踏んでAWSをログインして、EC2インスタンスを作成する。
[https://github.com/matsuu/aws-isucon#ami](https://github.com/matsuu/aws-isucon#ami)

[ISUCON11予選の場合](https://console.aws.amazon.com/ec2/home?region=ap-northeast-1#ImageDetails:imageId=ami-0796be4f4814fc3d5)

参考
https://repost.aws/ja/knowledge-center/launch-instance-custom-ami

インスンス設定は下記のようにして起動する。(言及していない所はデフォルトのまま)

- インスタンスタイプ: 本番に合わせるなら`C5.large`、安く済ませるなら`t2.micro`
- キーペア: 持っていないなら作成する
  - 作成したキー(key_name.pem)は、`~/.ssh/`に保存して、`chmod 400 key_name.pem`で権限を変更する
  - SSH接続する際に使用する

### SSH接続

~/.ssh/configに下記の設定を追記する。

```bash
Host 適当な名前(例:isucon11)
  HostName インスタンスのパブリックIPv4 DNS(ec2-35-72-8-225.ap-northeast-1.compute.amazonaws.com)
  IdentityFile ~/.ssh/key_name.pem
  User ユーザー名(READMEに書いてある。isucon11予選の場合はubuntu)
```

### ベンチマークの実行

基本的にREADMEに書いてある通りに実行する。
[ISCUON11の場合](https://github.com/matsuu/aws-isucon/tree/main/isucon11-qualify#bench)
