# Chapter 6 — 하향식 구문분석 (Top-down Parsing) Part I

---

## 1. First and Follow Sets 개요

하향식 파서는 현재 lookahead 토큰을 보고 **어떤 생성 규칙을 선택할지** 결정해야 한다.

### 동기 예시

```
S → (S)S | a | ε
```

| lookahead | 선택 규칙 |
|-----------|-----------|
| `(`       | S → (S)S  |
| `a`       | S → a     |
| `ε`       | 정보 없음 → FOLLOW 필요 |

- **`FIRST(S) = { (, a, ε }`**
  - `ε`는 lookahead 정보를 제공하지 않는다.
  - `S → ε` 규칙은 **언제** 선택해야 하는가? → **FOLLOW(S)** 를 이용한다.

- **`FOLLOW(S) = { ), $ }`**
  - `$` : EOF (End of File), 입력 스트림의 끝을 나타내는 endmarker

> **핵심 원리**  
> - lookahead가 `FIRST(α)`에 속하면 `A → α` 규칙을 선택한다.  
> - `A`가 nullable(ε 유도 가능)하고, lookahead가 `FOLLOW(A)`에 속하면 `A → ε` 규칙을 선택한다.

---

## 2. FIRST 집합

### 2.1 기본 정의

> **FIRST(α)** : 기호열 α로부터 파생(derive)되어 만들어진 문자열의 **맨 앞에 나타나는 terminal 기호들의 집합**

$$
\text{FIRST}(\alpha) = \{ a \in V_T \mid \alpha \Rightarrow^* a\beta \} \cup (\{\varepsilon\} \text{ if } \alpha \Rightarrow^* \varepsilon)
$$

- $V_T$ : terminal 기호 집합
- $V = V_T \cup V_N$ (terminal + nonterminal)

---

### 2.2 계산 규칙

#### 기본 케이스

| 경우 | FIRST 계산 |
|------|-----------|
| `α = ε` (empty string) | `FIRST(ε) = { ε }` |
| `X ∈ Σ` (terminal) | `FIRST(X) = { X }` |
| `X → a` (생성 규칙 RHS 맨 앞이 terminal `a`) | `FIRST(X)에 a 추가` |
| `X → α₁ \| α₂ \| … \| αₙ` | `FIRST(X) = FIRST(α₁) ∪ FIRST(α₂) ∪ … ∪ FIRST(αₙ)` |

#### Nullable이 있을 때: `A → X₁ X₂ … Xₖ`

```
FIRST(A) ⊇ FIRST(X₁)
X₁이 nullable → FIRST(A) ⊇ FIRST(X₂)
X₁, X₂가 모두 nullable → FIRST(A) ⊇ FIRST(X₃)
…
X₁ ~ Xₖ₋₁이 모두 nullable → FIRST(A) ⊇ FIRST(Xₖ)
X₁ ~ Xₖ가 모두 nullable → ε ∈ FIRST(A)
```

> **Nullable** : `X →* ε` 이 성립하는 기호 X

---

### 2.3 Ring Sum (⊕)

연속된 기호들의 FIRST를 순차적으로 합칠 때 사용하는 연산.

$$
A \oplus B = \begin{cases} A & \text{if } \varepsilon \notin A \\ (A - \{\varepsilon\}) \cup B & \text{if } \varepsilon \in A \end{cases}
$$

**FIRST(A₁A₂…Aₙ) 계산:**

```
FIRST(A₁A₂…Aₙ) = FIRST(A₁) ⊕ FIRST(A₂) ⊕ … ⊕ FIRST(Aₙ)
```

- 첫 번째 기호 A₁이 nullable하면 → A₂의 FIRST도 포함
- 순차적으로 nullable이 아닌 기호를 만날 때까지 진행
- 결과에 ε를 포함시킬지 여부는 모든 기호가 nullable할 때만 해당

---

### 2.4 예제 모음

#### 예제 1 — type/simple 문법

```
type  → simple
      | ^ id
      | array [ simple ] of type

simple → integer
       | char
       | num dotdot num
```

| 기호 | FIRST |
|------|-------|
| `simple` | `{ integer, char, num }` |
| `type` | `FIRST(simple) ∪ {^} ∪ {array}` = `{ integer, char, num, ^, array }` |

---

#### 예제 2 — nullable 포함

**문법 1:**
```
A → aB | B
B → bC | C
C → c
```

| 기호 | FIRST |
|------|-------|
| `C` | `{ c }` |
| `B` | `{ b, c }` (bC의 b, 그리고 C의 c) |
| `A` | `{ a, b, c }` (aB의 a, 그리고 B의 FIRST) |

