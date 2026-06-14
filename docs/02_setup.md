# 2章: 環境構築

## 前提条件

- **Ubuntu 22.04 LTS** がインストールされていること（仮想マシン・WSL2 でも可）
- インターネットに接続できること
- `sudo` コマンドが使えること

> **WSL2 を使う場合の注意**: WSL2 でも ROS2 は動作しますが，GUI ツール（rviz2 等）の表示には追加設定が必要です．RViz2 を使う 10 章では GUI が必要になるため，Ubuntu のネイティブ環境または GUI を有効にした WSL2 環境（WSLg など）を推奨します．

---

## ROS2 Humble のインストール

ターミナルを開いて，以下のコマンドを上から順に実行してください．

### 1. ロケールの設定

```bash
sudo apt update && sudo apt install locales -y
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

### 2. ROS2 リポジトリの追加

```bash
sudo apt install software-properties-common -y
sudo add-apt-repository universe

sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc \
    | sudo apt-key add -

sudo sh -c 'echo "deb [arch=$(dpkg --print-architecture)] \
    http://packages.ros.org/ros2/ubuntu \
    $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
    > /etc/apt/sources.list.d/ros2-latest.list'
```

### 3. ROS2 Humble のインストール

```bash
sudo apt update
sudo apt install ros-humble-desktop -y
```

> `desktop` を選ぶと，rviz2 などの GUI ツールも一緒にインストールされます．インストールには数分〜十数分かかります．

### 4. 追加ツールのインストール

```bash
sudo apt install python3-colcon-common-extensions -y
sudo apt install python3-rosdep -y
sudo rosdep init
rosdep update
```

`colcon` は ROS2 のビルドツールです（ROS1 の `catkin build` に相当します）．

### 5. 環境変数の設定

ROS2 のコマンドを使えるようにするため，`.bashrc` に追記します．

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## インストールの確認

以下のコマンドを実行してバージョンが表示されれば成功です．

```bash
ros2 --version
```

出力例：
```
ros2, version 0.18.X
```

また，環境変数も確認します：

```bash
echo $ROS_DISTRO
```

出力：
```
humble
```

---

## colcon ワークスペースの作成

ROS2 では，自分で書いたコードを **ワークスペース** という専用のフォルダにまとめて管理します．

### ワークスペースの作成とビルド

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
```

コマンドを実行すると `build/`・`install/`・`log/` フォルダが生成されます．

```
~/ros2_ws/
├── src/        ← ここに自分のパッケージ（コード）を置く
├── build/      ← ビルドの中間ファイル（自動生成）
├── install/    ← ビルド結果（実行ファイルなど）（自動生成）
└── log/        ← ビルドログ（自動生成）
```

> ROS1 の `devel/` が ROS2 では `install/` になっています．

### ワークスペースを常に使えるように設定

```bash
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

> **なぜ `source` が必要なのか?**
> `source` は「このファイルに書いてある環境変数の設定を今のターミナルに読み込む」コマンドです．
> `.bashrc` に書いておくことで，ターミナルを開くたびに自動で実行されます．

---

## 動作確認

### デモノードを起動してみる

ROS2 インストール時に含まれるデモノードで動作確認できます．

**ターミナル 1（talker）：**
```bash
ros2 run demo_nodes_cpp talker
```

**ターミナル 2（listener）：**
```bash
ros2 run demo_nodes_cpp listener
```

talker が "Hello World: X" を送信し，listener が受信して表示されれば成功です．

`Ctrl + C` で停止できます．

> **ROS1 との違い**: ROS2 では `roscore` を起動しなくても通信できます．

---

## よくあるエラーと対処法

### `ros2: command not found`

→ `source /opt/ros/humble/setup.bash` が実行されていません．
```bash
source /opt/ros/humble/setup.bash
```
または `.bashrc` への追記が正しく行われているか確認してください．

### `colcon: command not found`

→ `python3-colcon-common-extensions` がインストールされていません．
```bash
sudo apt install python3-colcon-common-extensions
```

### ビルド後のノードが `ros2 run` で見つからない

→ `source ~/ros2_ws/install/setup.bash` が必要です．ビルド後は必ず実行してください（`.bashrc` に書いてあれば新しいターミナルを開くだけで OK）．

---

環境の準備ができました．次の章では，最初のパッケージを作成します．

[→ 3章: 最初のパッケージを作る](03_first_package.md)
