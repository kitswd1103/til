# プログラミングRust 第2版メモ

読んで気になったところをメモしています。
6章まではメモをとっていないためありません。

## 7章

### エラー型のないResult型がある

```Rust
Result<()>
```

これらは型エイリアスが使われているため、使用するモジュールに

```Rust
pub type Result<T> = result::Result<T, Error>;
```

のような定義がされている。

### カスタムエラー

独自のエラー宣言は構造体を定義する。

例:

```Rust
#[derive(Debug, Clone)]
pub struct HogeError {
    pub message: String,
}

fn hoge(make_error: bool) -> Result<(), HogeError> {

    if make_error {
        Err(HogeError { message: "error hoge.".to_string() })
    } else {
        Ok(())
    }
}

fn main() {
    // エラーを起こさない
    match hoge(false) {
        Ok(()) => println!("hoge Ok"),
        Err(err) => eprintln!("hoge error message: {}", err.message)
    }
    // エラーを起こす
    match hoge(true) {
        Ok(()) => println!("hoge Ok"),
        Err(err) => eprintln!("hoge error message: {}", err.message)
    }
}
```

実行結果:

```console
hoge Ok
hoge error message: error hoge.
```

ただし上のコードだと標準エラー型のように書くことが出来ない

```Rust
error[E0277]: `HogeError` doesn't implement `std::fmt::Display`
Err(err) => eprintln!("hoge error message: {}", err)
                                              ^^^ `HogeError` cannot formatted with the default formatter
```

のようにHogeError構造体変数を出力する等のことが出来ない。

HogeError構造体を出力するための方法として下記の二つがある

方法1 標準機能で行う

```Rust
use std::fmt;

impl fmt::Display for HogeError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        write!(f, "{}", self.message)
    }
}
impl std::error::Error for HogeError { }
```

方法2 `thiserror`クレートを使用すして構造体を定義する

```Rust
use thiserror::Error;
#[derive(Error, Debug)]
#[error("{message:}")]
pub struct HogeError {
    pub message: String,
}
```

### 複数種類のエラーへの対応

エラーを伝播させる際に?演算子を使用するが戻り値と違う型のエラーだとエラーを吐く。

例:

```Rust

use std::fmt;

#[derive(Debug, Clone)]
pub struct HogeError {
    pub message: String,
}

#[derive(Debug, Clone)]
pub struct FugaError {
    pub message: String,
}

fn hoge(make_error: bool) -> Result<(), HogeError> {
    if make_error {
        Err(HogeError { message: "error hoge.".to_string() })
    } else {
        Ok(())
    }
}
fn fuga(make_error: bool) -> Result<(), FugaError> {
    if make_error {
        Err(FugaError { message: "error fuga.".to_string() })
    } else {
        Ok(())
    }
}

impl fmt::Display for HogeError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        write!(f, "{}", self.message)
    }
}
impl std::error::Error for HogeError { }

impl fmt::Display for FugaError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        write!(f, "{}", self.message)
    }
}
impl std::error::Error for FugaError { }

fn func() -> Result<(), HogeError> {
    hoge(false)?;
    fuga(false)?;

    Ok(())
}

fn main() {
    func();
}

```

ビルド結果の一部:

```rust
error[E0277]: `?` couldn't convert the error to `HogeError`
  --> src\main.rs:45:16
   |
43 | fn func() -> Result<(), HogeError> {
   |              --------------------- expected `HogeError` because of this
44 |     hoge(false)?;
45 |     fuga(false)?;
   |                ^ the trait `From<FugaError>` is not implemented for `HogeError`   
   |
```

解決方法としてthiserro等のクレートを使うか標準機能で行う方法がある。  
標準方法では、関数の戻り値 `Result<(), HogeError>` を `Result<(), Box<dyn std::error::Error + Send + Sync + 'static>>` に変える。

```Rust
fn func() -> Result<(), Box<dyn std::error::Error + Send + Sync + 'static>>
```

使いやすくしたい場合は上記のResultのエイリアスを作成すると便利。

