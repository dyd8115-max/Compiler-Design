# Chapter 2. 컴파일러 구조: Part II

---

## 1. Review: 문법(Grammar) 기초

### 1-1. 예제 문법 (생성 규칙, Production Rules)

```
1  | E → I
2  | E → E + E
3  | E → E * E
4  | E → ( E )
5  | I → a
6  | I → b
7  | I → I a
8  | I → I b
9  | I → I 0
10 | I → I 1
```

- **비단말기호(Nonterminal)**: `E`, `I` (대문자, 생성 규칙의 좌변에 등장)
- **단말기호(Terminal)**: `a`, `b`, `0`, `1`, `+`, `*`, `(`, `)` (실제 입력 기호)
- 위 규칙을 **BNF(Backus-Naur Form)** 방식으로 축약하면:

```
E → I | E + E | E * E | ( E )
I → a | b | Ia | Ib | I0 | I1
```

### 1-2. 올바른 문자열 판별

아래 문자열이 위 문법에서 유도(derivation)되는지 확인한다.

| 문자열 | 올바른가? | 이유 |
|--------|-----------|------|
| `a` | ✅ | `E → I → a` |
| `b` | ✅ | `E → I → b` |
| `b0` | ✅ | `E → I → I0 → b0` |
| `b00` | ✅ | `E → I → I0 → I00 → b00` |
| `a+b00` | ✅ | `E → E+E → I+I0... → a+b00` |
| `(a+b00)` | ✅ | `E → (E) → (E+E) → ...` |
| `a(a+b00)` | ❌ | 괄호 앞에 연산자 없음, 문법 위반 |

---

## 2. 컴파일러 단계 개요

이 챕터에서 다루는 간단한 컴파일러(ac 언어)의 단계:

```
소스코드
   ↓
[Scanner]           ← 어휘 분석(Lexical Analysis)
   ↓ 토큰 스트림(Token stream)
[Parser]            ← 구문 분석(Syntax Analysis) — 순환 하강(Recursive Descent) 방식
   ↓ 구문 트리(Parse Tree)
[Semantic Analysis] ← 의미 분석(Semantic Analysis)
   ↓ 주석 달린 AST(Annotated AST)
[Code Generation]
   ↓
목적코드
```

---

## 3. Scanner (어휘 분석기)

### 3-1. ac 언어 예제 입력

```
f b
i a
a = 5
b = a + 3.2
p b
```

- `f b` : b를 실수(float)로 선언
- `i a` : a를 정수(integer)로 선언
- `a = 5` : a에 5 할당
- `b = a + 3.2` : b에 a+3.2 할당
- `p b` : b 출력

### 3-2. 단말기호(Terminal)와 토큰(Token)

스캐너(Scanner)는 입력 문자를 읽어 **토큰(Token)** 을 만든다. 토큰 = 단말기호.

| 단말기호(Terminal) | 정규식(Regular Expression) | 의미 |
|--------------------|----------------------------|------|
| `floatdcl` | `"f"` | 실수 선언 |
| `intdcl` | `"i"` | 정수 선언 |
| `print` | `"p"` | 출력 |
| `id` | `[a-e] \| [g-h] \| [j-o] \| [q-z]` | 변수명 (i, f, p 제외) |
| `assign` | `"="` | 대입 연산자 |
| `plus` | `"+"` | 덧셈 |
| `minus` | `"-"` | 뺄셈 |
| `inum` | `[0-9]+` | 정수 상수 |
| `fnum` | `[0-9]+.[0-9]+` | 실수 상수 |
| `blank` | `(" ")+` | 공백 (무시) |

> **토큰 구조**: `{ type, value }`  
> 예: 입력 `f` → `{ type: floatdcl, value: 'f' }`

### 3-3. SCANNER() 의사코드(Pseudocode)

```
function SCANNER() returns Token
    while s.peek() = blank do
        call s.Advance()          -- 공백 건너뜀
    if s.EOF()
    then ans.type ← $            -- 입력 끝 (EOF)
    else
        if s.peek() ∈ {0, 1, ..., 9}
        then ans ← ScanDigits()  -- 숫자이면 정수/실수 처리
        else
            ch ← s.Advance()     -- 한 문자 읽어옴
            switch (ch)
                case {a, b, ..., z} − {i, f, p}
                    ans.type ← id
                    ans.val  ← ch
                case f
                    ans.type ← floatdcl
                case i
                    ans.type ← intdcl
                case p
                    ans.type ← print
                case =
                    ans.type ← assign
                case +
                    ans.type ← plus
                case -
                    ans.type ← minus
                case default
                    call LexicalError()
    return (ans)
end
```

