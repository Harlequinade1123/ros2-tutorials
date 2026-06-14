# 7章: カスタムメッセージ

これまでは `std_msgs::msg::String`（文字列）を使いました．実際のロボット開発では「センサー名・計測値・ID を 1 つにまとめて送りたい」といったケースがよくあります．このような場合に **カスタムメッセージ** を定義します．

---

## メッセージ定義ファイル（.msg）の作成

パッケージ内に `msg/` フォルダを作ります．

```bash
mkdir -p ~/ros2_ws/src/ros_tutorial/msg
```

`~/ros2_ws/src/ros_tutorial/msg/SensorData.msg` を作成：

```
string name
float64 value
int32 id
```

**使える基本型：**

| 型 | 意味 |
|----|------|
| `bool` | 真偽値 |
| `int8`, `int16`, `int32`, `int64` | 整数（ビット数が異なる） |
| `float32`, `float64` | 浮動小数点 |
| `string` | 文字列 |
| `uint8[]` | バイト列（画像データなど） |
| `Header` | タイムスタンプ・フレームID（よく使う）|

他のメッセージ型を入れ子にすることもできます：

```
# 例：geometry_msgs/Point を含む場合
geometry_msgs/Point position
float64 speed
```

---

## ビルド設定の変更

### CMakeLists.txt の変更

`find_package(ament_cmake REQUIRED)` などの後に追加します：

```cmake
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SensorData.msg"
  DEPENDENCIES std_msgs
)
```

既存の `rosidl_generate_interfaces` があればそこに追記します：

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SensorData.msg"
  "srv/AddTwoInts.srv"
  "action/CountDown.action"
  DEPENDENCIES std_msgs
)
```

カスタムメッセージを使うノードにインターフェースをリンクします：

```cmake
rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")

add_executable(sensor_publisher src/sensor_publisher.cpp)
ament_target_dependencies(sensor_publisher rclcpp)
target_link_libraries(sensor_publisher ${cpp_typesupport_target})

add_executable(sensor_subscriber src/sensor_subscriber.cpp)
ament_target_dependencies(sensor_subscriber rclcpp)
target_link_libraries(sensor_subscriber ${cpp_typesupport_target})

install(TARGETS
  sensor_publisher
  sensor_subscriber
  DESTINATION lib/${PROJECT_NAME})
```

### package.xml の変更

（5章でまだ追加していない場合）以下を追加：

```xml
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>
```

---

## カスタムメッセージを使うノード

### Publisher（sensor_publisher.cpp）

`~/ros2_ws/src/ros_tutorial/src/sensor_publisher.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "ros_tutorial/msg/sensor_data.hpp"   // 自分で定義したメッセージ（snake_case）

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    auto node = rclcpp::Node::make_shared("sensor_publisher");

    auto pub = node->create_publisher<ros_tutorial::msg::SensorData>("sensor_topic", 10);
    rclcpp::Rate rate(1);  // 1Hz

    int id = 0;
    while (rclcpp::ok())
    {
        auto msg = ros_tutorial::msg::SensorData();
        msg.name  = "temperature";
        msg.value = 25.0 + id * 0.5;   // 疑似センサー値
        msg.id    = id++;

        RCLCPP_INFO(node->get_logger(), "送信: name=%s, value=%.1f, id=%d",
                    msg.name.c_str(), msg.value, msg.id);
        pub->publish(msg);

        rclcpp::spin_some(node);
        rate.sleep();
    }

    rclcpp::shutdown();
    return 0;
}
```

### Subscriber（sensor_subscriber.cpp）

`~/ros2_ws/src/ros_tutorial/src/sensor_subscriber.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "ros_tutorial/msg/sensor_data.hpp"

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    auto node = rclcpp::Node::make_shared("sensor_subscriber");

    auto sub = node->create_subscription<ros_tutorial::msg::SensorData>(
        "sensor_topic", 10,
        [node](const ros_tutorial::msg::SensorData::SharedPtr msg) {
            RCLCPP_INFO(node->get_logger(),
                        "受信: name=%s, value=%.1f, id=%d",
                        msg->name.c_str(), msg->value, msg->id);
        });

    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```

---

## ビルドと実行

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros_tutorial
source install/setup.bash
```

**ターミナル 1：**
```bash
ros2 run ros_tutorial sensor_publisher
```

**ターミナル 2：**
```bash
ros2 run ros_tutorial sensor_subscriber
```

### メッセージ型の確認

```bash
ros2 interface show ros_tutorial/msg/SensorData
```

出力：
```
string name
float64 value
int32 id
```

---

## 型名の変換規則まとめ

ROS2 では `.msg` / `.srv` / `.action` ファイル名と C++ の型名・ヘッダ名の変換規則があります．

| ファイル名（CamelCase）| C++ 型名 | ヘッダファイル名（snake_case）|
|---------------------|----------|-------------------------|
| `SensorData.msg` | `ros_tutorial::msg::SensorData` | `ros_tutorial/msg/sensor_data.hpp` |
| `AddTwoInts.srv` | `ros_tutorial::srv::AddTwoInts` | `ros_tutorial/srv/add_two_ints.hpp` |
| `CountDown.action` | `ros_tutorial::action::CountDown` | `ros_tutorial/action/count_down.hpp` |

---

## 現時点での CMakeLists.txt 全体像

混乱しないよう，この時点での主要な部分を示します．

```cmake
cmake_minimum_required(VERSION 3.8)
project(ros_tutorial)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_action REQUIRED)
find_package(std_msgs REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SensorData.msg"
  "srv/AddTwoInts.srv"
  "action/CountDown.action"
  DEPENDENCIES std_msgs
)

rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")

add_executable(hello_node src/hello.cpp)
ament_target_dependencies(hello_node rclcpp)

add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)

add_executable(listener src/listener.cpp)
ament_target_dependencies(listener rclcpp std_msgs)

add_executable(relay src/relay.cpp)
ament_target_dependencies(relay rclcpp std_msgs)

add_executable(add_two_ints_server src/add_two_ints_server.cpp)
ament_target_dependencies(add_two_ints_server rclcpp)
target_link_libraries(add_two_ints_server ${cpp_typesupport_target})

add_executable(add_two_ints_client src/add_two_ints_client.cpp)
ament_target_dependencies(add_two_ints_client rclcpp)
target_link_libraries(add_two_ints_client ${cpp_typesupport_target})

add_executable(count_down_server src/count_down_server.cpp)
ament_target_dependencies(count_down_server rclcpp rclcpp_action)
target_link_libraries(count_down_server ${cpp_typesupport_target})

add_executable(count_down_client src/count_down_client.cpp)
ament_target_dependencies(count_down_client rclcpp rclcpp_action)
target_link_libraries(count_down_client ${cpp_typesupport_target})

add_executable(sensor_publisher src/sensor_publisher.cpp)
ament_target_dependencies(sensor_publisher rclcpp)
target_link_libraries(sensor_publisher ${cpp_typesupport_target})

add_executable(sensor_subscriber src/sensor_subscriber.cpp)
ament_target_dependencies(sensor_subscriber rclcpp)
target_link_libraries(sensor_subscriber ${cpp_typesupport_target})

install(TARGETS
  hello_node talker listener relay
  add_two_ints_server add_two_ints_client
  count_down_server count_down_client
  sensor_publisher sensor_subscriber
  DESTINATION lib/${PROJECT_NAME})

ament_package()
```

---

[→ 8章: パラメータ](08_parameters.md)
