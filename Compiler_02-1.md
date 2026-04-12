# 컴파일러 — Chapter 2-1 컴파일러 구조 Part I

---

## 1. 컴파일러 구조 (개요)

```
소스 프로그램
       ↓
  ┌──────────────────┐
  │      전단부       │
  │   어휘 분석       │
  │      ↓           │
  │   구문 분석       │
  │      ↓           │
  │   의미 분석       │
  │      ↓           │
  │  중간 코드 생성   │
  └──────────────────┘
       ↓
  ┌──────────────────┐
  │      후단부       │
  │   코드 최적화     │
  │      ↓           │
  │   코드 생성       │
  └──────────────────┘
       ↓
목적 프로그램
```

---

## 2. 컴파일러 상세 구조

```
                    소스 프로그램
                         ↓
         ┌───────────────────────────────┐
         │  어휘 분석기의 어휘 분석        │ ←── 기호표 관리    에러 처리
         │         ↓ 토큰 스트림          │          ↕              ↕
         │  구문 분석기의 구문 분석        │ ←─────────────────────────
         │         ↓ 파스 트리           │
         │  의미 분석기의 의미 분석        │ ←─────────────────────────
         └───────────────────────────────┘
                    ↓ 구문 트리
         ┌───────────────────────────────┐
         │  중간 코드 생성기의 중간 코드 생성│ ←─────────────────────────
         │         ↓ 중간 코드 표현       │
         │  코드 최적화기의 코드 최적화    │ ←─────────────────────────
         │         ↓ 중간 코드 표현       │
         │  코드 생성기의 목적 코드 생성   │ ←─────────────────────────
         └───────────────────────────────┘
                    ↓ 목적 기계 코드
                목적 프로그램
```

- **전반부 (분석 단계)** : 어휘 분석 → 구문 분석 → 의미 분석
- **후반부 (합성 단계)** : 중간 코드 생성 → 코드 최적화 → 코드 생성
- 좌측 : 기호표 관리 (모든 단계 접근)
- 우측 : 에러 처리 (모든 단계 접근)

---

## 3. ac 언어 소개

아주 간단한 언어. 컴파일러 구현 실습용.

**ac (adding calculator)**

- **데이터 타입** : 정수, 실수
  - 실수는 소수점 이하 5자리까지만 허용
- **키워드** : `f` (float), `i` (integer), `p` (print)
- **변수** : 알파벳 소문자 23자 (키워드 3개 `f`, `i`, `p` 제외)
  - 변수는 사용하기 전에 먼저 선언해야 함
- **형 변환** :
  - 정수형 → 실수형 변환은 자동
  - 다른 종류의 형 변환은 허용하지 않음

**목적 코드 : dc (desk calculator)**
- 스택 기반 계산기
- 역폴란드 표기법으로 코드 생성

```
ac 프로그램 : 2 + 3
코드 생성   : 2 3 +    ← 연산자가 뒤에 옴 (역폴란드 표기법)
실행 결과   : 5
```

> - **스택 기반 계산기** : 스택(stack)에 값을 쌓아두고, 연산자가 나오면 스택에서 값을 꺼내 계산하는 방식
> - **역폴란드 표기법** : 연산자를 피연산자 뒤에 쓰는 표기법. `2 + 3` 을 `2 3 +` 로 표현. 괄호 없이도 연산 순서가 명확해서 컴퓨터가 처리하기 쉬움

---

## 4. 문맥 자유 문법 (CFG for ac)

**CFG (Context-Free Grammar)** : ac 언어의 문법을 정의하는 15개의 생성 규칙

```
 1  Prog  → Dcls Stmts $
 2  Dcls  → Dcl Dcls
 3        | λ
 4  Dcl   → floatdcl id
 5        | intdcl id
 6  Stmts → Stmt Stmts
 7        | λ
 8  Stmt  → id assign Val Expr
 9        | print id
10  Expr  → plus Val Expr
11        | minus Val Expr
12        | λ
13  Val   → id
14        | inum
15        | fnum
```

---

## 5. 생성 규칙 (Production / Rewriting Rule)

**생성 규칙이란?**

1. 화살표(`→`) **왼쪽**에 놓인 기호는 화살표 **오른쪽** 문자열로 확장해서 나타낼 수 있다
2. `|` : **또는(or)** 이란 뜻

```
Stmt → id assign Val Expr
     | print id
```

위 규칙의 의미:
- `Stmt` 는 `id assign Val Expr` 로 표현할 수 있다
- 또는 `Stmt` 는 `print id` 로도 표현할 수 있다

