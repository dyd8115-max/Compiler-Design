# Chapter 6. 하향식 구문분석 (Top-Down Parsing) — Part II

---

## 1. Top-Down Parsing 동작 원리

하향식 파싱은 **시작 기호(S)에서 출발해 아래로 파스 트리를 만들어 나가는** 방식이다.

```
입력: abac
문법:
  1. S → XY
  2. X → aX
  3. X → b
  4. Y → aY
  5. Y → c
```

### 파싱 과정

```
S ⇒ XY ⇒ aXY ⇒ abY ⇒ abaY ⇒ abac
    (2)    (3)    (4)    (5)
```

이것이 **좌측 유도(Leftmost Derivation)** — 가장 왼쪽 비단말을 먼저 전개.  
파싱 과정에서 생성 규칙 번호를 순서대로 나열한 것이 **좌 파스(Left Parse)**: `1 2 3 4 5`

> 트리 관점: 루트 S에서 시작해 lookahead 토큰을 보면서 어느 생성 규칙을 적용할지 결정하고 아래로 트리를 확장해 나간다.

---

## 2. 좋은 구문 분석이란?

### 문제: Backtracking

- 파서는 lookahead 하나만 보고 생성 규칙을 선택해야 함
- 잘못 선택하면 **지금까지의 유도 과정을 취소하고 되돌아가야 함 (backtracking)**
- Backtracking은 비효율적이고 구현이 복잡함

### 목표: 결정적(Deterministic) 구문 분석

> lookahead symbol과 현재 파싱 상태만을 갖고 **backtracking 없이** 한 번에 올바른 생성 규칙을 선택

### 예시

```
S → (S)S | a | ε

FIRST(S)  = { (, a }
FOLLOW(S) = { ), $ }
```

| lookahead | 선택 규칙 | 근거 |
|-----------|----------|------|
| `(` | `S → (S)S` | `(` ∈ FIRST((S)S) |
| `a` | `S → a` | `a` ∈ FIRST(a) |
| `)` | `S → ε` | `)` ∈ FOLLOW(S) |
| `$` | `S → ε` | `$` ∈ FOLLOW(S) |

---

## 3. Predict Set

### LL(k) Parser

- **LL(k)**: 한 번에 k개의 토큰을 lookahead해서 생성 규칙을 결정
- 실용적으로 **LL(1)** (lookahead 1개) 이 가장 많이 사용됨

### Predict Set 정의

생성 규칙 p에 대해:

```
Predict(p) = lookahead 기호 a가 왔을 때, 규칙 p를 선택해야 하는 terminal 집합
```

집합 P를 이용한 결정:

| 상황 | 의미 |
|------|------|
| `P = ∅` | Syntax Error (적용 가능한 규칙 없음) |
| `\|P\| > 1` | 비결정적 — LL(1) 불가 |
| `\|P\| = 1` | 결정적 파싱 가능 ✅ |

### Predict Set 계산

```
function Predict(p : A → X1...Xm) : Set
    ans ← First(X1...Xm)
    if RuleDerivesEmpty(p)          // RHS 전체가 ε를 유도 가능하면
    then
        ans ← ans ∪ Follow(A)       // A의 FOLLOW도 포함
    return (ans)
end
```

> `RuleDerivesEmpty(p)`: 규칙 p의 RHS가 ε를 유도할 수 있는지 여부

#### 예시: `S → ε`

```
ans = FIRST(ε) = { ε }
RuleDerivesEmpty(S → ε) = true
→ ans = ans ∪ FOLLOW(S) = { ), $ }
∴ Predict(S → ε) = { ), $ }
```

### Predict Set 예제

```
1  S → A C $
2  C → c
3     | λ
4  A → a B C d
5     | B Q
6  B → b B
7     | λ
8  Q → q
9     | λ
```

| Nonterminal | FIRST | FOLLOW |
|-------------|-------|--------|
| S | { a, b, c, q, $ } | { $ } |
| C | { c } | { d, $ } |
| A | { a, b, q } | { c, $ } |
| B | { b } | { c, d, q, $ } |
| Q | { q } | { c, $ } |

> S는 생성 규칙이 하나(single rule)이므로 Predict Set을 구할 필요 없음

**FOLLOW(B) 계산 근거:**
- `A → aBCd` 에서 B 뒤에 Cd → `FIRST(Cd) = { c, d }`
- `A → BQ` 에서 B 뒤에 Q → `FIRST(Q) = { q }`
- `A → BQ` 에서 Q가 nullable이면 → `FOLLOW(A) = { c, $ }`