> **핵심 개념**  
> - `s.peek()` : 현재 가리키고 있는 입력 문자를 읽음 (= **앞보기(lookahead)**, 1개)  
> - `s.Advance()` : 다음 문자로 이동하며 현재 문자를 반환  
> - `s.EOF()` : 입력 끝 여부 확인  
> - i, f, p는 `id`가 아니라 각각 `intdcl`, `floatdcl`, `print` 토큰으로 처리됨에 주의

### 3-4. ScanDigits() — 정수/실수 구분

```
function ScanDigits() returns token
    tok.val ← ""
    while s.peek() ∈ {0, 1, ..., 9} do
        tok.val ← tok.val + s.Advance()   -- 숫자 계속 이어붙임 (문자열 연결)
    if s.peek() ≠ "."
    then tok.type ← inum                  -- 소수점 없으면 정수(inum)
    else
        tok.type ← fnum
        tok.val  ← tok.val + s.Advance()  -- '.' 읽어옴
        while s.peek() ∈ {0, 1, ..., 9} do
            tok.val ← tok.val + s.Advance() -- 소수점 이하 숫자 읽음
    return (tok)
end
```

> **정수(inum)**: 한 개 이상의 숫자(0~9)로 이루어짐. 예: `1`, `123`, `3579`  
> **실수(fnum)**: 소수점(`.`)이 있는 숫자. 예: `0.0`, `12.0`, `123.456`  
> `+`는 덧셈 연산이 아니라 **문자열 연결(string concatenation)** 로 사용됨  
> 예: `"alpha" + "go"` → `"alphago"` (tok.val 문자열 이어붙이기)

#### 🤔 헷갈리는 케이스

| 입력 | 결과 | 이유 |
|------|------|------|
| `+2` | ❌ 정수로 인식 안 됨 | `+`는 `plus` 토큰, `2`는 별도 `inum` 토큰 |
| `-19` | ❌ 정수로 인식 안 됨 | `-`는 `minus` 토큰, `19`는 별도 `inum` 토큰 |
| `.7` | ❌ 실수로 인식 안 됨 | 먼저 숫자가 와야 ScanDigits 진입 가능 |
| `0000002.0000005` | ✅ 실수(fnum) | `[0-9]+.[0-9]+` 규칙에 부합 |

---

## 4. Parser (구문 분석기) — 순환 하강 파싱(Recursive-Descent Parsing)

### 4-1. 순환 하강 파싱 개념

- **가장 단순한 파싱 기법**
- 구문 트리(syntax tree)에서 **부모 노드 → 자식 노드** 방향(위→아래)으로 내려감(descent)
- 이 과정에서 **재귀(recursive) 호출**이 발생
- **각 비단말기호(Nonterminal)마다 자신의 이름을 가진 파싱 프로시저(parsing procedure)를 가짐**
- 각 프로시저는 해당 비단말기호의 오른쪽(RHS) 기호들을 순서대로 처리
- 토큰 스트림이 해당 비단말기호로부터 유도(derivable)되는 토큰 시퀀스인지 확인

### 4-2. ac 언어의 문맥 자유 문법(Context-Free Grammar)

```
1   Prog  → Dcls Stmts $
2   Dcls  → Dcl Dcls
3         | λ                   -- λ: 빈 문자열 (epsilon)
4   Dcl   → floatdcl id
5         | intdcl id
6   Stmts → Stmt Stmts
7         | λ
8   Stmt  → id assign Val Expr
9         | print id
10  Expr  → plus Val Expr
11        | minus Val Expr
12        | λ
13  Val   → id
14        | inum
15        | fnum
```

> `$` : 입력 끝을 나타내는 특수 기호  
> `λ` : 빈 생성(empty production) — 아무것도 없음을 의미

### 4-3. STMT() 프로시저

`Stmt → id assign Val Expr | print id` 에 대응

