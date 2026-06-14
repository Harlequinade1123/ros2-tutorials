# 付録A: ROS2 コマンドリファレンス

ROS2 のコマンドは `ros2 <サブコマンド>` の形式に統一されています（ROS1 の `rosnode`/`rostopic` などが `ros2 node`/`ros2 topic` に変わった）．

---

## ros2 node

| コマンド | 説明 |
|---------|------|
| `ros2 node list` | 起動中のノード一覧を表示 |
| `ros2 node info /node_name` | ノードの Publisher・Subscriber・Service・Action の情報を表示 |

---

## ros2 topic

| コマンド | 説明 |
|---------|------|
| `ros2 topic list` | トピック一覧を表示 |
| `ros2 topic list -t` | トピック一覧と型を表示 |
| `ros2 topic echo /topic` | トピックの内容をリアルタイム表示 |
| `ros2 topic info /topic` | トピックの型・Publisher 数・Subscriber 数を表示 |
| `ros2 topic pub /topic msg_type '{data: value}'` | コマンドラインからメッセージを送信 |
| `ros2 topic hz /topic` | トピックの配信周波数を計測 |
| `ros2 topic bw /topic` | トピックの帯域幅を計測 |

**例：**
```bash
# /chatter に文字列を 1 回送信
ros2 topic pub --once /chatter std_msgs/msg/String '{data: "hello"}'

# /chatter に 1 Hz で送信し続ける
ros2 topic pub -r 1 /chatter std_msgs/msg/String '{data: "hello"}'
```

---

## ros2 service

| コマンド | 説明 |
|---------|------|
| `ros2 service list` | サービス一覧を表示 |
| `ros2 service list -t` | サービス一覧と型を表示 |
| `ros2 service type /srv` | サービスの型を表示 |
| `ros2 service call /srv type '{args}'` | サービスを呼び出す |

**例：**
```bash
ros2 service call /add_two_ints ros_tutorial/srv/AddTwoInts '{a: 3, b: 5}'
ros2 service call /reset_count std_srvs/srv/Empty
```

---

## ros2 action

| コマンド | 説明 |
|---------|------|
| `ros2 action list` | アクション一覧を表示 |
| `ros2 action list -t` | アクション一覧と型を表示 |
| `ros2 action info /action` | アクションの情報を表示 |
| `ros2 action send_goal /action type '{goal}'` | ゴールを送信 |
| `ros2 action send_goal /action type '{goal}' --feedback` | フィードバックも表示 |

**例：**
```bash
ros2 action send_goal /count_down ros_tutorial/action/CountDown '{target: 5}' --feedback
```

---

## ros2 param

| コマンド | 説明 |
|---------|------|
| `ros2 param list` | 全ノードのパラメータ一覧を表示 |
| `ros2 param list /node` | 指定ノードのパラメータ一覧 |
| `ros2 param get /node param_name` | パラメータの値を表示 |
| `ros2 param set /node param_name value` | パラメータを変更 |
| `ros2 param dump /node` | ノードの全パラメータを YAML 形式で表示 |
| `ros2 param load /node config.yaml` | YAML ファイルからパラメータを読み込む |

**例：**
```bash
ros2 param get /counter_node rate
ros2 param set /counter_node rate 2.0
ros2 param dump /counter_node > params.yaml
```

---

## ros2 bag

| コマンド | 説明 |
|---------|------|
| `ros2 bag record -a` | 全トピックを記録 |
| `ros2 bag record -o name /topic` | トピック指定・ディレクトリ名指定で記録 |
| `ros2 bag play dir_name` | 記録を再生 |
| `ros2 bag play -l dir_name` | ループ再生 |
| `ros2 bag play -r 2.0 dir_name` | 2倍速で再生 |
| `ros2 bag info dir_name` | 記録の内容（時間・トピック数など）を確認 |

---

## ros2 run / launch

| コマンド | 説明 |
|---------|------|
| `ros2 run package executable` | 単体でノードを起動 |
| `ros2 run package executable --ros-args -p name:=value` | パラメータを指定して起動 |
| `ros2 run package executable --ros-args -r old:=new` | トピックをリマップして起動 |
| `ros2 launch package launch_file.launch.py` | launch ファイルで複数ノードを起動 |
| `ros2 launch package launch_file.launch.py key:=value` | 引数を渡して launch |

**例：**
```bash
ros2 run ros_tutorial talker --ros-args -r chatter:=/my_chatter
ros2 launch ros_exercises monitor_system.launch.py
```

---

## ros2 pkg

| コマンド | 説明 |
|---------|------|
| `ros2 pkg create name --build-type ament_cmake --dependencies dep` | パッケージ作成 |
| `ros2 pkg list` | インストール済みパッケージ一覧 |
| `ros2 pkg prefix package` | パッケージのインストール先パスを表示 |
| `ros2 pkg executables package` | パッケージ内の実行ファイル一覧 |

---

## ros2 interface

| コマンド | 説明 |
|---------|------|
| `ros2 interface list` | 利用可能なメッセージ・サービス・アクション一覧 |
| `ros2 interface show std_msgs/msg/String` | 型の定義を表示 |
| `ros2 interface package std_msgs` | パッケージ内の型一覧 |

---

## colcon

colcon は ROS2 のビルドツールです（ROS1 の `catkin_make` に相当）．

| コマンド | 説明 |
|---------|------|
| `colcon build` | ワークスペース全体をビルド |
| `colcon build --packages-select pkg1 pkg2` | 指定パッケージだけビルド |
| `colcon build --symlink-install` | Python スクリプトなどをシンボリックリンクでインストール |
| `colcon test --packages-select pkg` | テストを実行 |
| `colcon graph` | パッケージの依存関係グラフを表示 |

> **ビルド後は必ず `source install/setup.bash` を実行してください．**

---

## ROS1 ↔ ROS2 コマンド対応表

| ROS1 | ROS2 |
|------|------|
| `rosnode list` | `ros2 node list` |
| `rostopic list` | `ros2 topic list` |
| `rostopic echo /topic` | `ros2 topic echo /topic` |
| `rostopic pub /topic type value` | `ros2 topic pub /topic type value` |
| `rosservice list` | `ros2 service list` |
| `rosservice call /srv args` | `ros2 service call /srv type args` |
| `rosparam get /param` | `ros2 param get /node param` |
| `rosbag record /topic` | `ros2 bag record /topic` |
| `rosbag play data.bag` | `ros2 bag play data_dir` |
| `rosrun pkg exec` | `ros2 run pkg exec` |
| `roslaunch pkg file.launch` | `ros2 launch pkg file.launch.py` |
| `catkin_make` | `colcon build` |
| `source devel/setup.bash` | `source install/setup.bash` |
