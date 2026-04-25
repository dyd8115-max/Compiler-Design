# 컴파일러 — Chapter 1 컴파일러 구조

---

## 0. 기초 개념

**컴파일러 (Compiler)**
- 특정 고급 프로그래밍 언어로 작성된 프로그램을 특정 대상 컴퓨터의 실행 가능한 코드로 번역하는 컴퓨터 프로그램

**컴파일 (Compile)**
- 사람이 쓴 코드(소스)를 컴퓨터가 이해할 수 있는 언어(기계어)로 번역하는 과정
- 의미(semantics)는 보존하고 표현(syntax)만 바꿈

**프로그래밍 언어 (Programming Language)**
- 알고리즘과 계산을 형식적이고 모호하지 않게 표현하여 컴퓨터가 실행할 수 있도록 하는 인공 언어

**인간 언어 (Human Language)**
- 표기 방식과 구조가 언어마다 다양한 자연 발생 언어
- 모호성이 존재하기 때문에 컴퓨터용으로는 형식적인 프로그래밍 언어가 필요

**전반부 구성 요소 간략 소개** (각 섹션에서 상세히 다룸)

| 구성 요소 | 역할 |
|----------|------|
| 스캐너 (Scanner) | 소스 코드를 토큰 단위로 쪼갬 |
| 파서 (Parser) | 토큰을 구문 트리로 변환 |
| 의미 분석기 (Semantic Analyzer) | 구문 트리에 타입 정보를 추가 |

---

## 1. 컴파일러 구조 (Abstract View)

```
Source Programs
       ↓
  ┌─────────────┐
  │  Front-End  │  ← 언어 의존적 (language-dependent)
  └─────────────┘
       ↓
  중간 코드 (IC, Intermediate Code)
       ↓
  ┌─────────────┐
  │  Back-End   │  ← 기계 의존적 (machine-dependent)
  └─────────────┘
       ↓
Target Programs
```

- **전반부 (Front-end)** : 소스 언어에 종속. 분석 담당 → 이론적(Theoretic)
- **후반부 (Back-end)** : 목적 기계에 종속. 합성 담당 → 경험적 규칙(Rule-of-Thumb)

**언어 의존적 (language-dependent)**
- 소스 언어의 문법에 따라 동작이 달라짐
- C와 Java는 문법이 다르기 때문에 각각 다른 전반부(Front-end)가 필요함

**기계 의존적 (machine-dependent)**
- 목적 기계(CPU)에 따라 동작이 달라짐
- Intel과 ARM은 명령어(기계어) 체계가 다르기 때문에 각각 다른 후반부(Back-end)가 필요함

**중간 코드 (IC, Intermediate Code)**
- 전반부(Front-end)와 후반부(Back-end) 사이에서 주고받는 기계 독립적인 중간 표현
- 특정 언어나 기계에 종속되지 않음
- 전반부와 후반부를 독립적으로 개발하고 조합할 수 있게 해줌

**실제 IC 예시**
- **Java/Kotlin 바이트코드** (`.class`) : JVM이 읽고 실행. Intel이든 ARM이든 JVM만 있으면 동작. Kotlin도 동일한 바이트코드로 변환되어 Java와 호환
- **LLVM IR** : C/C++/Rust 등의 IC. LLVM 백엔드가 각 CPU 기계어로 변환
- **Python 바이트코드** (`.pyc`) : 파이썬 인터프리터가 실행 시점에 해석

> - C : 컴파일 시점에 기계어까지 완전 변환
> - Java/Python : IC에서 멈추고 인터프리터가 실행 시점에 해석

---

## 2. 컴파일러 구조 (Detailed View)

```
Source Program
       ↓
  ┌──────────────────────────────────────────┐
  │              Front-end (분석)            │
  │                                          │
  │  스캐너 ──→ 파서 ──→ 의미 분석기           │
  │  (Scanner)  (Parser)  (Semantic          │
  │                        Analyzer)         │
  └──────────────────────────────────────────┘
       ↓
  중간 코드 (IC)
       ↓
  ┌──────────────────────────────────────────┐
  │              Back-end (합성)             │
  │                                          │
  │  소스 코드     코드        목적 코드       │
  │  최적화기 ──→ 생성기 ──→  최적화기         │
  └──────────────────────────────────────────┘
       ↓
Target Program
```

**공유 자료 구조 (모든 단계가 접근)**