```
procedure STMT()
    if ts.peek() = id
    then
        call Match(ts, id)       -- ② 생성 규칙의 단말기호 id와 입력 토큰 비교
        call Match(ts, assign)   -- ③
        call Val()               -- ④ 비단말기호 → 해당 프로시저 호출
        call Expr()              -- ⑤
    else
        if ts.peek() = print     -- ⑥
        then
            call Match(ts, print)
            call Match(ts, id)
        else
            call Error()         -- ⑦
end
```

> **핵심 원리**  
> - **단말기호(Terminal)**: `Match(ts, 기호)` — 입력에서 읽어 온 토큰과 직접 비교  
> - **비단말기호(Nonterminal)**: 해당 이름의 프로시저를 호출 (`call Val()` 등)  
> - `ts` : 입력 토큰 스트림(token stream)을 저장하는 전역 변수  
> - `ts.peek()` : 다음 토큰을 미리 보는 앞보기(lookahead)

### 4-4. STMTS() 프로시저

`Stmts → Stmt Stmts | λ` 에 대응

```
procedure STMTS()
    if ts.peek() = id or ts.peek() = print  -- ⑧ Stmt가 시작될 수 있는 토큰
    then
        call STMT()    -- ⑨
        call STMTS()   -- ⑩ 재귀 호출
    else
        if ts.peek() = $    -- ⑪ 입력 끝
        then
            /* do nothing for λ-production */  -- ⑫
        else
            call Error()
end
```

> **λ-생성 처리**: `$`가 왔다는 것은 더 이상 Stmt가 없다는 뜻 → 아무것도 하지 않음

### 4-5. 나머지 비단말기호(Nonterminal) 프로시저

#### DCLS() — `Dcls → Dcl Dcls | λ`

```
procedure DCLS()
    if ts.peek() = floatdcl or ts.peek() = intdcl
    then
        call DCL()
        call DCLS()
    else
        if ts.peek() = id or ts.peek() = print   -- Hint: 3번 규칙(λ)는 id, print일 때
        then
            /* do nothing for λ-production */
        else
            call Error()
end
```

#### DCL() — `Dcl → floatdcl id | intdcl id`

```
procedure DCL()
    if ts.peek() = floatdcl
    then
        call Match(ts, floatdcl)
        call Match(ts, id)
    else
        if ts.peek() = intdcl
        then
            call Match(ts, intdcl)
            call Match(ts, id)
        else
            call Error()
end
```

#### EXPR() — `Expr → plus Val Expr | minus Val Expr | λ`

```
procedure EXPR()
    if ts.peek() = plus
    then
        call Match(ts, plus)
        call Val()
        call EXPR()
    else
        if ts.peek() = minus
        then
            call Match(ts, minus)
            call Val()
            call EXPR()
        else
            if ts.peek() = id or ts.peek() = print   -- Hint: 12번 규칙(λ)는 id, print일 때
            then
                /* do nothing for λ-production */
            else
                call Error()
end
```

#### VAL() — `Val → id | inum | fnum`

```
procedure VAL()
    if ts.peek() = id
    then
        call Match(ts, id)
    else
        if ts.peek() = inum
        then
            call Match(ts, inum)
        else
            if ts.peek() = fnum
            then
                call Match(ts, fnum)
            else
                call Error()
end
```

### 4-6. 구문 트리(Parse Tree) 예시

입력:
```
f b
i a
a = 5
b = a + 3.2
p b
```

구문 트리 (부분):

```
Prog
├── Dcls
│   ├── Dcl
│   │   ├── floatdcl (f)
│   │   └── id (b)
│   └── Dcls
│       ├── Dcl
│       │   ├── intdcl (i)
│       │   └── id (a)
│       └── Dcls → λ
├── Stmts
│   ├── Stmt
│   │   ├── id (a)
│   │   ├── assign (=)
│   │   ├── Val → inum (5)
│   │   └── Expr → λ
│   └── Stmts
│       ├── Stmt
│       │   ├── id (b)
│       │   ├── assign (=)
│       │   ├── Val → id (a)
│       │   └── Expr
│       │       ├── plus (+)
│       │       ├── Val → fnum (3.2)
│       │       └── Expr → λ
│       └── Stmts
│           ├── Stmt
│           │   ├── print (p)
│           │   └── id (b)
│           └── Stmts → λ
└── $
```

---

## 5. 추상 구문 트리 (AST, Abstract Syntax Tree)

