# 10章: RViz2 ── データを視覚化する

ROS2 の各種データをリアルタイムに 3D 表示するツール **RViz2** を学びます．

---

## RViz2 とは

**RViz2**（ROS2 Visualization）は ROS2 標準の 3D ビジュアライザです．

| 用途 | 例 |
|------|----|
| センサーデータの確認 | LiDAR スキャン・点群・カメラ画像 |
| ロボットの状態監視 | 位置・姿勢・座標フレーム（TF）|
| アルゴリズムのデバッグ | 経路・コストマップ・障害物 |
| 独自データの表示 | マーカー（矢印・球・テキストなど）|

ターミナルの `RCLCPP_INFO` では数値しか見えませんが，RViz2 を使うと**空間的なデータを瞬時に把握**できます．

---

## 起動方法

```bash
rviz2
```

> RViz2 自体も「トピックを受け取って表示するノード」です．

---

## 画面構成

```
┌─────────────────────────────────────────────────────────┐
│  [Toolbar]  Interact / Move Camera / ...                │
├──────────────────┬────────────────────────┬─────────────┤
│                  │                        │             │
│  Displays        │   3D Viewport          │  Views      │
│  Panel           │                        │  Panel      │
│                  │  <- Main view          │  (Camera)   │
│  [Add]           │                        │             │
│  [Remove]        │                        │             │
│                  │                        │             │
├──────────────────┴────────────────────────┴─────────────┤
│  [Time]                                                 │
└─────────────────────────────────────────────────────────┘
```

| パネル | 役割 |
|--------|------|
| **Displays** | 表示する内容（Display）を追加・削除・設定する |
| **3D Viewport** | データが描画されるメインの画面 |
| **Views** | 視点（カメラ）の種類と設定 |

---

## 基本的なマウス操作

| 操作 | 動作 |
|------|------|
| 左ドラッグ | 視点を回転 |
| 中ドラッグ（または Shift+左）| 視点を平行移動 |
| スクロール | ズームイン・アウト |
| `0` キー | 視点をリセット |

---

## Fixed Frame の設定

Displays パネル上部の「**Fixed Frame**」は，表示の基準座標フレームを指定します．

| 状況 | よく使う Fixed Frame |
|------|---------------------|
| 地図なし・絶対位置基準 | `map` |
| ロボット中心で表示 | `base_link` |
| オドメトリ基準 | `odom` |

> Fixed Frame に存在しないフレームを設定すると，Displays の項目に赤いエラーが表示されます．

---

## 実際に試す：Marker を表示する

**Marker** は，C++ コードから RViz2 に好きな図形（球・矢印・テキストなど）を描くためのメッセージ型です．

まず `visualization_msgs` の依存を追加します．

### CMakeLists.txt の変更

```cmake
find_package(visualization_msgs REQUIRED)

add_executable(marker_publisher src/marker_publisher.cpp)
ament_target_dependencies(marker_publisher rclcpp visualization_msgs)

install(TARGETS marker_publisher DESTINATION lib/${PROJECT_NAME})
```

### package.xml の変更

```xml
<depend>visualization_msgs</depend>
```

### marker_publisher.cpp

`~/ros2_ws/src/ros_tutorial/src/marker_publisher.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "visualization_msgs/msg/marker.hpp"

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    auto node = rclcpp::Node::make_shared("marker_publisher");

    auto pub = node->create_publisher<visualization_msgs::msg::Marker>(
        "visualization_marker", 1);

    rclcpp::Rate rate(1);

    while (rclcpp::ok())
    {
        auto marker = visualization_msgs::msg::Marker();

        marker.header.frame_id = "map";
        marker.header.stamp    = node->get_clock()->now();   // 現在時刻
        marker.ns              = "tutorial";
        marker.id              = 0;

        // 表示タイプ：SPHERE（球）
        marker.type   = visualization_msgs::msg::Marker::SPHERE;
        marker.action = visualization_msgs::msg::Marker::ADD;

        // 位置・向き
        marker.pose.position.x    = 0.0;
        marker.pose.position.y    = 0.0;
        marker.pose.position.z    = 0.0;
        marker.pose.orientation.w = 1.0;

        // サイズ [m]
        marker.scale.x = 0.5;
        marker.scale.y = 0.5;
        marker.scale.z = 0.5;

        // 色（RGBA，0.0〜1.0）
        marker.color.r = 1.0f;
        marker.color.g = 0.0f;
        marker.color.b = 0.0f;
        marker.color.a = 1.0f;

        pub->publish(marker);

        rclcpp::spin_some(node);
        rate.sleep();
    }

    rclcpp::shutdown();
    return 0;
}
```

> **`node->get_clock()->now()` について**: ROS2 の現在時刻を返す方法です．ROS1 の `ros::Time::now()` に相当します．

### ビルドと実行

**ターミナル 1（marker_publisher）：**
```bash
cd ~/ros2_ws && colcon build --packages-select ros_tutorial
source install/setup.bash
ros2 run ros_tutorial marker_publisher
```

**ターミナル 2（RViz2）：**
```bash
rviz2
```

### RViz2 での確認手順

1. Fixed Frame を `map` に変更
2. **[Add]** → **「By topic」タブ** → `/visualization_marker` → **Marker** → **[OK]**

原点（0, 0, 0）に赤い球が表示されます．

### 複数の Marker を出す

`marker.id` を変えると同時に複数の図形を表示できます．また `marker.type` を変えると形が変わります．

| 定数 | 形状 |
|------|------|
| `SPHERE` | 球 |
| `CUBE` | 直方体 |
| `CYLINDER` | 円柱 |
| `ARROW` | 矢印 |
| `LINE_STRIP` | 折れ線 |
| `TEXT_VIEW_FACING` | テキスト（常に正面を向く）|

---

## 主な Display タイプ一覧

| タイプ | 対応メッセージ型 | 用途 |
|--------|----------------|------|
| **Grid** | なし | グリッドライン（床面の目安）|
| **TF** | なし | 座標フレーム間の関係を矢印で表示 |
| **Axes** | なし | 特定フレームを XYZ 軸で表示 |
| **Marker** | `visualization_msgs/msg/Marker` | 自由な図形を描く |
| **Odometry** | `nav_msgs/msg/Odometry` | 位置・速度を矢印で表示 |
| **Path** | `nav_msgs/msg/Path` | 軌跡を線で表示 |
| **LaserScan** | `sensor_msgs/msg/LaserScan` | LiDAR のスキャンデータ |
| **PointCloud2** | `sensor_msgs/msg/PointCloud2` | 3D 点群 |
| **Image** | `sensor_msgs/msg/Image` | カメラ画像 |

---

## 設定の保存・読み込み

RViz2 の Displays 設定は `.rviz` ファイルに保存できます．

```bash
# GUI から保存：File → Save Config As
# GUI から読み込み：File → Open Config

# コマンドラインで設定ファイルを指定して起動
rviz2 -d ~/my_config.rviz
```

launch ファイルから設定ファイル付きで RViz2 を起動することもできます：

```python
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os

Node(
    package='rviz2',
    executable='rviz2',
    name='rviz2',
    arguments=['-d', os.path.join(
        get_package_share_directory('ros_tutorial'),
        'rviz', 'default.rviz')]
)
```

---

## rqt_graph との使い分け

| ツール | 見えるもの |
|--------|-----------|
| **rqt_graph** | ノードとトピックの**接続関係**（グラフ構造）|
| **RViz2** | トピックのデータ内容（**空間データ**）|
| **ros2 topic echo** | トピックのデータ内容（**テキスト**）|

---

[→ 11章: rosbag2](11_rosbag2.md)
