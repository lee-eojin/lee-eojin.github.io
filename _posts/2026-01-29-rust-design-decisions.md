---
layout: article
title: "왜 Rust는 이런 규칙들을 만들었을까."
category: "TIL"
tag: "Rust"
comment: true
key: 20260129
---

Rust 한 2주정도 개념을 공부해보고 처음 써봤는데, 다른 언어랑 다른 규칙들이 많음. 처음엔 왜 이래 싶었긴한데 알고 보니 다 이유가 있었음. <a href="https://github.com/lee-eojin/crab-szn/tree/main/baseball" target="_blank">숫자야구</a> 만들면서 배운 것들 + 왜 Rust가 이런 design decision을 했는지 정리.

(갑자기 Rust를 왜 해봤냐고? 그냥 어차피 2월 말까지 자유야 나는 ㅋ)

## Rust가 해결하려는 문제

Rust 쓰기 전에 "메모리 안전한 시스템 언어", "C++ 대체" 이런 말만 들었음. 직접 써보니까 왜 이런 말이 나오는지 막상 해봐야 체감됨.

핵심은 **compile time에 메모리 안전성을 보장**한다는 것. 보통 언어들은:

- **C/C++** - 프로그래머가 직접 메모리 관리. 빠르지만 실수하면 segfault, memory leak, dangling pointer 등 터짐
- **Java, Go, JS 등** - GC(Garbage Collector)가 메모리 관리. 편하지만 GC pause 발생, 메모리 사용량 예측 어려움

Rust는 제3의 길을 택함. **Ownership System**이라는 규칙을 만들어서, 이 규칙을 지키면 compile time에 메모리 안전성이 보장됨. GC 없이. 실제로 써보니까 compiler가 진짜 엄격하게 잡아줌. 대신 learning curve가 좀 있음.

## Project Init

```bash
cargo new baseball
```

`cargo`는 Rust의 공식 build tool + package manager. npm이나 pip 같은 건데, build까지 담당함.

생성되는 구조:

```
baseball/
├── Cargo.toml   # manifest file
└── src/
    └── main.rs  # entry point
```

### Cargo.toml 뜯어보기

```toml
[package]
name = "baseball"
version = "0.1.0"
edition = "2024"

[dependencies]
```

**edition이 뭔가?**

Rust는 매년 새 기능이 추가되는데, 가끔 breaking change가 필요할 때가 있음. 근데 기존 코드를 깨뜨리고 싶진 않음. 그래서 edition 개념을 도입함.

- edition마다 문법이 조금씩 다름
- 하지만 다른 edition 코드끼리 서로 의존 가능 (!)
- compiler가 내부적으로 호환성 처리함

이게 진짜 좋은 점은, 10년 된 Rust 코드도 최신 compiler로 빌드 가능하면서, 새 프로젝트는 최신 문법 쓸 수 있다는 것.

**[dependencies]는 언제 fetch되나?**

npm처럼 `cargo install` 같은 거 안 해도 됨. `cargo build`나 `cargo run` 하면 Cargo.toml 읽어서 필요한 거 알아서 다운받음. 그것도 첫 빌드 때만. 이후엔 캐시됨.

다운받는 곳은 <a href="https://crates.io" target="_blank">crates.io</a>. npm registry 같은 역할. 참고로 Rust에서 package는 **crate**라고 부름.

### build는 어떻게?

```bash
cargo build              # debug build
cargo build --release    # release build (최적화)
cargo run                # build + 실행
cargo check              # type check만 (binary 안 만듦)
```

`cargo check`가 꿀인 게, 전체 compile 안 하고 type check만 해서 엄청 빠름. 코드 수정하면서 자주 돌리면 좋음.

**debug vs release 차이**

처음에 "왜 mode가 두 개지?" 싶었는데, 실제로 성능 차이가 큼.

debug mode:
- 최적화 없음
- overflow check 같은 runtime check 활성화
- compile 빠름, 실행 느림

release mode:
- 최적화 적용 (inlining, loop unrolling 등)
- 일부 runtime check 비활성화
- compile 느림, 실행 빠름

숫자야구 정도는 차이 못 느끼겠지만, CPU-intensive한 작업에서는 10배 이상 차이 나기도 함.

## Random Number Generation

