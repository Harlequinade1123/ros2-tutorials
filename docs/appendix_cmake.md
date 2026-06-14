# 付録B: ament_cmake ビルドシステム

ROS2 のビルドシステム **ament_cmake** の基本を説明します．`CMakeLists.txt` と `package.xml` の書き方を理解することで，エラーが出たときに自分で対処できるようになります．

---

## ament_cmake とは

ROS1 では `catkin_cmake` が使われていましたが，ROS2 では **ament_cmake** が標準です．

| ビルドシステム | 使用場面 |
|--------------|---------|
| `ament_cmake` | C++ パッケージ（この チュートリアルで使用）|
| `ament_python` | Pure Python パッケージ |
| `ament_cmake_python` | C++ と Python が混在するパッケージ |

---

## CMakeLists.txt の構造

典型的な CMakeLists.txt の全体像：

```cmake
cmake_minimum_required(VERSION 3.8)
project(ros_tutorial)

# ① 依存パッケージを見つける
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

# ② インターフェースの生成（.msg/.srv/.action がある場合）
find_package(rosidl_default_generators REQUIRED)
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SensorData.msg"
  "srv/AddTwoInts.srv"
  "action/CountDown.action"
  DEPENDENCIES std_msgs
)

# ③ 実行ファイルの作成
add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)

add_executable(listener src/listener.cpp)
ament_target_dependencies(listener rclcpp std_msgs)

# ④ カスタムインターフェースとのリンク（同一パッケージ内）
rosidl_get_typesupport_target(cpp_typesupport_target
  ${PROJECT_NAME} "rosidl_typesupport_cpp")

add_executable(service_server src/service_server.cpp)
ament_target_dependencies(service_server rclcpp)
target_link_libraries(service_server ${cpp_typesupport_target})

# ⑤ インストール先の指定
install(TARGETS talker listener service_server
  DESTINATION lib/${PROJECT_NAME})

install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME})

# ⑥ ament のフィナライズ（必ず最後に書く）
ament_package()
```

---

## 各セクションの説明

### ① find_package

依存するパッケージを cmake に教えます．

```cmake
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
```

- `REQUIRED` を付けると，パッケージが見つからないときにビルドエラーになります
- `find_package` に書いたパッケージは `package.xml` にも書く必要があります（後述）

### ② rosidl_generate_interfaces

`.msg`，`.srv`，`.action` ファイルから C++ ヘッダを生成します．

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SensorData.msg"
  "srv/AddTwoInts.srv"
  DEPENDENCIES std_msgs  # 使用している型のパッケージを記述
)
```

> **DEPENDENCIES** には，メッセージ定義内で使っている外部型（例: `std_msgs/Header`）のパッケージを列挙します．`std_msgs` 組み込み型のみの場合でも `std_msgs` を書くのが安全です．

### ③ add_executable + ament_target_dependencies

実行ファイルを作り，依存ライブラリを紐づけます．

```cmake
add_executable(my_node src/my_node.cpp)
ament_target_dependencies(my_node rclcpp std_msgs)
```

`ament_target_dependencies` を使うと，インクルードパスとリンクが自動で設定されます．

### ④ rosidl_get_typesupport_target

**同一パッケージ内で定義したカスタム型を使う場合**に必要です．

```cmake
rosidl_get_typesupport_target(cpp_typesupport_target
  ${PROJECT_NAME} "rosidl_typesupport_cpp")

target_link_libraries(my_node ${cpp_typesupport_target})
```

> 別パッケージのカスタム型を使う場合は `find_package(other_pkg REQUIRED)` と `ament_target_dependencies` だけで OK です．

### ⑤ install

`ros2 run` でノードを見つけるために，インストール先を指定します．

```cmake
# 実行ファイルのインストール先
install(TARGETS talker listener
  DESTINATION lib/${PROJECT_NAME})

# ディレクトリのインストール先
install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME})

# 単一ファイルのインストール
install(FILES some_file.yaml
  DESTINATION share/${PROJECT_NAME})