| 테이블 | 역할 | 상세 |
|--------|------|------|
| 심볼 테이블 (Symbol Table) | 변수, 함수 등 식별자 정보 저장 | 식별자 이름, 타입, 메모리 주소 등을 기록. 어휘 분석 단계부터 채워지고 모든 단계에서 참조 |
| 리터럴 테이블 (Literal Table) | 상수값 저장 | `"Park"`, `3.14` 같은 상수를 한 곳에 모아 관리. 중복 저장 방지 |
| 에러 처리기 (Error Handler) | 각 단계에서 발생하는 에러 처리 | 어휘/구문/의미 분석 중 발견된 오류를 수집하고 보고. 오류가 있어도 분석을 계속 진행할 수 있게 해줌 |

> **리터럴 (literal)** : 상수. 문자열 상수 `"Park"`, 숫자 상수 `3.14` 등

---

## 3. 컴파일러 단계 (Phases of a Compiler)

**단계 (Phase)** : 독립적인 처리가 가능한 모듈.

```
Source Code
     ↓
┌─────────────┐
│   스캐너     │  ← 어휘 분석 (Lexical Analysis)
│  (Scanner)  │
└─────────────┘
     ↓ 토큰 (Tokens)
┌─────────────┐
│    파서      │  ← 구문 분석 (Syntax Analysis)    ┐ 전반부
│   (Parser)  │                                   │ (Front-End)
└─────────────┘                                   │
     ↓ 구문 트리 (Syntax Tree)                     │
┌──────────────────┐                              │
│   의미 분석기     │  ← 의미 분석                 ┘
│(Semantic Analyzer│
└──────────────────┘
     ↓ 주석 트리 (Annotated Tree)
┌──────────────────────┐
│   소스 코드 최적화기  │  ← 코드 최적화              ┐
│(Source Code Optimizer│                            │
└──────────────────────┘                            │
     ↓ 중간 코드 (Intermediate Code)                 │ 후반부
┌────────────────┐                                  │ (Back-End)
│   코드 생성기  │  ← 코드 생성                      │
│(Code Generator)│                                  │
└────────────────┘                                  │
     ↓ 목적 코드 (Target Code)                       │
┌──────────────────────┐                            │
│  목적 코드 최적화기   │  ← 목적 코드 최적화          ┘
│(Target Code Optimizer│
└──────────────────────┘
     ↓
Target Code
```

> **주석 달기 (Annotation)** : 트리의 각 노드에 상세한 정보(타입 등)를 추가하는 것

---

## 4. 어휘 분석 (Scanner, Lexical Analyzer)

**토큰 (Token)** : 의미 있는 최소 단위의 문자열

예시: `a[index] = 4 + 2`

| 토큰 | 종류 |
|------|------|
| `a` | 식별자 (identifier, 변수) |
| `[` | 왼쪽 대괄호 (left bracket) |
| `index` | 식별자 (identifier) |
| `]` | 오른쪽 대괄호 (right bracket) |
| `=` | 대입 연산자 (assignment) |
| `4` | 숫자 상수 (number, literal) |
| `+` | 더하기 기호 (plus sign) |
| `2` | 숫자 상수 (number) |

> 영어 문장 `He eats an apple.` 에서 철자 검사(spell check) 하듯,
> 소스 코드를 읽어 의미 있는 단위(토큰)로 쪼개는 작업.

---

## 5. 구문 분석 (Parser, Syntax Analyzer)

소스 코드가 어떤 구조의 문장인지 분석. 구문 트리(Parse Tree)를 생성.

`a[index] = 4 + 2` 를 보고:
- 어떤 종류의 문장인가? → 대입문 형태 (좌변 = 우변, assignment statement)
- 좌변 (LHS, Left Hand Side): 배열의 원소
- 우변 (RHS, Right Hand Side): 덧셈 수식

### 문장(Statement) vs 수식(Expression)

- **수식 (Expression)**: 값을 만들어내는 식 → `4 + 2`, `a[index]`
- **문장 (Statement)**: 실행 단위 → `if (조건식) 문장1 else 문장2`
- 수식은 문장의 일부

### 구체 구문 트리 (Concrete Syntax Tree)

```
                    expression (수식)
                        |
               assign-expression (대입 수식)
              /         |          \
        expression      =        expression
        (수식)                    (수식)
            |                        |
    subscript-expression      additive-expression
    (배열 첨자 수식)            (덧셈 수식)
       /    |    |    \          /    |    \
  expr   [   expr   ]    expr    +    expr
    |           |           |            |
identifier  identifier   number       number
(식별자)    (식별자)     (숫자)       (숫자)
    a          index         4            2
```

### 추상 구문 트리 (AST, Abstract Syntax Tree)

