# 9章: launch ファイル

4 章では talker・listener を **2 つのターミナル** で別々に起動しました．ノードが増えるたびにターミナルを開くのは不便です．**launch ファイル** を使うと，複数のノードを 1 コマンドで起動できます．

---

## launch ファイルとは

**Python** で書かれた設定ファイルで，以下のことができます：

- 複数のノードをまとめて起動する
- パラメータを設定する
- トピック名を変更する（リマップ）
- 別の launch ファイルを読み込む
- 起動時に引数を受け取る

> **ROS1 との違い**: ROS1 の launch ファイルは XML でしたが，ROS2 では **Python** を使います．また ROS2 では `roscore` が不要なため，`roslaunch` に相当する `ros2 launch` を実行するだけで起動できます．

---

## launch ファイルの作成

パッケージ内に `launch/` フォルダを作って，そこに置くのが慣例です．

```bash
mkdir -p ~/ros2_ws/src/ros_tutorial/launch
```

### 基本的な launch ファイル

`~/ros2_ws/src/ros_tutorial/launch/talker_listener.launch.py` を作成：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        # talker ノードを起動
        Node(
            package='ros_tutorial',
            executable='talker',
            name='talker',
            output='screen',
        ),
        # listener ノードを起動
        Node(
            package='ros_tutorial',
            executable='listener',
            name='listener',
            output='screen',
        ),
    ])
```

**`Node()` の主なパラメータ：**

| 引数 | 意味 |
|------|------|
| `package` | パッケージ名 |
| `executable` | 実行ファイル名（`ros2 run` の第2引数と同じ）|
| `name` | このノードにつける名前（ROS 内での識別名）|
| `output='screen'` | ログをターミナルに表示する |

### CMakeLists.txt への追加

`ament_package()` の前に追記：

```cmake
install(DIRECTORY launch
  DESTINATION share/${PROJECT_NAME})
```

### ビルドと実行

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros_tutorial
source install/setup.bash
ros2 launch ros_tutorial talker_listener.launch.py
```

1 コマンドで talker と listener が同時に起動します．`Ctrl+C` で全ノードが終了します．

---

## パラメータを渡す

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='ros_tutorial',
            executable='talker',
            name='talker',
            output='screen',
            parameters=[{'publish_rate': 5}],   # パラメータを辞書で渡す
        ),
        Node(
            package='ros_tutorial',
            executable='listener',
            name='listener',
            output='screen',
        ),
    ])
```

---

## 引数（argument）を使う

起動時に値を変えられる**引数**を定義できます．

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration

def generate_launch_description():
    # 引数の定義（デフォルト値あり）
    rate_arg = DeclareLaunchArgument('rate', default_value='10',
                                    description='publish rate in Hz')

    return LaunchDescription([
        rate_arg,
        Node(
            package='ros_tutorial',
            executable='talker',
            name='talker',
            output='screen',
            parameters=[{'publish_rate': LaunchConfiguration('rate')}],
        ),
        Node(
            package='ros_tutorial',
            executable='listener',
            name='listener',
            output='screen',
        ),
    ])
```

起動時に引数を変えたい場合：

```bash
# デフォルト値（rate=10）で起動
ros2 launch ros_tutorial talker_listener.launch.py

# rate を 1 に変えて起動
ros2 launch ros_tutorial talker_listener.launch.py rate:=1
```

---

## トピック名の変更（remap）

```python
Node(
    package='ros_tutorial',
    executable='talker',
    name='talker',
    output='screen',
    remappings=[
        ('chatter', 'my_topic'),   # "chatter" を "my_topic" に変更
    ],
),
```

---

## 別の launch ファイルを読み込む

```python
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    ros_tutorial_dir = get_package_share_directory('ros_tutorial')

    return LaunchDescription([
        IncludeLaunchDescription(
            PythonLaunchDescriptionSource(
                os.path.join(ros_tutorial_dir, 'launch', 'talker_listener.launch.py')
            )
        ),
    ])
```

`get_package_share_directory('パッケージ名')` でそのパッケージのインストールパスを取得できます（ROS1 の `$(find パッケージ名)` に相当）．

---

## 名前空間を設定する

同じ実行ファイルを複数起動するときに **名前空間（namespace）** を使うと，ノード名やトピック名が重複しません．

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='ros_tutorial',
            executable='listener',
            name='listener',
            namespace='ns1',       # /ns1/listener というノード名になる
            output='screen',
        ),
        Node(
            package='ros_tutorial',
            executable='listener',
            name='listener',
            namespace='ns2',       # /ns2/listener というノード名になる
            output='screen',
        ),
    ])
```

---

## launch ファイルの主な要素まとめ

| 要素 | 用途 |
|------|------|
| `LaunchDescription` | launch ファイルのルート要素 |
| `Node` | ノードの起動 |
| `DeclareLaunchArgument` | 引数の定義 |
| `LaunchConfiguration` | 引数の値を参照 |
| `IncludeLaunchDescription` | 別の launch ファイルを読み込む |
| `GroupAction` | 複数のアクションをグループ化（名前空間の設定等）|

---

[→ 10章: RViz2](10_rviz2.md)