**AST** : 구문 트리에서 의미 있는 정보만 추려낸 트리. 불필요한 비단말기호 노드를 제거해 구조를 간결하게 만든 것으로, 이후 의미 분석과 코드 생성 단계에서 사용하는 프로그램의 중간 표현(IR, Intermediate Representation)

### 5-1. 구문 트리(Parse Tree) vs AST

| | 구문 트리(Parse Tree) | AST |
|--|----------------------|-----|
| 특징 | 문법의 모든 기호 포함 | 의미 있는 정보만 포함 |
| 크기 | 크고 복잡함 | 간결함 |
| 비단말기호(Nonterminal) | 모두 포함 | 제거됨 |
| 역할 | 파싱 과정 표현 | 프로그램의 중간 표현(IR, Intermediate Representation) |

> AST는 프로그램의 **공통 중간 표현(common intermediate representation)** 으로 활용된다.

### 5-2. AST 구성 규칙 (5가지)

#### ① 실행 순서를 구체적으로 표현

- 프로그램의 실행 순서가 트리 구조에 명확히 드러나야 함
- `+`, `assign` 등 연산 노드가 실행 흐름을 결정

#### ② 데이터 형 선언은 하나의 노드(node)로 표시

```
구문 트리:            AST:
Dcls                  floatdcl    intdcl
├── Dcl               b           a
│   ├── floatdcl
│   └── id(b)
└── Dcls
    └── Dcl
        ├── intdcl
        └── id(a)
```

- `Dcls`, `Dcl` 같은 중간 비단말기호 노드는 제거
- `floatdcl b`, `intdcl a` 처럼 **단일 노드**로 압축

#### ③ 할당문 노드: 왼쪽 피연산자(LHS) 식별자가 왼쪽 자식(left child)

```
         assign
        /       \
    id(a)      inum(5)
```

- `a = 5` → assign 노드의 **왼쪽 자식(left child)** = `id a`, **오른쪽 자식(right child)** = `inum 5`

#### ④ 연산 노드: 연산 종류만 표시

```
       plus
      /     \
   id(a)   fnum(3.2)
```

- `a + 3.2` → plus 노드만 있으면 충분 (Val, Expr 같은 비단말기호 노드 불필요)

#### ⑤ print문: 출력할 변수만 표시

```
   print
     b
```

- `p b` → `print` 노드에 변수명 `b`만 저장

### 5-3. 전체 AST 결과

입력 `f b / i a / a = 5 / b = a + 3.2 / p b`에 대한 AST:

```
Program
├── floatdcl b
├── intdcl a
├── assign
│   ├── id a            (왼쪽 피연산자, LHS)
│   └── inum 5          (오른쪽 피연산자, RHS)
├── assign
│   ├── id b            (LHS)
│   └── plus
│       ├── id a
│       └── fnum 3.2
└── print b
```

---

## 6. 의미 분석 (Semantic Analysis)

### 6-1. 역할

구문 분석에서 처리하지 못한 언어 정의 상세 내용을 처리:

- **식별자(Identifier) 선언 및 범위(scope)** 가 적절한가?
- 사용자 정의 **형(type)** 이 알맞게 선언되었는가?
- 연산 및 메모리 참조 과정에서 **형(type)이 일치**하는가?
  - 예: `a + b` → 실수 + 정수 → 실수 덧셈 → 결과를 실수 형으로 저장

### 6-2. 형 검사(Type Checking) 개요

- 모든 변수는 사용 전에 **형(type)을 미리 선언**해야 함
- 의미 분석 단계에서 **심볼 테이블(Symbol Table)** 을 구성 → 변수 형 추적
- **형 계층(Type Hierarchy, 자동 형 변환 규칙)**:
  - `float`은 `integer`보다 **더 넓은(wider) 타입** (더 일반적)
  - 모든 정수는 실수로 표현 가능 → `integer → float` 변환 **O.K.**
  - `float → integer` 변환은 정밀도 손실 → **오류(Error)**
- 형 검사는 AST를 **아래에서 위로(bottom-up)** 순회하며 수행 (잎 → 루트)

### 6-3. 심볼 테이블(Symbol Table) 구성

`f b` / `i a` 선언 처리 후:

| 심볼(Symbol) | 형(Type) |
|--------------|----------|
| `a` | integer |
| `b` | float |

### 6-4. 방문자 메서드(Visitor methods) — 노드 종류별 형(type) 결정

