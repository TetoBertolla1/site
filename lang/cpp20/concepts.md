# コンセプト
* cpp20[meta cpp]

## 概要
(執筆中)

## 仕様
### concept
- `concept`キーワードによって定義されたコンセプトは、`bool`型のprvalueを持つ定数式である：
    ```cpp
    template <class T>
    concept C = true;

    static_assert(C<int>);
    ```

(執筆中)

### requires式
- 「requires式 (Requires expressions)」は、「型`T`がメンバ関数`f()`を持っていなければならない」「型`T`がメンバ型`value_type`を持っていなければならない」といったテンプレートパラメータの型がもつプロパティを検査し、要件を定義するための機能である
- requires式の結果は、`bool`型のprvalueを持つ定数式である
- requires式のサンプルをいくつか示す：
    ```cpp
    template <typename T>
    concept R = requires (T i) {       // 型Tの値iがあるとして、
      typename T::type;                // 型Tがメンバ型typeを持つこと。
      {*i} -> const typename T::type&; // 型Tの値に対して式*iが妥当であり、
                                       // その式の戻り値型としてconst typename T::type&が返ること
    };
    ```

    - ここでは、関数形式でローカルパラメータをひとつ (`T i`) とるrequires式によってコンセプト`R`を定義している
    - ローカルパラメータである`T i`の変数定義では、`T`型に対して「コピー構築可能であること」といった要求は行わず、そのような評価はコンパイル時にも行われない。これは[`std::declval()`](/reference/utility/declval.md)関数と同様に、「`T`型のなんらかの値」であることのみを表し、特定の値は持たず、構築もされない
- パラメータリスト後の波括弧 { } で囲まれた本体中には、要件のシーケンスをセミコロン区切りで記述する。各要件は、ローカルパラメータ、テンプレートパラメータ、およびその文脈から見えるほかの宣言を参照できる
- ローカルパラメータは、リンケージ、記憶域、生存期間をもたない
- ローカルパラメータのスコープは、名前が導入されてから本体の閉じカッコまでである
- ローカルパラメータはデフォルト引数を持ってはならず、パラメータリストの最後が `...` であってはならない
- 型`T`がrequires式で列挙された要件は定義順に検査され、全ての要件を満たす場合、`bool`型の定数`true`が返る。requires式内では、無効な型や式が形成されたり、意味論的な制約に違反する場合があるが、そういった場合はプログラムが不適格にはならず、`false`が返る
    - テンプレートパラメータに関わらず無効になる型や式が現れた場合、プログラムは不適格となる
- 対象となる型がひとつである場合でも、requires式のパラメータは複数とることができる：
    ```cpp
    template <typename T>
    concept C = requires (T a, T b) { // 型Tの値aとbがあるとして、
      a + b; // 式a + bが妥当であること
    };
    ```

- ローカルパラメータをとらないrequires式も定義できる：
    ```cpp
    template <typename T> concept C = requires {
      typename T::inner; // メンバ型innerの存在を要求
    };
    ```

- requires式で定義できる要件の種類は、以下である：
    - 単純要件
    - 型要件
    - 複合要件
    - 入れ子要件

#### 単純要件
- 「単純要件 (Simple requirements)」は、式の妥当性を表明する要件である。テンプレート引数で型を置き換えた結果として式が無効な場合、`false`に評価される
    ```cpp
    template <typename T>
    concept C = requires (T a, T b) { // 型Tの値aとbがあるとして、
      a + b; // 式a + bが妥当であること
    };
    ```

- この要件には、任意の定数式を含めることができるが、直接的にそのような方法をとったとしても、その定数式の評価された結果が`true`であること、というような要件にはできない。あくまで式が妥当であることの要件である


#### 型要件
- 「型要件 (Type requirements)」は、型の妥当性を表明する要件である。テンプレート引数で型を置き換えた結果として型が無効な場合、`false`に評価される
- 型要件の構文は以下のようになる：
    ```cpp
    typename 入れ子指定(省略可) 要求する型名;
    ```

    - つまり、先頭に`typename`が記述されていれば型要件である。テンプレートパラメータの型が保持するメンバ型を`typename T::nested_type;`のように要求することもできるが、特定のクラステンプレートにテンプレート引数を渡した結果が妥当であること、というような要求もできる

    ```cpp
    template <typename T, typename T::type = 0> struct S;
    template <typename T> using Ref = T&;

    template <typename T>
    concept C = requires {
      typename T::inner; // メンバ型innerの存在を要求
      typename S<T>;     // クラステンプレートの特殊化を要求
      typename Ref<T>;   // エイリアステンプレートRefに型Tを渡せることを要求
                         // (Tがvoidだったら失敗することを意図)
    };
    ```

- ただし特殊化の要求は、テンプレート引数を渡した結果として完全型になること、という要求ではない
    - そのため、宣言のみのプライマリテンプレートと、定義をもつ特殊化、という構成になっているクラステンプレートは、特殊化されていないテンプレート引数に対しては不完全型になるのみで非妥当ではない


