# 컴파일러 — Chapter 1 컴파일러 구조

---

## 0. 기초 개념

**컴파일러 (Compiler)**
> 특정 고급 프로그래밍 언어로 작성된 프로그램을 특정 대상 컴퓨터의 실행 가능한 코드로 번역하는 컴퓨터 프로그램

**컴파일 (Compile)**
> 사람이 쓴 코드(소스)를 컴퓨터가 이해할 수 있는 언어(기계어)로 번역하는 과정
> 의미(semantics)는 보존하고 표현(syntax)만 바꿈

**프로그래밍 언어 (Programming Language)**
> 알고리즘과 계산을 형식적이고 모호하지 않게 표현하여 컴퓨터가 실행할 수 있도록 하는 인공 언어

**인간 언어 (Human Language)**
> 표기 방식과 구조가 언어마다 다양한 자연 발생 언어
> 모호성이 존재하기 때문에 컴퓨터용으로는 형식적인 프로그래밍 언어가 필요

---

## 1. 컴파일러 구조 (Abstract View)

```
Source Programs
       ↓
  ┌─────────────┐
  │  Front-End  │  ← language-dependent (언어에 종속)
  └─────────────┘
       ↓
  IC (Intermediate Code, 중간 코드)
       ↓
  ┌─────────────┐
  │  Back-End   │  ← machine-dependent (기계에 종속)
  └─────────────┘
       ↓
Target Programs
```

- **전반부 (Front-end)** : 소스 언어에 종속. 분석(Analysis) 담당 → 이론적(Theoretic)
- **후반부 (Back-end)** : 목적 기계에 종속. 합성(Synthesis) 담당 → 경험적 규칙(Rule-of-Thumb)

**언어 의존적 (language-dependent)**
- 소스 언어의 문법에 따라 동작이 달라짐
- C와 Java는 문법이 다르기 때문에 각각 다른 전반부(Front-end)가 필요함

**기계 의존적 (machine-dependent)**
- 목적 기계(CPU)에 따라 동작이 달라짐
- Intel과 ARM은 명령어 체계가 다르기 때문에 각각 다른 후반부(Back-end)가 필요함

**중간 코드 (IC, Intermediate Code)**
- 전반부(Front-end)와 후반부(Back-end) 사이에서 주고받는 기계 독립적인 중간 표현
- 특정 언어나 기계에 종속되지 않음
- 전반부와 후반부를 독립적으로 개발하고 조합할 수 있게 해줌

---

## 2. 컴파일러 구조 (Detailed View)

```
Source Program
       ↓
  ┌──────────────────────────────────────────┐
  │              Front-end (분석)            │
  │                                          │
  │  Scanner ──→ Parser ──→ Semantic         │
  │                         Analyzer         │
  └──────────────────────────────────────────┘
       ↓
  IC (Intermediate Code)
       ↓
  ┌──────────────────────────────────────────┐
  │              Back-end (합성)             │
  │                                          │
  │  Source Code    Code       Target Code   │
  │  Optimizer  ──→ Generator ──→ Optimizer  │
  └──────────────────────────────────────────┘
       ↓
Target Program
```

**공유 자료 구조 (모든 단계가 접근)**

| 테이블 | 역할 |
|--------|------|
| Symbol Table | 변수, 함수 등 식별자 정보 저장 |
| Literal Table | 상수값 저장 (`"Park"`, `3.14` 등) |
| Error Handler | 각 단계에서 발생하는 에러 처리 |

> **literal** : 상수. 문자열 상수 `"Park"`, 숫자 상수 `3.14` 등

---

## 3. The Phases of a Compiler

**Phase** : 독립적인 처리가 가능한 모듈.

```
Source Code
     ↓
┌─────────────┐
│   Scanner   │  ← 어휘 분석 (Lexical Analysis)
└─────────────┘
     ↓ Tokens
┌─────────────┐
│   Parser    │  ← 구문 분석 (Syntax Analysis)    ┐ Front-End
└─────────────┘                                   │
     ↓ Syntax Tree                                │
┌──────────────────┐                              │
│ Semantic Analyzer│  ← 의미 분석                 ┘
└──────────────────┘
     ↓ Annotated Tree
┌──────────────────────┐
│ Source Code Optimizer│  ← 코드 최적화            ┐
└──────────────────────┘                           │
     ↓ Intermediate Code                           │
┌────────────────┐                                 │ Back-End
│ Code Generator │  ← 코드 생성                    │
└────────────────┘                                 │
     ↓ Target Code                                 │
┌──────────────────────┐                           │
│ Target Code Optimizer│  ← 목적 코드 최적화        ┘
└──────────────────────┘
     ↓
Target Code
```

