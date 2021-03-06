# fetch_xor
* atomic[meta header]
* std[meta namespace]
* atomic[meta class]
* function[meta id-type]
* cpp11[meta cpp]

```cpp
T fetch_xor(T operand, memory_order order = memory_order_seq_cst) volatile noexcept;
T fetch_xor(T operand, memory_order order = memory_order_seq_cst) noexcept;
```
* memory_order[link /reference/atomic/memory_order.md]
* memory_order_seq_cst[link /reference/atomic/memory_order.md]

## 概要
XOR演算を行う


## 要件
- `std::atomic<T*>`の場合、型`T`がオブジェクト型であること。型`T`が`void*`や関数ポインタであってはならない (C++17)


## 効果
`order`で指定されたメモリオーダーにしたがって、現在の値に`operand`をXORした値でアトミックに置き換える


## 戻り値
変更前の値が返される


## 例外
投げない


## 備考
- この関数は、`atomic`クラスの整数型に対する特殊化で定義される。
- 符号付き整数型に対しては、符号なし整数型に変換されたかのようにしたあと演算が行われ、結果は符号付き整数型になる。未定義動作はない


## 例
### 基本的な使い方
```cpp example
#include <iostream>
#include <atomic>
#include <bitset>

int main()
{
  int a = 0x0b;
  int b = 0x0e;

  std::atomic<int> x(a);

  x.fetch_xor(b);

  std::cout << std::bitset<4>(a).to_string() << std::endl;
  std::cout << std::bitset<4>(b).to_string() << std::endl;
  std::cout << std::bitset<4>(x.load()).to_string() << std::endl;
}
```
* fetch_xor[color ff0000]
* x.load()[link load.md]
* to_string()[link /reference/bitset/bitset/to_string.md]

#### 出力
```
1011
1110
0101
```

### 複数スレッドからビット複合演算を行う例 (C++14)
```cpp example
#include <iostream>
#include <atomic>
#include <thread>

int main()
{
  int ar[] = {1, 2, 3, 4, 5};
  std::atomic<int> hash{ar[0]};

  // XORで配列のハッシュ値を並列に計算する。
  // 複数スレッドでビット複合演算を呼んでも、
  // 最終的に全てのスレッドでのビット複合演算が処理された値になる
  std::thread t1 {[&hash, ar] {
    hash.fetch_xor(ar[1]);
    hash.fetch_xor(ar[2]);
  }};
  std::thread t2 {[&hash, ar] {
    hash.fetch_xor(ar[3]);
    hash.fetch_xor(ar[4]);
  }};

  t1.join();
  t2.join();

  int value = hash.load();
  std::cout << std::hex << value << std::endl;
}
```
* fetch_xor[color ff0000]
* hash.load()[link load.md]

#### 出力
```
1
```

## バージョン
### 言語
- C++11

### 処理系
- [Clang](/implementation.md#clang): 3.2
- [GCC](/implementation.md#gcc): 4.7.0
- [Visual C++](/implementation.md#visual_cpp): 2012, 2013


## 参照
- [P0558R1 Resolving `atomic<T>` named base class inconsistencies](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0558r1.pdf)
