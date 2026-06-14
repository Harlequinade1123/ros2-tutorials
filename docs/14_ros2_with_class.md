# 14章: クラスを使った ROS2 プログラミング

4 章で書いた talker・listener をクラスで書き直します．ROS2 では **`rclcpp::Node` を継承したクラス** として書くのが標準スタイルです．

> クラスの基本（コンストラクタ・メンバ変数・public/private など）は **[付録C: C++ クラス入門](appendix_cpp_class.md)** で詳しく説明しています．

---

## なぜクラスで書くのか

4 章の talker はシンプルでしたが，ノードが複雑になると：
- Publisher・Subscriber・Timer など多くのオブジェクトが増える
- コールバック関数が増えてきたとき，必要なデータをどう渡すか困る
- グローバル変数が増えてコードが追いにくくなる

**クラスを使うと：**
- 関連するデータと処理をひとまとめにできる
- コールバックからメンバ変数に自然にアクセスできる
- `rclcpp::Node` の全機能を `this->` で使える

---

## クラス版 talker（class_talker.cpp）

`~/ros2_ws/src/ros_tutorial/src/class_talker.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class Talker : public rclcpp::Node   // rclcpp::Node を継承
{
public:
    Talker() : Node("talker"), count_(0)
    {
        // コンストラクタで Publisher と Timer を初期化
        pub_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);

        // 100ms（10Hz）ごとに timer_callback を呼ぶタイマー
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(100),
            std::bind(&Talker::timer_callback, this));
    }

private:
    void timer_callback()
    {
        auto msg = std_msgs::msg::String();
        msg.data = "hello world " + std::to_string(count_++);
        RCLCPP_INFO(this->get_logger(), "%s", msg.data.c_str());
        pub_->publish(msg);
    }

    rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    int count_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<Talker>());   // クラスのインスタンスを spin
    rclcpp::shutdown();
    return 0;
}
```

### ポイント

| コード | 意味 |
|--------|------|
| `class Talker : public rclcpp::Node` | `rclcpp::Node` を継承したクラスを作る |
| `Talker() : Node("talker"), count_(0)` | コンストラクタでノード名 "talker" を指定して親クラスを初期化 |
| `this->create_publisher<>(...)` | 継承したノードの機能を `this->` で呼ぶ |
| `this->create_wall_timer(...)` | タイマーを作る（`ros::Timer` に相当）|
| `std::bind(&Talker::timer_callback, this)` | メンバ関数をコールバックとして登録 |
| `this->get_logger()` | ノードのロガーを取得（`RCLCPP_INFO` で使う）|
| `rclcpp::spin(std::make_shared<Talker>())` | `main()` は 1 行で OK |

> **ROS1 との違い**:
> - `ros::NodeHandle nh_` をメンバに持つ → `rclcpp::Node` を継承
> - `nh_.createTimer(...)` → `this->create_wall_timer(...)`
> - `ros::Rate` + `while` ループ → タイマーコールバックが推奨

---

## クラス版 listener（class_listener.cpp）

`~/ros2_ws/src/ros_tutorial/src/class_listener.cpp` を作成：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class Listener : public rclcpp::Node
{
public:
    Listener() : Node("listener"), receive_count_(0)
    {
        // メンバ関数をコールバックとして登録
        sub_ = this->create_subscription<std_msgs::msg::String>(
            "chatter", 10,
            std::bind(&Listener::chatter_callback, this, std::placeholders::_1));
    }

private:
    void chatter_callback(const std_msgs::msg::String::SharedPtr msg)
    {
        receive_count_++;
        RCLCPP_INFO(this->get_logger(), "受信 [%d]: [%s]",
                    receive_count_, msg->data.c_str());
    }

    rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
    int receive_count_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<Listener>());
    rclcpp::shutdown();
    return 0;
}
```

### `std::bind` + `std::placeholders::_1` について

```cpp
std::bind(&Listener::chatter_callback, this, std::placeholders::_1)
```

| 部分 | 意味 |
|------|------|
| `&Listener::chatter_callback` | `Listener` クラスの `chatter_callback` 関数のアドレス |
| `this` | 「このオブジェクト」（どのインスタンスのメンバ関数か）|
| `std::placeholders::_1` | 「第1引数はコールバック呼び出し時に渡される」（msg に対応）|

`std::bind` の詳細は **[付録C](appendix_cpp_class.md)** を参照してください．

ラムダ式でも同様に書けます：

```cpp
sub_ = this->create_subscription<std_msgs::msg::String>(
    "chatter", 10,
    [this](const std_msgs::msg::String::SharedPtr msg) {
        chatter_callback(msg);
    });
```

---

## CMakeLists.txt に追加

```cmake
add_executable(class_talker src/class_talker.cpp)
ament_target_dependencies(class_talker rclcpp std_msgs)

add_executable(class_listener src/class_listener.cpp)
ament_target_dependencies(class_listener rclcpp std_msgs)