```
/* Visitor methods */

procedure Visit(Computing n)         -- 연산 노드 (plus, minus 등)
    n.type ← Consistent(n.child1, n.child2)
end

procedure Visit(Assigning n)         -- 할당 노드 (assign)
    n.type ← Convert(n.child2, n.child1.type)
end

procedure Visit(SymReferencing n)    -- 변수 노드 (id)
    n.type ← LookupSymbol(n.id)     -- 심볼 테이블(Symbol Table) 참조
end

procedure Visit(IntConsting n)       -- 정수 상수 노드 (inum)
    n.type ← integer
end

procedure Visit(FloatConsting n)     -- 실수 상수 노드 (fnum)
    n.type ← float
end
```

### 6-5. 형 검사 유틸리티 함수

```
/* Type-checking utilities */

function Generalize(t1, t2) returns type
    if t1 = float or t2 = float
    then ans ← float      -- 하나라도 float이면 결과는 float
    else ans ← integer
    return (ans)
end

function Consistent(c1, c2) returns type
    m ← Generalize(c1.type, c2.type)
    call Convert(c1, m)   -- child1을 공통 형(type) m으로 변환
    call Convert(c2, m)   -- child2를 공통 형(type) m으로 변환
    return (m)
end

procedure Convert(n, t)
    if n.type = float and t = integer
    then call Error("Illegal type conversion")   -- float → integer: 오류!
    else
        if n.type = integer and t = float
        then
            /* replace node n by convert-to-float of node n */
            -- n을 int2float 변환 노드로 교체
        else
            /* nothing needed */  -- 형(type) 동일, 변환 불필요
end
```

### 6-6. 형 분석(Type Analysis) 단계별 추적

#### 입력: `b = a + 3.2`

**Step 1: 잎 노드(leaf node) 형 결정**

```
id(a)     → LookupSymbol("a") → integer
fnum(3.2) → float
```

**Step 2: plus 노드 (Computing)**

```
Visit(Computing: plus)
  → Consistent(id(a): integer, fnum(3.2): float)
    → Generalize(integer, float) = float
    → Convert(id(a), float)      → integer → float 변환: O.K.
                                 → int2float 노드 삽입
    → Convert(fnum(3.2), float)  → 이미 float, 변환 불필요
  → plus.type = float
```

**Step 3: assign 노드 (Assigning)**

```
Visit(Assigning: assign)
  → n.child1 = id(b), n.child1.type = LookupSymbol("b") = float
  → Convert(n.child2=plus, n.child1.type=float)
    → plus.type = float, t = float → 동일, 변환 불필요
  → assign.type = float
```

#### 입력: `a = 5`

```
id(a).type   = LookupSymbol("a") = integer
inum(5).type = integer
Visit(Assigning: assign)
  → Convert(inum(5), integer) → 동일, 변환 불필요
  → assign.type = integer
```

### 6-7. 의미 분석 후 AST

```
Program
├── floatdcl b
├── intdcl a
├── assign [integer]              ← a = 5
│   ├── id a
│   └── inum 5
├── assign [float]                ← b = a + 3.2
│   ├── id b
│   └── plus [float]
│       ├── int2float [float]     ← a (integer → float 자동 변환)
│       │   └── id a
│       └── fnum 3.2
└── print b
```

> **핵심 변화**: `plus` 노드의 자식인 `id(a)`가 `integer`이므로, `float`으로 변환하는 **`int2float` 노드**가 자동으로 삽입됨.

---

## 정리: 컴파일 단계 요약

| 단계 | 입력 | 출력 | 주요 작업 |
|------|------|------|-----------|
| **스캐너(Scanner)** | 문자 스트림 | 토큰 스트림 | 어휘 분석, 공백 제거, 토큰 분류 |
| **파서(Parser)** | 토큰 스트림 | 구문 트리(Parse Tree) | 문법 검사, 순환 하강(Recursive Descent) |
| **AST 생성** | 구문 트리 | AST | 불필요한 노드 제거, 구조 간결화 |
| **의미 분석(Semantic Analysis)** | AST | 주석 달린 AST | 심볼 테이블 구성, 형 검사, 자동 형 변환 노드 삽입 |

---

> 📁 참고: `ch2-2.pdf` — Compiler Structure Part II (홍윤식 교수)
