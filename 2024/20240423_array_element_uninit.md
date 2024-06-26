# ［C++］ std::arrayを初期化せずに初期化する

初期化せずに初期化する。一見矛盾しているようにしか思えない行いはしかし、生配列の場合は次のように初期化しながら初期化しないことによって行うことができます

```cpp
int main() {
  int array_uninit[5];      // 各要素は未初期化
  int array_zeroinit[5]{};  // 各要素は0で初期化
}
```

この時`std::array`で同様に初期化しながら初期化しないことを行うにはどうすればいいのでしょうか？クラス型の場合、初期化をしない初期化（デフォルト初期化）の場合でもデフォルトコンストラクタが呼ばれてしまうため、なんとなくできないような気がしてしまいます。

先に結論を書いておくと、生配列と全く同様の書き方によって全く同様の初期化を行うことができます。

```cpp
int main() {
  std::array<int, 5> array_uninit;      // 各要素は未初期化
  std::array<int, 5> array_zeroinit{};  // 各要素は0で初期化
}
```

[:contents]

### デフォルト初期化

`int n;`のように変数の初期化子を指定せずに変数を宣言した場合、この形式の初期化はデフォルト初期化という初期化方法に分類されます。デフォルト初期化によってその変数（オブジェクト）の生存期間は開始されますが、非クラス型の場合はその値は初期化されず不定となります。

クラス型の変数をデフォルト初期化すると（例えば、`std::string str;`）そのクラスのデフォルトコンストラクタが呼ばれることによってその値が初期化されます。

配列型の場合は、各要素がデフォルト初期化されます。要素型が非クラス型ならばその値は初期化されず、クラス型の場合はデフォルトコンストラクタが呼ばれます。

デフォルト初期化における値が初期化されないとは、そのオブジェクトが占めるメモリ領域はその初期化時に一切書き込みがなされないということでもあります。変数初期化の直後で特定の値を書き込むことが分かっている場合など、初期化時のゼロ埋めを省くためにあえてデフォルト初期化したい場合が稀に良くあります。

一方で、`int n{};`のように空の初期化子を指定すると、これは値初期化（もしくは空のリストによる集成体体初期化）と呼ばれる形式の初期化になり、非クラス型の場合はその値はゼロ初期化（`0`あるいはそれに相当する値によって初期化）されます。クラス型の値初期化はデフォルト初期化と同様にデフォルトコンストラクタを呼びだし、配列型の場合は各要素が値初期化されます。

値初期化の場合はほとんどの場合、その領域はゼロ埋めされています。

```cpp
int main() {
  // デフォルト初期化
  int n1;             // 未初期化、値は不定
  char carray1[5];    // 未初期化、各要素の値は不定
  std::string str1;   // デフォルトコンストラクタ呼び出し

  // 値初期化
  int n1{};           // 値は0で初期化済
  char carray2[5]{};  // 集成体初期化、各要素は0で初期化済
  std::string str2{}; // デフォルトコンストラクタ呼び出し
}
```

### 集成体型のデフォルト初期化

`std::array`はクラス型であるので、デフォルト初期化においてもデフォルトコンストラクタが呼ばれてしまい何かしら初期化されてしまうような気がします。ただし一方で`std::array`は集成体型でもあり、C++17以降はデフォルトコンストラクタを宣言することはできません。`std::array`でデフォルト初期化を行うにはどうすればいいのか？という問いの答えを知るには、集成体型でデフォルト初期化を行うと何が起こるのかを知る事で近づくことができます。

集成体型は通常一切のコンストラクタをユーザーが宣言することができず、全てのコンストラクタ（デフォルト/コピー/ムーブ）はコンパイラによって暗黙に宣言・定義されています。集成体初期化は特定のコンストラクタを呼び出しているわけではありません。

したがって、集成体型の変数をデフォルト初期化した場合、クラス型の変数をデフォルト初期化したのと同じことが起こります。その場合、暗黙に定義されたデフォルトコンストラクタが呼び出され、暗黙に定義されたデフォルトコンストラクタは、引数なしでコンストラクタ初期化子リストが空で本体も空、なユーザー定義コンストラクタとほぼ同じ振る舞いをします。

