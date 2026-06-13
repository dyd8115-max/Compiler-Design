# Chapter 6. 하향식 구문분석 (Top-Down Parsing) — Part I

---

## 1. First / Follow Sets 개요

하향식 파서는 현재 nonterminal을 어느 생성 규칙으로 전개할지 **lookahead(미리 보기)** 토큰을 보고 결정한다.

```
S → (S)S | a | ε
```

| lookahead | 선택 규칙 |
|-----------|----------|
| `(` | `S → (S)S` |
| `a` | `S → a` |
| `)` 또는 `$` | `S → ε` |

### 왜 FIRST만으로는 부족한가?

- `FIRST(S) = { (, a, ε }` 에서 **ε는 아무 정보도 제공하지 않음**
- `S → ε` 를 언제 선택해야 하는지 알 수 없음
- → **FOLLOW(S)** 가 필요: S 바로 뒤에 나오는 기호를 찾아 결정

```
FOLLOW(S) = { ), $ }
$ : EOF (end of file)
```

> **핵심 원칙**: lookahead ∈ FOLLOW(A) 이면 `A → ε` 를 선택한다.

---

## 2. FIRST: 기본 정의

> **FIRST(α)**: 기호열 α 로부터 유도(derive)되어 만들어진 문자열의 **맨 앞에 나타나는 terminal 기호** 집합

```
FIRST(α) = { a ∈ VT | α →* aV* }
```

- `VT` : terminal 기호 집합
- `V*` : 임의의 기호열
- ε (빈 문자열) 도 포함될 수 있음

---

## 3. FIRST: 정의 (공식)

### 기본 규칙

```
FIRST(a) = { a }        (a ∈ VT, terminal은 자기 자신)
```

생성 규칙 `X → a ...` 에서 맨 앞에 terminal `a`가 나오면 → `FIRST(X) ∋ a`

### 복수 대안이 있을 때

```
X → α1 | α2 | ... | αn 이면
FIRST(X) = FIRST(α1) ∪ FIRST(α2) ∪ ... ∪ FIRST(αn)
```

### Nullable 기호가 포함된 연속 기호열 `A → X1 X2 ... Xk`

```
FIRST(A) ⊇ FIRST(X1)
X1이 nullable(→* ε) 하면: FIRST(A) ⊇ FIRST(X2)
X2도 nullable 하면:       FIRST(A) ⊇ FIRST(X3)
... 순차적으로 nullable이 아닐 때까지
```

> **nullable**: `X →* ε` 인 경우, X가 nullable하다고 한다.

---

## 4. Ring Sum (⊕)

연속 기호열의 FIRST를 구할 때 사용하는 연산.

```
A ⊕ B = A           (ε ∉ A 인 경우 — A 자체)
A ⊕ B = (A – {ε}) ∪ B   (ε ∈ A 인 경우 — ε 제거 후 B 추가)
```

### 적용: `FIRST(A1 A2 ... An)`

```
FIRST(A1 A2 ... An) = FIRST(A1) ⊕ FIRST(A2) ⊕ ... ⊕ FIRST(An)
```

- A1이 nullable하면 → A2의 FIRST도 포함
- 순차적으로 nullable이 아닐 때까지 이어서 구함

---

## 5. FIRST: 예제

### 예제 1 — type / simple 문법

```
type   → simple
       | ^ id
       | array [ simple ] of type

simple → integer
       | char
       | num dotdot num
```

| 기호 | 계산 | FIRST |
|------|------|-------|
| `simple` | 각 대안의 첫 terminal | `{ integer, char, num }` |
| `type` | `FIRST(simple) ∪ {^} ∪ {array}` | `{ integer, char, num, ^, array }` |

---

### 예제 2 — nullable 포함

```
A → aB | B
B → bC | C
C → c
```

| 기호 | 계산 | FIRST |
|------|------|-------|
| `C` | `c` | `{ c }` |
| `B` | `FIRST(bC) ∪ FIRST(C)` = `{b} ∪ {c}` | `{ b, c }` |
| `A` | `FIRST(aB) ∪ FIRST(B)` = `{a} ∪ {b,c}` | `{ a, b, c }` |

---

```
S → ABc
A → bA | ε
B → c
```