---

## 4. LL(1) 조건

문법이 LL(1)이 되려면, 동일한 비단말 A의 두 대안 α, β에 대해 다음을 **모두** 만족해야 한다:

```
조건 1: FIRST(α) ∩ FIRST(β) = ∅
조건 2: ε ∈ FIRST(α) 이면, FOLLOW(A) ∩ FIRST(β) = ∅
```

즉, **어떤 lookahead 기호를 보더라도 선택할 규칙이 유일**해야 한다.

### 예시 1 — LL(1) O

```
S → aAb
A → aS | b

FIRST(aS) ∩ FIRST(b) = {a} ∩ {b} = ∅  ✅
```

```
S → ABc
A → bA | ε
B → c

FIRST(bA) ∩ FOLLOW(A) = {b} ∩ {c} = ∅  ✅
```

### 예시 2 — LL(1) O

```
A → iB → e
B → SB | ε
S → [eC] | .i
C → eC | ε

FIRST(SB) ∩ FOLLOW(B) = {[, .} ∩ {→} = ∅  ✅
FIRST([eC]) ∩ FIRST(.i) = {[} ∩ {.} = ∅   ✅
FIRST(eC) ∩ FOLLOW(C) = {e} ∩ {]} = ∅     ✅
```

### Quiz — LL(1) X

```
E → E + id | id
```

**좌재귀(Left Recursion)** 존재 → LL(1) 불가

- `FIRST(E + id)` = `FIRST(E)` = 같은 집합 → 조건 1 위반
- 좌재귀는 LL(1) 파싱에서 항상 문제 → **좌재귀 제거** 필요

---

## 5. Recursive Descent Parser (RDP)

LL(1) 문법에 대해 **각 비단말마다 하나의 함수**를 작성해 구현하는 파서.

### 구성 원칙

| 상황 | 처리 |
|------|------|
| `A → λ` | 즉시 return |
| RHS의 Xi가 단말 | `match(ts, Xi)` 호출 |
| RHS의 Xi가 비단말 | `Xi()` 함수 호출 |

### match 프로시저

```
procedure match(ts, token)
    if ts.PEEK() = token
    then call ts.ADVANCE()      // 다음 토큰으로 이동
    else call ERROR("Expected token")
end
```

- **PEEK**: 입력 포인터를 이동시키지 않고 다음 토큰만 확인
- **ADVANCE**: 입력 포인터를 한 칸 앞으로 이동

### RDP 예시 — procedure Q()

```
Q → q | λ
FIRST(Q) = {q},  FOLLOW(Q) = {c, $}
```

```
procedure Q()
    switch (...)
        case ts.peek() ∈ {q}:
            call match(q)
        case ts.peek() ∈ {c, $}:
            return ()       // Q → λ
end
```

### RDP 예시 — procedure B()

```
B → bB | λ
FIRST(B) = {b},  FOLLOW(B) = {c, d, q, $}
```

```
procedure B()
    switch (...)
        case ts.peek() ∈ {b}:
            call match(b)
            call B()            // 재귀 호출 (B → bB)
        case ts.peek() ∈ {q, c, d, $}:
            return ()           // B → λ
end
```

> **재귀 하강(Recursive Descent)** 이라는 이름의 유래: 각 비단말 함수가 서로를 재귀적으로 호출하며 파스 트리를 하강 방향으로 만들어 나가기 때문.

---

## 6. Table-driven LL(1) Parser

RDP와 달리 **파싱 테이블(LL Table)** 을 사용해 명시적 스택으로 구현하는 방식.  
Non-recursive parsing이라고도 한다.

### 구성 요소

| 요소 | 설명 |
|------|------|
| **Parsing Table M[A, a]** | 비단말 A와 lookahead a에 대해 적용할 생성 규칙 번호 저장 |
| **Stack** | 문법 기호 저장. 지금까지 파싱 과정에서 만들어진 sentential form |
| **Input Buffer** | 입력 문자열. 한 번에 하나의 lookahead를 읽어 옴 |

### LL 파싱 테이블 구성 — FILLTABLE

```
procedure FILLTABLE(LLtable)
    foreach A ∈ N do
        foreach a ∈ Σ do  LLtable[A][a] ← 0    // 초기화
    foreach A ∈ N do
        foreach p ∈ ProductionsFor(A) do
            foreach a ∈ Predict(p) do
                LLtable[A][a] ← p               // Predict Set으로 테이블 채움
end
```

