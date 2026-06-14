# 3章: 最初のパッケージを作る

## パッケージとは

ROS2 では，ひとまとまりのプログラムを **パッケージ** という単位で管理します．
Visual Studio のプロジェクトに相当するものだと思ってください．

パッケージには以下が含まれます：

- C++ のソースファイル（`.cpp`）
- ビルド設定ファイル（`CMakeLists.txt`）
- パッケージ情報ファイル（`package.xml`）
- メッセージ・サービス定義ファイル（後の章で説明）
- launch ファイル（9章で説明）

---

## パッケージを作成する

### 作業ディレクトリへ移動

```bash
cd ~/ros2_ws/src
```

### パッケージ作成コマンド

```bash
ros2 pkg create ros_tutorial --build-type ament_cmake --dependencies rclcpp std_msgs
```

**コマンドの意味：**
- `ros2 pkg create` ：ROS2 のパッケージ作成コマンド
- `ros_tutorial` ：作成するパッケージの名前
- `--build-type ament_cmake` ：C++ パッケージに使うビルドシステム
- `--dependencies rclcpp std_msgs` ：このパッケージが使用する依存パッケージ
  - `rclcpp`：C++ で ROS2 を使うために必要（ROS1 の `roscpp` に相当）
  - `std_msgs`：標準的なメッセージ型（文字列・数値など）を使うために必要

実行すると `src/ros_tutorial/` フォルダが作成されます．

```
~/ros2_ws/src/ros_tutorial/
├── CMakeLists.txt    ← ビルド設定
├── package.xml       ← パッケージ情報
├── include/          ← ヘッダファイルを置く場所
│   └── ros_tutorial/
└── src/              ← C++ ソースファイルを置く場所（最初は空）
```

---

## 各ファイルの役割

### CMakeLists.txt

**「どのファイルをどうビルドするか」** を記述するファイルです．

生成された内容（コメント行は省略）：

```cmake
cmake_minimum_required(VERSION 3.8)
project(ros_tutorial)

# C++17 の有効化
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 依存パッケージの検索
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

# ここに実行ファイルの設定を追加していく（後述）

ament_package()   # ← 必ず最後に書く
```

> **ROS1 との違い**: `find_package(catkin ...)` の代わりに `find_package(ament_cmake ...)` を使います．`catkin_package()` の代わりに `ament_package()` が最後に必要です．

### package.xml

**パッケージのメタ情報（名前・バージョン・依存関係など）** を記述するファイルです．

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd"
            schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>ros_tutorial</name>
  <version>0.0.0</version>
  <description>ROS2 Tutorial Package</description>
  <maintainer email="your@email.com">Your Name</maintainer>
  <license>MIT</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <!-- ビルド時・実行時に必要なパッケージ -->
  <depend>rclcpp</depend>
  <depend>std_msgs</depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

> **ROS1 との違い**: `format="3"` を使います．`<buildtool_depend>catkin</buildtool_depend>` の代わりに `<buildtool_depend>ament_cmake</buildtool_depend>` を書きます．`<depend>` タグ 1 つでビルド時と実行時の両方を宣言できます（`<build_depend>` + `<exec_depend>` を兼ねます）．

---

## ビルドしてみる

```bash
cd ~/ros2_ws
colcon build --symlink-install
```

エラーなく完了すれば成功です．

> **よく使うビルドオプション：**
> ```bash
> colcon build --symlink-install --packages-select ros_tutorial
> ```
> - `--packages-select ros_tutorial` : ワークスペース全体ではなく，指定したパッケージだけをビルドします．開発中は頻繁に使います．
> - `--symlink-install` : launch ファイル・YAML・Python スクリプトなどの**非コンパイルファイルをコピーではなくシンボリックリンクでインストール**します．これらのファイルを編集したとき，**再ビルドなしで変更が即座に反映**されます（C++ ソースの変更は引き続きリビルドが必要です）．

---

## 試しにソースファイルを置いてみる

動作確認のために，簡単な C++ ファイルを作ってビルドしてみましょう．

### ソースファイルの作成

`~/ros2_ws/src/ros_tutorial/src/hello.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    auto node = rclcpp::Node::make_shared("hello_node");

    RCLCPP_INFO(node->get_logger(), "Hello, ROS2!");

    rclcpp::shutdown();
    return 0;
}
```

**`RCLCPP_INFO`** は ROS2 用のログ出力マクロです．`printf` や `std::cout` の代わりに使います．タイムスタンプやノード名も自動でつきます．

> **ROS1 との違い**:
> - `#include <ros/ros.h>` → `#include "rclcpp/rclcpp.hpp"`
> - `ros::init(argc, argv, "hello_node")` → `rclcpp::init(argc, argv)` + `rclcpp::Node::make_shared("hello_node")`
> - `ROS_INFO(...)` → `RCLCPP_INFO(node->get_logger(), ...)`
> - `ros::shutdown()` → `rclcpp::shutdown()`

### CMakeLists.txt に実行ファイルの設定を追加

`~/ros2_ws/src/ros_tutorial/CMakeLists.txt` を開き，`ament_package()` の**前**に以下を追加します：

```cmake
add_executable(hello_node src/hello.cpp)
ament_target_dependencies(hello_node rclcpp std_msgs)

install(TARGETS hello_node
  DESTINATION lib/${PROJECT_NAME})
```

**意味：**
- `add_executable(hello_node src/hello.cpp)` → `src/hello.cpp` をコンパイルして `hello_node` という実行ファイルを作る
- `ament_target_dependencies(hello_node rclcpp std_msgs)` → 依存ライブラリをリンクする（ROS1 の `target_link_libraries(... ${catkin_LIBRARIES})` に相当）
- `install(TARGETS hello_node DESTINATION lib/${PROJECT_NAME})` → ビルドした実行ファイルを `install/` 以下にインストールする（**必須**：これがないと `ros2 run` で見つからない）

> **`install()` が必要な理由**: `colcon build` のビルド結果は `build/` 以下に置かれますが，`ros2 run` は `install/` 以下を探します．`install()` を書くことで実行ファイルが正しい場所にコピーされます．

### ビルドと実行

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select ros_tutorial
source install/setup.bash
```

```bash
ros2 run ros_tutorial hello_node
```

出力例：
```
[INFO] [1234567890.123] [hello_node]: Hello, ROS2!
```

`ros2 run <パッケージ名> <ノード名>` でノードを起動します（ROS1 の `rosrun` に相当）．

> **ROS2 では `roscore` 不要**：ROS1 では `roscore` を先に起動する必要がありましたが，ROS2 では不要です．

---

## まとめ

パッケージを作る基本的な流れをまとめます．

```
1. ros2 pkg create でパッケージ作成
2. src/ に .cpp ファイルを書く
3. CMakeLists.txt に add_executable / ament_target_dependencies / install を追加
4. colcon build --symlink-install でビルド
5. source install/setup.bash
6. ros2 run で実行
```

この流れは以降の章でも同じです．

---

[→ 4章: Publisher / Subscriber](04_publisher_subscriber.md)
