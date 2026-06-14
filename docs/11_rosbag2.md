# 11章: rosbag2 ── トピックの記録と再生

## rosbag2 とは

**rosbag2** は，ROS2 トピックのデータを **ディレクトリに記録（録画）して後から再生** できるツールです．

> **ROS1 との違い**: ROS1 の `rosbag` は 1 つの `.bag` ファイルに保存しましたが，ROS2 の `rosbag2` は**ディレクトリ**に保存します（`mcap` や `sqlite3` などの形式）．コマンド名も `rosbag` から `ros2 bag` に変わりました．

### 何に使うのか

| 用途 | 説明 |
|------|------|
| **デバッグ** | 実機でセンサーデータを収録し，PCだけで何度でも再現して解析できる |
| **開発効率化** | ロボットがなくても収録済みデータでアルゴリズムを試せる |
| **記録・比較** | 改良前後の動作データを保存して比較できる |
| **データセット作成** | 機械学習の学習データとして活用できる |

---

## インストール確認

ROS2 Humble の `desktop` インストールには rosbag2 が含まれています．含まれていない場合：

```bash
sudo apt install ros-humble-rosbag2 -y
```

---

## 基本的な使い方

### ros2 bag record ── 記録

```bash
# 全トピックを記録（ディレクトリ名は自動生成）
ros2 bag record -a

# 特定のトピックだけ記録
ros2 bag record /chatter /odom

# ディレクトリ名を指定して記録
ros2 bag record -o my_data /chatter

# 指定した秒数だけ記録して自動終了
ros2 bag record --max-bag-duration 10 -o sensor_10s /odom
```

> `Ctrl + C` で記録を終了します．

### ros2 bag play ── 再生

```bash
# 再生（記録時と同じトピック名で publish される）
ros2 bag play my_data

# ループ再生（Ctrl+C で終了）
ros2 bag play -l my_data

# 再生速度を変える（0.5 = 半速，2.0 = 2倍速）
ros2 bag play -r 0.5 my_data

# 指定秒数後から再生
ros2 bag play --start-offset 5 my_data

# 一時停止状態で起動（スペースキーで再生/一時停止）
# ※ Humble では GUI ツール経由
```

### ros2 bag info ── 内容確認

```bash
ros2 bag info my_data
```

出力例：
```
Files:             my_data_0.mcap
Bag size:          45.2 KiB
Storage id:        mcap
Duration:          10.200s
Start:             Jun 14 2026 12:00:00.000 (1718352000.000)
End:               Jun 14 2026 12:00:10.200 (1718352010.200)
Messages:          102
Topic information: Topic: /chatter | Type: std_msgs/msg/String | Count: 102 | Serialization Format: cdr
```

---

## 実際に試してみる

talker / listener を使って記録・再生の流れを体験します．

### 手順 1: talker を動かしながら記録する

**ターミナル 1（talker を起動）：**
```bash
ros2 run ros_tutorial talker
```

**ターミナル 2（記録開始）：**
```bash
mkdir -p ~/rosbag2_data
cd ~/rosbag2_data
ros2 bag record -o chatter_data /chatter
```

10 秒ほど待ってから `Ctrl+C` で記録終了．

### 手順 2: talker を止めてバッグを確認する

ターミナル 1 の talker を `Ctrl+C` で終了．

```bash
ros2 bag info ~/rosbag2_data/chatter_data
```

### 手順 3: バッグを再生して listener で受け取る

**ターミナル 1（listener を起動）：**
```bash
ros2 run ros_tutorial listener
```

**ターミナル 2（バッグを再生）：**
```bash
ros2 bag play ~/rosbag2_data/chatter_data
```

listener のターミナルに，記録済みのメッセージが再生されて表示されます．**talker を動かしていなくてもデータが届く** ことを確認してください．

---

## rosbag2 と ros2 topic echo の組み合わせ

再生中に `ros2 topic echo` でリアルタイム確認できます．

```bash
# ターミナル A: 再生
ros2 bag play my_data

# ターミナル B: 内容を確認
ros2 topic echo /chatter
```

---

## よく使うオプションまとめ

### record オプション

| オプション | 説明 | 例 |
|-----------|------|----|
| `-a` | 全トピックを記録 | `ros2 bag record -a` |
| `-o <名前>` | ディレクトリ名を指定 | `ros2 bag record -o data /odom` |
| `--max-bag-duration <秒>` | 指定秒数で自動停止 | `--max-bag-duration 30` |
| `--max-bag-size <バイト>` | 指定サイズで分割保存 | `--max-bag-size 104857600` |
| `-x <正規表現>` | 除外するトピックを指定 | `-x "/camera/.*"` |

### play オプション

| オプション | 説明 | 例 |
|-----------|------|----|
| `-l` | ループ再生 | `ros2 bag play -l data` |
| `-r <倍率>` | 再生速度の変更 | `-r 2.0` |
| `--start-offset <秒>` | 開始位置を指定 | `--start-offset 10` |
| `--topics <トピック>` | 指定トピックだけ再生 | `--topics /odom /chatter` |

---

## トピック名を変えて再生する

再生時にトピック名を変更（remap）することもできます．

```bash
# /chatter を /old_chatter として再生
ros2 bag play data --remap /chatter:=/old_chatter
```

---

## よくあるトラブル

### 再生しても何も届かない

**確認**:
```bash
ros2 node list    # listener などが起動しているか
ros2 topic list   # 再生中は /chatter が見えるはず
```

### 記録ファイルの形式について

ROS2 Humble のデフォルト保存形式は `mcap` です．`sqlite3` 形式で保存したい場合：

```bash
ros2 bag record -o my_data --storage sqlite3 /chatter
```

---

[→ 12章: tf2](12_tf2.md)
