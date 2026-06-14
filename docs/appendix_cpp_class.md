# 付録C: C++ クラス入門

ROS2 のノードをクラスで書くために必要な C++ クラスの基礎をまとめます．  
[14章: クラスを使った ROS2 プログラミング](14_ros2_with_class.md) の前に読んでおくことを推奨します．

---

## クラスとは

クラスは「**データ（変数）と処理（関数）を一つにまとめた設計図**」です．

設計図から実際のモノを作ることを **インスタンス化（オブジェクトの生成）** と呼びます．

```
クラス      ＝ 設計図（ロボットの仕様書）
オブジェクト ＝ 実際のモノ（その仕様書で作ったロボット）
```

1 つの設計図から複数のロボットを作れるように，1 つのクラスから複数のオブジェクトを生成できます．

---

## 最初の例

まずコード全体を見て，それからパーツごとに説明します．

```cpp
#include <iostream>
#include <string>

class Robot
{
public:
    // コンストラクタ
    Robot(std::string name, int speed)
        : name_(name), speed_(speed), distance_(0)
    {
        std::cout << name_ << " が起動しました" << std::endl;
    }

    // デストラクタ
    ~Robot()
    {
        std::cout << name_ << " がシャットダウンしました" << std::endl;
    }

    // メンバ関数（メソッド）
    void move(int steps)
    {
        distance_ += steps * speed_;
        std::cout << name_ << " が " << steps << " 歩移動（累計: "
                  << distance_ << "）" << std::endl;
    }

    void showStatus() const
    {
        std::cout << "名前: " << name_
                  << "  速度: " << speed_
                  << "  累計距離: " << distance_ << std::endl;
    }

private:
    std::string name_;
    int         speed_;
    int         distance_;
};

int main()
{
    Robot robot1("Taro", 10);
    Robot robot2("Hanako", 5);

    robot1.move(3);
    robot2.move(5);
    robot1.move(2);

    robot1.showStatus();
    robot2.showStatus();

    return 0;
}
```

出力：
```
Taro が起動しました
Hanako が起動しました
Taro が 3 歩移動（累計: 30）
Hanako が 5 歩移動（累計: 25）
Taro が 2 歩移動（累計: 50）
名前: Taro  速度: 10  累計距離: 50
名前: Hanako  速度: 5  累計距離: 25
Hanako がシャットダウンしました
Taro がシャットダウンしました
```

デストラクタは**生成の逆順**で呼ばれます（`robot2` → `robot1`）．

---

## パーツごとの解説

### クラスの定義

```cpp
class Robot
{
    // ここに中身を書く
};  // ← セミコロンが必要！
```

クラス名は慣習として**大文字始まり**にします（`Robot`・`MyClass`・`SensorNode` など）．

---

### public と private

クラスの中身は **公開（public）** か **非公開（private）** に分けます．

```cpp
class Robot
{
public:
    // ← ここは外から使える
    void move(int steps) { ... }

private:
    // ← ここは外から使えない（クラスの内部だけで使う）
    int distance_;
};
```

```cpp
Robot robot("Taro", 10);
robot.move(3);       // OK（public なので外から呼べる）
robot.distance_;     // エラー！（private なので外からアクセス不可）
```

**なぜ private にするのか？**
- 内部の変数を外から直接変えられると，予期しないバグが起きやすい
- 「このクラスを使う人は `move()` だけ呼べばよい」という設計にできる

---

### メンバ変数

クラスが「持っているデータ」です．

```cpp
private:
    std::string name_;
    int         speed_;
    int         distance_;
```

慣習として末尾に `_` をつけてメンバ変数だとわかるようにします（必須ではありません）．

---

### コンストラクタ

**オブジェクトを生成したときに自動で呼ばれる特別な関数**です．

- クラス名と同じ名前
- 戻り値の型は書かない

```cpp
Robot(std::string name, int speed)
    : name_(name), speed_(speed), distance_(0)    // 初期化リスト
{
    std::cout << name_ << " が起動しました" << std::endl;
}
```

`: name_(name), speed_(speed), distance_(0)` の部分を**初期化リスト**と呼びます．  
メンバ変数をここで初期化するのは C++ の慣習です（`{ }` の中で代入するより効率的）．

---

### デストラクタ

**オブジェクトが破棄されるときに自動で呼ばれる特別な関数**です．

- クラス名の前に `~` をつけた名前
- 戻り値の型・引数ともに書かない

```cpp
~Robot()    // デストラクタ
{
    std::cout << name_ << " がシャットダウンしました" << std::endl;
}
```

上の例では `main` の `}` を抜けた瞬間に自動で呼ばれ，`Hanako がシャットダウンしました` → `Taro がシャットダウンしました` と出力されます．

`delete` を使う場合はそのタイミングで呼ばれます：

```cpp
Robot* ptr = new Robot("Jiro", 8);
ptr->move(2);
delete ptr;   // ← ここでデストラクタが呼ばれる
```

**ROS2 との関連**

