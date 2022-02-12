# Fedora CoreOSのインストール

## 手順

### ISO イメージのダウンロード

[Download Fedora CoreOS](https://getfedora.org/ja/coreos/)からダウンロードする。

### 仮想マシンの登録

スペックは以下の通りとする。今回は、自宅ラボの ESXi 7 で実行する。

- コア割り当て：6 コア
- メモリ：24 GB
- ストレージ：128 GB

### 簡易 Web サーバの立ち上げ

Fedora CoreOS のインストール時に使う Ignition ファイルをホストするために WEB サーバを立ち上げる。

```shell
mkdir ~/fcos_work/
cd ~/fcos_work/
python3 -m http.server
Serving HTTP on :: port 8000 (http://[::]:8000/) ...
```

この状態で`curl http://localhost:8000/`を実行して以下のようなレスポンスが返ってくることを確認する。

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title>Directory listing for /</title>
  </head>
  <body>
    <h1>Directory listing for /</h1>
    <hr />
    <ul></ul>
    <hr />
  </body>
</html>
```

### Ignition ファイルの作成

`buファイル`を`Ignitionファイル`に変換する流れで`Ignitionファイル`を作成するので、まずは`buファイル`を作成する。

#### install-config.bu

[System Configuration](https://docs.fedoraproject.org/en-US/fedora-coreos/)を参考に、最低限の内容を記載する。

```yaml
variant: fcos
version: 1.4.0
passwd:
  users:
    - name: sonohen
      groups:
        - docker
        - sudo
      ssh_authorized_keys:
        - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJw8VQlDD10wx3PPuC8l8T9mhYmDOikMicWhgH6JkAd1 sonohen@mbpro.local
storage:
  files:
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: container-server
    - path: /etc/NetworkManager/system-connections/ens192.nmconnection
      mode: 0600
      contents:
        inline: |
          [connection]
          id=ens192
          type=ethernet
          interface-name=ens192
          [ipv4]
          address1=192.168.1.11/24,192.168.1.1
          dhcp-hostname=container-server
          dns=192.168.1.1;
          dns-search=
          may-fail=false
          method=manual
  links:
    - path: /etc/localtime
      target: ../usr/share/zoneinfo/Asia/Tokyo
```

### Ignition ファイルへの変換

コンテナを使って変換する。

```shell
docker run --interactive --rm quay.io/coreos/butane:release --pretty --strict < install-config.bu > install-config.ign
```

#### `install-config.ign`の出力例

```json
{
  "ignition": {
    "version": "3.3.0"
  },
  "passwd": {
    "users": [
      {
        "groups": [
          "docker",
          "sudo"
        ],
        "name": "sonohen",
        "sshAuthorizedKeys": [
          "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJw8VQlDD10wx3PPuC8l8T9mhYmDOikMicWhgH6JkAd1 sonohen@mbpro.local"
        ]
      }
    ]
  },
  "storage": {
    "files": [
      {
        "path": "/etc/hostname",
        "contents": {
          "source": "data:,container-server"
        },
        "mode": 420
      },
      {
        "path": "/etc/NetworkManager/system-connections/ens192.nmconnection",
        "contents": {
          "source": "data:,%5Bconnection%5D%0Aid%3Dens192%0Atype%3Dethernet%0Ainterface-name%3Dens192%0A%5Bipv4%5D%0Aaddress1%3D192.168.1.11%2F24%2C192.168.1.1%0Adhcp-hostname%3Dcontainer-server%0Adns%3D192.168.1.1%3B%0Adns-search%3D%0Amay-fail%3Dfalse%0Amethod%3Dmanual%0A"
        },
        "mode": 384
      }
    ],
    "links": [
      {
        "path": "/etc/localtime",
        "target": "../usr/share/zoneinfo/Asia/Tokyo"
      }
    ]
  }
}
```

### インストール

Fedora CoreOSのコンソールに戻り、以下のコマンドを実行する。

```shell
sudo coreos-installer install /dev/sda ¥
    --insecure-igntion ¥
    --ignition-url http://192.168.1.118:8000/install-config.ign
```

`Install Complete`と表示されたらインストール完了なので、メディアを外す等、仮想マシンを再構成してから再度起動する。

### インストール後の確認

```shell
# Fedora CoreOSへの接続
ssh sonohen@192.168.1.11

# タイムゾーンの確認
> date
Sat Feb 12 10:34:43 JST 2022

# ホスト名の確認
> hostname
container-server

# 再起動できるかの確認
> sudo systemctl reboot

# dockerの操作ができるかの確認
> docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

# 試しにJenkinsを立ち上げてみる
# 以下のコマンドを実行したあとに、`http://192.168.1.11:3000/`にアクセスしてJenkinsの画面が表示されれば成功
> docker run -d ¥
    --name myjenkins ¥
    -v `pwd`/jenkins_home:/var/lib/jenkins_home ¥
    -p "3000:8080" ¥
    -p "50000:50000" ¥
    jenkins/jenkins:lts-jdk11
```