`A`가 nullable이므로 Ring Sum 적용:

```
FIRST(S) = FIRST(A) ⊕ FIRST(B) ⊕ FIRST(C)
         = { b, ε } ⊕ { c }
         = { b } ∪ { c } = { b, c }
```

> 예시 유도: `S ⇒ ABc ⇒ bABc` / `S ⇒ ABc ⇒ Bc ⇒ cc`

---

### 예제 3

```
1  E      → Prefix ( E )
2         | v Tail
3  Prefix → f
4         | λ
5  Tail   → + E
6         | λ
```

| 기호 | 계산 과정 | FIRST |
|------|----------|-------|
| `Tail` | terminal `+` | `{ + }` |
| `Prefix` | terminal `f` (+ ε) | `{ f, ε }` |
| `E` | `E ⇒ Prefix(E)` → `FIRST(Prefix)={f}`, Prefix nullable → `FIRST((E))={(` | |
| | `E ⇒ v Tail` → `FIRST(v Tail)={v}` | |
| | 합산 | `{ f, (, v }` |

---

## 6. function: First / InternalFirst

### First(α) — 래퍼 함수

```
function First(α) returns Set
    foreach A ∈ Nonterminals() do
        VisitedFirst(A) ← false   // 모든 Nonterminal 초기화
    ans ← InternalFirst(α)
    return (ans)
end
```

> **VisitedFirst**: 재귀 호출 시 무한 루프 방지용 방문 체크 플래그

---

### InternalFirst(Xβ) — 핵심 로직

```
function InternalFirst(Xβ) returns Set
    if Xβ = ⊥                          // (10) Xβ가 empty string이면
        then return (∅)                 //      공집합 반환

    if X ∈ Σ                            // (11) X가 terminal이면
        then return ({X})               //      자기 자신 반환

    /* X is a nonterminal. */           // (12)
    ans ← ∅
    if not VisitedFirst(X)              // X에 대해 First를 구한 적 없으면
    then
        VisitedFirst(X) ← true          // (13) 방문 표시
        foreach rhs ∈ ProductionsFor(X) do  // X의 모든 생성 규칙에 대해
            ans ← ans ∪ InternalFirst(rhs)  // (14) 각 rhs의 First를 합산

    if SymbolDerivesEmpty(X)            // (15) X가 nullable이면
    then ans ← ans ∪ InternalFirst(β)  //      나머지 β의 First도 포함

    return (ans)                        // (16)
end
```

#### 각 단계 요약

| 번호 | 조건 | 처리 |
|------|------|------|
| (10) | Xβ = ⊥ (empty) | `∅` 반환 |
| (11) | X ∈ Σ (terminal) | `{X}` 반환 |
| (13)~(14) | X가 nonterminal, 미방문 | X의 모든 생성 규칙에 재귀 적용 |
| (15)~(16) | X가 nullable | β의 InternalFirst도 추가 |

---

## 7. FOLLOW: 기본 정의

> **FOLLOW(A)**: nonterminal A 바로 다음에 올 수 있는 **terminal symbol들의 집합**

```
FOLLOW(A) = { a ∈ VT ∪ {$} | S →* αAaβ,  α, β ∈ V* }
```

- `VT` : terminal symbol 집합
- `α, β` : 임의의 symbol열
- `$` : 입력 스트림의 endmarker (EOF)

### 왜 필요한가?

- A가 nullable이면 FIRST만으로 생성 규칙 선택 불가
- A 뒤에 어떤 terminal이 나오는지 알면 → `A → ε` 선택 가능

---

## 8. FOLLOW: 정의 (공식)

### 규칙 1 — 시작 기호

```
FOLLOW(S) ⊇ { $ }
```

시작 기호 S의 FOLLOW는 반드시 `$`를 포함한다.

### 규칙 2 — `A → αBβ` 형태 (B 뒤에 β 존재)

```
FOLLOW(B) ⊇ FOLLOW(B) ∪ FIRST(β) – {ε}
```

β의 FIRST에서 ε을 제외한 것이 B의 FOLLOW에 포함된다.

### 규칙 3 — `A → αB` 또는 `A → αBβ`이고 β →* ε 인 경우

