# ROS2 入門チュートリアル

C++ の基礎知識はあるが，ROS2 は初めてという方を対象にした入門教材です．

## 動作環境

| 項目 | バージョン |
|------|-----------|
| OS | Ubuntu 22.04 LTS |
| ROS | Humble Hawksbill（ROS2 LTS） |
| 言語 | C++17 |

## 目次

| # | 章タイトル | 概要 |
|---|-----------|------|
| 1 | [ROS2 とは](docs/01_introduction.md) | ROS2 の概要・基本概念 |
| 2 | [環境構築](docs/02_setup.md) | インストール・ワークスペース作成 |
| 3 | [最初のパッケージを作る](docs/03_first_package.md) | パッケージ作成・ビルドの流れ |
| 4 | [Publisher / Subscriber](docs/04_publisher_subscriber.md) | ROS2 の基本通信（トピック） |
| 5 | [サービス](docs/05_service.md) | リクエスト・レスポンス型の通信 |
| 6 | [アクション通信](docs/06_action.md) | 長時間処理のフィードバックとキャンセル |
| 7 | [カスタムメッセージ](docs/07_custom_messages.md) | 独自のメッセージ型を定義する |
| 8 | [パラメータ](docs/08_parameters.md) | 実行時に値を設定・取得する |
| 9 | [launch ファイル](docs/09_launch_files.md) | 複数ノードをまとめて起動する |
| 10 | [RViz2](docs/10_rviz2.md) | データを 3D で視覚化する |
| 11 | [rosbag2](docs/11_rosbag2.md) | トピックの記録と再生 |
| 12 | [tf2](docs/12_tf2.md) | 座標フレーム間の変換を管理する |
| 13 | [C++ クラス入門](docs/13_cpp_class_basics.md) | クラスの概要（詳細は付録C） |
| 14 | [クラスを使った ROS2 プログラミング](docs/14_ros2_with_class.md) | ノードをクラスで書く・サービス・アクションのクラス版 |
| 15 | [ROS2 総合演習](docs/15_ros2_exercises.md) | 2 ノードシステムの構築から，カスタムメッセージ・アクション・ウォッチドッグなどの発展演習まで |
| 付録A | [コマンドリファレンス](docs/appendix_commands.md) | Ubuntu・ROS2 コマンド早見表 |
| 付録B | [ament_cmake とビルドの仕組み](docs/appendix_cmake.md) | コンパイルの基礎・CMake・ament_cmake の違い |
| 付録C | [C++ クラス入門](docs/appendix_cpp_class.md) | クラス・コンストラクタ・メンバ変数の基礎 |
| 付録D | [ROS2 のコードを C++ として読む](docs/appendix_ros_cpp.md) | 名前空間・テンプレート・スマートポインタ・ラムダ式 |

## 読み進め方

- **1 章から順番に**読み進めることを推奨します
- 詰まったときは `source` コマンドを実行したか，ワークスペースをビルドしたかを先に確認しましょう

## パッケージ名について

このチュートリアルでは `ros_tutorial` というパッケージ名を使います．

---

> **ROS1 との主な違い**: ROS2 では `roscore` が不要，ビルドシステムが `colcon`，launch ファイルが Python になりました．ノードは `rclcpp::Node` を継承するクラスとして書きます．概念（ノード・トピック・サービス・アクション等）は共通しています．