---

**문법 2:**
```
S → ABc
A → bA | ε
B → c
```

```
FIRST(S) = FIRST(A) ⊕ FIRST(B) ⊕ FIRST(c)
         = { b, ε } ⊕ { c } ⊕ { c }
         = { b } ⊕ { c }       ← A가 nullable이므로 B의 FIRST 포함
         = { b, c }             ← B는 nullable이 아니므로 c는 ⊕되지 않음
```

> 파생 예시:  
> `S ⇒ ABc ⇒ bABc` (FIRST에 b 포함 확인)  
> `S ⇒ ABc ⇒ Bc ⇒ cc` (FIRST에 c 포함 확인)

| 기호 | FIRST |
|------|-------|
| `A` | `{ b, ε }` |
| `B` | `{ c }` |
| `S` | `{ b, c }` |

---

#### 예제 3 — 함수형 표현식 문법

```
1  E      → Prefix ( E )
2         | v Tail
3  Prefix → f
4         | λ (ε)
5  Tail   → + E
6         | λ (ε)
```

| 기호 | FIRST | 이유 |
|------|-------|------|
| `Tail` | `{ + }` | `+ E`의 첫 기호 |
| `Prefix` | `{ f, ε }` | f 또는 ε |
| `E` | `{ f, (, v }` | 아래 파생 참고 |

**FIRST(E) 계산 과정:**

| E의 파생 | 첫 기호 |
|---------|--------|
| `E ⇒ Prefix ( E )` | `FIRST(Prefix) = { f }` |
| `E ⇒ Prefix ( E ) ⇒ ( E )` | `FIRST(( E )) = { ( }` (Prefix가 nullable이므로) |
| `E ⇒ v Tail` | `FIRST(v Tail) = { v }` |

→ **`FIRST(E) = { f, (, v }`**

---

### 2.5 알고리즘: function First / InternalFirst

#### function First(α)

```
function First(α) returns Set
  foreach A ∈ Nonterminals() do
    VisitedFirst(A) ← false    // 모든 Nonterminal에 대해 초기화 (방문 여부 false)
  ans ← InternalFirst(α)
  return (ans)
end
```

> `VisitedFirst` 초기화: **재귀 호출 시 무한루프 방지**를 위해 First를 계산한 적 있는 Nonterminal을 추적한다.

---

#### function InternalFirst(Xβ)

```
function InternalFirst(Xβ) returns Set
  // Case 1: Xβ가 empty string (⊥)이면 공집합 반환
  if Xβ = ⊥
    then return (∅)                      // (10)

  // Case 2: X가 terminal 기호 (X ∈ Σ)이면 {X} 반환
  if X ∈ Σ
    then return ({X})                     // (11)

  /★ X is a nonterminal. ★/              // (12)

  ans ← ∅
  if not VisitedFirst(X)
  then
    VisitedFirst(X) ← true               // (13) 방문 표시
    foreach rhs ∈ ProductionsFor(X) do
      ans ← ans ∪ InternalFirst(rhs)     // (14) X의 모든 생성규칙에 대해 First 계산
  
  // Case 3: X가 nullable하면 β의 First도 포함
  if SymbolDerivesEmpty(X)
    then ans ← ans ∪ InternalFirst(β)   // (15)
  
  return (ans)                           // (16)
end
```

**각 단계 요약:**

| 단계 번호 | 조건 | 동작 |
|-----------|------|------|
| (10) | Xβ = ⊥ (empty string) | ∅ 반환 |
| (11) | X ∈ Σ (terminal) | {X} 반환 |
| (12) | X는 nonterminal | 이후 처리 |
| (13) | VisitedFirst(X) = false | true로 표시 (재방문 방지) |
| (14) | X의 모든 production | 각 RHS에 대해 재귀적으로 InternalFirst 호출 |
| (15) | X가 nullable | β의 InternalFirst도 합산 |
| (16) | — | ans 반환 |

> **VisitedFirst의 역할**: 예를 들어 `A → A b`처럼 좌재귀 규칙이 있을 때, 이미 A를 방문했다면 재귀 호출을 건너뛰어 **무한루프를 방지**한다.

---

### 2.6 Quiz #1 풀이

```
1  S → A B c
2  A → a
3     | λ (ε)
4  B → b
5     | λ (ε)
```

#### FIRST(A), FIRST(B) 계산

| 기호 | FIRST | 이유 |
|------|-------|------|
| `A` | `{ a, ε }` | `A → a` (terminal a), `A → ε` |
| `B` | `{ b, ε }` | `B → b` (terminal b), `B → ε` |

