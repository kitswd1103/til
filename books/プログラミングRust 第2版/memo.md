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