**순환(반복) 정의** : 자기 자신을 포함하는 규칙

```
2  Dcls → Dcl Dcls    ← Dcls 안에 Dcls가 다시 등장 → 반복(순환) 정의
```

**상세 정의** : 더 구체적인 규칙으로 확장

```
6  Stmts → Stmt Stmts
8  Stmt  → id assign Val Expr  ← Stmts를 Stmt로 상세하게 정의
```

---

## 6. 비단말 기호 (Nonterminal)

생성 규칙의 **왼쪽(LHS)** 에 나오는 기호. 왼쪽, 오른쪽 모두 사용 가능.

```
Nonterminals = { Prog, Dcls, Dcl, Stmts, Stmt, Expr, Val }
```

- **시작 기호 (Start symbol)** : 비단말 기호 중 하나
  - ac의 시작 기호 = `Prog` (생성 규칙 1번 왼쪽)
  - 단, 시작 기호는 생성 규칙 왼쪽에 **한 번만** 사용할 수 있다

```
1  Prog → Dcls Stmts $    ← Prog가 시작 기호
```

---

## 7. 단말 기호 (Terminal)

생성 규칙의 **오른쪽(RHS)에만** 나오는 기호. 더 이상 확장할 수 없음 (생성 규칙 없음).

```
terminals = { floatdcl, intdcl, id, assign, print, plus, minus, inum, fnum, $, λ }
```

| 단말 기호 | 실제 입력 기호 |
|----------|-------------|
| floatdcl | `f` |
| intdcl | `i` |
| assign | `=` |
| plus | `+` |
| minus | `-` |
| print | `p` |
| id | `a`, `b`, `c`, … |
| inum | `12`, `345`, … |
| fnum | `0.1`, `3.14`, … |

**특수 단말 기호**
- **`$`** : 입력 스트림의 끝 (the end of input stream). 실제로 입력하지 않지만 끝까지 읽었는지 확인하는 용도
- **`λ` (람다) 또는 `ε` (입실론)** : 빈 문자열(empty string / null string). 생략 가능함을 의미

**비단말 vs 단말 기호 정리**

| 구분 | 위치 | 생성 규칙 |
|------|------|---------|
| 비단말 기호 (Nonterminal) | 왼쪽, 오른쪽 모두 | 있음 |
| 단말 기호 (Terminal) | 오른쪽만 | 없음 |
| 시작 기호 (Start symbol) | 왼쪽에 한 번만 | 있음 |

> 실제 언어 문법 예시:
> - XML 명세 : http://www.w3.org/TR/REC-xml/
> - Python 문법 : https://docs.python.org/3/reference/grammar.html

---

## 8. ac 프로그램 예시

```
f b        ← b를 실수(float)로 선언
i a        ← a를 정수(integer)로 선언
a = 5      ← a에 5 대입
b = a + 3.2 ← b에 a + 3.2 대입 (정수→실수 자동 형변환)
p b        ← b 출력
```

**단말 기호와 실제 입력 대응**

```
f b        →  floatdcl id   (선언부)
i a        →  intdcl id     (선언부)
a = 5      →  id assign inum (문장)
b = a + 3.2 → id assign id plus fnum (문장)
p b        →  print id      (문장)
```

---

## 9. 유도 (Derivation)

시작 기호부터 생성 규칙을 적용해서 실제 입력 프로그램으로 변환하는 과정.
**한 번에 한 개의 비단말 기호에 대한 생성 규칙 적용**.

`f b / i a / a = 5 / b = a + 3.2 / p b` 의 유도 과정:

```
단계 | 문장 형태 (Sentential Form)                                    | 적용 규칙
-----|------------------------------------------------------------------|----------
  1  | <Prog>                                                          |
  2  | <Dcls> Stmts $                                                  | 1
  3  | <Dcl> Dcls Stmts $                                              | 2
  4  | floatdcl id <Dcls> Stmts $                                      | 4
  5  | floatdcl id <Dcl> Dcls Stmts $                                  | 2
  6  | floatdcl id intdcl id <Dcls> Stmts $                            | 5
  7  | floatdcl id intdcl id <Stmts> $                                 | 3
  8  | floatdcl id intdcl id <Stmt> Stmts $                            | 6
  9  | floatdcl id intdcl id id assign <Val> Expr Stmts $              | 8
 10  | floatdcl id intdcl id id assign inum <Expr> Stmts $             | 14
 11  | floatdcl id intdcl id id assign inum <Stmts> $                  | 12
 12  | floatdcl id intdcl id id assign inum <Stmt> Stmts $             | 6
 13  | floatdcl id intdcl id id assign inum id assign <Val> Expr Stmts $| 8
 14  | floatdcl id intdcl id id assign inum id assign id <Expr> Stmts $ | 13
 15  | floatdcl id intdcl id id assign inum id assign id plus <Val> Expr Stmts $ | 10
 16  | floatdcl id intdcl id id assign inum id assign id plus fnum <Expr> Stmts $| 15
 17  | floatdcl id intdcl id id assign inum id assign id plus fnum <Stmts> $     | 12
 18  | floatdcl id intdcl id id assign inum id assign id plus fnum <Stmt> Stmts $| 6
 19  | floatdcl id intdcl id id assign inum id assign id plus fnum print id <Stmts> $ | 9
 20  | floatdcl id intdcl id id assign inum id assign id plus fnum print id $    | 7
```