コンストラクタ初期化子リストが空の場合、各非静的メンバ変数の初期化は、そのデフォルトメンバ初期化子があればそれによって、無ければデフォルト初期化されます。

したがって、集成体型変数のデフォルト初期化において各メンバ変数は、デフォルトメンバ初期化子が指定されていればそれによって、そうでないならばデフォルト初期化されます。

```cpp
// 非集成体のクラス型
struct S {
  int n = 0;
  int m;

  S(){}
};

// 集成体型
struct A {
  int n = 0;
  int m;
};

int main() {
  // デフォルト初期化
  // どちらも、メンバnは0で初期化され
  // メンバmは未初期化
  S s;
  A a;

  // これは保証されている
  assert(s.n == 0 and a.n == 0);

  // 未初期化値の読み取りは未定義動作（C++26以降はErroneous Behaviour）
  int n = s.m;
  int m = a.m;
}
```

### std::arrayの場合

`std::array`はとても単純には、次のような集成体型です

```cpp
template<typename T, std::size_t N>
struct array {
  T m_inner_array[N]; // 内部配列

  ...
};
```

そのため、`std::array`のデフォルト初期化時の挙動はこの内部配列がどう初期化されるかによって決まります。それはつまり、この内部配列にデフォルトメンバ初期化子があるかないかによって変化します。

```cpp
template<typename T, std::size_t N>
struct array {
  // どう宣言されている？？
  T m_inner_array[N]{}; // デフォルトメンバ初期化子あり
  T m_inner_array[N];   // 初期化子なし

  ...
};
```

デフォルトメンバ初期化子がある場合（`{}`とします）、`std::array`のデフォルト初期化は内部配列を空のリストによって集成体初期化し、生配列を空のリストによって集成体初期化すると各要素も`{}`によって初期化され、非クラス型の場合はゼロ初期化されます。

デフォルトメンバ初期化子がない場合、`std::array`のデフォルト初期化は内部配列をデフォルト初期化し、各要素もデフォルト初期化され、非クラス型の場合は初期化されません。

実は規格には`std::array`の内部配列がどのように宣言されるべきかについて規定が無いのですが、C++11時点では集成体型の非静的メンバのデフォルトメンバ初期化を行うことができなかったため、少なくともC++11時点の`std::array`の内部配列は初期化子を持っていません。そして、後からデフォルトメンバ初期化子を追加すると`std::array`のデフォルト初期化時の挙動が変化してしまうためそれは行われていないとみなすことができ、C++23時点の`std::array`も同様にその内部配列は初期化子を持っていないはずです。

主要な3実装（gcc/clang/msvc）を調べると、いずれも`std::array`の内部配列は初期化子を持っていません。

よって、要素型が非クラス型の`std::array`をデフォルト初期化すると、その各要素は未初期化のまま初期化を完了することができます。もっと言えば、`std::array`の初期化周りの挙動は生配列と同じになります（たぶん）。

```cpp
int main() {
  // デフォルト初期化、各要素は未初期化
  int raw_array_uninit[5];
  std::array<int, 5> array_uninit;
  
  // 空のリストによる集成体初期化、各要素は0で初期化
  int raw_array_zeroinit[5]{};
  std::array<int, 5> array_zeroinit{};
}
```

要素型がクラス型の場合でも、デフォルト初期化によって初期化されないメンバを持つクラス型の場合はそのメンバは未初期化とすることができます。

### 必ず初期化されるstd::array？

現在ではそのような実装はありませんが、C++23時点の標準としては特に禁止してはいないように見えます。通常あえてそのような実装を取る必要はないのですが、デフォルト初期化した時に各要素は未初期化となるため、その値の読み取りはUB（C++26からはEB）となります。

これは挙動としては安全ではないので、例えばコンパイラフラグのデバッグレベルなどによって、`std::array`の内部配列をデフォルトメンバ初期化するようにして、`std::array`がデフォルト初期化された時でもその各要素をゼロ初期化しておく安全に倒した挙動を取るようにする、ということは無意味ではないかもしれません。