구체 구문 트리에서 코드 생성에 **필요한 부분만 남기고** 나머지(괄호, 구두점 등)를 제거한 트리.

> **왜 추상(abstract)이라고 부를까?**
> 코드 생성에 필요한 부분만 남기고 나머지 문장 구성 요소를 없앴기 때문.

```
               assign-expression (대입 수식)
              /                   \
  subscript-expression      additive-expression
  (배열 첨자 수식)            (덧셈 수식)
      /          \               /           \
 identifier   identifier      number        number
 (식별자)     (식별자)        (숫자)        (숫자)
     a           index           4              2
```

제거된 것: `[`, `]`, `=`, `+`, `expression` 노드 등

---

## 6. 의미 분석 (Semantic Analyzer)

추상 구문 트리(AST)의 각 노드에 **데이터 타입** 정보를 추가 → **주석 트리 (Annotated Tree)** 생성

> **주석 달기 (Annotate)** : 트리 노드에 타입 등 상세 정보를 추가

```
               assign-expression (대입 수식)
              /                   \
  subscript-expression          additive-expression
  (배열 첨자 수식)                (덧셈 수식)
       integer                       integer
      /          \                 /           \
 identifier   identifier        number        number
 (식별자)     (식별자)          (숫자)        (숫자)
     a           index              4              2
array of integer  integer        integer        integer
(정수 배열)       (정수)          (정수)          (정수)
```

- `a` → 정수 배열 (array of integer)
- `index` → 정수 (integer)
- `4`, `2` → 정수 (integer)
- `subscript-expression` → 정수 (integer)
- `additive-expression` → 정수 (integer)

---

## 7. 전반부 용어 정리

| 용어 | 의미 | 담당 |
|------|------|------|
| **어휘적 (Lexical)** | 단어/철자 수준 분석 | 스캐너 (Scanner) |
| **구문적 (Syntactic)** | 문장 구조 분석 | 파서 (Parser) |
| **의미적 (Semantic)** | 의미/타입 분석 | 의미 분석기 (Semantic Analyzer) |

---

## 8. 중간 코드 (Intermediate Code)

구문 트리를 직접 목적 코드로 번역하는 대신 **기계 독립적인 중간 코드**로 번역.

흐름: 중간 코드 생성 → 코드 최적화 → 기계 코드 생성

### 3주소 코드 (3-address code)

IC의 대표적인 형태.

- 한 줄에 연산자 하나, 피연산자 최대 3개 (`결과 = 피연산자1 op 피연산자2`)
- 임시 변수(t)를 사용해 복잡한 수식을 단순한 연산으로 쪼갬
- 특정 CPU 명령어와 무관하게 표현됨

```
// a[index] = 4 + 2 의 3주소 코드
t = 4 + 2      // t : 임시 변수
a[index] = t
```

### 중간 코드가 필요한 이유

```
중간 코드 없이:
  C → Intel 기계어      (1개)
  C → ARM 기계어        (1개)
  Java → Intel 기계어   (1개)
  Java → ARM 기계어     (1개)
  Python → Intel 기계어 (1개)
  Python → ARM 기계어   (1개)
  → 총 6개 필요 (3언어 × 2기계)

중간 코드 있으면:
  C → IC                      (전반부 1개)
  Java → IC (Java 바이트코드) (전반부 1개)
  Python → IC                 (전반부 1개)
  IC → Intel                  (후반부 1개)
  IC → ARM                    (후반부 1개)
  → 총 5개로 해결 (3언어 + 2기계)
```

> - CPU마다 기계어 체계가 달라서 같은 소스 코드도 Intel용, ARM용 따로 변환 필요
> - 중간 코드를 두면 전반부(언어→IC)와 후반부(IC→CPU)를 독립적으로 개발·조합 가능
> - C/C++은 IC 없이 바로 기계어로 컴파일하는 방식도 존재 → IC는 필수가 아닌 선택 (속도 vs 이식성 트레이드오프)

---

## 9. 코드 최적화 (Source Code Optimizer)

주석 트리에서 최적화 수행. **전역 최적화 (global optimization)** 라고도 함.

**상수 접기**: 컴파일 시점에 이미 결과를 알 수 있는 상수 표현식을 미리 계산해서 결과값으로 대체

```
// 상수 접기 전 (3주소 코드)
t = 4 + 2
a[index] = t

// 상수 접기 후 (4+2를 미리 계산해서 6으로 대체)
a[index] = 6
```

트리 변화:
```
Before:
  additive-expression
  /    |    \
 4     +     2

After (상수 접기 적용):
  number
    6
```