최종 결과: `floatdcl id intdcl id id assign inum id assign id plus fnum print id $`
= `f b i a a = 5 b = a + 3.2 p b`  ✅

---

## 10. 토큰 명세 (Token Specification)

각 단말 기호를 **정규 표현식(Regular Expression)** 으로 형식적으로 정의.

| 단말 기호 (토큰 형, type) | 정규 표현식 (토큰 값, value) | 의미 |
|--------------------------|--------------------------|------|
| floatdcl | `"f"` | 키워드 |
| intdcl | `"i"` | 키워드 |
| print | `"p"` | 키워드 |
| id | `[a-e] \| [g-h] \| [j-o] \| [q-z]` | 변수 (키워드 제외 소문자) |
| assign | `"="` | 대입 연산자 |
| plus | `"+"` | 덧셈 |
| minus | `"-"` | 뺄셈 |
| inum | `[0-9]+` | 정수 (1개 이상의 숫자) |
| fnum | `[0-9]+.[0-9]+` | 실수 (소수점 포함) |
| blank | `(" ")+` | 공백 (1개 이상) |

> 정규 표현에 **메타 심볼(meta symbol)** 을 사용할 수 있음
> - `+` : 1개 이상 반복
> - `|` : 또는(or)
> - `[a-z]` : a부터 z 사이의 문자 중 하나

- **토큰의 형 (type)** : 단말 기호 이름 (floatdcl, id, inum 등)
- **토큰의 값 (value)** : 실제 입력된 문자열 (f, a, 5 등)

---

## 11. 파스 트리 (Parse Tree)

유도 과정을 트리 구조로 시각화한 것.

`f b / i a / a = 5 / b = a + 3.2 / p b` 의 파스 트리:

```
                              Prog
                    /          |           \
                 Dcls         Stmts          $
                /    \           \
              Dcl    Dcls        Stmts
              / \    /    \         \
        floatdcl id  Dcl  Dcls     Stmts
             |   |  / \    |          \
             f   b intdcl id λ        Stmt       Stmts
                    |    |              |              \
                    i    a           Stmt            Stmts
                                   /  |  \           /    \
                                  id assign Val   Expr  Stmt  Stmts
                                  |    |    |      |     |
                                  a    =   Val   Expr  print  id  λ
                                        |    |      |         |
                                       inum  id   plus  Val  λ   p   b
                                        |    |     |     |
                                        5    b     +    Val  Expr
                                                        |      |
                                                       fnum    λ
                                                        |
                                                       3.2
```

> 리프 노드(가장 아래 노드)를 왼쪽부터 읽으면 실제 입력 프로그램이 됨:
> `f b i a a = 5 b = a + 3.2 p b`

---

## 핵심 요약

```
ac 언어
  키워드  : f(float), i(integer), p(print)
  변수    : 소문자 알파벳 23자 (f, i, p 제외)
  목적 코드: dc (역폴란드 표기법)

CFG 구성 요소
  비단말 기호 (Nonterminal) : 생성 규칙 LHS. 확장 가능
  단말 기호   (Terminal)    : 생성 규칙 RHS만. 더 이상 확장 불가
  시작 기호   (Start symbol): 비단말 중 하나. LHS에 한 번만 등장
  생성 규칙   (Production)  : A → B 형태. | 는 or

특수 단말 기호
  $ : 입력 스트림의 끝
  λ : 빈 문자열 (생략 가능)

유도 (Derivation)
  시작 기호에서 생성 규칙을 적용해 입력 프로그램으로 변환
  한 번에 비단말 기호 하나씩 적용

토큰 명세
  토큰 형(type)  : 단말 기호 이름
  토큰 값(value) : 정규 표현식으로 정의된 실제 입력 패턴
```