install(TARGETS class_talker class_listener DESTINATION lib/${PROJECT_NAME})
```

---

## ビルドと実行

```bash
cd ~/ros2_ws && colcon build --symlink-install --packages-select ros_tutorial
source install/setup.bash
```

**ターミナル 1：**
```bash
ros2 run ros_tutorial class_talker
```

**ターミナル 2：**
```bash
ros2 run ros_tutorial class_listener
```

---

## Publisher と Subscriber を両方持つクラス

実際のノードでは，Publisher と Subscriber を両方持つことが多いです．

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/float64.hpp"

class FilterNode : public rclcpp::Node
{
public:
    FilterNode() : Node("filter_node"), sum_(0.0), count_(0)
    {
        sub_ = this->create_subscription<std_msgs::msg::Float64>(
            "raw_value", 10,
            std::bind(&FilterNode::input_callback, this, std::placeholders::_1));
        pub_ = this->create_publisher<std_msgs::msg::Float64>("filtered_value", 10);
    }

private:
    void input_callback(const std_msgs::msg::Float64::SharedPtr msg)
    {
        sum_ += msg->data;
        count_++;

        auto out = std_msgs::msg::Float64();
        out.data = sum_ / count_;

        pub_->publish(out);
        RCLCPP_INFO(this->get_logger(),
                    "入力: %.2f  出力（平均）: %.2f", msg->data, out.data);
    }

    rclcpp::Subscription<std_msgs::msg::Float64>::SharedPtr sub_;
    rclcpp::Publisher<std_msgs::msg::Float64>::SharedPtr    pub_;
    double sum_;
    int    count_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<FilterNode>());
    rclcpp::shutdown();
    return 0;
}
```

このパターン（**受け取って → 処理して → 送る**）は ROS2 で最もよく使われる構造です．

---

## クラスを使った ROS2 ノードのまとめ

```
クラスを使った ROS2 ノードの典型的な構造：

class MyNode : public rclcpp::Node
{
public:
    MyNode() : Node("my_node")
    {
        // Publisher・Subscription・Timer の初期化
        pub_   = this->create_publisher<...>("topic", 10);
        sub_   = this->create_subscription<...>("topic", 10, ...);
        timer_ = this->create_wall_timer(std::chrono::milliseconds(100), ...);
    }

private:
    // コールバック関数たち
    void timer_callback()   { ... }
    void sub_callback(...)  { ... }

    rclcpp::Publisher<...>::SharedPtr    pub_;
    rclcpp::Subscription<...>::SharedPtr sub_;
    rclcpp::TimerBase::SharedPtr         timer_;
    // その他のメンバ変数
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<MyNode>());
    rclcpp::shutdown();
    return 0;
}
```

この構造を基本テンプレートとして覚えておくと，どんなノードを書くときも迷いにくくなります．

---

## サービスをクラスで書く

```cpp
#include "rclcpp/rclcpp.hpp"
#include "ros_tutorial/srv/add_two_ints.hpp"

class AddTwoIntsServer : public rclcpp::Node
{
public:
    AddTwoIntsServer() : Node("add_two_ints_server")
    {
        service_ = this->create_service<ros_tutorial::srv::AddTwoInts>(
            "add_two_ints",
            std::bind(&AddTwoIntsServer::add, this,
                      std::placeholders::_1, std::placeholders::_2));
        RCLCPP_INFO(this->get_logger(), "add_two_ints サービスの準備完了");
    }

private:
    void add(const std::shared_ptr<ros_tutorial::srv::AddTwoInts::Request> request,
             std::shared_ptr<ros_tutorial::srv::AddTwoInts::Response> response)
    {
        response->sum = request->a + request->b;
        RCLCPP_INFO(this->get_logger(), "a=%ld, b=%ld → sum=%ld",
                    request->a, request->b, response->sum);
    }

    rclcpp::Service<ros_tutorial::srv::AddTwoInts>::SharedPtr service_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<AddTwoIntsServer>());
    rclcpp::shutdown();
    return 0;
}
```

---

## アクションをクラスで書く