### 未初期化領域の読み取りとEB

`std::array`に限らず、デフォルト初期化によって初期化されなかった領域を初期化する前に読み取ることはC++23までは未定義動作となるので、その読み取りに関しては注意が必要です。

```cpp
void f(int);

int main() {
  int n;  // 未初期化、これそのものは問題ない

  f(n);   // 初期化前の値の読み取り、これがUB（C++23まで）

  // 一度初期化すれば問題ない
  n = 10; // 未初期化領域への書き込みは当然ok（オブジェクトが生存期間内にあれば）
  f(n);   // ok、初期化済の領域の読み取り
}
```

C++26からはこの未初期化領域の読み取りによるUBが緩和されてErroneous BehaviourとなりUB（何が起こるかわからない、最適化に悪用される）ではなくなります。このEBとして読み取られる値は実装定義とされますが、おそらくほとんどの場合0が読み取られます。

```cpp
void f(int);

int main() {
  // 以下、C++26から

  int n;  // 未初期化、引き続き問題ない

  f(n);   // 初期化前の値の読み取り、EBとして実装定義の値（おそらく0）が読み取られる
}
```

UBあるいはEBとなるのは未初期化領域の読み取りのみであり、この場合のEBは不定値を読み取る代わりに特定の値（おそらく0）を読み取るようにするものですが、その実体は未初期化領域が特定の値によって初期化されるようになることによって実現されるはずです。

```cpp
int main() {
  // C++26以降、どの初期化においても領域は0で初期化されるようになる

  // デフォルト初期化
  int raw_array_uninit[1];
  std::array<int, 1> array_uninit;

  // 集成体初期化
  int raw_array_zeroinit[1]{};
  std::array<int, 1> array_zeroinit{};
}
```

この動作は現在でも新しめのGCC/Clangで`-ftrivial-auto-var-init=zero`オプションを使用すると先取することができます。なお、`-ftrivial-auto-var-init=pattern`とすると、未初期化値の読み取りを検知することができます。おそらくよく似たオプションによって、C++26以降のEB時の振る舞いも制御できるはずです。

- [[C++] gcc HEAD 14.0.1 20240422 (experimental) - Wandbox](https://wandbox.org/permlink/0osf6DX3X5hkXgib)

パフォーマンスのために従来未初期化だった領域のゼロ埋めが好ましくないなど、C++26以降でもこれまで通りの未初期化領域が欲しい場合は、`[[indeterminate]]`属性を指定することで挙動を維持することができます。

```cpp
void f(int);

int main() {
  // デフォルト初期化、かつ未初期化
  int raw_array_uninit[1] [[indeterminate]];
  std::array<int, 1> array_uninit [[indeterminate]];

  // C++26でもどちらも未定義動作（EBではない）
  f(raw_array_uninit[0]);
  f(array_uninit[0]);
}
```

EB環境の下ではこのように、意図的に初期化せず不定値を持つ変数（初期化が必要な変数）を明示することを強制します。

コンパイラは不明な属性を無視することが規定されているため、`[[indeterminate]]`属性そのものは今日から使用し始めることができます。これはC++26でコンパイルするまでは全く効果はありませんが、今すぐC++26に移行しないコードでも未初期化変数を目立たせる目的で使用することができ、C++26における`[[indeterminate]]`属性の役割の一部を先取りすることができます。そして、そうしておくと将来C++26に移行したときでも`[[indeterminate]]`が指定された変数は変わらず未初期化のままになり挙動が変更されることがなく、すんなりとC++26に移行することができるでしょう（この部分に関してだけは）。

### 参考文献

- [Default-initialization - cppreference.com](https://en.cppreference.com/w/cpp/language/default_initialization)
- [Default constructors - cppreference.com](https://en.cppreference.com/w/cpp/language/default_constructor)
- [［C++］UBとEB - 地面を見下ろす少年の足蹴にされる私](https://onihusube.hatenablog.com/entry/2023/12/11/000000)

[この記事のMarkdownソース](https://github.com/onihusube/blog/blob/master/2024/20240423_array_element_uninit.md)
