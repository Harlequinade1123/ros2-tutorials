# 付録D: ROS2のコードをC++として読む

ROS2 のノードを初めて読むと「これは ROS2 だけの特別な書き方なのか」と感じる構文が出てきます．実はそれらはすべて**標準的なC++の機能**の組み合わせです．この付録では，ROS2 のコードに頻出するパターンを C++ の観点から整理します．

---

## 早見表

| ROS2 コードで見えるもの | C++ としての正体 |
|---|---|
| `auto node = rclcpp::Node::make_shared("name")` | `shared_ptr<rclcpp::Node>` の生成 |
| `node->create_publisher<std_msgs::msg::String>(...)` | テンプレートメンバ関数の呼び出し |
| `std_msgs::msg::String` | 名前空間（`::`） + 構造体 |
| `msg.data = "hello"` | 構造体のメンバアクセス |
| `::SharedPtr` | `shared_ptr<T>` の `using` 宣言（型の別名） |
| `msg->data` | スマートポインタの `->` 演算子 |
| `std::bind(&Class::f, this, _1)` | `std::function` オブジェクトの生成 |
| `[this](auto msg){ ... }` | ラムダ式でのコールバック登録 |

---

## 名前空間 `::`

`rclcpp::Node` や `std_msgs::msg::String` の `::` は**C++の名前空間の区切り文字**です．

```cpp
rclcpp::Node              // rclcpp 名前空間の Node クラス
std_msgs::msg::String     // std_msgs::msg 名前空間の String 構造体
ros_tutorial::srv::AddTwoInts  // パッケージ名 + インターフェース種別 + 型名
```

標準ライブラリの `std::string`・`std::cout` と同じ仕組みです．  
ROS2 は型名を `<パッケージ名>::<msg|srv|action>::<型名>` の形式で統一しています．

> **ROS1 との違い**: `std_msgs::String` → `std_msgs::msg::String`（`msg` の名前空間が追加された）

---

## テンプレート `<>`

```cpp
auto pub = node->create_publisher<std_msgs::msg::String>("chatter", 10);
```

`<std_msgs::msg::String>` の部分はC++の**テンプレート引数**です．「どの型のメッセージを扱うか」を関数に伝えています．

```cpp
// 標準ライブラリでも同じ書き方
std::vector<int>         v;  // int 型の vector
std::vector<std::string> s;  // string 型の vector
```

サービス・アクションでも同じパターンが出てきます：

```cpp
node->create_service<ros_tutorial::srv::AddTwoInts>("add_two_ints", ...)
rclcpp_action::create_server<ros_tutorial::action::CountDown>(...)
```

---

## メッセージ型は自動生成された構造体

`.msg` ファイルは**C++の構造体にビルド時に自動変換**されます．

`std_msgs/String.msg`（定義）：
```
string data
```

ビルド時に次のような構造体が生成されます（概念的な表現）：
```cpp
namespace std_msgs {
  namespace msg {
    struct String {
      std::string data;
    };
  }
}
```

だから `msg.data = "hello"` は普通の**構造体メンバアクセス**です．`.srv` や `.action` ファイルも同様に構造体へ変換されます．

---

## `SharedPtr` とスマートポインタ

コールバックの引数やメンバ変数に出てくる型：

```cpp
// コールバック引数
void callback(const std_msgs::msg::String::SharedPtr msg)
{
    msg->data;   // ドット「.」ではなくアロー「->」を使う
}

// メンバ変数
rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
```

`SharedPtr` は `std::shared_ptr<T>` の **using 宣言（型の別名）** です．ROS2 が生成するコードの中で次のように定義されています：

```cpp
using SharedPtr = std::shared_ptr<std_msgs::msg::String>;
```

`shared_ptr` は**スマートポインタ**の一種で，生ポインタ（`T*`）と同様に `->` でメンバにアクセスします．`delete` を書かなくても自動でメモリを解放してくれます．

```
       生ポインタ：     T*         ptr;   ptr->member
  スマートポインタ：shared_ptr<T>  ptr;   ptr->member  ← 同じ書き方
```

### `make_shared`

```cpp
rclcpp::spin(std::make_shared<Talker>());
```

`std::make_shared<T>(args...)` は「`T` のオブジェクトを `shared_ptr` で作る」C++ 標準の関数です．`new` を使わずにスマートポインタを生成するための推奨パターンです．

---

## コールバック登録の 2 つの方法

### 方法 1: `std::bind`（メンバ関数を使う場合）

```cpp
sub_ = this->create_subscription<std_msgs::msg::String>(
    "chatter", 10,
    std::bind(&Listener::callback, this, std::placeholders::_1));
```

`std::bind` は「関数を後で呼ぶための準備をする」C++ 標準の仕組みです．

| 引数 | 意味 |
|------|------|
| `&Listener::callback` | `Listener` クラスの `callback` 関数のアドレス |
| `this` | 「このオブジェクト」に対して呼ぶ |
| `std::placeholders::_1` | 「第1引数はコールバック呼び出し時に渡される」|

### 方法 2: ラムダ式（C++11 以降）

```cpp
sub_ = this->create_subscription<std_msgs::msg::String>(
    "chatter", 10,
    [this](const std_msgs::msg::String::SharedPtr msg) {
        callback(msg);
    });
```

`[this]` の部分は**キャプチャ**といい，「このラムダ式の中で `this` を使えるようにする」という指定です．  
どちらの方法でも動作は同じです．シンプルな処理ならラムダ，複雑な処理なら `std::bind` + メンバ関数が読みやすいです．

---

## `rclcpp::Node` の継承

```cpp
class Talker : public rclcpp::Node
{
public:
    Talker() : Node("talker")  // 親クラスのコンストラクタを呼ぶ
    {
        pub_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
    }
};
```

`public rclcpp::Node` は「`rclcpp::Node` クラスを継承する」という宣言です（**継承**）．  
継承すると，`rclcpp::Node` が持つすべての機能（`create_publisher()`・`get_logger()`・`get_clock()` など）を `this->` で使えるようになります．

```
rclcpp::Node             ←  create_publisher() などを提供する親クラス
       ↑ 継承
Talker                   ←  それらの機能を使って独自の動作を追加する子クラス
```

> **ROS1 との違い**: ROS1 では `ros::NodeHandle nh` をメンバ変数として持っていましたが，ROS2 では `rclcpp::Node` を継承するのが標準スタイルです．

---

## `auto` キーワード

```cpp
auto node = rclcpp::Node::make_shared("talker");
auto pub  = node->create_publisher<std_msgs::msg::String>("chatter", 10);
```

`auto` は**型推論**です．コンパイラが右辺から型を自動で判断します．  
`create_publisher<std_msgs::msg::String>()` の戻り値は `rclcpp::Publisher<std_msgs::msg::String>::SharedPtr` という長い型名ですが，`auto` を使うと省略できます．

---

この付録で挙げたパターンは Publisher・Subscription・Service・Action のすべてに共通して現れます．「ROS2 ならでは」ではなく「C++ を ROS2 が使っている」と読むと，コードの見通しがよくなります．

---

[← 付録C: C++ クラス入門](appendix_cpp_class.md)
