Bootable Container Image とは ~ プラットフォームをコンテナイメージとして提供する ~
---

2024 年 5 月 6 日 ~ 5 月 9 日（米国時間）に行われた Red Hat Summit 2024 にて、RHEL の新しいデプロイ方法として [Image mode for Red Hat Enterprise Linux](https://www.redhat.com/ja/technologies/linux-platforms/enterprise-linux/image-mode) が発表されました。

これにはどうやら Bootable Container Image という技術が使われているようなのですが、Bootable Container Image ってなんだ...？となったので Image mode と併せて調べてみました。


## 概要

[Bootable Container Image](https://containers.github.io/bootable/) は、その名のとおりブート可能なコンテナイメージです。  
通常のコンテナイメージと比較しながら、その特徴を見てみましょう。

### 通常のコンテナイメージ

- コンテナに対応した OS の上で実行することを前提としており、最低限のものしか含んでいない（つまり、それ単独で実行することはできない）
- フォーマットは OCI によって標準化されている

### Bootable Container Image

- カーネルやデーモンなど、単独で実行するための OS としての機能が含められている
- ベアメタルサーバや仮想マシンなどでブート可能
- OCI のフォーマットに従っている

通常のコンテナと異なるのは、OSとしての機能がコンテナイメージに含まれている点です。（コンテナイメージに含められているのはファイルシステムを再現するための情報のみなので、OS として起動するためには初期構築時のみディスクイメージに変換する作業が必要になります）

また、`OCI のフォーマットに従って` おり、まるで通常のコンテナイメージのように扱うことができるのも大きな特徴ではないでしょうか。  
つまり、コンテナレジストリの利用はもちろん、テストや GitOps ワークフローなどの標準的なコンテナプラクティスやツールなどを利用して Linux システムを構築することができます。


## 使い方 (構築・運用フロー)

次に、Bootable Container Imageの構築から運用までのフローを見ていきましょう。
<div style="text-align:center">
  <img src="https://github.com/hyorimitsu/tech-talk/blob/main/2024/0529_BootableContainerImage/img/bootable-container-image-flow.png?raw=true" alt="bootable-container-image-flow" /><br>
  <a href="https://developers.redhat.com/products/rhel-image-mode/overview" target="_blank">https://developers.redhat.com/products/rhel-image-mode/overview</a> より引用
</div><br>

フローは上図のような形になっています。順番に見ていきましょう。

### 1. Build

まずは、コンテナイメージの構築です。基本的には通常のコンテナイメージと同様に以下の手順で行います。

#### 1.1. Containerfile (または Dockerfile) の作成

以下のような形で Containerfile (または Dockerfile) を作成します。  

```dockerfile
FROM quay.io/fedora/fedora-bootc:40

RUN dnf -y install [software] [dependencies] && dnf clean all
# e.g.) RUN dnf install -y nodejs postgresql16-server && dnf clean all

ADD [application]
ADD [configuration files]

RUN [config scripts]
# e.g.) RUN systemctl enable postgresql-16.service
```

ベースイメージで指定できるものとしては以下のとおりです。

- quay.io/centos-bootc/centos-bootc:stream9
- quay.io/fedora/fedora-bootc:40
- registry.redhat.io/rhel9/rhel-bootc:9.4
- ...

通常の Dockerfile の場合のベストプラクティスとして、用途ごとにコンテナを分ける（アプリケーションを複数のコンテナに切り離す）というものがありますが、Bootable Container Image の場合はサーバとしての姿を記述しているのが特徴です。

#### 1.2. Containerfile (または Dockerfile) のビルド

以下のコマンドにて Containerfile (または Dockerfile) をビルドします。

```shell
$ sudo podman build -f Dockerfile -t ${CONTAINER_REGISTRY}:latest
```

#### 1.3. 動作確認

概要で記述したとおり、Bootable Container Image は通常のコンテナイメージと同様に扱うことができるため、以下のようにして実行することができます。

```shell
$ sudo podman run -d --rm --name ${APP_NAME} -p 3000:3000 ${CONTAINER_REGISTRY}:latest /sbin/init
```

もちろん localhost でアクセスできます。

```shell
$ curl -IL localhost:3000
HTTP/1.1 200 OK
Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Accept-Encoding
link: </_next/static/media/e11418ac562b8ac1-s.p.woff2>; rel=preload; as="font"; crossorigin=""; type="font/woff2"
X-Powered-By: Next.js
Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
Content-Type: text/html; charset=utf-8
Date: Tue, 28 May 2024 10:46:33 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

※ Bootable Container Image として実行した場合と通常のコンテナとして実行した場合とで動作が異なる部分があるので注意が必要です。例えば、Bootable Container Image として実行した場合には `ENTRYPOINT` や `CMD`, `EXPOSE` などが無視され、通常のコンテナとして実行した場合には `/usr/lib/modules/$kver/vmlinuz` の Linux カーネルバイナリが無視されます。（その他詳細は[こちら](https://containers.github.io/bootc/building/bootc-runtime.html)）

#### 1.4. コンテナレジストリへプッシュ

ビルドしたコンテナイメージを以下のコマンドでコンテナレジストリへプッシュします。

```shell
$ sudo podman push ${CONTAINER_REGISTRY}:latest
```

### 2. Deploy

次に、Build セクションで作成したコンテナイメージをもとにサーバを構築します。

#### 2.1. コンテナレジストリからプル

以下のコマンドで、コンテナイメージを取得します。

```shell
$ sudo podman image pull ${CONTAINER_REGISTRY}:latest
```

#### 2.2. コンテナイメージをディスクイメージへ変換

概要で記述したとおり、コンテナイメージに入っているのはファイルシステムを再現するための情報のみなので、ディスクイメージへ変換し、OS 起動に必要な仕込みを行う必要があります。

ここでは [bootc-image-builder](https://github.com/osbuild/bootc-image-builder) というツールを使って変換をします。

コマンド例は以下のとおりです。

```shell
$ sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    -v $(pwd)/config.json:/config.json \  # 追加するユーザ情報
    -v $(pwd)/output:/output \            # イメージの出力先
    quay.io/centos-bootc/bootc-image-builder:latest \
    --type qcow2 \                        # 出力するイメージの形式
    --local \
    --rootfs xfs \
    --log-level info \
    ${CONTAINER_REGISTRY}:latest
```

##### 2.2.1 ユーザの追加

volume のマウントで指定している `config.json` (または toml でも可) には追加するユーザの情報を記述することができ、例えば以下のように記述することができます。

```json
{
  "blueprint": {
    "customizations": {
      "user": [
        {
          "name": "user",
          "password": "password",
          "key": "ssh-rsa AAAAB3...",
          "groups": ["wheel"]
        }
      ]
    }
  }
}
```

それぞれのフィールドの説明は以下のとおりです。

| フィールド名 | 説明 |
| --- | --- |
| name | ユーザ名 |
| password | 暗号化していないパスワード |
| key | SSH 公開キーの中身 |
| groups | ユーザが属するグループ |

##### 2.2.2 イメージ形式の指定

`type` で出力するイメージの形式を指定することができます。  
対応している形式は以下のとおりです。

- ami
- qcow2
- vmdk
- anaconda-iso
- raw

#### 2.3. イメージからマシンを起動

作成したイメージをもとにマシンを起動します。

### 3. Manage

Build, Deploy セクションでアプリケーションの初期構築はできました。  
最後に、アプリケーションのアップデート手順です。

手順は以下のとおりです。

1. Containerfile (または Dockerfile) の更新
2. Containerfile (または Dockerfile) のビルド
3. コンテナレジストリへプッシュ

これだけで完了です。

systemd の timer によって定期的にイメージの更新状態のチェックが行われるようになっており、更新があれば自動的にイメージをプルして適用する仕組みになっています。  
もちろん、この機構を停止して任意のタイミングでアップデートすることも可能です。

### 4. 補足

上記で一通りのフローを紹介していますが、[Red Hat の方のリポジトリ](https://github.com/kubealex/bootable-container-images)にいくつかのユースケースとサンプルが上げられているので、こちらも見てみるとよりイメージが付きやすいかもしれません。


## まとめ

Bootable Container Image によって、OS レベルでアプリケーションをコンテナイメージとして管理できるようになります。もしかしたら、本番、開発、テスト環境の差分を減らしたり、既存のコンテナスキャンツールと併せて使うことでスキャンの範囲を OS レベルにまで広げたりすることができる未来があるかもしれません。

また、今ではコンテナ技術はアプリケーション開発に欠かせない存在になっているので、今後の発展次第ではその波が OS にまで広がるかも？と考えると夢が広がります。

弊社では AWS の場合は ECS (Fargate) を、Google Cloud の場合は Cloud Run を使うことが比較的多いですが、お客様の要件次第では EC2 や Compute Engine を利用することもありますので、Bootable Container Image の今後の発展にも注目していきたいと思います。


## 参考文献

- [Bootable Container Images](https://containers.github.io/bootable/)
- [bootc](https://containers.github.io/bootc/intro.html)
- [Image mode for Red Hat Enterprise Linux - Red Hat Developer](https://developers.redhat.com/products/rhel-image-mode/overview)
- [osbuild/bootc-image-builder - GitHub](https://github.com/osbuild/bootc-image-builder)
- [kubealex/bootable-container-images - GitHub](https://github.com/kubealex/bootable-container-images)
- [Image mode for Red Hat Enterprise Linuxの中身を見てみる - 赤帽エンジニアブログ](https://rheb.hatenablog.com/entry/try-rhel-image-mode)
- [コンテナイメージなのにブート可能な新技術による「Image mode for Red Hat Enterprise Linux」、Red Hatが発表。レジストリなどのコンテナ関連ツールがそのまま利用可能 - Publickey](https://www.publickey1.jp/blog/24/image_mode_for_red_hat_enterprise_linuxred_hat.html)
