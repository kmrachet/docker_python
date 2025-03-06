# DockerによるPython GPUディープラーニング環境構築

なんもわからん人向けのUbuntu&DockerによるPython実行環境構築

## メモ
- `$アルファベット大文字`は変数を表すので、状況に応じて自分で変更する(例: `$USER`)
- スクリプト行頭の`$`は一般ユーザ、`#`はrootユーザで実行する。<br>rootユーザへのログインは、一般ユーザでログインしている状態で`sudo -i`→ログインしている一般ユーザのパスワードを入力

## 環境
サーバ
- OS : Ubuntu Server 22.04 LTS
- CPU: 64bit
- GPU: Nvidia GPU
- Docker: 20.0 〜
- オンボードでのモニタ出力が可能なPCが望ましい<br>
  **※Nvidia Driverのみをインストールすると、以降GPUは画面出力できなくなる**
  `nvidia-toolkit`全体を入れる場合はどちらでも可
  バラ完する場合iGPUつきのCPUが必要

クライアント(手元の端末)
- Windows10 v1803～
- macOS / Linux

## Ubuntuのインストール
BIOS(UEFI)の操作はマザーボードのメーカーによって異なる。
1. [Ubuntuのダウンロードページ](https://jp.ubuntu.com/download)からUbuntu ServerのISOイメージをダウンロードする
2. DVDやUSBにマウントしてLiveメディアを作成する
3. メディアをサーバにする端末に入れて再起動
4. BIOSを起動し、ブートメニューでメディアを選択→Ubuntu Serverのインストールが始まる
4. OpenSSH Serverを一緒にインストールしておくと後でインストールする必要がなくなる

## Ubuntuの初期設定

### パッケージのアップグレード
```shell
# apt-get update
# apt-get upgrade
```

### タイムゾーンの変更
1. `timedatectl` で現在設定を確認
2. タイムゾーンが違う場合、`timedatectl set-timezone Asia/Tokyo` でタイムゾーンを変更

### 起動時設定
起動時に最大数分待ちが出る場合、`systemd-networkd-wait-online.service`を変更することで解決できる。
LANポートが複数あるとすべてのポートでセッションが確立するかタイムアウトするまで待つことがある。

1. `sudo -i`
2. `systemctl status systemd-networkd-wait-online` でステータス確認
    `Loaded:` 欄の設定ファイルのファイルパスを確認
3. `networkctl` で`configuring`になっているポート名をメモする
4. `/lib/systemd/system/NetworkManager-wait-online.service`を編集<br>
   ※`/etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service`からシンボリックリンクされている
    `[Service]`セクションの`ExecStart=/lib/...` 行の末尾に `--ignore=$3.でメモしたポート名` を入力
5. `sudo reboot`したときに`systemctl status systemd-networkd-wait-online`で動作確認する。`Active: active (exited)`になっていればOK

一定時間後にサーバがスリープする場合、以下のコードをシェルで実行([参考](https://ocg.aori.u-tokyo.ac.jp/member/daigo/comp/memo/?val=valid&typ=all&nbr=2021052501))<br>
`sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target`

### ネットワーク設定
#### IPアドレスの固定
1. `ip link` でイーサネットのポート名を調べる<br>
   ポート名 例:`eth0`
2. `ip route show` でデフォルトゲートウェイを調べる<br>
   例 `192.168.2.1`
3. `sudo resolvectl status` でDNSサーバのIPアドレスを調べる<br>
   例 `192.168.1.1`
4. `vi /etc/netplan/99_config.yaml` でconfigを作成<br>
   固定したいIPアドレスと上記の調べたものを記入

```yml
network:
  version: 2
  ethernets:
    eno0:
      dhcp4: false
      dhcp6: false
      addresses: [192.168.2.2/24]
      gateway4: 192.168.2.1
      nameservers:
        addresses: [192.168.1.1]
```

5. 作成した`99_config.yaml`を適用する<br>
   `sudo netplan apply`
6. `ip addr`で確認、`dynamic`がなくなってたらOK

#### Open SSHで他端末からリモートアクセス
**インストール**<br>Ubuntu Server初期インストール時に入れてる場合不要

```shell
# apt install -y openssh-server
```

##### パスワードログイン
1. クライアント側で`ssh $USER名@$IP_ADDRESS`コマンドを実行<br>

   `$USER`: ログインしたいユーザ名<br>

   `$IP_ADDRESS`: サーバのIPアドレス

2. はじめてサーバに接続するときは`known_hosts`でない端末へ接続するかを確認されるので`yes` -> サーバ側の`USER`ユーザのパスワードを入力

3. ログイン完了

WindowsでOpenSSHがインストールされていない場合
- [Windows10 v1803以降](https://docs.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_install_firstuse)
- Windows10 v1803以前↓
1. [GitHub](https://github.com/PowerShell/Win32-OpenSSH/releases)からOpenSSH-Win64(64bitの場合)をダウンロードしてくる
2. 解凍してC:直下にフォルダを配置
3. システム環境変数Pathに配置したフォルダのパスを追加

##### 鍵交換ログイン
**鍵の作成**<br>
クライアント側で`ssh-keygen -t ed25519` でED25519鍵を生成(RSA他も指定できる)<br>
パスフレーズを設定しない場合そのまま飛ばす

**鍵の登録**<br>
Mac

1. `brew install ssh-copy-id`
2. `ssh-copy-id id_ed25519 $USER@$IP_ADDRESS` でサーバに公開鍵を転送

手動で登録
1. サーバ側で`mkdir ~/.ssh`で一般ユーザのホームディレクトリ直下に`.ssh`ディレクトリを作成(ある場合は不要)
2. `touch`で`.ssh`フォルダの中に`authorized_keys`という空ファイルを作成
3. クライアント側で以下を実行<br>
   `cat ~/.ssh/id_ed25519.pub | ssh $USER@$IP_ADDRESS 'cat >> .ssh/authorized_keys'`<br>
   ※`scp`等で`id_ed25519.pub`ファイルを転送して`authorized_keys`に追記してもOK

**キーが変更された場合**
1. クライアント側で`ssh-keygen -R $IP_ADDRESS`を実行して`known_hosts`に登録した鍵の情報を削除
2. つなぎなおす

### ユーザ設定
ユーザ作成
```shell
# adduser $USER # ユーザを追加→指示に従ってパスワードなどを入力
# adduser $USER sudo # sudoグループに追加
$ cat /etc/passwd | grep $USER # ユーザ名一覧に名前があるかを確認
$ cat /etc/group | grep sudo # グループに追加されているかを確認
# adduser $USER docker # dockerグループにユーザを追加(Dockerをインストールしたあと)
```

### CUI環境を整える
#### byobu
`tmux/screen`のラッパー。複数のシェルを同時に使用したり、シェルの動作を維持したままデタッチできる。

1. Ubuntu Serverには標準でインストールされている。Ubuntu Desktopの場合、`apt install byobu`
2. `byobu`コマンドでbyobuをアクティブにする。`byobu-activate`でログイン時に起動する設定を追加できる。
3. `Ctrl + A`でエスケープシーケンスを設定する(デフォルトは`Ctrl + A` or `F12キー`)
4. `F6`キーでデタッチ
5. `F2`キーでウィンドウを増やす。`Option + F3`キーでウィンドウの名前を変更

エスケープシーケンス
- エスケープシーケンス + `d`でデタッチ
- エスケープシーケンス + `%`で縦に分割
- エスケープシーケンス + `Tab`で分割したウィンドウを移動
- エスケープシーケンス + `&`でウィンドウをkillする
- エスケープシーケンス + `数字`でその番号のウィンドウに移動
- エスケープシーケンス + `A`でウィンドウの名前を変更

画面分割
- エスケープシーケンス + `>`で画面分割・移動に関するヘルプを表示
- `u` : 今選択しているスプリットを上に移動
- `d` : 今選択しているスプリットを下に移動

ウィンドウ
- エスケープシーケンス + `<`でウィンドウに関するヘルプ
- `l` : 今開いているウィンドウを左に移動
- `r` : 今開いているウィンドウを右に移動

#### gotop
CPU、メモリ、ネットワーク等をリアルタイムにモニタリングできる。
1. [GitHub](https://github.com/cjbassi/gotop/releases)から`.deb`ファイルをダウンロードする
2. `scp gotop_3.0.0_linux_amd64.deb user@192.168.1.1:/home/usr`でファイルをコピー
3. `dpkg -i gotop_3.0.0_linux_amd64.deb`でインストール
4. `gotop`コマンドで起動、`q`で終了

#### htop
プロセスビューワ
1. Ubuntu Serverには標準でインストールされている。
2. `htop`コマンドで起動、`q`で終了

#### Nvidia GPUのモニタリング
Nvidia製GPUのドライバがインストールされると`nvidia-smi`コマンドが使用できる。`watch`コマンドと組み合わせることで、GPUをリアルタイムでモニタリングできる(デフォルトは2秒更新)。

1. インストールは後述
2. `watch nvidia-smi`で起動&継続的なモニタリングができる。他にもAPIあり([参考](https://dev.classmethod.jp/articles/monitor-nvidia-gpu-usage-with-nvidia-smi-nvsmi/))

#### フォントのインストール

1. `wget`でフォントをダウンロード
2. フォントファイルをディレクトリに配置<br>
`.ttfファイル`→`/usr/share/fonts/truetype`<br>
`.otfファイル`→`/usr/share/fonts/opentype`
3. `fc-cache -fv`で更新
3. `fc-list` でインストールされているフォント一覧を確認

### ユーザ間の共有

#### アカウント作成時にスクリプトを自動でコピーする

1. `/etc/skel` にコピーしたいファイルを置いておく
2. `usermod` コマンドでユーザを作成

#### 共有フォルダをつくる

1. `mkdir /home/shared` ディレクトリを作成
2. `groupadd shared` でユーザグループを作成
3. `chown root:shared /home/shared` でディレクトリの所有者をsharedに変更
4. `chmod 2770 /home/shared` でSGIDビットを立てる
5. `gpasswd -a ${ユーザ名} shared` でグループにユーザを追加
6. `cat /etc/group | grep shared` で`shared`にユーザが追加されているか確認
7. `ln -s /home/shared /home/${ユーザ名}` で`/home/${ユーザ名}`ディレクトリにシンボリックリンクを作成(作成時`root`ログアウトしたほうがよい？)
8. `chown -h ${ユーザ名}:${ユーザ名} /home/${ユーザ名}/shared`でシンボリックリンクの所有者を`username`に変更

## Nvidia GPU設定

[参考資料](https://medium.com/nvidiajapan/nvidia-docker-%E3%81%A3%E3%81%A6%E4%BB%8A%E3%81%A9%E3%81%86%E3%81%AA%E3%81%A3%E3%81%A6%E3%82%8B%E3%81%AE-20-09-%E7%89%88-558fae883f44)

### Nvidia Driverのインストール

[CUDA Toolkitのウェブサイト](https://developer.nvidia.com/cuda-downloads?target_os=Linux)のウェブサイトから該当するOSやアーキテクチャを選択する。<br>
[Ubuntu 20.04LTSにインストールする場合](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=20.04&target_type=deb_local)、以下が表示される。<br>
これに従って、**最終行を`sudo apt-get -y install cuda-drivers`とする。**

```shell
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/11.5.1/local_installers/cuda-repo-ubuntu2004-11-5-local_11.5.1-495.29.05-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2004-11-5-local_11.5.1-495.29.05-1_amd64.deb
sudo apt-key add /var/cuda-repo-ubuntu2004-11-5-local/7fa2af80.pub
sudo apt-get update
sudo apt-get -y install cuda # -> sudo apt-get -y install cuda-driversへ変更し実行
```

セキュアブートをオンにしている場合パスワードを求められるので、適当なものを設定し覚えておく(アカウントのログインパスワードと同一である必要はない)

### Nvidia Container Toolkitのインストール

[Nvidiaの公式ドキュメント](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#installing-docker-ce)に沿って、`nvidia-docker2`パッケージをインストールする

```shell
curl https://get.docker.com | sh \
&& sudo systemctl --now enable docker
```

```shell
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

```shell
sudo apt-get update
sudo apt-get install -y nvidia-docker2
sudo systemctl restart docker
```

### 再起動→セキュアブートの設定

[参考](https://nanbu.marune205.net/2021/12/ubuntu-secure-boot.html?m=1)

1. `sudo reboot`で再起動し、`Press any key`のメッセージが出てきたら適当なキーを押し、画面に従って以下の通りに操作する

2. 以下の画面が出てきたら`Enroll MOK`を選択

   ```
   Perform MOK Management
   
   Continue boot
   Enroll MOK
   Enroll key from disk
   Enroll hash from disk
   ```

3. 以下の画面が出てきたら`Continue`を選択

   ```
   [Enroll MOK]
   
   Veiw Key 0
   Continue

4. 先程のパスワードを入力する

   ```
   [Enroll the key(s)?]
     
   password:

5. `Reboot`を選択し、再起動する。

   ```
   Perform MOK management
   
   Reboot
   Enroll key from disk
   Enroll hash from disk

6. ログイン後、`nvidia-smi`コマンドでGPUが認識されているか確認する。

   ```
   +-----------------------------------------------------------------------------+
   | NVIDIA-SMI 495.29.05    Driver Version: 495.29.05    CUDA Version: 11.5     |
   |-------------------------------+----------------------+----------------------+
   | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
   | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
   |                               |                      |               MIG M. |
   |===============================+======================+======================|
   |   0  Quadro RTX 4000     On   | 00000000:B3:00.0 Off |                  N/A |
   | 30%   30C    P8     6W / 125W |     16MiB /  7973MiB |      0%      Default |
   |                               |                      |                  N/A |
   +-------------------------------+----------------------+----------------------+
                                                                                  
   +-----------------------------------------------------------------------------+
   | Processes:                                                                  |
   |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
   |        ID   ID                                                   Usage      |
   |=============================================================================|
   |    0   N/A  N/A      1264      G   /usr/lib/xorg/Xorg                  8MiB |
   |    0   N/A  N/A      1483      G   /usr/bin/gnome-shell                5MiB |
   +-----------------------------------------------------------------------------+

## DockerでPython環境構築

### Dockerのインストール

[Docker公式が用意しているUbuntu用のスクリプト](https://docs.docker.com/engine/install/ubuntu/#install-using-the-convenience-script)を利用してインストールする。

1. `curl -fsSL https://get.docker.com -o get-docker.sh`
2. `sudo sh get-docker.sh`

Dockerのサービス開始および自動起動設定をする

3. `curl https://get.docker.com | sh`
4. `sudo systemctl start docker && sudo systemctl enable docker`

### コンテナを動かす

[参考](https://jupyter-docker-stacks.readthedocs.io/en/latest/)

Jupyterの公式ドキュメントに沿ってJupyter LabをDockerコンテナで動かしてみる。

```
要件
・Dockerホスト(サーバ)のIPアドレス 192.168.2.2
・Dockerホストのポート番号 8000
・Dockerコンテナのポート番号 8888
・バインドマウント(Dockerホストにデータを保存する)
```

1. 以下のコードを実行する

   ```shell
   docker container run \
   -it \
   --rm \
   --name test_container \
   --user root \
   -p 8000:8888 \
   -e JUPYTER_ENABLE_LAB=yes \
   --mount type=bind,source=/home/user,target=/home/jovyan/work \
   jupyter/datascience-notebook:latest /bin/bash
   ```
   
2. `root`ユーザでDockerコンテナの`bash`が起動したら、Jupyter Labを起動する。このとき、IPアドレスは`0.0.0.0`で固定、ポート番号はDockerコンテナのものを指定する。

   ```shell
   jupyter lab --ip=0.0.0.0 --port=8888 --allow-root
   ```

3. Jupyter Labが起動するので、`To access the server, open this file in a browser:`の`Or copy and paste one of these URLs`のURLをコピーする。

   ```shell
   [I 2022-01-18 10:35:41.420 ServerApp] jupyterlab | extension was successfully linked.
   〜〜省略〜〜
   [C 2022-01-18 10:35:41.630 ServerApp] 
       
       To access the server, open this file in a browser:
           file:///home/jovyan/.local/share/jupyter/runtime/jpserver-7-open.html
       Or copy and paste one of these URLs:
           http://4a3d5c481647:8888/lab?token=XXXX...
        or http://127.0.0.1:8888/lab?token=XXXX...
   
   ```

3. コピーしたURLを`http://DockerホストのIPアドレス:Dockerホストのポート番号/lab?token=XXXX...`に変更してアクセスする。

   ```http
   http://192.168.2.2:8000/lab?token=XXXX...
   ```

4. Jupyter Labを停止する場合は、Jupyter Lab側でFile→Shut Downを選択するか、コマンドライン上で`Ctrl+C`→`Shutdown this Jupyter server (y/[n])?`のメッセージが表示されたら`Y`キーを押下する。
   Dockerコンテナの`bash`に戻ってくるので、`exit`コマンドを入力してコンテナを停止させる。
   コンテナからデタッチ(コンテナを停止せずにコンテナから抜ける)する場合は`Ctrl+P`→`Ctrl+Q`を押下する

### `docker container run`のつかいかた

構文: `docker container run $オプション $コンテナ名:$タグ名 $シェル名`

よく使うオプション
- `-it` / `-dit`: 対話モードで起動し、コンテナにログインする / デタッチする(どちらかを指定)
- `--rm`: Dockerコンテナが停止した際自動でコンテナを削除
- `--gpus`: どのGPUを使用するかを引数に指定(GPUをサポートしているコンテナのみ、`nvidia-smi`で出てきた番号`0, 1, 2..., all`を指定する。)
- `--name`: Dockerコンテナの名前を指定(ない場合自動でランダムな名前がつく)
- `-p`: Dockerホストのポート:Dockerコンテナのポート を指定
- `-e`: 環境変数を変更 

マウント
- `--mount`: Dockerコンテナの指定したディレクトリをマウントする。
- `type`: `bind` / `volume` バインドマウント(Dockerホストへのマウント)かボリュームマウント(Dockerボリュームへのマウント)かを選択
- `source`: Dockerホストのディレクトリ(バインドマウント)か、ボリューム名(ボリュームマウント)を指定する
- `target`: Dockerコンテナのマウント先(ディレクトリ)をフルパスで指定

### Dockerfileで自分のDockerイメージを作成

pullするイメージによって環境変数や入っているパッケージが異なるため、動作確認時に足りないものをメモしてDockerfileに記載し、本番用イメージをビルドする。
Docker Hubなどのドキュメントを確認したり、目的のDockerコンテナを`run`したりして、

- パッケージ管理は`pip`か`conda`か
- インストールされているパッケージ
- Nvidia GPUが使用可能か

などをチェックする。

例)`tensorflow/tensorflow`イメージをビルドするDockerfile
- `FROM`: Dockerイメージのビルドに使用するイメージを指定
- `LABEL`: ラベルを記載
- `RUN`: 実行したいコマンドを記載

```dockerfile
FROM tensorflow/tensorflow:latest-gpu-jupyter
LABEL version="1.0.2" \
      maintainer="hogehoge"

RUN apt update && apt install -y graphviz

# install python packages
RUN pip install -U pip ipython ipykernel && \
pip install imbalanced-learn matplotlib openpyxl optuna pydot scikit-learn seaborn shap swifter tableone xgboost lckr-jupyterlab-variableinspector

# generate JupyterLab configuration file and add the setting
RUN jupyter notebook --generate-config && \
echo "c.NotebookApp.default_url = '/lab'" >> /root/.jupyter/jupyter_lab_config.py

# generate JupyterLab advansed configuration file and add the settings
RUN mkdir -p /root/.jupyter/lab/user-settings/@jupyterlab/notebook-extension && \
echo -n '{"codeCellConfig": {"fontFamily": "HackGen35 Console", "lineNumbers": true},}' > /root/.jupyter/lab/user-settings/@jupyterlab/notebook-extension/tracker.jupyterlab-settings
```

Dockerfileの実行

`docker image build -t tensorflow/tensorflow:hoge .`でイメージ名`tensorflow/tensorflow`、タグ名`hoge`のイメージが作成される。<br>
`-t`オプションで`docker image ls`コマンドを実行したときに表示されるイメージ名とタグ名を設定する。<br>
`-f`オプションで任意のDockefileを指定することができる。

### シェルスクリプトでDockerコンテナの操作を簡略化する

実行は`bash ****.sh $ARG1 $ARG2`

シェルスクリプトによるユーザ作成&設定ファイルのコピー<br>
先に`adduser $USER`を実行しておく

```shell
#!/bin/bash
USER=$1

if [ -z $USER ] ; then
  echo "please input an user name for an argument."
  exit
fi

usermod --shell /bin/bash
usermod -aG sudo,docker,share ${USER}
chown -R ${USER}:${USER} /home/${USER}/docker
```


例) ポート、`--rm`するかどうかを指定してTensorFlowのコンテナを立ち上げる
※`$HOST_IP`を自分の環境に合わせて変更しておく

```shell
# run_tensorflow.sh
#!/bin/bash
PORT=$1
RM=$2

SCRIPT_DIR=$(cd $(dirname ${BASH_SOURCE:-$0}); pwd)
USER=$(whoami)
CONT_NAME="${USER}_tensorflow"
CONT_PORT="8888"
HOST_IP="192.168.2.2"

if [ -z $PORT ]; then
  echo "please input a port number."
  exit
fi

if [ "$RM" = "no-remove" ]; then
  docker container run \
  --name ${CONT_NAME} \
  -dit \
  --gpus all \
  -e TZ=Asia/Tokyo \
  --mount type=bind,source=${SCRIPT_DIR}/../,target=/tf/workdir \
  --mount type=bind,source=/usr/share/fonts/truetype,target=/usr/share/fonts/truetype \
  -p ${PORT}:${PORT} \
  tensorflow/tensorflow:hoge
elif [ -z $RM ]; then
  docker container run \
  --name ${CONT_NAME} \
  -dit \
  --rm \
  --gpus all \
  -e TZ=Asia/Tokyo \
  -e JUPYTER_ENABLE_LAB=yes \
  --mount type=bind,source=${SCRIPT_DIR}/../,target=/tf/workdir \
  --mount type=bind,source=/usr/share/fonts/truetype,target=/usr/share/fonts/truetype \
  -p ${CONT_PORT}:${PORT} \
  tensorflow/tensorflow:hoge
else
  echo "Invarid option. Please input like \"bash ${BASH_SOURCE:-$0} $PORT no-remove\""
  exit
fi

sleep 5

GET_TOKEN=$(/bin/bash ./get_token.sh ${CONT_NAME} ${CONT_PORT})
echo -e "Access below URL.\n${GET_TOKEN}"
```
```shell
# get_token.sh
#!/bin/bash

CONT_NAME=$1
CONT_PORT=$2

HOST_IP="10.35.141.120"
JUPYTER_URL="http://${HOST_IP}:${CONT_PORT}/lab?token="
JUPYTER_LOG="$(docker container logs ${CONT_NAME} | grep -m1 token)"
JUPYTER_TOKEN="$(echo $JUPYTER_LOG | awk -F '[=]' '{print $2}')"

if [ -n "$JUPYTER_TOKEN" ]; then
  echo "${JUPYTER_URL}${JUPYTER_TOKEN}"
else
  echo -e "Failed to check the container logs. Please run command \"docker container logs ${CONT_NAME} | grep -m1 token\" and check your Jupyter token."
fi
```

jupyter/datascience-notebookコンテナを立ち上げて、JupyterLabのトークンを自動で取得する

```shell
#!/bin/bash

NAME=$1
PORT=$2

HOST_IP="192.168.2.165"
USER=$(whoami)
DIR=$(cd $(dirname ${BASH_SOURCE:-$0}); pwd)
CONT_NAME="${USER}_${NAME}"
CONT_PORT="8888"

echo "your container ID is below."
docker container run \
  -dit \
  --rm \
  -e JUPYTER_ENABLE_LAB=yes \
  -e TZ=Asia/Tokyo \
  -p $PORT:$CONT_PORT \
  --name $CONT_NAME \
  --user root \
  --mount type=bind,source=$DIR/../,target=/home/jovyan/work \
  --mount type=bind,source=/usr/local/share/fonts/,target=/usr/local/share/fonts/ \
  jupyter/datascience-notebook

sleep 2

GET_TOKEN=$(/bin/bash ./get_token.sh ${HOST_IP} ${PORT} ${CONT_NAME} ${CONT_PORT})
echo $GET_TOKEN
```

### TensorBoardによる監視
[TensorBoard](https://www.tensorflow.org/tensorboard/get_started)を使うことで、機械学習の進捗をGUIコンソールでリアルタイムに確認することができる。DockerコンテナではTensorBoardをインラインで実行できないため、専用のDockerコンテナを動かす必要がある。

**TensorBoard用コンテナの定義のDockerfile**<br>

`CMD`: コンテナを`run`するときに自動で実行するコマンドを記述する

```dockerfile
FROM tensorflow/tensorflow:latest
ENV LOG_DIR=/tf/workdir/logs
CMD tensorboard --logdir $LOG_DIR --host 0.0.0.0 --port 8888
```

イメージをビルドする

`````shell
docker image build -t tensorflow/tensorboard:hoge　.
`````

**TensorFlowの定義**<br>
プロジェクトのディレクトリが下記の場合(`logs`ディレクトリと`training_checkpoints`ディレクトリは機械学習が回るときに自動で作成される)

```shell
~/test_project
├ test_project.ipynb
├ logs
└ training_checkpoints
```

TensorFlowの`callbacks`にディレクトリを指定

```python
test_project.ipynb

callbacks = [
    TensorBoard(log_dir="logs"), 
    ModelCheckpoint(filepath="training_checkpoints/checkpoint_{epoch}", save_weights_only=True), 
    EarlyStopping(monitor="val_loss", patience=5),
]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
model.fit(
  train_dataset, validation_data=test_dataset, 
  epochs=10, callbacks=callbacks, verbose=0
)
```

TensorBoard用Dockerコンテナを`run`する。このとき`--mount`オプションの`source`引数には、TensorFlowのログが出力されるディレクトリのひとつ上を指定する。`$LOG_DIR`を変更する場合は`-e`オプションで環境変数を再定義する。

```shell
docker container run \
	-dit \
	--rm \
	--name tensorboard \
	--mount type=bind,source=~/test_project,target=/tf/workdir
	-p 8889:8888 \
	tensorflow/tensorboard:hoge
```

### GPUの分散処理

[参考1](https://www.tensorflow.org/api_docs/python/tf/distribute/Strategy) <br>
[参考2](https://zenn.dev/ozora/articles/tensorflow_strategy)

Tensorflowはデフォルトでは単一のGPUを用いる。複数枚GPUを使用する場合、TensorflowのDockerコンテナを起動する際`--gpus`オプションで複数枚または`all`を指定して、モデルのコンパイル/ビルドまでをスコープで括るように記述する。

```Python
import tensorflow as tf

strategy = tf.strategy.MirroredStrategy()
with strategy.scope():
  inputs = ...
  ...
  ...
  model.compile(...)
  
model.fit(...)
```

`MirroredStrategy`以外にも、複数マシンでの動作やGoogle TPUにも対応するAPIがある。<br>
[Bazelによる分散処理](https://www.tensorflow.org/install/source?hl=ja)もできるが、コンパイルに関するWarningが出る。バージョンが揃ってない？