Rust stdlib에 random이 없음. 처음엔 "왜 이런 기본적인 것도 없지?" 싶었는데, 이유가 있음.

- stdlib을 최대한 minimal하게 유지하는 철학
- random의 use case가 다양함 (게임용 빠른 RNG vs 암호화용 secure RNG)
- ecosystem에서 각자 필요에 맞는 거 쓰라는 것

de facto standard는 `rand` crate.

```toml
[dependencies]
rand = "0.8"
```

### Version Syntax 잠깐

```toml
rand = "0.8"       # >=0.8.0, <0.9.0
rand = "=0.8.5"    # exactly 0.8.5
rand = ">=0.8"     # 0.8 이상 아무거나
```

`"0.8"`이 `>=0.8.0, <0.9.0` 의미인 게 처음엔 헷갈렸는데, <a href="https://semver.org" target="_blank">SemVer</a> 규칙 따르는 것.

MAJOR.MINOR.PATCH 중 MAJOR가 바뀌면 breaking change, MINOR는 backward compatible 기능 추가, PATCH는 버그 수정. 그래서 같은 MINOR 내에서는 업데이트해도 안전하다고 가정하는 것.

### 실제 코드

```rust
use rand::seq::SliceRandom;

fn generate_number() -> Vec<u8> {
    let mut nums: Vec<u8> = (1..=9).collect();
    nums.shuffle(&mut rand::thread_rng());
    nums[0..3].to_vec()
}
```

한 줄씩 뜯어보면:

**`use rand::seq::SliceRandom;`**

`SliceRandom`은 trait. slice에 `shuffle()` 메서드를 추가해줌.

여기서 Rust의 재밌는 점 하나. trait을 scope에 가져와야 해당 메서드를 쓸 수 있음. 이게 처음엔 귀찮았는데, 생각해보니까:
- 어떤 메서드가 어디서 온 건지 명확함
- 이름 충돌 방지

**`let mut nums: Vec<u8> = (1..=9).collect();`**

`1..=9`는 1부터 9까지 range. `=` 붙이면 끝값 포함 (inclusive).

`.collect()`는 iterator를 collection으로 바꿔줌. 근데 어떤 collection으로? 그건 type annotation 보고 추론함. 여기선 `Vec<u8>`.

**`nums.shuffle(&mut rand::thread_rng());`**

`thread_rng()`는 thread-local RNG 반환. 왜 thread-local이냐면, RNG가 internal state를 가지고 있어서 여러 thread에서 공유하면 문제 생김.

`&mut`가 붙는 이유는 shuffle이 RNG의 state를 변경하기 때문. Rust에서는 뭔가를 변경하려면 명시적으로 mutable reference를 넘겨야 함.

**`nums[0..3].to_vec()`**

앞에서 3개만 잘라서 새 Vec로. `[0..3]`은 slicing.

### Vec 좀 더 깊게

Vec는 Rust의 dynamic array. heap에 할당됨.

```rust
let v: Vec<i32> = Vec::new();           // empty
let v = vec![1, 2, 3];                   // macro로 초기화
let v: Vec<u8> = (1..=9).collect();      // iterator에서
```

내부적으로 이렇게 생김:

```
Stack:                   Heap:
┌──────────────┐        ┌───┬───┬───┬───┬───┐
│ ptr ─────────┼───────>│ 1 │ 2 │ 3 │   │   │
│ len: 3       │        └───┴───┴───┴───┴───┘
│ capacity: 5  │
└──────────────┘
```

- `ptr` - heap 데이터 시작 주소
- `len` - 현재 element 수
- `capacity` - 할당된 공간

capacity 초과하면? 새 메모리 할당하고 기존 데이터 copy. 그래서 `push()`가 평균 O(1)이지만 가끔 O(n).

미리 크기 알면 `Vec::with_capacity(n)`으로 할당해두면 좋음.

### Integer Types

Rust integer type이 진짜 많음.

```
Signed:    i8    i16    i32    i64    i128    isize
Unsigned:  u8    u16    u32    u64    u128    usize
```

왜 이렇게 세분화했을까? 시스템 프로그래밍 언어라서.

- 메모리 사용량 정확히 제어 가능
- 특정 크기가 필요한 경우 (network protocol, file format 등)
- overflow 동작 명확

`isize`, `usize`는 architecture dependent. 64bit 시스템이면 64bit. array indexing에는 항상 `usize` 씀.