```Rust
type GenericError = Box<dyn std::error::Error + Send + Sync + 'static>;
type GenericResult = Result<(), GenericError>;

fn func() -> GenericResult
```

上記の型を使用して特定の種類のエラーだけを対処したい場合は `err.downcast_ref` を使用する。

例:

```rust
fn func(make_hoge_error: bool, make_fuga_error: bool) -> GenericResult {
    hoge(make_hoge_error)?;
    fuga(make_fuga_error)?;

    Ok(())
}

fn check_func(make_hoge_error: bool, make_fuga_error: bool) {
    println!("HogeError: {}, FugaError: {} で実行", make_hoge_error, make_fuga_error);
    match func(make_hoge_error, make_fuga_error) {
        Ok(()) => println!("> エラーなし"),
        Err(err) => {
            if err.downcast_ref::<HogeError>().is_some() {
                println!("> Hogeのエラー");
            }
            else {
                println!("> Hoge以外のエラー");
            }
        }
    }
}

fn main() {
    check_func(false, false);
    check_func(true,  false);
    check_func(false,  true);
}

```

実行結果:

```console
HogeError: false, FugaError: false で実行
> エラーなし
HogeError: true, FugaError: false で実行
> Hogeのエラー
HogeError: false, FugaError: true で実行
> Hoge以外のエラー
```

---

エラー処理は何らかのクレートを使用したほうが便利そう。

## 8章

### エディション

互換性を保つためにある。
Cargo.toml の `[package]` セクションにどのエディションで書かれたか書いてある。

```toml
edition = "2021"
```

