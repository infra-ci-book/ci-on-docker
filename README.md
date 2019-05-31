# インフラCI実践ガイドのDockerオンリーバージョン（Nested VM未使用）

この手順書は第1版本編のp65(3.5.3)-p66(3.5.4の途中) までの手順をNested VM未使用な環境で置き換えるものです。
その他の手順は本紙とほぼ同等です。

## 制限事項
- ホストOSとしてCentOS7を想定しています。
- 最新版(compose の 3.6 が動かせる) Docker を必要とします。
- Windows, MacOS X の Docker では動きません（ホストOSとDockerデーモンの動作する場所の違い）
  - その場合は、VirtualBox等でCentOSを起動して、その中でDockerを動かしてください。
- 各演習章の最後に登場するクリーンアップで登場する `vagrant` を使った仮想OSの再構築は実行できませんので飛ばしてください。手順に沿った演習をしている限り問題にはなりません。

## 環境構築

以下の操作を全て `root` で実施します。

Prepare the latest docker & docker-compose for CentOS7
```
yum update -y
yum install -y yum-utils device-mapper-persistent-data lvm2 git
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce

vi /usr/lib/systemd/system/docker.service
---
ExecStart=/usr/bin/dockerd -H unix://
  ↓
ExecStart=/usr/bin/dockerd --insecure-registry 192.168.33.10:4567 -H unix://
---

systemctl enable docker


# Check the latest version by https://github.com/docker/compose/releases/
VERSION=1.23.2
curl -L https://github.com/docker/compose/releases/download/${VERSION}/docker-compose-`uname -s`-`uname -m` -o /bin/docker-compose
chmod +x /bin/docker-compose


reboot
```


CI環境の起動（3-5分）
```
mkdir -p ~/prepare && cd ~/prepare
git clone https://github.com/infra-ci-book/ci-on-docker.git
cd ci-on-docker/docker/env_build/docker-compose/
docker-compose up -d
```
パスワードやトークンを変更する場合は `infra-ci/ci-on-docker/docker/env_build/docker-compose/volumes/gitlab.rb` の値を編集する。



GitLabへログイン可能になったら以下を実行（ログインはブラウザで `サーバーのIPアドレス:8080` へアクセスします）
```
RUNNER_TOKEN=token-AABBCCDD

docker exec gitlab-runner \
       gitlab-runner register \
       --non-interactive \
       --url http://192.168.33.10 \
       --registration-token ${RUNNER_TOKEN} \
       --tag-list docker \
       --executor docker \
       --locked=false \
       --docker-image docker:latest \
       --clone-url http://192.168.33.10/ \
       --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
       --docker-privileged=true \
       --docker-network-mode docker-compose_infraci_nw
```


環境全体の設定作業（書籍版の環境と合わせるための作業）

- 新規プロジェクトを作成して import project -> repo by URL から `https://github.com/infra-ci-book/ci-on-docker.git` をインポートする。
- ci-on-docker のプロジェクトページから CI/CD -> pipelines -> run pipeline からパインプラインを実行する。
- 全ての処理が成功すると本編と同じ環境に設定される。



## 演習の開始

演習を開始するには、以下のコマンドでコンソールサーバーから作業する（今回の手順ではCIホストにAnsible等が設定されず、代わりに console コンテナに設定が行われます）

```
docker exec -it -u vagrant console bash

root になる必要はありません。

```

P84 の `VAGRANT_PRIVATE_KEY` に設定する鍵の内容は紙面と同じ `cat ~/.ssh/infraci` で参照でいます。


## 環境の再起動等


CI演習ホスト上で以下を実行
```
cd ci-on-docker/docker/env_build/docker-compose/

# 停止
docker-compose stop

# 開始
docker-compose start

# 削除（やり直し)
docker-compose down
```