#### FIRST(S) 계산 — `S → A B c` 에서 InternalFirst 추적

```
InternalFirst(A B c) 호출
  X = A, β = B c
  A가 nonterminal → ProductionsFor(A) = { a, ε }
    InternalFirst(a) = { a }  → ans = { a }
    InternalFirst(ε) = ∅      (ε는 ⊥로 처리)
  ans = { a }
  SymbolDerivesEmpty(A) = true → InternalFirst(B c) 호출
    X = B, β = c
    ProductionsFor(B) = { b, ε }
      InternalFirst(b) = { b }  → ans = { b }
      InternalFirst(ε) = ∅
    ans = { b }
    SymbolDerivesEmpty(B) = true → InternalFirst(c) 호출
      X = c (terminal) → return { c }
    ans = { b } ∪ { c } = { b, c }
  ans = { a } ∪ { b, c } = { a, b, c }
```

| 기호 | FIRST |
|------|-------|
| `A` | `{ a, ε }` |
| `B` | `{ b, ε }` |
| `S` | `{ a, b, c }` |

---

## 3. FOLLOW 집합

### 3.1 기본 정의

> **nullable한 nonterminal A**가 있을 때, FIRST만으로는 생성 규칙을 선택할 수 없다.  
> `A → ε` 규칙을 선택해야 하는 시점 = lookahead가 **A 바로 뒤에 오는 terminal**일 때

$$
\text{FOLLOW}(A) = \{ a \in V_T \cup \{\$\} \mid S \Rightarrow^* \alpha A \, a \, \beta, \; \alpha, \beta \in V^* \}
$$

- $V_T$ : terminal 기호 집합
- `$` : 입력 스트림의 endmarker (EOF)
- FOLLOW는 **nonterminal에 대해서만** 정의된다.

---

### 3.2 계산 규칙

**규칙 1 — 시작 기호:**

$$
\text{FOLLOW}(S) \supseteq \{\$\}
$$

시작 기호 S의 FOLLOW에는 무조건 `$`가 포함된다.

---

**규칙 2 — A → α B β 형태의 생성 규칙:**

$$
\text{FOLLOW}(B) \supseteq \text{FIRST}(\beta) - \{\varepsilon\}
$$

B 뒤에 β가 오므로, β의 첫 기호들이 B의 FOLLOW가 된다.  
단, ε는 FOLLOW에 포함하지 않는다.

---

**규칙 3 — B가 규칙의 끝에 위치하거나, β가 nullable한 경우:**

$$
\text{FOLLOW}(B) \supseteq \text{FOLLOW}(A)
$$

- `A → α B` (B 뒤에 아무것도 없음)
- `A → α B β` 이고 `β →* ε` (β가 nullable)

두 경우 모두 A 다음에 오는 것이 B 다음에도 올 수 있으므로, FOLLOW(A)를 FOLLOW(B)에 포함시킨다.

---

**정리 (세 규칙 통합):**

| 상황 | FOLLOW에 추가할 것 |
|------|------------------|
| A가 시작 기호 | `$` |
| `X → α A β`, β ≠ ε, β nullable 아님 | `FIRST(β) - {ε}` |
| `X → α A β`, β →* ε (또는 β = ε) | `FIRST(β) - {ε}` **+** `FOLLOW(X)` |
| `X → α A` | `FOLLOW(X)` |

---

### 3.3 예제

#### 예제 1

```
S → aAb
A → aS | b
```

**FOLLOW(S) 계산:**
- S는 시작 기호 → `$` 포함
- `A → aS` : S 뒤에 아무것도 없음 → `FOLLOW(A)` 포함
- `FOLLOW(A)` = ? → `S → aAb` : A 뒤에 b → `{ b }`
- `FOLLOW(A) = { b }`
- `FOLLOW(S) = { $, b }`

| 기호 | FOLLOW |
|------|--------|
| `S` | `{ $, b }` |
| `A` | `{ b }` |

---

#### 예제 2

```
S → ABc
A → bA | ε
B → c
```

| 기호 | FOLLOW | 이유 |
|------|--------|------|
| `S` | `{ $ }` | 시작 기호 |
| `A` | `{ c }` | `S → ABc`: A 뒤에 B가 오고 FIRST(B)={c}, B는 nullable이 아님 → {c} |
| `B` | `{ c }` | `S → ABc`: B 뒤에 c → {c} |

---

### 3.4 알고리즘: function Follow / InternalFollow

#### function Follow(A)

```
function Follow(A) returns Set
  foreach A ∈ Nonterminals() do
    VisitedFollow(A) ← false     // 모든 Nonterminal에 대해 초기화 (17)
  ans ← InternalFollow(A)
  return (ans)
end
```