一つのプログラムで複数エディションを混在させることが出来る。エディションの詳細は 「[The Rust Edition Guide](https://doc.rust-lang.org/stable/edition-guide/)」に載っている。

古いエディションは `cargo fix` コマンドで自動的にアップグレードできる場合がある。詳細は「The Rust Edition Guide」に書かれている。

### モジュール

Rustの名前空間。 `pub` をつけることで外部からアクセスできる。
また `pub(super)` などをつけることで公開範囲を制限する。

```Rust
mod hoge {

    pub mod fuga {
        fn piyo() {
            println!("piyo");
        }
        pub fn piyo_pub() {
            println!("piyo_pub");
        }
        pub(super) fn piyo_super() {
            println!("piyo_super");
        }
        pub(in crate::hoge) fn piyo_hoge() {
            println!("piyo_hoge");
        }
        fn func() {
            piyo();
            piyo_pub();
            piyo_super();
            piyo_hoge();
        }
    }
    pub fn func() {
        // fuga::piyo(); // error
        fuga::piyo_pub();
        fuga::piyo_super();
        fuga::piyo_hoge();
    }
}

fn main() {
    hoge::func();
    // hoge::fuga::piyo(); // error
    hoge::fuga::piyo_pub();
    // hoge::fuga::piyo_super(); // error
    // hoge::fuga::piyo_hoge(); // error
}
```

#### モジュールの分割

モジュールを分割するときは src ディレクトリに rsファイルを作成する方法とディレクトリの作成+mod.rsを使用する方法、*module _name*.mod+補助ディレクトリを使用する方法がある。
分割したモジュールを使用する時は `main.rs` サブモジュールの場合は親モジュールのソースファイルに `mod mod_name;` と記述しインポートする必要がある。

##### rsファイルを作成

rsファイルの名前がモジュール名になる。

ファイル構成:

```tree
┣src
┃ ┣ hoge.rs
┃ ┗ main.rs
```

main.rs

```rust
pub mod hoge;

fn main() {
    hoge::hoge_func();
}
```

hoge.rs

```rust
pub fn hoge_func() {
    println!("hoge_func");
}
```

##### ディレクトリを作成

モジュール名のディレクトリを作成する。その直下に `mod.rs` を作成しその中にモジュール機能を書く。サブモジュールを使う場合は `mod.rs` 使用するサブモジュールを書きファイルを作成する。

```tree
┣src
┃ ┣ main.rs
┃ ┗ hoge        // モジュールディレクトリ
┃     ┣ mod.rs  // hogeのモジュールを記述
┃     ┗ fuga.rs // hogeのサブモジュール
```

main.rs

```rust
mod hoge;

fn main() {
    hoge::fuga::fuga_func();
    hoge::hoge_func();
}
```

hoge/mod.rs

```rust
pub mod fuga; // サブモジュールのインポート

pub fn hoge_func() {
    println!("hoge_func");
}
```

hoge/fuga.rs

```rust
pub fn fuga_func() {
    println!("fuga_piyo");
}
```

#### rsの作成+ディレクトリ作成

```tree
┣src
┃ ┣ main.rs
┃ ┣ hoge.rs     // hogeのモジュールを記述
┃ ┗ hoge        // モジュールの補助ディレクトリ
┃     ┗ fuga.rs // hogeのサブモジュール
```

main.rs

```rust
mod hoge;

fn main() {
    hoge::fuga::fuga_func();
    hoge::hoge_func();
}
```

hoge.rs

```rust
pub mod fuga; // サブモジュールのインポート

pub fn hoge_func() {
    println!("hoge_func");
}
```

hoge/fuga.rs

```rust
pub fn fuga_func() {
    println!("fuga_piyo");
}
```

モジュール分割は上のどれかの方法を使う。ただし *module_name*.rs の作成+ディレクトリに mod.rs を同時に作成するとエラーを吐くので注意。

親モジュールや別モジュールを使用するときは `use` を使って機能をインポートする。親モジュールの場合は `use super::module_name;` と書き別モジュールかクレートルートからの相対パスで指定するときは `use crate::module_name;` と記述する 

```tree
┣src
┃ ┣ main.rs
┃ ┣ hoge.rs     // hogeのモジュールを記述
┃ ┗ hoge        // モジュールの補助ディレクトリ
┃     ┗ fuga.rs // hogeのサブモジュール
```

上ような構成の時にfugaからhogeのモジュールをするときは

```rust
use super::hoge_func;
```

```rust
use crate::hoge::hoge_func;
```

のどちらかで使用することが出来る。

外部モジュールと自身のモジュール名が同一の場合はインポートが曖昧になるため明示的に指定する必要がある。

外部モジュールの場合は :: で書き始める。

```rust
use ::hoge::hoge_func;
```

自身のモジュールの場合は self を使用する。

```rust
use self::hoge::hoge_func;
```

### ライブラリ

既存プログラムをライブラリ化するには `lib.rs` を作成する。その中に外で使う機能にpubをつけることでライブラリ化できる。

### 属性

プログラム中のアイテムへ属性をつけることが出来る。警告を無効化したり条件付きコンパイル等いくつか属性がある。
属性の詳細はRustの[リファレンスサイト](https://doc.rust-lang.org/reference/attributes.html)に乗っている。

### ドキュメント

`cargo doc` コマンドを使うことでドキュメントを作成することが出来る。
ドキュメントは `///` から始まるコメントを書くことで書くことが出来る。ドキュメントの詳細はRustの[ドキュメント](https://doc.rust-lang.org/rustdoc/index.html)から確認することが出来る。

## 9章

### ユニット構造体

```rust
struct Hoge;
```

のように要素を宣言しにない構造体。

### self

self の型は明示的に書くことが出来る

例:

```rust
use std::{rc::Rc, sync::Arc};

struct Hoge;

impl Hoge {
    pub fn func(self: Rc<Self>) { }
    pub fn func2(self: Arc<Self>) { }
    pub fn func3(self: Box<Self>) { }
}

fn main() {
    let rc_hoge = Rc::new(Hoge);
    let arc_hoge = Arc::new(Hoge);
    let box_hoge = Box::new(Hoge);
    rc_hoge.func();
    arc_hoge.func2();
    box_hoge.func3();
}
```

### 型関連定数

```rust
struct Hoge {
    pub a: u32,
}

impl Hoge {
    pub const ZERO: Hoge = Hoge { a: 0 };
    pub const FUGA: u32 = 1;
    pub const PIYO: &'static str = "PIYO";
}

fn main() {
    println!("{}", Hoge::ZERO.a);
    println!("{}", Hoge::FUGA);
    println!("{}", Hoge::PIYO);
}
```

### ジェネリック

implの後ろにも型パラメータを書く必要がある。書かない場合は特定の型にのみ実装するようになる。

```rust
struct Hoge<T> {
    pub a: T,
}

impl<T> Hoge<T> {
    pub fn fuga(&self) { println!("fuga generic") }
}

impl Hoge<u32> {
    pub fn piyo(&self) { println!("piyo u32") }
}

fn main() {
    let hoge = Hoge { a: 10u8 };
    let fuga = Hoge { a: 10u32 };

    hoge.fuga();
    // hoge.piyo(); // error piyo関数はHoge<u32>でない場合実装されない。

    fuga.fuga();
    fuga.piyo();    // fuga変数はHoge<u32>のためコンパイル可能
}
```

### 定数パラメータ

```rust
struct Hoge<const N: usize> {
    pub a: [u32; N],
}
impl<const N: usize> Hoge<N> {
    pub fn new(arr: [u32; N]) -> Self {
        Hoge {a: arr }
    }
}

fn main() {
    let hoge = Hoge::new([0; 10]);
    let fuga = Hoge::new([0; 20]);

    println!("{}", hoge.a.len());   // 10
    println!("{}", fuga.a.len());   // 20
}
```

複数のジェネリックパラメータを使用するときは生存期間、型パラメータ、定数パラメータの順番で書く。

### 一般的トレイとの自動実装

作成した構造体にCopyやClone、演算子動作の機能を追加するには #[derive] 属性を付与する。

```rust
#[derive(Copy, Clone, PartialEq)]
struct Hoge{
    pub a: u32,
}
```

属性を付与することで自動実装されるが、構造体の全てのフィールドが付与するトレイトを実装する必要がある。

### 内部可変性

通常 Rc を使用した場合中身の書き換えができない。

```rust
use std::rc::Rc;

struct Hoge {
    pub a: u32,
}

struct Fuga {
    hoge: Rc<Hoge>,
}

impl Hoge {
    pub fn new() -> Self {
        Hoge { a: 0 }
    }
}

impl Fuga {
    pub fn new() -> Self {
        Fuga { hoge: Rc::new(Hoge::new()) }
    }
    pub fn add_hoge_a(&mut self, add_value: u32) {
        self.hoge.a = self.hoge.a + add_value; // <- hogeがRcのため書き換えできない
    }
    pub fn print_hoge_a(&self) {
        println!("hoge_a: {}", self.hoge.a)
    }
}

fn main() {
    let mut fuga = Fuga::new();
    fuga.print_hoge_a();
    fuga.add_hoge_a(10);
    fuga.print_hoge_a();
}
```

値を書き換えたい場合は Cell または RefCell を使用する。

Cellを使用する場合は直接書き換えるのではなく set() を使用することで書き換えることが出来る。値をとる場合は get() を使用するが参照の取得はできないため、Cellの型パラメータには Copy トレイトが実装されている必要がある。

Cell:

```rust
use std::{rc::Rc, cell::Cell};

struct Hoge {
    pub a: Cell<u32>,   // u32はCopyトレイトが実装されているためget()で値を取得可能
}

struct Fuga {
    hoge: Rc<Hoge>,
}

impl Hoge {
    pub fn new() -> Self {
        Hoge { a: Cell::new(0) }
    }
}

impl Fuga {
    pub fn new() -> Self {
        Fuga { hoge: Rc::new(Hoge::new()) }
    }
    pub fn add_hoge_a(&mut self, add_value: u32) {
        self.hoge.a.set(self.hoge.a.get() + add_value);
    }
    pub fn print_hoge_a(&self) {
        println!("hoge_a: {}", self.hoge.a.get())
    }
}

fn main() {
    let mut fuga = Fuga::new();
    fuga.print_hoge_a();    // 0
    fuga.add_hoge_a(10);
    fuga.print_hoge_a();    // 10
}
```

RefCellの場合は `brrow()`, `brrow_mut()` を使用することで参照、可変参照を取得することが出来る。ただし既に参照が借用されている場合はパニックを起こす。
パニックを起こしたくない場合は `try_brrow()`, `try_brrow_mut()` を使用することでResult型として取得でき、既に借用されている場合はErrが帰ってくる

RefCell:

```rust
use std::{rc::Rc, cell::RefCell};

struct Hoge {
    pub a: RefCell<u32>,
}

struct Fuga {
    hoge: Rc<Hoge>,
}

impl Hoge {
    pub fn new() -> Self {
        Hoge { a: RefCell::new(0) }
    }
}

impl Fuga {
    pub fn new() -> Self {
        Fuga { hoge: Rc::new(Hoge::new()) }
    }
    pub fn add_hoge_a(&mut self, add_value: u32) {
        let mut a = self.hoge.a.borrow_mut();   // RefMut型が帰る
        *a = *a + add_value;                    // RefMut型は通常の &mut と同じ制約のため * をつける
    }
    pub fn print_hoge_a(&self) {
        println!("hoge_a: {}", self.hoge.a.borrow())
    }
}

fn main() {
    let mut fuga = Fuga::new();
    fuga.print_hoge_a();    // 0
    fuga.add_hoge_a(10);
    fuga.print_hoge_a();    // 10

}
```

```rust
use std::cell::RefCell;

fn main() {
    let a = RefCell::new(0);

    {
        let mut b = a.borrow_mut();
        *b = *b + 10;
        // println!("{}", a.borrow());  // <= 既にaの借用を受けているためパニックを起こす
        match a.try_borrow() {
            Ok(value) => println!("Ok. value: {}", value),
            Err(msg) => println!("Err. msg: {}", msg)       // < こちらが呼ばれる。
        }; // Err. msg: already mutably borrowed
    }

    // ブロックを抜けたことで a を借用した b がドロップするため a の借用が再度可能になる
    match a.try_borrow() {
        Ok(value) => println!("Ok. value: {}", value),  // < こちらが呼ばれる。
        Err(msg) => println!("Err. msg: {}", msg)
    }; // Ok. value: 10
}
```

## 10章

### 列挙型

ヴァリアントをインポートすることが出来る。同じモジュールの場合はSelfインポートをする。

```rust
use std::cell::RefCell;

enum Hoge {
    Foo,
    Bar,
}

use self::Hoge::Bar; // Barヴァリアントをインポート

fn main() {
    let _foo = Hoge::Foo;
    let _bar = Bar;         // Barをインポートしているため Hoge:: を省略することができる
}
```

Cスタイルの列挙型のように整数を割り当てることが可能。
整数割り当てがない場合、先頭は0、それ以外は上の番号+1で割り振られる

```rust
enum Hoge {
    Foo,        // 先頭で指定がないため0
    Bar = 10,   // 10
    Baz,        // 11
}
```

Cスタイルの列挙型は整数型へのキャストは可能だが、整数型から列挙型への変換は自分でチェックする必要がある。

```rust
enum Hoge {
    Foo = 10,
    Bar,
    Baz = 20,
}

impl Hoge {
    fn from_u32(n: u32) -> Option<Hoge> {
        match n {
            10 => Some(Hoge::Foo),
            11 => Some(Hoge::Bar),
            20 => Some(Hoge::Baz),
            _ => None,
        }
    }

    fn to_string(self) -> String {
        match self {
            Hoge::Foo => "Foo".to_string(),
            Hoge::Bar => "Bar".to_string(),
            Hoge::Baz => "Baz".to_string(),
        }
    }

    fn print(self) {
        println!("hoge: {}", self.to_string());
    }

    fn print_to_u32(self) {
        println!("hoge as u32: {}", self as u32);
    }

    fn print_from_u32(n: u32) {
        match Self::from_u32(n) {
            Some(hoge) => println!("{} to Hoge: {}", n, hoge.to_string()),
            None => println!("{} convert failed.", n),
        }
    }
}

fn main() {
    Hoge::Foo.print(); // hoge: Foo
    Hoge::Bar.print(); // hoge: Bar
    Hoge::Baz.print(); // hoge: Baz

    Hoge::Foo.print_to_u32(); // hoge as u32: 10
    Hoge::Bar.print_to_u32(); // hoge as u32: 11
    Hoge::Baz.print_to_u32(); // hoge as u32: 20

    Hoge::print_from_u32(10); // 10 to Hoge: Foo
    Hoge::print_from_u32(11); // 11 to Hoge: Bar
    Hoge::print_from_u32(20); // 20 to Hoge: Baz
    Hoge::print_from_u32(1); // 1 convert failed.
}
```

列挙型の変換については [enum_primitive](https://crates.io/crates/enum_primitive)クレートを使用することもできる。

==演算子等はコンパイラが提供しているが、使用する場合は明示的に要求する必要がある。

```rust
#[derive(Copy, Clone, PartialEq)]
enum Hoge {
    Foo,
}
```

列挙型はのヴァリアントは3種類ありデータを持たないもの、タプル型、構造体型ヴァリアントがある。

```rust
enum Hoge {
    Foo,                // データなし
    Bar(u32, u32),      // タプル型ヴァリアント
    Baz { value: u32 }, // 構造体ヴァリアント
}

fn main() {
    let _foo = Hoge::Foo;
    let _bar = Hoge::Bar(0, 0);
    let _baz = Hoge::Baz { value: 0 };
}
```

列挙型はツリー構造を簡単に書くことが出来る。

---

他にも便利そうな機能があるため実際に使う場面で再度この章を読み返す。

## 11章

### トレイトの衝突

トレイトの名前の衝突は、同じ型に同じ名前でメソッドを定義しないと衝突しない。衝突が起きた場合は完全修飾メソッド記法を使用する。

### トレイトオブジェクト

トレイトオブジェクトは動的ディスパッチ、トレイト型の参照で、dyn キーワードを使用する。トレイト型はコンパイル時にサイズが決まらずコンパイルが出来ないため参照だと明示する必要がある。

```rust
struct Hoge;

trait Fuga {
    fn print(&self) { println!("Fuga trait for Hoge") }
}

impl Fuga for Hoge { 
}

fn main() {
    let mut hoge = Hoge {};
    //let mut fuga: dyn Fuga = hoge; // dyn Fuga型は型サイズがコンパイル時に定まらないためエラー
    let fuga: &mut dyn Fuga = &mut hoge; // 参照型はコンパイル時にサイズが決まるためコンパイル可能
    fuga.print();
}
```

トレイトオブジェクトは通常の参照として使用することが出来るが、基本的に実際の型情報の取得やダウンキャストはできない。調べたらクレートを使用したらできるらしい。

メモリ配置として仮想テーブルへのポインタは構造体が持たず、トレイトオブジェクトが所有する。そのため小さな構造体に大量のトレイトを実装しても構造体自体のメモリサイズは変わらない。

c++で仮想関数を使用した構造体のサイズ出力コード(paiza.ioで実行):

```c++
#include <iostream>

struct Foo {
    void func() const { }
};
struct Hoge {
    virtual void func() const { }
};

struct Fuga : public Hoge { };

int main(void){
    Foo foo;
    std::cout << "Foo size: " << sizeof(foo) << std::endl;     // Foo size: 1
    
    Hoge hoge;
    std::cout << "Hoge size: " << sizeof(hoge) << std::endl;   // Hoge size: 8
    
    Fuga fuga;
    std::cout << "Fuga size: " << sizeof(fuga) << std::endl;   // Fuga size: 8
}
```

普通の関数だけの構造体はサイズ1[^1]、仮想関数をもつ構造体はvtableを持つ為サイズ8となる。

rustでトレイトオブジェクトを使用したコード

```rust
use std::mem;

struct Foo;

impl Foo {
    fn _func() { }
}

struct Hoge;

trait Fuga {
    fn _func(&self) { }
}

impl Fuga for Hoge { }

fn main() {
    let foo = Foo {};
    println!("foo size: {}", mem::size_of_val(&foo));   // foo size: 0
    
    let mut hoge = Hoge {};
    println!("hoge size: {}", mem::size_of_val(&hoge)); // hoge size: 0
    
    let fuga: &mut dyn Fuga = &mut hoge;
    println!("&mut dyn fuga size: {}", mem::size_of_val(&fuga)); // &mut dyn fuga size: 16
    println!("fuga size: {}", mem::size_of_val(fuga)); // fuga size: 16
}
```

implで実装したユニット構造体およびトレイトを実装したユニット構造体はサイズ0、トレイトオブジェクトはデータ構造体へのptr+vptrのサイズ16で構造体自体のサイズは0となる。[^2]

通常の参照やBox,Rc等のポインタ型は必要に応じてトレイトオブジェクトに変換される。

```rust
use std::rc::Rc;

struct Hoge;

trait Fuga {
    fn _func(&self) { }
}

impl Fuga for Hoge { }

fn fuga_ref(_fuga: &dyn Fuga) { println!("fuga ref") }
fn fuga_box(_fuga: Box<dyn Fuga>) { println!("fuga box") }
fn fuga_rc(_fuga: Rc<dyn Fuga>) { println!("fuga rc") }

fn main() {
    let hoge = Hoge {};
    let hoge_box = Box::new( Hoge{} );
    let hoge_rc = Rc::new( Hoge{} );

    fuga_ref(&hoge);    // fuga ref
    fuga_box(hoge_box); // fuga box
    fuga_rc(hoge_rc);   // fuga rc
}
```

[^1]: c++では空構造体はサイズ1らしい(もしかしたら環境依存かも)
[^2]: 正直c++とrustのコードを書いたが、あまり自信がないため間違っているかもしれない。

### トレイトオブジェクトとジェネリック関数について

複数の型が入り混じっているコレクションを使う場合はトレイトオブジェクトのほうが良い。ジェネリックを使ったコレクションの場合は一つのコレクションに対して一つの型しか作れない。またトレイトオブジェクトはジェネリックに比べコードサイズが少なくなるため制限がある場合は利用したほうがいい。

```rust
trait Hoge {
    fn _func(&self) { }
}

struct Fuga;
struct Piyo;

impl Hoge for Fuga { }
impl Hoge for Piyo { }

struct Foo {
    hoge_vec: Vec<Box<dyn Hoge>>,
}

struct Bar<T: Hoge> {
    hoge_vec: Vec<T>
}

impl Foo {
    pub fn add(&mut self, hoge: Box<dyn Hoge>) { self.hoge_vec.push(hoge) }
}
impl<T: Hoge> Bar<T> {
    pub fn add(&mut self, hoge: T) { self.hoge_vec.push(hoge) }
}

fn main() {
    let mut foo = Foo{ hoge_vec: Vec::default() };  // Hogeトレイトを実装したものならなんでも入る
    foo.add(Box::new(Fuga{}));
    foo.add(Box::new(Piyo{}));

    let mut bar = Bar{ hoge_vec: Vec::<Fuga>::default() }; // Bar<Fuga>型で作成される
    bar.add(Fuga{});
    //bar.add(Piyo{});  // Bar::hoge_vecは Vec<Fuga> になるため Fuga型ではないPiyo型のものを入れることが出来ない
}

```

ジェネリックの利点は、実行速度が早い、レイトオブジェクトで使用できないトレイトの機能が使用できる、型パラメータに対する制約を書くことが出来る。

### 拡張トレイト

既存の型やトレイトに対して実装するトレイトを拡張トレイトと呼ぶ。

```rust
use std::fmt::Binary;


trait Hoge {
    fn get_size(&self) -> usize { std::mem::size_of_val(self) }
}

impl Hoge for u32 { }  // u32型の拡張

trait Fuga {
    fn get_size2(&self) -> usize { std::mem::size_of_val(self)}
}
impl<T: Binary> Fuga for T { } // Binaryトレイトを実装しているものに対して拡張

fn main() {
    println!("{}", 10u32.get_size());   // 4
    //println!("{}", 10u64.get_size()); // Hogeトレイトはu32の拡張トレイトしか実装していないためエラー
    println!("{}", 10u32.get_size2());  // 4
    println!("{}", 10u64.get_size2());  // 8
    // u32やu64などBinaryトレイトを実装しているものはFugaトレイトの関数を呼び出せる
}
```

### Self

Self型を使う場合はトレイトオブジェクトを使用できない。

### サブトレイト

あるトレイトに対して他のトレイトの拡張として宣言する

```rust
trait Hoge {
    fn get_zero(&self) -> u32 { 0 }
}

trait Fuga: Hoge { // Hogeの拡張としてFugaを作成
    fn print_zero(&self) { println!("{}", self.get_zero() )}
}

struct Foo;

impl Hoge for Foo {} // この行をコメントアウトするとしたの行でエラーを吐く
impl Fuga for Foo {} // エラーの原因はFugaトレイトはHogeトレイトを実装しているものにしか実装できないため

fn main() {
    Foo.print_zero(); // 0
}

```

### 型関連

型関連はトレイトで型の関係を定義するもの。[^3]

```rust
trait Hoge {
    type ReturnType;

    fn get_value(&self) -> Self::ReturnType;
}

struct Fuga {
    value: u32,
}

impl Hoge for Fuga {
    type ReturnType = u32;

    fn get_value(&self) -> Self::ReturnType {
        self.value
    }
}

fn main() {
    let fuga = Fuga { value: 0 };
    let x = fuga.get_value();

    println!("{}", x)
}
```

[^3]: ジェネリックとあまり違いが判らない。もう少し調査して必要に応じて使えるようにしておきたい。

### impl trait

impl traitは引数や戻り値に特定のトレイトを実装したものをだけを指定留守ことが出来る

```rust
trait Hoge { 
    fn print_type(&self);
}

struct Foo;
struct Bar;

impl Hoge for Foo {
    fn print_type(&self) {
        println!("Foo")
    }
}
impl Hoge for Bar {
    fn print_type(&self) {
        println!("Bar")
    }
}

fn print_type(value: &impl Hoge) { value.print_type() } // 引数にHogeトレイトを指定
//fn print_type<T: Hoge>(value: &T) { value.print_type() } // ジェネリックで書いた場合

fn main() {
    let foo = Foo {};
    let bar = Bar {};

    print_type(&foo); // Foo
    print_type(&bar); // Bar
}
```

また制限としてトレイトメソッドでは使用することができない。

```rust
trait Hoge { 
}
trait Fuga {
    fn func(&self) -> impl Hoge; // トレイトメソッドでimpl trait を使っているためコンパイルエラー
}
```

### トレイト関連定数

トレイトでも定数を宣言することが出来る。構造体などとの違いはトレイト内で値を指定せず、実装時に値を指定することが出来る

```rust
trait Value { 
    const VALUE: u32 = 10;
}
trait MaxValue {
    const VALUE: Self; // 宣言だけ　
}

impl MaxValue for i32 {
    const VALUE: i32 = i32::MAX;
}
impl MaxValue for i64 {
    const VALUE: i64 = i64::MAX;
}
```

## 13章

### Drop

C++のデストラクタのようなもの。明示的に呼び出すことが出来ず、暗黙的に呼び出される。
値が移動されるなどして未初期化状態になったものは呼び出されない。

```rust
struct Hoge {
    name: String,
}

impl Hoge {
struct Hoge {
    name: String,
}

impl Hoge {
    fn new(name: &str) -> Self {
        println!("new {}", name);
        Hoge { name: name.to_string() }
    }
}

impl Drop for Hoge {
    fn drop(&mut self) {
        println!("Dropping {}", self.name)
    }
}

fn main() {
    let foo = Hoge::new("Foo"); // new Foo
    {
        let _foo2 = foo;
    } // Dropping Foo
    println!("end block");
} // fooの変数がブロック内で移動し、未初期化状態になったためDropが呼び出されない
```
