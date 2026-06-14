# 8章: パラメータ

**パラメータ** は，ソースコードを変更・再ビルドせずに実行時に値を変えられる仕組みです．ゲインの調整・閾値の変更・ロボット名の設定など，変更頻度が高い値に使います．

---

## ROS2 のパラメータの仕組み

ROS1 では中央の **パラメータサーバー** にすべてのパラメータが保存されていました．  
ROS2 では **各ノードが自分のパラメータを管理** します．ノードは起動時にパラメータを「宣言（declare）」してから使います．

---

## C++ でパラメータを使う

### `declare_parameter` + `get_parameter`

```cpp
// まずパラメータを宣言（デフォルト値つき）
this->declare_parameter("my_param", 10);

// 値を取得
int value = this->get_parameter("my_param").as_int();
```

`declare_parameter` の第2引数がデフォルト値です．

### 型に応じた取得メソッド

| 型 | 取得メソッド |
|----|------------|
| `int` | `.as_int()` |
| `double` | `.as_double()` |
| `std::string` | `.as_string()` |
| `bool` | `.as_bool()` |

---

## パラメータを使うノード（param_example.cpp）

`~/ros2_ws/src/ros_tutorial/src/param_example.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    auto node = rclcpp::Node::make_shared("param_example");

    // パラメータの宣言（名前，デフォルト値）
    node->declare_parameter("robot_name",   std::string("robot_A"));
    node->declare_parameter("publish_rate", 10);
    node->declare_parameter("threshold",    0.5);

    // パラメータの取得
    std::string robot_name  = node->get_parameter("robot_name").as_string();
    int         publish_rate = node->get_parameter("publish_rate").as_int();
    double      threshold   = node->get_parameter("threshold").as_double();

    RCLCPP_INFO(node->get_logger(), "ロボット名    : %s",  robot_name.c_str());
    RCLCPP_INFO(node->get_logger(), "パブリッシュレート: %d Hz", publish_rate);
    RCLCPP_INFO(node->get_logger(), "閾値          : %.2f", threshold);

    auto pub = node->create_publisher<std_msgs::msg::String>("status", 10);
    rclcpp::Rate rate(publish_rate);

    while (rclcpp::ok())
    {
        auto msg = std_msgs::msg::String();
        msg.data = robot_name + " is running";
        pub->publish(msg);

        rclcpp::spin_some(node);
        rate.sleep();
    }

    rclcpp::shutdown();
    return 0;
}
```

### CMakeLists.txt に追記

```cmake
add_executable(param_example src/param_example.cpp)
ament_target_dependencies(param_example rclcpp std_msgs)

install(TARGETS param_example DESTINATION lib/${PROJECT_NAME})
```

---

## launch ファイルでパラメータを設定する

`launch/` ディレクトリを作成します：

```bash
mkdir -p ~/ros2_ws/src/ros_tutorial/launch
```

`~/ros2_ws/src/ros_tutorial/launch/param_example.launch.py` を作成：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='ros_tutorial',
            executable='param_example',
            name='param_example',
            output='screen',
            parameters=[{
                'robot_name':   'my_robot',
                'publish_rate': 2,
                'threshold':    0.8,
            }]
        )
    ])
```

launch フォルダをインストールに追加します（`CMakeLists.txt` の `ament_package()` の前）：

```cmake
install(DIRECTORY launch
  DESTINATION share/${PROJECT_NAME})
```

### 実行

```bash
cd ~/ros2_ws
colcon build --packages-select ros_tutorial
source install/setup.bash
ros2 launch ros_tutorial param_example.launch.py
```

出力：
```
[INFO] [...] [param_example]: ロボット名    : my_robot
[INFO] [...] [param_example]: パブリッシュレート: 2 Hz
[INFO] [...] [param_example]: 閾値          : 0.80
```

---

## コマンドラインからパラメータを渡す

launch ファイルを使わず，`ros2 run` 実行時にパラメータを渡すこともできます：

```bash
ros2 run ros_tutorial param_example \
    --ros-args -p robot_name:=my_robot -p publish_rate:=2 -p threshold:=0.8
```

---

## ros2 param コマンド

```bash
# ノードのパラメータ一覧
ros2 param list /param_example

# パラメータの値を確認
ros2 param get /param_example robot_name

# パラメータの値を変更（ノードが declare_parameter している必要がある）
ros2 param set /param_example publish_rate 5

# パラメータを YAML ファイルに保存
ros2 param dump /param_example --output-dir ./

# YAML ファイルからパラメータを読み込む
ros2 param load /param_example param_example.yaml
```

---

## YAML ファイルでパラメータをまとめる

複数のパラメータは YAML ファイルにまとめると管理しやすいです．

```bash
mkdir -p ~/ros2_ws/src/ros_tutorial/config
```

`~/ros2_ws/src/ros_tutorial/config/robot_params.yaml` を作成：

```yaml
param_example:
  ros__parameters:
    robot_name: "cool_robot"
    publish_rate: 5
    threshold: 0.75
```

> **ROS2 の YAML フォーマット**: ROS2 では必ず `ノード名:` → `ros__parameters:` の階層構造を使います（`ros__parameters` はアンダースコア 2 つ）．

launch ファイルから YAML を読み込む：

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    config = os.path.join(
        get_package_share_directory('ros_tutorial'),
        'config', 'robot_params.yaml')

    return LaunchDescription([
        Node(
            package='ros_tutorial',
            executable='param_example',
            name='param_example',
            output='screen',
            parameters=[config]
        )
    ])
```

YAML ファイルも忘れずにインストール設定に追加：

```cmake
install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME})
```

---

## パラメータの動的変更

`ros2 param set` でパラメータを変更しても，**すでに起動しているノードが `declare_parameter` + `get_parameter` のみで読んでいる場合はリアルタイムには反映されません**（起動時の一度だけ読み取り）．

動的変更に対応するには `on_set_parameters_callback` を登録します（このチュートリアルの範囲外）．

---

[→ 9章: launch ファイル](09_launch_files.md)
