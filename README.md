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
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
  ↓
ExecStart=/usr/bin/dockerd --insecure-registry 192.168.33.10:4567 -H fd:// --containerd=/run/containerd/containerd.sock
---

systemctl enable docker


# Check the latest version by https://github.com/docker/compose/releases/
VERSION=1.29.1
curl -L https://github.com/docker/compose/releases/download/${VERSION}/docker-compose-`uname -s`-`uname -m` -o /bin/docker-compose
chmod +x /bin/docker-compose


reboot
```


CI環境の起動
```
mkdir -p ~/prepare && cd ~/prepare
git clone https://github.com/infra-ci-book/ci-on-docker.git
cd ci-on-docker/docker/env_build/docker-compose/
```

パスワードやトークンを変更する場合は `volumes/gitlab.rb` の値を編集する。その後 compose を起動(3-5分かかります)
```
docker-compose up -d
```


GitLabへログイン可能になったら以下を実行（ログインはブラウザで `サーバーのIPアドレス:8080` へアクセスします）

`RUNNER_TOKEN` には `volumes/gitlab.rg` で設定した値を使います。
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

成功時の出力例
``` 
Running in system-mode.

Registering runner... succeeded                     runner=token-AA
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```


環境全体の設定作業（書籍版の環境と合わせるための作業）

- gitlab にログインする。ユーザーは `root` パスワードは `volumes/gitlab.rb` で設定した値。
- 新規プロジェクトを作成して import project -> repo by URL から `https://github.com/infra-ci-book/ci-on-docker.git` をインポートする。
- 作成されたプロジェクト `ci-on-docker` のプロジェクトページから CI/CD -> pipelines -> run pipeline からパインプラインを実行する。
- 全ての処理が成功すると本編と同じ環境に設定される。



## 演習の開始

演習を開始するには、以下のコマンドでコンソールサーバーへ接続してから行います（今回の手順ではCIホストにAnsible等が設定されず、代わりに console コンテナに設定が行われます）

```
docker exec -it -u vagrant console bash
```

P84 の `VAGRANT_PRIVATE_KEY` に設定する鍵の内容は紙面と同じ `cat ~/.ssh/infraci` で参照でいます。

## 本編との差分

本編ではホストマシンから vagrant ssh コマンドを利用してサーバーにログインする操作が含まれています。コンテナ環境を用いた場合は vagrant コマンドが利用できないため、代わりにホストマシンから docker exec コマンドか、コンソールから ssh コマンドを利用してください。


## TIPs

本編のパイプラインは毎回コンテナのビルドが走るため、負荷が高く時間もかかります。.gitlab-ci.yml を`Unit_Package`を以下のように編集することで初回のみビルドが走るように変更できます。

```yaml
Unit_Package:
  stage: unit_prepare
  script:
    - |
        IMAGE_CHECK=`docker images -q ${CONTAINER_IMAGE_PATH}`
        if [ "${IMAGE_CHECK}" = "" ]; then
            docker login -u gitlab-ci-token -p ${CI_BUILD_TOKEN} ${CI_REGISTRY}
            docker build . -t ${CONTAINER_IMAGE_PATH}
            docker push ${CONTAINER_IMAGE_PATH}
        fi
  tags:
    - docker

```

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

## インフラCIのデモとしてこの環境を使う場合

1. 上記の「環境構築」を実施する
2. GitLabへユーザーを追加する
   - 「user」を「Regular」権限で追加（その他はデフォルト）
3. プロジェクトの作成 → インポートを行う
   - インポート元: https://github.com/infra-ci-book/ketchup-vagrant-ansible.git
   - Visibility Level を 「Public」に設定
4. 「ketchup-vagrant-ansible 」プロジェクトに「user」を「developer」権限で追加する
5. プロジェクトのSettings -> CI/CD -> Secret Variables に変数を追加
   - 変数名に「VAGRANT_PRIVATE_KEY」を追加
   - 変数の内容は「~/.ssh/infraci」を設定する
     - 確認方法
     - `docker exec -it -u vagrant console bash`
     - `cat ~/.ssh/infraci`
6. 上記の「TIPS」を実施し、2回目以降はビルドが走らないようにする
7. pipeline を一回動かしてみる