```
FOLLOW(B) ⊇ FOLLOW(B) ∪ FOLLOW(A)
```

B 뒤가 사라질 수 있으면, A의 FOLLOW가 그대로 B의 FOLLOW로 전파된다.

---

## 9. FOLLOW: 예제

### 예제 1

```
S → aAb
A → aS | b
```

- `FOLLOW(S) = { $, b }` — S는 시작 기호이므로 `$` 포함, `A → aS`에서 S 뒤에 `b`
- `FOLLOW(A) = { b }` — `S → aAb`에서 A 뒤에 `b`

---

### 예제 2

```
S → ABc
A → bA | ε
B → c
```

- `FOLLOW(S) = { $ }`
- `FOLLOW(A)`: `S → ABc`에서 A 뒤에 B, `FIRST(Bc) = {c}` → `{ c }`
- `FOLLOW(B)`: `S → ABc`에서 B 뒤에 `c` → `{ c }`

---

## 10. function: Follow / InternalFollow

### Follow(A) — 래퍼 함수

```
function Follow(A) returns Set
    foreach A ∈ Nonterminals() do
        VisitedFollow(A) ← false   // (17) 모든 Nonterminal 초기화
    ans ← InternalFollow(A)
    return (ans)
end
```

### AllDeriveEmpty(γ) — 보조 함수

```
function AllDeriveEmpty(γ) returns Boolean
    foreach X ∈ γ do
        if not SymbolDerivesEmpty(X) or X ∈ Σ
        then return (false)
    return (true)
end
```

> γ에 속한 모든 기호가 nullable(→* ε)이면 true. Nonterminal이면서 nullable하지 않거나, terminal이면 false.

---

### InternalFollow(A) — 핵심 로직

```
function InternalFollow(A) returns Set
    ans ← ∅
    if not VisitedFollow(A)          // (18) A에 대해 Follow를 구한 적 없으면
    then
        VisitedFollow(A) ← true      // (19)
        foreach a ∈ Occurrences(A) do   // (20) A가 RHS에 포함된 모든 규칙
            ans ← ans ∪ First(TAIL(a))  // (21) TAIL의 FIRST를 추가

            if AllDeriveEmpty(TAIL(a))  // (22) TAIL이 전부 ε를 유도 가능하면
            then
                targ ← LHS(Production(a))   // (23) 해당 규칙의 LHS nonterminal
                ans ← ans ∪ InternalFollow(targ)  // (24) LHS의 Follow를 전파
    return (ans)
end
```

#### 핵심 개념 정리

| 개념 | 설명 |
|------|------|
| `TAIL(y)` | 기호 y 뒤에 놓인 기호 리스트. `A → αyβ` 일 때 `TAIL(y) = β` |
| `LHS(p)` | 생성 규칙 p의 LHS nonterminal. `p: A → α` 일 때 `LHS(p) = A` |
| `Occurrences(A)` | A가 RHS에 등장하는 모든 규칙의 occurrence |

#### 규칙 대응 요약

| 상황 | 처리 |
|------|------|
| `A → αBβ` | `FOLLOW(B) ∪= FIRST(β)` (규칙 2) |
| `A → αBβ`, β →* ε | `FOLLOW(B) ∪= FOLLOW(A)` (규칙 3) |
| `A → αB` (TAIL = ∅) | `FOLLOW(B) ∪= FOLLOW(A)` (규칙 3) |

---

## 11. Quiz & Practice 풀이

### Quiz #1

```
1  S → A B c
2  A → a
3     | λ
4  B → b
5     | λ
```

**풀이:**

```
FIRST(A) = { a, ε }     (A → a | ε)
FIRST(B) = { b, ε }     (B → b | ε)

FIRST(S) = FIRST(A) ⊕ FIRST(B) ⊕ FIRST(c)
         = { a, ε } ⊕ { b, ε } ⊕ { c }
         = { a } ∪ { b, ε } ⊕ { c }      (ε ∈ A → 제거 후 B 추가)
         = { a, b } ∪ { c }              (ε ∈ B → 제거 후 c 추가)
         = { a, b, c }
```

---

### Quiz #2 (동일 문법, Follow 추가)