---

## 10. 코드 생성 (Code Generator)

목적 기계의 속성에 따라 실제 기계 명령어로 변환.

고려 사항:
- 정수형/실수형 변수는 몇 바이트를 차지하는가?
- 배열 인덱싱을 위한 주소 지정 (addressing) 방식은?
- CPU 내부에 레지스터 (register)는 몇 개인가?

예: `a[index] = 4 + 2` → `a[index] = 6` (상수 접기 후)

```asm
MOV R0, index  ;; index 값 → R0
MOV R1, &a     ;; a의 주소 → R1  (R1 = a[0])
MUL R0, 2      ;; R0 = index × 2  (정수가 2bytes를 차지)
ADD R1, R0     ;; R1 = &a + R0  →  a[index]
MOV *R1, 6     ;; 상수 6 → R1이 가리키는 주소
```

---

## 11. 목적 코드 최적화 (Target Code Optimizer)

생성된 기계 코드를 더 효율적으로 변환.

최적화 방법:
- 성능 향상을 위한 주소 지정 (addressing) 모드 선택
- 실행 속도가 빠른 명령어로 대체
- 중복 또는 불필요한 연산 제거

예:
```asm
;; Before (MUL 사용)
MOV R0, index
MUL R0, 2      ;; 곱셈 명령어 (느림)
MOV &a[R0], 6

;; After (SHL로 대체)
MOV R0, index
SHL R0, 2      ;; 시프트 명령어 (빠름) — MUL 대신 SHL 사용
MOV &a[R0], 6  ;; 인덱스 주소 지정 모드 사용
```

> `MUL` (곱셈) → `SHL` (왼쪽 시프트, shift left): 2의 거듭제곱 곱셈은 시프트가 훨씬 빠름

---

## 12. 단계별 컴파일 과정 전체 예시

소스 코드: `position := initial + rate * 60`

> `:=` : 대입 연산자. C/Java의 `=` 와 동일한 의미. 이 예시에서 사용하는 표기 방식.

### 1단계 — 어휘 분석

```
position := initial + rate * 60
     ↓
심볼 테이블에 식별자 등록:
  1: position
  2: initial
  3: rate

토큰으로 변환:
  id1 := id2 + id3 * 60
```

### 2단계 — 구문 분석

```
구문 트리 생성:

        id1
         |
        :=
       /    \
     id2     *
    /    \
  id3    60
```

### 3단계 — 의미 분석

```
60은 정수인데 rate는 실수 → 타입 불일치
→ inttoreal(60) 삽입 (정수를 실수로 변환)

        id1
         |
        :=
       /    \
     id2     *
    /         \
  id3       inttoreal
                |
               60
```

### 4단계 — 중간 코드 생성

```
temp1 := inttoreal(60)
temp2 := id3 * temp1
temp3 := id2 + temp2
id1   := temp3
```

### 5단계 — 코드 최적화

```
temp1 := id3 * 60.0   // inttoreal 제거, 60.0으로 직접 처리
id1   := id2 + temp1
```

### 6단계 — 코드 생성

```asm
MOVF id3, R2    ;; id3 → R2
MULF #60.0, R2  ;; R2 = R2 * 60.0
MOVF id2, R1    ;; id2 → R1
ADDF R2, R1     ;; R1 = R1 + R2
MOVF R1, id1    ;; R1 → id1
```

---

## 핵심 요약

```
컴파일러 구조
  전반부 (언어 의존적) : 스캐너 → 파서 → 의미 분석기
  후반부 (기계 의존적) : 최적화기 → 코드 생성기 → 목적 코드 최적화기

공유 자료 구조
  심볼 테이블  : 식별자 정보
  리터럴 테이블 : 상수 정보
  에러 처리기  : 에러 처리

각 단계 출력물
  스캐너       → 토큰 (Tokens)
  파서         → 구문 트리 (Concrete / AST)
  의미 분석기  → 주석 트리 (Annotated Tree)
  소스 최적화기 → 중간 코드
  코드 생성기  → 목적 코드
  목적 최적화기 → 최적화된 목적 코드

주요 개념
  상수 접기 : 컴파일 시점에 상수 표현식 미리 계산 (4+2 → 6)
  3주소 코드 (3-address code)  : 중간 코드의 한 형태 (t = a op b)
  추상 구문 트리 (AST)         : 코드 생성에 필요한 부분만 남긴 트리
  inttoreal                    : 정수 → 실수 타입 변환 (의미 분석 단계)
```