#### 複合要件
- 「複合要件 (Compound requirements)」は、式のプロパティを表明する要件である。式の妥当性、`noexcept`、式の戻り値型に対する要件を順に検査する
- 複合要件の構文は以下のようになる：
    ```cpp
    { 妥当性を検査する式 } noexcept(省略可) -> 戻り値型、もしくはCV修飾された戻り値型の制約(省略可);
    ```

- この要件は、以下のように検査される：
    - テンプレート引数で型を置き換えて式を評価し、妥当でなければ`false`に評価される
    - `noexcept`を指定した場合、式は例外送出の可能性がある場合は`false`に評価される
    - 戻り値の型要件が指定された場合、
        - テンプレート引数で型を置き換えて型を評価し、妥当でなければ`false`に評価される
        - 制約ではなく戻り値型が指定された場合、式の戻り値型が指定された戻り値型に変換可能であること。変換できなければ`false`に評価される
        - 制約が指定された場合、その制約された型をパラメータにとり`void`を返す関数テンプレートが擬似的に定義され、その関数に式を渡した結果が妥当であること。妥当でなければ`false`に評価される
- 例として、式のみを指定する場合、単純要件と等価である：
    ```cpp
    template <typename T>
    concept C1 = requires(T x) {
      {x++}; // 型Tの値xに対して式x++が妥当であること
    };
    ```

- 式と戻り値型を指定した場合：
    ```cpp
    template <typename T>
    concept C2 = requires(T x) {
      {*x} -> typename T::inner; // 型Tの値xに対して式*xが妥当であり、
                                 // その戻り値型がtypename T::inner型に暗黙変換可能であること
    };
    ```

- 式と戻り値型の制約を指定した場合：
    ```cpp
    template <typename T, typename U> concept C3 = false;
    template<typename T>
    concept C4 = requires(T x) {
      {*x} -> C3<int> const&;
    };
    ```

    - この場合、式`*x`の妥当性が検査されたあと、以下の関数テンプレートが擬似的に定義され、

    ```cpp
    template<C3<int> X> void f(X const&);
    ```

    - 式`*x`の戻り値型をその制約された関数テンプレートに渡せるかが検査される。この場合、制約`C3`は常に`false`に評価されるため、制約`C4`は`false`となる

- 式と`noexcept`を指定した場合、指定した式`g(x)`が例外送出の可能性がないことが検査される：
    ```cpp
    template <typename T>
    concept C5 = requires(T x) {
      {g(x)} noexcept;
    };
    ```


#### 入れ子要件
- 「入れ子要件 (Nested requirements)」は、requires式内で`bool`型の定数式で制約する要件である。コンセプトには`bool`型の定数式を直接指定できるため入れ子要件と等価な指定ができるが、こちらは先に述べた単純要件、型要件、複合要件に対する追加の制約として使用する。要件は定義順に検査されるため、要件Aが成り立たなければ要件Bを検査しない、というような状況で入れ子要件を使用できるだろう
- 入れ子要件の構文は以下のようになる：
    ```cpp
    requires 制約式;
    ```

    - ここでの制約式とは、`concept C = 制約定数式;`のようになっているコンセプト定義に指定する`bool`型の定数式と同じである

    ```cpp
    template <typename U>
    concept C = sizeof(U) == 1;

    template <typename T>
    concept D = requires (T t) {
      requires C<decltype (+t)>; // コンセプトを指定できる
    };
    ```

- 入れ子要件では、requires式で導入したローカルパラメータを使用できる。ただし、ローカルパラメータは特定の値を意味しない「その型のなんらかの値をもつオブジェクト」でしかないため、ローカルパラメータの値を参照しようとする式は不適格となる。ローカルパラメータを使用できるのは、値が評価されない文脈のみである (`sizeof`、`decltype`、`alignof`など)
    ```cpp
    template <typename T>
    concept C = requires (T a) {
      requires sizeof(a) == 4; // OK : 値が評価されない文脈でローカルパラメータを使用
      //requires a == 0;       // コンパイルエラー！: 制約変数は値を評価できない
    }
    ```


### 制約テンプレート
### requires節
- 「requires節 (Requires clauses)」は、requires式を持てる
    ```cpp
    template <typename T>
    requires requires (T x) { x + x; } // ひとつめのrequiresはrequires節
    T add(T a, T b) { return a + b; }  // ふたつめのrequiresはrequires式
    ```

(執筆中)


## 備考
- GCC 9.1では、コンセプトが正式サポートされていないため、コンパイルオプションとして`-fconcepts`を付ける必要がある


## 例
```cpp example
```

### 出力
```
```


## この機能が必要になった背景・経緯


## 関連項目
- [`<concepts>`](/reference/concepts.md)
- [`<iterator>`](/reference/iterator.md)
- [`<ranges>`](/reference/ranges.md.nolink)


## 参照
- [P0734R0 Wording Paper, C++ extensions for Concepts](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0734r0.pdf)
- [P0857R0 Wording for "functionality gaps in constraints"](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0857r0.html)
- [P1084R2 Today's return-type-requirement s Are Insufcient](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1084r2.pdf)
- [P1141R2 Yet another approach for constrained declarations](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1141r2.html)