```
FIRST(S) = { a, b, c }
FIRST(A) = { a, ε }
FIRST(B) = { b, ε }

FOLLOW(S) = { $ }
FOLLOW(A) = FIRST(Bc) – {ε} = { b, c }     (S → ABc 에서 A 뒤에 Bc)
            A가 nullable이더라도 Bc가 뒤에 있으므로
            → FIRST(B) ∪ FIRST(c) = { b } ∪ { c } = { b, c }
FOLLOW(B) = { c }                            (S → ABc 에서 B 뒤에 c)
```

---

### Quiz #3

```
1  E      → Prefix ( E )
2         | v Tail
3  Prefix → f
4         | λ
5  Tail   → + E
6         | λ
```

**FIRST:**

```
FIRST(Prefix) = { f, ε }
FIRST(Tail)   = { +, ε }
FIRST(E)      = { f, (, v }
  (E → Prefix(E): Prefix nullable → ( 도 포함 → { f, ( })
  (E → v Tail  :                              → { v })
  합산 → { f, (, v }
```

**FOLLOW:**

```
FOLLOW(E) = { $, ), + }
  - 시작 기호이므로 $
  - E → Prefix(E) 에서 E 뒤에 ) → )
  - Tail → +E 에서 E 뒤가 없으므로 FOLLOW(Tail) 전파
    → FOLLOW(Tail) = { ), $ }  (아래 참고)

FOLLOW(Prefix) = { ( }
  (E → Prefix(E) 에서 Prefix 뒤에 ()

FOLLOW(Tail) = FOLLOW(E) 일부
  (E → vTail 에서 Tail이 맨 끝 → FOLLOW(E) 전파)
  = { ), $ }
```

---

### Practice #1

```
S → a S A | ε
A → b
```

```
FIRST(A) = { b }
FIRST(S) = { a, ε }

FOLLOW(S) = { $ } ∪ FOLLOW(A 의 맥락)
  - S → aSA 에서 S 뒤에 A, FIRST(A) = {b} → FOLLOW(S) ⊇ { b }
  - S가 맨 끝에 올 수도 있으므로 FOLLOW(S) ⊇ FOLLOW(S) (자기 참조, 이미 포함)
  ∴ FOLLOW(S) = { a, b, $ }

FOLLOW(A) = FOLLOW(S) = { a, b, $ }
  (S → aSA 에서 A가 맨 끝 → FOLLOW(S) 전파)
```

> 보충: `S → aSA` 에서 S 뒤에 A가 있으므로 `FOLLOW(S) ⊇ FIRST(A) = {b}`. 또한 A 뒤에 아무것도 없으므로 `FOLLOW(A) = FOLLOW(S)` 가 된다. S가 nullable이어서 `S →* ε` 이므로 주의.

---

### Practice #2

```
S → a R T b | b R R
R → c R d | ε
T → R S | T a T
```

**좌재귀 제거 (T → TaT 포함):**

```
T → T a T | R S
      ↓ 좌재귀 제거
T  → R S T'
T' → a T T' | ε
```

**FIRST:**

```
FIRST(S) = { a, b }
FIRST(R) = { c, ε }
FIRST(T) = FIRST(RS) = FIRST(R) ⊕ FIRST(S)
         = { c, ε } ⊕ { a, b }
         = { c } ∪ { a, b } = { a, b, c }
```

**FOLLOW:**

```
FOLLOW(T):
  - S → aRTb 에서 T 뒤에 b → { b }
  - T → TaT  에서 앞 T 뒤에 a → { a }
  ∴ FOLLOW(T) = { a, b }

FOLLOW(S):
  - 시작 기호 → $
  - T → RS 에서 S가 맨 끝 → FOLLOW(T) 전파
  ∴ FOLLOW(S) = { a, b, $ }

FOLLOW(R):
  - S → aRTb 에서 R 뒤에 T, FIRST(T) = {a,b,c} → { a, b, c }
  - R → cRd  에서 R 뒤에 d → { d }
  - S → bRR  에서 앞 R 뒤에 R, FIRST(R)={c,ε} → { c }
               + R nullable이면 FOLLOW(S) 전파 → { a, b, $ }
  ∴ FOLLOW(R) = { a, b, c, d, $ }
```

---

