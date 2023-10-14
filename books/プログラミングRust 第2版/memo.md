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