**overflow가 나면?**

```rust
let x: u8 = 255;
let y = x + 1;  // debug mode에서 panic!
```

debug mode에서는 panic. release mode에서는 wrap around (255 + 1 = 0).

명시적으로 처리하고 싶으면:

```rust
x.wrapping_add(1);    // 0 (항상 wrap)
x.saturating_add(1);  // 255 (최대값에서 멈춤)
x.checked_add(1);     // None (Option 반환)
```

처음엔 "귀찮게 왜 이래" 싶었는데, integer overflow 버그가 실제로 많이 터지는 걸 생각하면 합리적.

### let과 mut

Rust의 가장 특이한 점 중 하나. **변수가 기본적으로 immutable**.

```rust
let x = 5;
x = 6;        // compile error!

let mut y = 5;
y = 6;        // ok
```

왜 이런 design decision을 했을까?

1. **버그 방지** - 값이 안 바뀐다는 게 보장되면 코드 reasoning이 쉬워짐
2. **concurrency** - immutable이면 여러 thread에서 동시 접근해도 안전
3. **optimization** - compiler가 값이 안 바뀐다는 걸 알면 최적화 기회 늘어남

처음엔 귀찮았는데, 실제로 코드 짜다 보니까 "이거 실수로 바꾼 거 아닌가?" 하는 버그가 줄어듦.

**shadowing은 다름**

```rust
let x = 5;
let x = x + 1;    // 새 변수 (shadowing)
let x = "hello";  // type도 바꿀 수 있음
```

같은 이름으로 재선언하는 것. `mut`이랑 다른 점은, 아예 새 변수가 만들어진다는 것.

### Iterator가 lazy라는 것

```rust
let nums: Vec<u8> = (1..=9).collect();
```

`(1..=9)`만 쓰면 실제로 숫자가 생성되진 않음. `.collect()` 같은 consumer를 호출해야 실제 연산이 일어남.

이게 왜 좋냐면:

```rust
let result: Vec<i32> = (1..1000000)
    .map(|x| x * 2)
    .filter(|x| x > 100)
    .take(10)
    .collect();
```

이거 중간에 100만개짜리 Vec 안 만듦. 필요한 것만 그때그때 계산. memory efficient하고 빠름.

## User Input

```rust
use std::io::{self, Write};

fn get_input() -> Vec<u8> {
    print!("숫자를 입력하세요: ");
    io::stdout().flush().unwrap();

    let mut input = String::new();
    io::stdin().read_line(&mut input).unwrap();

    input
        .trim()
        .chars()
        .filter_map(|c| c.to_digit(10).map(|d| d as u8))
        .collect()
}
```

### flush가 왜 필요한가

`println!`은 자동으로 flush 되는데, `print!`는 안 됨. stdout이 보통 line-buffered라서 newline 만나야 실제로 출력됨.

```rust
print!("숫자를 입력하세요: ");  // 아직 화면에 안 나올 수 있음
io::stdout().flush().unwrap();  // 이제 나옴
```

이거 처음에 몰라서 프롬프트가 안 뜨는 줄 알았음.

### Result와 Error Handling

Rust에 exception 없음. 대신 `Result` enum.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`read_line()`의 return type은 `Result<usize, io::Error>`. 성공하면 `Ok(읽은 바이트 수)`, 실패하면 `Err(에러)`.

처리하는 방법이 여러 가지:

```rust
// 1. unwrap - 실패하면 panic
let value = result.unwrap();

// 2. expect - panic + custom message
let value = result.expect("파일 읽기 실패");

// 3. match로 직접 처리
match result {
    Ok(v) => println!("성공: {}", v),
    Err(e) => println!("실패: {}", e),
}

// 4. ? operator - 에러면 early return
fn foo() -> Result<i32, Error> {
    let v = some_operation()?;  // 에러면 여기서 바로 return Err
    Ok(v + 1)
}
```

`?` operator가 진짜 편함. error propagation을 한 글자로.

처음엔 "exception이 왜 없어?" 싶었는데, 써보니까 Result가 더 명확함. 어떤 함수가 에러를 낼 수 있는지 signature만 봐도 앎.

### Ownership과 Borrowing

Rust 핵심 중의 핵심. 이게 compile time memory safety의 비결.