```

> **`install()` は ROS2 で追加が必要な最重要設定の一つです．** 書き忘れると `ros2 run` が「実行ファイルが見つからない」エラーになります．

---

## package.xml の構造

```xml
<?xml version="1.0"?>
<package format="3">
  <name>ros_tutorial</name>
  <version>0.0.1</version>
  <description>ROS2 tutorial package</description>
  <maintainer email="your@email.com">Your Name</maintainer>
  <license>Apache-2.0</license>

  <!-- ビルド時にだけ必要な依存 -->
  <buildtool_depend>ament_cmake</buildtool_depend>

  <!-- ビルド時・実行時の両方で必要な依存 -->
  <depend>rclcpp</depend>
  <depend>std_msgs</depend>
  <depend>geometry_msgs</depend>

  <!-- ビルド時のみ必要な依存 -->
  <build_depend>rosidl_default_generators</build_depend>

  <!-- 実行時のみ必要な依存 -->
  <exec_depend>rosidl_default_runtime</exec_depend>

  <!-- インターフェースパッケージであることを宣言 -->
  <member_of_group>rosidl_interface_packages</member_of_group>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

### depend タグの種類

| タグ | 意味 | 使いどころ |
|------|------|-----------|
| `<depend>` | ビルド・実行の両方で必要 | ほとんどの場合これを使う |
| `<build_depend>` | ビルド時のみ必要 | `rosidl_default_generators` など |
| `<exec_depend>` | 実行時のみ必要 | `rosidl_default_runtime` など |
| `<buildtool_depend>` | ビルドツール | `ament_cmake` はこれを使う |
| `<test_depend>` | テスト時のみ必要 | `ament_lint_auto` など |

---

## よくあるエラーと対処法

### `ros2 run` で実行ファイルが見つからない

```
No executable found
```

**原因**: `CMakeLists.txt` に `install(TARGETS ...)` が書かれていない  
**対処**: `install(TARGETS my_node DESTINATION lib/${PROJECT_NAME})` を追加してビルドし直す

---

### カスタム型のヘッダが見つからない

```
fatal error: ros_tutorial/srv/add_two_ints.hpp: No such file or directory
```

**原因**: `rosidl_generate_interfaces()` の前に実行ファイルをビルドしようとしている  
**対処**: `CMakeLists.txt` でインターフェース生成を先に記述し，`target_link_libraries` を使う

---

### 依存パッケージが見つからない

```
Could not find a package configuration file provided by "xxx"
```

**原因**:
1. パッケージが未インストール → `sudo apt install ros-humble-xxx`
2. `source /opt/ros/humble/setup.bash` が実行されていない
3. `find_package(xxx REQUIRED)` が CMakeLists.txt にない

---

### `ament_package()` のあとにコードを書いた

```
CMake Error at ...
```

**原因**: `ament_package()` は必ず CMakeLists.txt の**最後**に書く必要があります  
**対処**: すべての `add_executable()`・`install()` を `ament_package()` より前に移動する

---

## ファイル変換の命名規則

ROS2 では，ファイル名と生成されるヘッダ・型名の間に変換ルールがあります．

| ファイル名 | 型名（C++） | ヘッダのインクルードパス |
|-----------|------------|----------------------|
| `msg/SensorData.msg` | `ros_tutorial::msg::SensorData` | `ros_tutorial/msg/sensor_data.hpp` |
| `srv/AddTwoInts.srv` | `ros_tutorial::srv::AddTwoInts` | `ros_tutorial/srv/add_two_ints.hpp` |
| `action/CountDown.action` | `ros_tutorial::action::CountDown` | `ros_tutorial/action/count_down.hpp` |

**変換ルール**: `CamelCase.msg` → ファイル名は `snake_case.hpp`，型名は `CamelCase`

---

## ROS1 との比較

| 項目 | ROS1 (catkin) | ROS2 (ament_cmake) |
|------|-------------|-------------------|
| ビルドシステム | catkin_cmake | ament_cmake |
| ワークスペースの初期化 | `catkin_init_workspace` | 不要（colcon が処理）|
| ビルドコマンド | `catkin_make` | `colcon build` |
| ビルド後のセットアップ | `source devel/setup.bash` | `source install/setup.bash` |
| 依存ライブラリの登録 | `catkin_package()` + `target_link_libraries()` | `ament_target_dependencies()` |
| 実行ファイルのインストール | 自動 | `install(TARGETS ... DESTINATION lib/${PROJECT_NAME})` が必要 |