> **Annotation** : 주석 (상세한 정보를 추가했다는 의미)

---

## 4. 어휘 분석 (Scanner, Lexical Analyzer)

**Token** : a sequence of characters (문자들의 연속)

예시: `a[index] = 4 + 2`

| Token | 종류 |
|-------|------|
| `a` | identifier (변수) |
| `[` | left bracket |
| `index` | identifier |
| `]` | right bracket |
| `=` | assignment (연산 기호) |
| `4` | number (literal, 숫자 상수) |
| `+` | plus sign (연산 기호) |
| `2` | number |

> 영어 문장 `He eats an apple.` 에서 철자 검사(spell check) 하듯,
> 소스 코드를 읽어 의미 있는 단위(token)로 쪼개는 작업.

---

## 5. 구문 분석 (Parser, Syntax Analyzer)

소스 코드가 어떤 구조의 문장인지 분석. Parse Tree(구문 트리)를 생성.

> `a[index] = 4 + 2` 를 보고:
> - 어떤 종류의 문장인가? → `… = …` 형태 → **할당문 (assignment statement)**
> - LHS (Left Hand Side): 배열의 원소
> - RHS (Right Hand Side): 덧셈 수식

### Statement vs Expression

- **Expression(수식)**: 값을 만들어내는 식 → `4 + 2`, `a[index]`
- **Statement(문장)**: 실행 단위 → `if (expr) stmt else stmt`
- 수식은 문장의 일부

### Parse Tree (Concrete Syntax Tree)

```
                    expression
                        |
               assign-expression
              /         |          \
        expression      =        expression
            |                        |
    subscript-expression      additive-expression
       /    |    |    \          /    |    \
  expr   [   expr   ]    expr    +    expr
    |           |           |            |
identifier  identifier   number       number
    a          index         4            2
```

### Abstract Syntax Tree (AST, 추상 구문 트리)

Parse Tree에서 코드 생성에 **필요한 부분만 남기고** 나머지 문장 구성 요소(괄호, 구두점 등)를 제거한 트리.

> **왜 abstract라고 부를까?**
> 코드 생성에 필요한 부분만 남기고 나머지 문장 구성 요소를 없앴기 때문.

```
               assign-expression
              /                   \
  subscript-expression      additive-expression
      /          \               /           \
 identifier   identifier      number        number
     a           index           4              2
```

Concrete Syntax Tree 대비 제거된 것: `[`, `]`, `=`, `+`, `expression` 노드 등

---

## 6. 의미 분석 (Semantic Analyzer)

AST의 각 노드에 **데이터 타입(Data Type)** 정보를 추가 → **Annotated Tree** 생성

> **Annotate** : 주석을 달다 (상세한 정보를 추가)

```
               assign-expression
              /                   \
  subscript-expression          additive-expression
       integer                       integer
      /          \                 /           \
 identifier   identifier        number        number
     a           index              4              2
array of integer  integer        integer        integer
```

- `a` → array of integer
- `index` → integer
- `4`, `2` → integer
- `subscript-expression` → integer
- `additive-expression` → integer

---

## 7. Front-End 용어 정리

| 용어 | 의미 | 담당 |
|------|------|------|
| **Lexical** | spelling or word (어휘, vocabulary) | Scanner |
| **Syntactic** | structure of a sentence (문장 구조) | Parser |
| **Semantic** | what does it mean? (의미) | Semantic Analyzer |

- Lexical analysis → **Scanner** (a pet name)
- Syntactic analysis → **Parser** / Parse : 분석(analysis)
- Semantic : what does it mean?

---

## 8. 중간 코드 (Intermediate Code)

구문 트리를 직접 목적 코드로 번역하는 대신 **기계 독립적인 중간 코드**로 번역.

흐름: 중간 코드 생성 → 코드 최적화 → 기계 코드 생성

예: 3-address code (3주소 코드)

```
// a[index] = 4 + 2 의 3-address code
t = 4 + 2      // t : 임시 변수

// constant folding 적용 (4+2를 미리 계산)
t = 6
a[index] = t

// 임시 변수 t 대신 상수 6을 직접 사용
a[index] = 6
```

### 중간 코드가 필요한 이유