**Ownership Rules**

1. 각 value는 owner(변수)가 정확히 하나
2. owner가 scope 벗어나면 value drop (메모리 해제)
3. ownership은 이동(move) 가능

```rust
let s1 = String::from("hello");
let s2 = s1;          // ownership이 s2로 이동
println!("{}", s1);   // compile error! s1은 더 이상 유효하지 않음
```

이게 처음에 개같이 헷갈렸음. 다른 언어에서 당연히 되는 게 안 되니까.

근데 생각해보면 합리적. s1과 s2가 같은 heap 메모리를 가리키면, 둘 중 하나가 해제할 때 문제 생김 (double free). Rust는 애초에 owner를 하나로 제한해서 이 문제를 원천 차단.

**Borrowing**

ownership 안 넘기고 참조만 빌려주는 것.

```rust
let s = String::from("hello");
let len = calculate_length(&s);  // s를 빌려줌
println!("{}", s);               // s 여전히 유효
```

**Reference Rules**

여기가 핵심.

- `&T` (immutable reference) - 여러 개 동시에 가능
- `&mut T` (mutable reference) - 오직 하나만
- 둘을 동시에 가질 수 없음

```rust
let mut s = String::from("hello");

let r1 = &s;      // ok
let r2 = &s;      // ok
let r3 = &mut s;  // compile error!
```

"읽는 사람 여러 명 ok, 쓰는 사람은 한 명만, 쓰는 사람 있으면 읽는 사람도 안 됨"

이 규칙 덕분에 **data race가 compile time에 방지**됨. data race 조건이:
1. 두 개 이상의 포인터가 같은 데이터 접근
2. 적어도 하나가 쓰기
3. 동기화 없음

Rust reference rules가 이걸 원천 차단.

### String과 &str

처음에 헷갈렸던 것 중 하나.

```rust
let s: String = String::from("hello");  // owned, heap
let s: &str = "hello";                   // borrowed, static
```

- `String` - owned, heap 할당, 변경 가능
- `&str` - borrowed string slice, 읽기 전용

함수 parameter로는 보통 `&str` 받음. `String`이든 `&str`이든 다 받을 수 있어서.

```rust
fn greet(name: &str) {
    println!("Hello, {}", name);
}

greet("world");                    // &str
greet(&String::from("world"));     // String에서 &str로 자동 변환
```

### Closure

```rust
let add = |a, b| a + b;
let result = add(1, 2);
```

`|args| body` 형태. JavaScript arrow function이랑 비슷.

재밌는 건 **environment capture**.

```rust
let x = 4;
let equal_to_x = |z| z == x;  // x를 캡처
```

캡처하는 방식이 세 가지:
- `Fn` - immutable borrow
- `FnMut` - mutable borrow
- `FnOnce` - ownership move

compiler가 알아서 가장 덜 제한적인 거 선택함. 강제로 move하고 싶으면 `move` keyword.

```rust
let s = String::from("hello");
let closure = move || println!("{}", s);  // s ownership 이동
// 여기서 s 못 씀
```

thread 넘길 때 자주 쓰는듯. thread가 언제 끝날지 모르니까 reference보다 ownership 넘기는 게 안전.

## Struct와 Impl

OOP의 class와 비슷하지만, data(struct)와 method(impl)가 분리되어 있음.

```rust
struct Game {
    answer: Vec<u8>,
    attempts: u32,
}

impl Game {
    // associated function (생성자 같은 거)
    fn new() -> Self {
        Game {
            answer: vec![1, 2, 3],
            attempts: 0,
        }
    }

    // method
    fn judge(&mut self, guess: &Vec<u8>) -> Score {
        self.attempts += 1;
        // ...
    }
}
```

**왜 분리했을까?**

- struct에 method를 여러 impl block으로 나눠서 정의 가능
- trait impl도 별도 impl block
- 코드 구조화에 유연함

**self의 종류**

```rust
fn method(self)        // ownership 가져감, 호출 후 원본 사용 불가
fn method(&self)       // immutable borrow
fn method(&mut self)   // mutable borrow
```

대부분 `&self`나 `&mut self` 씀.

### Derive Macro

common trait 자동 구현.

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

- `Debug` - `{:?}`로 출력 가능
- `Clone` - `.clone()` method
- `PartialEq` - `==` 비교 가능