```cpp
#include "rclcpp/rclcpp.hpp"
#include "rclcpp_action/rclcpp_action.hpp"
#include "ros_tutorial/action/count_down.hpp"

using CountDown = ros_tutorial::action::CountDown;
using GoalHandle = rclcpp_action::ServerGoalHandle<CountDown>;

class CountDownServer : public rclcpp::Node
{
public:
    CountDownServer() : Node("count_down_server")
    {
        action_server_ = rclcpp_action::create_server<CountDown>(
            this, "count_down",
            std::bind(&CountDownServer::handle_goal,     this, std::placeholders::_1, std::placeholders::_2),
            std::bind(&CountDownServer::handle_cancel,   this, std::placeholders::_1),
            std::bind(&CountDownServer::handle_accepted, this, std::placeholders::_1));
        RCLCPP_INFO(this->get_logger(), "CountDown アクションサーバー準備完了（クラス版）");
    }

private:
    rclcpp_action::GoalResponse handle_goal(
        const rclcpp_action::GoalUUID &,
        std::shared_ptr<const CountDown::Goal> goal)
    {
        RCLCPP_INFO(this->get_logger(), "ゴール受信: target=%d", goal->target);
        return rclcpp_action::GoalResponse::ACCEPT_AND_EXECUTE;
    }

    rclcpp_action::CancelResponse handle_cancel(const std::shared_ptr<GoalHandle>)
    {
        RCLCPP_INFO(this->get_logger(), "キャンセルリクエスト受信");
        return rclcpp_action::CancelResponse::ACCEPT;
    }

    void handle_accepted(const std::shared_ptr<GoalHandle> goal_handle)
    {
        std::thread{std::bind(&CountDownServer::execute, this, std::placeholders::_1),
                    goal_handle}.detach();
    }

    void execute(const std::shared_ptr<GoalHandle> goal_handle)
    {
        auto result   = std::make_shared<CountDown::Result>();
        auto feedback = std::make_shared<CountDown::Feedback>();
        const auto goal = goal_handle->get_goal();

        rclcpp::Rate rate(1.0);
        for (int i = goal->target; i >= 0; --i) {
            if (goal_handle->is_canceling()) {
                result->message = "キャンセルされました";
                goal_handle->canceled(result);
                return;
            }
            feedback->remaining = i;
            goal_handle->publish_feedback(feedback);
            RCLCPP_INFO(this->get_logger(), "残り: %d", i);
            rate.sleep();
        }
        result->message = "カウントダウン完了！";
        goal_handle->succeed(result);
        RCLCPP_INFO(this->get_logger(), "完了");
    }

    rclcpp_action::Server<CountDown>::SharedPtr action_server_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<CountDownServer>());
    rclcpp::shutdown();
    return 0;
}
```

5章で作ったクライアント（`count_down_client`）はそのまま使えます．

---

## パラメータ動的変更のクラス版

8章では**ラムダ**で `add_on_set_parameters_callback` を登録しました．クラスを使うと，コールバックを**メンバ関数**として書けるため，メンバ変数への反映も自然に書けます．

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"
#include "rcl_interfaces/msg/set_parameters_result.hpp"

class ParamNode : public rclcpp::Node
{
public:
    ParamNode() : Node("param_node")
    {
        this->declare_parameter("robot_name",   std::string("robot_A"));
        this->declare_parameter("publish_rate", 2);

        robot_name_   = this->get_parameter("robot_name").as_string();
        publish_rate_ = this->get_parameter("publish_rate").as_int();

        // メンバ関数をコールバックとして登録
        cb_handle_ = this->add_on_set_parameters_callback(
            std::bind(&ParamNode::on_set_parameters, this, std::placeholders::_1));

        pub_   = this->create_publisher<std_msgs::msg::String>("status", 10);
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(1000 / publish_rate_),
            std::bind(&ParamNode::timer_callback, this));

        RCLCPP_INFO(this->get_logger(), "ParamNode 起動");
    }

private:
    rcl_interfaces::msg::SetParametersResult
    on_set_parameters(const std::vector<rclcpp::Parameter> & params)
    {
        rcl_interfaces::msg::SetParametersResult result;
        result.successful = true;

        for (const auto & p : params) {
            if (p.get_name() == "publish_rate") {
                if (p.as_int() <= 0) {
                    result.successful = false;
                    result.reason     = "publish_rate は正の値にしてください";
                    return result;
                }
                // 受理されたらメンバ変数を更新
                publish_rate_ = p.as_int();
                timer_->cancel();
                timer_ = this->create_wall_timer(
                    std::chrono::milliseconds(1000 / publish_rate_),
                    std::bind(&ParamNode::timer_callback, this));
                RCLCPP_INFO(this->get_logger(), "publish_rate 変更 → %d Hz", publish_rate_);
            }
            if (p.get_name() == "robot_name") {
                robot_name_ = p.as_string();
                RCLCPP_INFO(this->get_logger(), "robot_name 変更 → %s", robot_name_.c_str());
            }
        }
        return result;
    }

    void timer_callback()
    {
        auto msg = std_msgs::msg::String();
        msg.data = robot_name_ + " is running (" + std::to_string(publish_rate_) + " Hz)";
        pub_->publish(msg);
        RCLCPP_INFO(this->get_logger(), "%s", msg.data.c_str());
    }

    rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    rclcpp::node_interfaces::OnSetParametersCallbackHandle::SharedPtr cb_handle_;

    std::string robot_name_;
    int         publish_rate_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<ParamNode>());
    rclcpp::shutdown();
    return 0;
}
```

### 8章のラムダ版との違い

| | 8章（ラムダ版） | 14章（クラス版）|
|--|--------------|--------------|
| コールバックの書き方 | ラムダ式 | メンバ関数（`std::bind`）|
| メンバ変数への反映 | ループで毎回 `get_parameter` | コールバック内でメンバ変数を直接更新 |
| タイマーの再生成 | 不可（ループが Rate を毎回作る）| 可能（`timer_->cancel()` + 新規生成）|

ハンドルの型 `rclcpp::node_interfaces::OnSetParametersCallbackHandle::SharedPtr` を**メンバ変数で保持**することが重要です．ローカル変数に置くとスコープを抜けた瞬間にコールバックが解除されてしまいます．

---

[→ 15章: ROS2 総合演習](15_ros2_exercises.md)