### LLPARSER 알고리즘

```
procedure LLPARSER(ts)
    call PUSH(S)
    accepted ← false
    while not accepted do
        if TOS() ∈ Σ                         // TOS가 단말이면
        then
            call MATCH(ts, TOS())            // 입력과 대조
            if TOS() = $
            then accepted ← true
            call POP()
        else                                 // TOS가 비단말이면
            p ← LLtable[TOS(), ts.PEEK()]   // 테이블 참조
            if p = 0
            then call ERROR("Syntax Error – no production applicable")
            else call APPLY(p)
end
```

### APPLY 프로시저

```
procedure APPLY(p : A → X1...Xm)
    call POP()
    for i = m downto 1 do    // 역순으로 PUSH (왼쪽 기호가 top에 오도록)
        call PUSH(Xi)
end
```

> 역순으로 push하는 이유: X1이 스택의 top에 있어야 가장 먼저 처리되기 때문

---

## 7. 하향식 구문분석 전체 예제

### 예제 1 — `S → (S)S | ε`, 입력: `()`

**파싱 테이블:**

| Nonterminal | `(` | `)` | `$` |
|-------------|-----|-----|-----|
| S | `S → (S)S` | `S → ε` | `S → ε` |

**파싱 과정:**

| 단계 | Parse Stack | Input | Action |
|------|-------------|-------|--------|
| 1 | `$S` | `()$` | Apply: `S → (S)S` |
| 2 | `$S)S(` | `()$` | Match `(` |
| 3 | `$S)S` | `)$` | Apply: `S → ε` |
| 4 | `$S)` | `)$` | Match `)` |
| 5 | `$S` | `$` | Apply: `S → ε` |
| 6 | `$` | `$` | **Accept** |

좌측 유도: `S ⇒ (S)S ⇒ ()S ⇒ ()`

---

### 예제 2 — 전체 문법, 입력: `abbdc$`

```
1  S → A C
2  C → c
3     | λ
4  A → a B C d
5     | B Q
6  B → b B
7     | λ
8  Q → q
9     | λ
```

**파싱 테이블 M[비단말, lookahead]:**

| | a | b | c | d | q | $ |
|--|---|---|---|---|---|---|
| S | 1 | 1 | 1 | | 1 | 1 |
| C | | | 2 | 3 | | 3 |
| A | 4 | 5 | 5 | | 5 | 5 |
| B | | 6 | 7 | 7 | 7 | 7 |
| Q | | | 9 | | 8 | 9 |

**파싱 과정 (입력: abbdc$):**

| Parse Stack | Action | Remaining Input |
|-------------|--------|-----------------|
| `$S` | Apply 1: `S → AC` | `abbdc$` |
| `$CA` | Apply 4: `A → aBCd` | `abbdc$` |
| `$CdCBa` | Match `a` | `abbdc$` |
| `$CdCB` | Apply 6: `B → bB` | `bbdc$` |
| `$CdCBb` | Match `b` | `bbdc$` |
| `$CdCB` | Apply 6: `B → bB` | `bdc$` |
| `$CdCBb` | Match `b` | `bdc$` |
| `$CdCB` | Apply 7: `B → λ` | `dc$` |
| `$CdC` | Apply 3: `C → λ` | `dc$` |
| `$Cd` | Match `d` | `dc$` |
| `$C` | Apply 2: `C → c` | `c$` |
| `$c` | Match `c` | `c$` |
| `$` | **Accept** | `$` |

---

## 8. Practice #3

```
S → aS | bA
A → aAb | ε
```

**FIRST / FOLLOW 계산:**

```
FIRST(A) = { a, ε }
FIRST(S) = { a, b }

FOLLOW(S) = { $ }
FOLLOW(A):
  - S → bA 에서 A가 맨 끝 → FOLLOW(S) = { $ }
  - A → aAb 에서 A 뒤에 b → { b }
  ∴ FOLLOW(A) = { b, $ }
```

**Predict Sets:**

```
Predict(S → aS)  = { a }
Predict(S → bA)  = { b }
Predict(A → aAb) = { a }
Predict(A → ε)   = { b, $ }
```

**LL(1) 검증:**

```
S에 대해:
  FIRST(aS) ∩ FIRST(bA) = {a} ∩ {b} = ∅  ✅

A에 대해:
  FIRST(aAb) ∩ Predict(A → ε) = {a} ∩ {b, $} = ∅  ✅
```

∴ 위 문법은 **LL(1) grammar** 이다.

---