---

#### function AllDeriveEmpty(γ)

```
function AllDeriveEmpty(γ) returns Boolean
  foreach X ∈ γ do
    if not SymbolDerivesEmpty(X) or X ∈ Σ  // Nonterminal이면서 nullable하지 않거나, terminal이면
      then return (false)
  return (true)
end
```

> γ에 속한 **모든 기호가 nullable**이면 true, 하나라도 아니면 false

---

#### function InternalFollow(A)

```
function InternalFollow(A) returns Set
  ans ← ∅
  if not VisitedFollow(A)                        // (18) A에 대해 Follow를 구한 적 없음
  then
    VisitedFollow(A) ← true                      // (19)
    foreach a ∈ Occurrences(A) do               // (20) A가 포함된 모든 rule의 RHS
      ans ← ans ∪ First(TAIL(a))                // (21) A 뒤에 오는 기호들의 FIRST
      if AllDeriveEmpty(TAIL(a))                 // (22) A 뒤가 모두 nullable이면
      then
        targ ← LHS(Production(a))               // (23) 해당 rule의 LHS nonterminal
        ans ← ans ∪ InternalFollow(targ)        // (23) LHS의 Follow도 포함
  return (ans)                                    // (24)
end
```

**보조 정의:**

| 함수 | 정의 |
|------|------|
| `TAIL(y)` | 기호 y 뒤에 놓인 기호 리스트. `A → α y β`일 때 `TAIL(y) = β` |
| `LHS(p)` | 생성 규칙 p의 LHS nonterminal. `p : A → α`일 때 `LHS(p) = A` |
| `Occurrences(A)` | A가 RHS에 포함된 모든 생성 규칙의 목록 |

**단계별 의미:**

| 단계 | 조건 | 동작 |
|------|------|------|
| (20) | A가 등장하는 모든 규칙 탐색 | 각 등장 위치에 대해 처리 |
| (21) | `A → α A β` | `ans ∪ FIRST(β)` (β의 FIRST가 A의 FOLLOW) |
| (22) | `AllDeriveEmpty(TAIL(a)) = true` | β가 모두 nullable |
| (23) | β가 nullable이므로 | LHS의 FOLLOW도 A의 FOLLOW에 포함 |

---

## 4. Quiz 모음

### Quiz #2

```
1  S → A B c
2  A → a
3     | λ (ε)
4  B → b
5     | λ (ε)
```

#### FIRST 계산 (Quiz #1과 동일)

| 기호 | FIRST |
|------|-------|
| `A` | `{ a, ε }` |
| `B` | `{ b, ε }` |
| `S` | `{ a, b, c }` |

#### FOLLOW 계산

**FOLLOW(S):**
- S는 시작 기호 → `{ $ }`

**FOLLOW(A):**
- `S → A B c`: A 뒤에 B가 있음
- `FIRST(B) = { b, ε }` → `{ b }` 추가 (ε 제외)
- B가 nullable → B 뒤의 c 포함: `{ c }` 추가
- B, c 모두 nullable인가? c는 nullable 아님 → FOLLOW(S) 전파 없음
- `FOLLOW(A) = { b, c }`

**FOLLOW(B):**
- `S → A B c`: B 뒤에 c
- `FIRST(c) = { c }` → `{ c }` 추가
- c는 nullable 아님 → FOLLOW(S) 전파 없음
- `FOLLOW(B) = { c }`

| 기호 | FOLLOW |
|------|--------|
| `S` | `{ $ }` |
| `A` | `{ b, c }` |
| `B` | `{ c }` |

---

### Quiz #3

```
1  E      → Prefix ( E )
2         | v Tail
3  Prefix → f
4         | λ (ε)
5  Tail   → + E
6         | λ (ε)
```

#### FIRST 계산 (예제 3에서 이미 계산)

| 기호 | FIRST |
|------|-------|
| `Prefix` | `{ f, ε }` |
| `Tail` | `{ +, ε }` |
| `E` | `{ f, (, v }` |

#### FOLLOW 계산

**FOLLOW(E):**
- E는 시작 기호 → `$` 포함
- `E → Prefix ( E )`: E 뒤에 `)` → `{ ) }` 추가
- `Tail → + E`: E 뒤에 아무것도 없음 → `FOLLOW(Tail)` 포함 (나중에 계산)
- `FOLLOW(E) = { $, ) } ∪ FOLLOW(Tail)`

**FOLLOW(Prefix):**
- `E → Prefix ( E )`: Prefix 뒤에 `(`
- `FIRST(() = { ( }` → `{ ( }` 추가
- `(` 는 nullable 아님 → FOLLOW(E) 전파 없음
- `FOLLOW(Prefix) = { ( }`