이거 없으면 직접 impl 해야 함. boilerplate 줄여줌.

## Module System

Rust module system이 처음엔 좀 낯설었음. **파일 만든다고 자동으로 인식되지 않음**.

```
src/
├── main.rs        # crate root
├── constants.rs
├── game.rs
└── view.rs
```

```rust
// main.rs
mod constants;  // constants.rs를 모듈로 선언
mod game;
mod view;

use game::Game;
```

`mod constants;` 이거 안 하면 constants.rs 파일이 있어도 compiler가 무시함. error도 안 남. 처음에 이거 몰라서 한참 헤맴.

### Visibility

기본이 **private**. `pub` 붙여야 외부에서 접근 가능.

```rust
pub struct Game { ... }      // struct는 public
    answer: Vec<u8>,         // field는 private
    pub attempts: u32,       // 이것만 public

pub fn new() -> Self { ... } // method도 pub 필요
```

이것도 처음엔 귀찮았는데, 생각해보면 좋은 default. 의도적으로 공개하는 것만 공개.

### crate, super, self

```rust
use crate::constants::DIGIT_COUNT;  // crate root부터
use super::something;                // parent module
use self::something;                 // current module
```

Node.js의 `../` `./` 같은 건데, 더 명시적.

## Strike/Ball Logic

```rust
pub struct Score {
    pub strike: u8,
    pub ball: u8,
}

impl Game {
    pub fn judge(&mut self, guess: &Vec<u8>) -> Score {
        self.attempts += 1;

        let mut strike = 0;
        let mut ball = 0;

        for (i, g) in guess.iter().enumerate() {
            if self.answer[i] == *g {
                strike += 1;
            } else if self.answer.contains(g) {
                ball += 1;
            }
        }

        Score { strike, ball }
    }
}
```

### enumerate

Python enumerate랑 같음. index랑 value 같이 줌.

```rust
for (i, g) in guess.iter().enumerate() {
    // i: usize, g: &u8
}
```

`g`가 `&u8`인 이유는 `.iter()`가 reference iterator를 반환해서. `.into_iter()`면 ownership 이동.

### Dereference

```rust
if self.answer[i] == *g { }
```

`g`가 `&u8`이니까 `*g`로 실제 값 꺼냄.

근데 사실 Rust가 자동으로 deref 해주는 경우도 많음. 여기선 명시적으로 씀.

## Game Loop

```rust
fn main() {
    let mut game = Game::new();

    loop {
        let guess = game.get_input();
        let score = game.judge(&guess);

        view::print_score(&score);

        if score.is_win() {
            view::print_win(game.attempts());
            break;
        }
    }
}
```

### loop vs while true

```rust
loop { }         // 이거 씀
while true { }   // 이거 안 씀
```

`loop`이 권장되는 이유:
1. compiler가 infinite loop임을 알아서 최적화
2. `loop`은 값을 반환할 수 있음

```rust
let result = loop {
    if done {
        break 42;  // loop의 결과값
    }
};
```

이거 다른 언어에선 못 본 feature. break로 값 반환.

## Final Structure

```
src/
├── main.rs        # entry point, game loop
├── constants.rs   # magic number 상수화
├── game.rs        # Game, Score struct
└── view.rs        # print functions
```

작은 프로젝트지만 모듈 분리 연습하기 좋았음.

## 마무리

Rust 공부하다가 처음 적용해본거긴한데, compiler가 strict한 게 처음엔 짜증났다가 나중엔 고마웠음. "이거 왜 안 돼?"하고 에러 메시지 읽으면 대부분 합리적인 이유가 있음.

특히 ownership/borrowing이 핵심인데, 이거 이해하면 나머지는 따라옴. "왜 Rust가 이런 규칙을 만들었을까?" 생각하면서 공부하면 더 와닿음.

## References

- <a href="https://doc.rust-lang.org/book/" target="_blank">The Rust Programming Language</a> - 공식 book. 이거 진짜 잘 되어있음
- <a href="https://doc.rust-lang.org/rust-by-example/" target="_blank">Rust by Example</a> - 예제로 배우기
- <a href="https://doc.rust-lang.org/std/" target="_blank">Standard Library Docs</a>
- <a href="https://cheats.rs/" target="_blank">Rust Cheat Sheet</a>