```
여러 언어 × 여러 기계 = 컴파일러 수가 폭발적으로 증가

중간 코드 없이:
  PASCAL/IBM, PASCAL/Mac
  FORTRAN/IBM, FORTRAN/Mac
  COBOL/IBM,   COBOL/Mac  → 6개의 컴파일러 필요

중간 코드 있으면:
  [PASCAL → IC], [FORTRAN → IC], [COBOL → IC]   (Front-End 3개)
  [IC → IBM],    [IC → Mac]                       (Back-End 2개)
  → 5개로 해결 (언어 + 기계 수만큼만 필요)
```

---

## 9. 코드 최적화 (Source Code Optimizer)

Annotated Tree에서 최적화 수행. **전역 최적화 (global optimization)** 라고도 함.

**Constant Folding**: 컴파일 시점에 미리 계산 가능한 상수 표현식을 계산

```
4 + 2  →  6  (실행 전에 컴파일러가 미리 계산)
```

트리 변화:
```
Before:
  additive-expression
  /    |    \
 4     +     2

After (Constant Folding):
  number
    6
```

---

## 10. 코드 생성 (Code Generator)

Target machine의 속성에 따라 실제 기계 명령어로 변환.

고려 사항:
- 정수형/실수형 변수는 몇 바이트를 차지하는가?
- 배열 인덱싱을 위한 addressing 방식은?
- CPU 내부에 register는 몇 개인가? (register allocation)

예: `a[index] = 4 + 2` → `a[index] = 6` (constant folding 후)

```asm
MOV R0, index  ;; value of index → R0
MOV R1, &a     ;; address of a → R1  (R1 = a[0])
MUL R0, 2      ;; R0 = index × 2  (정수가 2bytes를 차지)
ADD R1, R0     ;; R1 = &a + R0  →  a[index]
MOV *R1, 6     ;; constant 6 → address in R1
```

---

## 11. 목적 코드 최적화 (Target Code Optimizer)

생성된 기계 코드를 더 효율적으로 변환.

최적화 방법:
- 성능 향상을 위한 addressing 모드 선택
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
SHL R0, 2      ;; shift 명령어 (빠름) — MUL 대신 SHL 사용
MOV &a[R0], 6  ;; index addressing 모드 사용
```

> `MUL` (곱셈) → `SHL` (shift left): 2의 거듭제곱 곱셈은 shift가 훨씬 빠름

---

## 12. 단계별 컴파일 과정 전체 예시

소스 코드: `position := initial + rate * 60`

### 1단계 — 어휘 분석 (Lexical Analyzer)

```
position := initial + rate * 60
     ↓
Symbol Table에 식별자 등록:
  1: position
  2: initial
  3: rate

Token으로 변환:
  id1 := id2 + id3 * 60
```

### 2단계 — 구문 분석 (Syntax Analyzer)

```
Parse Tree 생성:

        id1
         |
        :=
       /    \
     id2     *
    /    \
  id3    60
```

### 3단계 — 의미 분석 (Semantic Analyzer)

```
60은 정수(int)인데 rate는 실수(real) → 타입 불일치
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

### 4단계 — 중간 코드 생성 (Intermediate Code Generator)

```
temp1 := inttoreal(60)
temp2 := id3 * temp1
temp3 := id2 + temp2
id1   := temp3
```

### 5단계 — 코드 최적화 (Code Optimizer)

```
temp1 := id3 * 60.0   // inttoreal 제거, 60.0으로 직접 처리
id1   := id2 + temp1
```

### 6단계 — 코드 생성 (Code Generator)

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
  Front-End (language-dependent) : Scanner → Parser → Semantic Analyzer
  Back-End  (machine-dependent)  : Optimizer → Code Generator → Target Optimizer

공유 자료 구조
  Symbol Table  : 식별자 정보
  Literal Table : 상수 정보
  Error Handler : 에러 처리

각 단계 출력물
  Scanner          → Tokens
  Parser           → Syntax Tree (Concrete / AST)
  Semantic Analyzer→ Annotated Tree
  Source Optimizer → Intermediate Code
  Code Generator   → Target Code
  Target Optimizer → 최적화된 Target Code

주요 개념
  Constant Folding : 컴파일 시점에 상수 표현식 미리 계산 (4+2 → 6)
  3-address code   : 중간 코드의 한 형태 (t = a op b)
  AST              : 코드 생성에 필요한 부분만 남긴 추상 구문 트리
  inttoreal        : 정수 → 실수 타입 변환 (의미 분석 단계)
```