**FOLLOW(Tail):**
- `E → v Tail`: Tail 뒤에 아무것도 없음 → `FOLLOW(E)` 포함
- `FOLLOW(Tail) = FOLLOW(E) = { $, ) }`

**FOLLOW(E) 완성:**
- `FOLLOW(E) = { $, ) } ∪ FOLLOW(Tail) = { $, ) }`

| 기호 | FOLLOW |
|------|--------|
| `E` | `{ $, ) }` |
| `Prefix` | `{ ( }` |
| `Tail` | `{ $, ) }` |

---

## 5. Practice 문제 및 풀이

### Practice #1

```
S → a S A | ε
A → b
```

#### FIRST 계산

| 기호 | FIRST | 이유 |
|------|-------|------|
| `A` | `{ b }` | `A → b` |
| `S` | `{ a, ε }` | `S → aSA` (a), `S → ε` |

#### FOLLOW 계산

**FOLLOW(S):**
- S는 시작 기호 → `$` 포함
- `S → a S A`: S 뒤에 A가 있음
  - `FIRST(A) = { b }` → `{ b }` 추가
  - A는 nullable이 아님 → FOLLOW(S) 전파 없음
- `FOLLOW(S) = { $, b }`

**FOLLOW(A):**
- `S → a S A`: A는 RHS의 끝 → `FOLLOW(S)` 포함
- `FOLLOW(A) = FOLLOW(S) = { $, b }`

| 기호 | FIRST | FOLLOW |
|------|-------|--------|
| `S` | `{ a, ε }` | `{ $, b }` |
| `A` | `{ b }` | `{ $, b }` |

---

### Practice #2 (풀이 포함)

```
S → a R T b | b R R
R → c R d | ε
T → R S | T a T
```

> **주의**: `T → T a T`는 **좌재귀(Left Recursion)**가 있다. 이를 제거하면:
> ```
> T  → R S T'
> T' → a T T' | ε
> ```

#### FIRST 계산

**FIRST(R):**
- `R → c R d`: 첫 기호 c (terminal)
- `R → ε`
- `FIRST(R) = { c, ε }`

**FIRST(S):**
- `S → a R T b`: 첫 기호 a → `{ a }`
- `S → b R R`: 첫 기호 b → `{ b }`
- `FIRST(S) = { a, b }`

**FIRST(T):** (좌재귀 제거 후 `T → R S T'` 기준)
- `FIRST(T) = FIRST(R) ⊕ FIRST(S)`
- R이 nullable → S의 FIRST도 포함
- `= { c, ε } ⊕ { a, b } = { c } ∪ { a, b } = { a, b, c }`

| 기호 | FIRST |
|------|-------|
| `S` | `{ a, b }` |
| `R` | `{ c, ε }` |
| `T` | `{ a, b, c }` |

#### FOLLOW 계산

**FOLLOW(S):**
- S는 시작 기호 → `$` 포함
- `T → R S` (원래 규칙): S 뒤에 아무것도 없음 → `FOLLOW(T)` 포함
- `FOLLOW(S) = { $, a, b }` (아래 FOLLOW(T) 참조)

**FOLLOW(T):**
- `S → a R T b`: T 뒤에 b → `{ b }` 추가
- `T → T a T` (원래 규칙): 두 번째 T 뒤에 아무것도 없음 → `FOLLOW(T)` 자기 자신 (이미 포함됨)
- 첫 번째 T 뒤에 a → `{ a }` 추가
- `FOLLOW(T) = { a, b }`

→ `FOLLOW(S) = { $, a, b }`

**FOLLOW(R):**
- `R → c R d`: R 뒤에 d → `{ d }` 추가
- `S → a R T b`: R 뒤에 T, `FIRST(T) = { a, b, c }` → `{ a, b, c }` 추가 (T가 nullable이 아님)
- `S → b R R`: 첫 번째 R 뒤에 R, `FIRST(R) = { c, ε }` → `{ c }` 추가
  - R이 nullable → 그 다음의 FOLLOW(S) 포함: `{ a, b, $ }`
- 마지막 R 뒤에 아무것도 없음 → `FOLLOW(S)` 포함: `{ a, b, $ }`
- `FOLLOW(R) = { d, a, b, c, $, a, b } = { a, b, c, d, $ }`

| 기호 | FOLLOW |
|------|--------|
| `S` | `{ $, a, b }` |
| `R` | `{ a, b, c, d, $ }` |
| `T` | `{ a, b }` |

---