ROS2 のノードをクラスで書く場合，デストラクタはあまり明示的に書きません．`rclcpp::Publisher`・`rclcpp::Subscription`・`rclcpp::TimerBase` などは `SharedPtr`（スマートポインタ）で管理されており，クラスが破棄されると自動でリソースを解放してくれるからです．

---

### メンバ関数（メソッド）

クラスが「できること」を定義した関数です．

```cpp
void move(int steps)
{
    distance_ += steps * speed_;
    // ← メンバ変数に直接アクセスできる
}
```

クラスの中（`{ }` 内）の関数は，同じクラスのメンバ変数に `this->` なしで直接アクセスできます．

#### `const` メンバ関数

```cpp
void showStatus() const { ... }
```

`const` をつけると「この関数はメンバ変数を変更しない」という約束になります．読み取り専用の関数には `const` をつける習慣があります．

---

### オブジェクトの生成と使い方

```cpp
// 生成（コンストラクタに引数を渡す）
Robot robot1("Taro", 10);

// メンバ関数の呼び出し（ドット演算子）
robot1.move(3);
robot1.showStatus();
```

ポインタ経由の場合はアロー演算子 `->` を使います：

```cpp
Robot* ptr = new Robot("Jiro", 8);
ptr->move(2);
delete ptr;   // new したら必ず delete
```

---

## もう少し現実的な例：カウンター

次の章で使うパターンに近い，「状態を持つ」クラスの例です．

```cpp
#include <iostream>

class Counter
{
public:
    Counter() : count_(0) {}

    void increment() { count_++; }
    void reset()     { count_ = 0; }
    int  getCount() const { return count_; }

private:
    int count_;
};

int main()
{
    Counter c;
    c.increment();
    c.increment();
    c.increment();
    std::cout << c.getCount() << std::endl;  // 3

    c.reset();
    std::cout << c.getCount() << std::endl;  // 0

    return 0;
}
```

**状態（`count_`）をクラスの中で管理** できるのがクラスの強みです．  
グローバル変数を使わずに済むので，複数のカウンターを独立して動かすこともできます．

---

## クラスを複数のファイルに分ける

大きなプログラムでは，クラスをヘッダファイル（`.h`）とソースファイル（`.cpp`）に分けて書きます．

**Robot.h**（クラスの宣言）：
```cpp
#pragma once
#include <string>

class Robot
{
public:
    Robot(std::string name, int speed);
    void move(int steps);
    void showStatus() const;

private:
    std::string name_;
    int         speed_;
    int         distance_;
};
```

**Robot.cpp**（クラスの実装）：
```cpp
#include "Robot.h"
#include <iostream>

Robot::Robot(std::string name, int speed)
    : name_(name), speed_(speed), distance_(0)
{
    std::cout << name_ << " が起動しました" << std::endl;
}

void Robot::move(int steps)
{
    distance_ += steps * speed_;
    std::cout << name_ << " が " << steps << " 歩移動（累計: "
              << distance_ << "）" << std::endl;
}

void Robot::showStatus() const
{
    std::cout << "名前: " << name_
              << "  速度: " << speed_
              << "  累計距離: " << distance_ << std::endl;
}
```

`Robot::move(...)` の `Robot::` は「Robot クラスの `move` 関数」という意味です．

---

## ROS2 で登場するクラス関連の構文

ROS2 のコードを読むと，見慣れない書き方がいくつか出てきます．付録として整理しておきます．

> テンプレート・名前空間・スマートポインタなど，クラス以外のC++の仕組みについては [付録D: ROS2のコードをC++として読む](appendix_ros_cpp.md) を参照してください．

### `this` キーワード

クラスの中で使える特別なキーワードで，「自分自身のオブジェクトへのポインタ」です．

```cpp
class Listener : public rclcpp::Node
{
public:
    Listener() : Node("listener")
    {
        // ROS2 にコールバックを登録するとき，
        // 「どのオブジェクトのメンバ関数か」を this で教える
        sub_ = this->create_subscription<std_msgs::msg::String>(
            "chatter", 10,
            std::bind(&Listener::callback, this, std::placeholders::_1));
    }
    ...
};
```

### メンバ関数ポインタ `&ClassName::functionName`

`&Listener::callback` は「`Listener` クラスの `callback` 関数のアドレス」という意味です．  
ROS2 がコールバックを呼び出す際に必要な情報です．

### `std::bind` と `std::placeholders::_1`

メンバ関数をコールバックとして登録するときに使います（ROS1 の `boost::bind` に相当）．

```cpp
std::bind(&MyClass::callback, this, std::placeholders::_1)
//             ↑                ↑          ↑
//     メンバ関数のアドレス  オブジェクト  「第1引数はあとで渡される」
```

`std::placeholders::_1` は「ここには呼び出し時に渡される引数が入る」を意味します．引数が 2 つある場合は `_2` も使います（サービスコールバックなど）：

```cpp
std::bind(&MyNode::service_callback, this,
          std::placeholders::_1, std::placeholders::_2)
```

---

[→ 14章: クラスを使った ROS2 プログラミング](14_ros2_with_class.md) ｜ [→ 付録D: ROS2のコードをC++として読む](appendix_ros_cpp.md)
