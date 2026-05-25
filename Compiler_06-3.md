# Chapter 5. 문맥 자유문법과 Pushdown Automata
운 오토마타

---

## 1. Parsers and Recognizers

### Recognizer
- 컴파일러는 입력 문자열 x가 문법 G에 속하는지 판단해야 한다: **x ∈ L(G)**

### Parser
- 문자열의 유효성뿐만 아니라 **구조(파스 트리)** 도 결정해야 한다

---

## 2. 두 가지 파싱 방법

사용 문법:
```
Program → begin Stmts end $
Stmts   → Stmt ; Stmts
        | λ
Stmt    → simplestmt
```
입력 문자열: `begin simplestmt ; simplestmt ; end $`

---

## 3. Top-Down Parsing (하향식 구문 분석)

### 개념
- **시작 기호(Start Symbol)** 에서 출발하여 입력 문자열을 유도
- **좌단 유도(Leftmost derivation)** 방식: 가장 왼쪽 Nonterminal을 먼저 전개
- **Lookahead** 기호를 보고 어느 생성 규칙을 선택할지 결정

### 유도 과정 (단계별)

```
Program
=> begin Stmts end $               [Stmts 전개 필요 → lookahead: simplestmt이므로 λ 아님]
=> begin Stmt ; Stmts end $        [Stmts → Stmt ; Stmts 선택]
=> begin simplestmt ; Stmts end $  [Stmt → simplestmt]
=> begin simplestmt ; Stmt ; Stmts end $   [Stmts → Stmt ; Stmts 선택]
=> begin simplestmt ; simplestmt ; Stmts end $  [Stmt → simplestmt]
=> begin simplestmt ; simplestmt ; end $   [lookahead: end → Stmts → λ 선택]
```

### 파스 트리 (Top-Down)

```
            Program
           /  |   |  \
        begin Stmts end $
              / | \
           Stmt ;  Stmts
            |       / | \
        simplestmt Stmt ;  Stmts
                    |          |
               simplestmt      λ
```

> Stmts가 λ(empty string)로 사라져야만 "end"가 나올 수 있음  
> → lookahead가 "end"이면 Stmts → λ 선택

### LL(k) 파싱
- **L**eftmost parse
- 첫 번째 L: 토큰 시퀀스를 **왼쪽에서 오른쪽**으로 처리
- k: 파싱 결정 시 참고하는 **lookahead 심볼의 수**
- LL(1): lookahead 1개로 파싱 결정

---

## 4. Bottom-Up Parsing (상향식 구문 분석)

### 개념
- **입력 문자열**에서 출발하여 시작 기호로 **환원(reduce)**
- 생성 규칙의 **오른쪽(RHS)** 기호를 왼쪽 Nonterminal로 교체
- 파스 트리의 **후위 순회(post-order traversal)** 에 해당

### 환원 과정 (단계별)

```
begin simplestmt ; simplestmt ; end $
=> begin Stmt ; simplestmt ; end $         [simplestmt → Stmt]
=> begin Stmt ; Stmt ; end $               [simplestmt → Stmt]
=> begin Stmt ; Stmt ; Stmts end $         [λ → Stmts]
=> begin Stmt ; Stmts end $                [Stmt ; Stmts → Stmts]
=> begin Stmts end $                       [Stmt ; Stmts → Stmts]
=> Program                                 [begin Stmts end $ → Program]
```

### 역방향으로 보면 (Top-Down 관점)

```
Program
→ begin Stmts end $
→ begin Stmt ; Stmts end $
→ begin Stmt ; Stmt ; Stmts end $
→ begin Stmt ; Stmt ; end $          [Stmts → λ]
→ begin Stmt ; simplestmt ; end $    [Stmt → simplestmt]
→ begin simplestmt ; simplestmt ; end $  [Stmt → simplestmt]
```

### 파스 트리 (Bottom-Up, 완성형)

```
            Program
           /  |   |  \
        begin Stmts end $
              /|\
           Stmt ; Stmts
            |     /  | \
        simplestmt Stmt ;  Stmts
                    |          |
               simplestmt      λ
```

### LR(k) 파싱
- **R**ightmost parse (역방향으로는 rightmost derivation)
- L: 토큰 시퀀스를 **왼쪽에서 오른쪽**으로 처리
- k: lookahead 심볼의 수
- 파스 트리의 post-order traversal

---

## 5. 문법 변환 (Grammar Transformation)

### 필요한 이유
- 구문 분석이 **효율적**으로 이루어지도록 문법을 변환
- Top-Down 파싱에서는 아래 변환이 **필수적**
- Bottom-Up 파싱에서는 해당 전처리가 필요하지 않음

### 변환 대상
1. ε(λ)을 유도하는 Nonterminal 찾기
2. 좌 인수분해 (Left Factoring)
3. 좌 순환(Left Recursion) → 우 순환(Right Recursion) 변환

---

## 6. ε을 유도하는 Nonterminal 찾기

### 개념
- 어떤 Nonterminal이 **여러 단계**를 거쳐 빈 문자열(λ)을 유도할 수 있는지 판별
- 파싱 도중 사라질 수 있으므로 **신중하게 처리**해야 함
- 예: B → λ, C → λ, D → λ 이면, A → BCD → BC → B → **λ**

---

### DerivesEmptyString 알고리즘

```
procedure DerivesEmptyString()
    // STEP 1: 모든 Nonterminal의 SymbolDerivesEmpty를 false로 초기화
    foreach A ∈ Nonterminals() do
        SymbolDerivesEmpty(A) ← false

    // STEP 2: 모든 Production을 순회하며 초기화
    foreach p ∈ Productions() do
        RuleDerivesEmpty(p) ← false
        Count(p) ← 0
        foreach X ∈ RHS(p) do Count(p) ← Count(p) + 1
        //  A → λ 형태이면 Count = 0
        call CheckForEmpty(p)

    // STEP 3: WorkList를 처리 (λ를 유도하는 Production의 LHS를 전파)
    foreach X ∈ WorkList do
        WorkList ← WorkList - {X}
        foreach x ∈ Occurrences(X) do
            p ← Productions(x)
            Count(p) ← Count(p) - 1
            call CheckForEmpty(p)
end
```

> **핵심 아이디어:** A → λ 형태의 production은 Count = 0이므로 즉시 처리됨  
> WorkList에 LHS가 추가되어 해당 기호가 등장하는 다른 rule의 Count를 감소시킴

---

### CheckForEmpty 프로시저

```
procedure CheckForEmpty(p)
    if Count(p) = 0
    then
        RuleDerivesEmpty(p) ← true      // Production p는 λ를 유도 가능
        A ← LHS(p)
        if not SymbolDerivesEmpty(A)
        then
            SymbolDerivesEmpty(A) ← true  // Nonterminal A는 λ를 유도 가능
            WorkList ← WorkList ∪ {A}
end
```

---

### DerivesEmptyString 실행 예시

#### 대상 문법
```
A → B C
B → λ
C → λ
```

#### STEP 1: 초기화

| Nonterminal | SymbolDerivesEmpty |
|:-----------:|:-----------------:|
| A | false |
| B | false |
| C | false |

#### STEP 2: Productions 순회 후 CheckForEmpty 호출

| p | Count(p) | RuleDerivesEmpty |
|:--:|:--------:|:----------------:|
| A → B C | 2 | false |
| B → λ | 0 | **true** |
| C → λ | 0 | **true** |

→ B, C는 Count = 0이므로 CheckForEmpty에 의해 처리됨  
→ WorkList = **{ B, C }**, SymbolDerivesEmpty(B) = true, SymbolDerivesEmpty(C) = true

#### STEP 3: WorkList 처리

- B, C가 WorkList에 있으므로, 각각을 RHS에서 사용하는 rule 탐색
- `Occurrences(B)` = { A → B C의 B }, `Occurrences(C)` = { A → B C의 C }
- A → B C의 Count: 2 → 1 → **0** (B와 C 모두 처리되므로)
- Count(A → B C) = 0 → CheckForEmpty 호출 → RuleDerivesEmpty(A → B C) = true

| Nonterminal | SymbolDerivesEmpty |
|:-----------:|:-----------------:|
| A | **true** |
| B | true |
| C | true |

> **Occurrences(X):** rule의 RHS에 문법 기호 X가 포함된 occurrence의 집합  
> 예: A → B C C D 에서 Occurrences(C) = { 2번째 C, 3번째 C }

---

### Practice #1

다음 문법에 대해 DerivesEmptyString 알고리즘의 각 단계별 실행 과정을 나타내시오.
```
A → B C D
B → C D
C → λ
D → λ
```

<details>
<summary>풀이 힌트</summary>

1. STEP 1: A, B, C, D 모두 SymbolDerivesEmpty = false
2. STEP 2: C → λ, D → λ는 Count = 0 → WorkList = {C, D}
3. STEP 3-1: C 처리 → B → C D의 Count: 2→1, A → B C D의 Count: 3→2
4. STEP 3-2: D 처리 → B → C D의 Count: 1→**0** → B도 WorkList 추가, A → B C D의 Count: 2→1
5. STEP 3-3: B 처리 → A → B C D의 Count: 1→**0** → A도 추가
6. 최종: A, B, C, D 모두 SymbolDerivesEmpty = **true**

</details>

---

## 7. 전처리 (Preprocessing for Top-Down Parsing)

하향식(Top-down) 구문 분석을 위해 아래 두 가지 전처리가 **필수적**:
- **좌 인수분해** (Left Factoring): 공통 prefix를 가진 생성 규칙 변환
- **좌 순환 제거** (Eliminate Left Recursion): 좌 순환 → 우 순환 변환

> 상향식(Bottom-up) 구문 분석에서는 이러한 전처리가 **필요하지 않음**

---

## 8. Left Factoring (좌 인수분해)

### 필요한 이유
- 두 생성 규칙이 **같은 prefix**로 시작하면 lookahead 1개만으로 어느 규칙인지 결정 불가
- 예:
  ```
  stmt → if expr then stmt else stmt
       | if expr then stmt
  ```
  → `if`를 보면 두 규칙 모두 해당, lookahead 1개로 선택 불가

### 변환 공식

```
일반형:
  A → α β₁ | α β₂

변환 후 (새로운 Nonterminal A' 추가):
  A  → α A'
  A' → β₁ | β₂
```

### 예시 1 (if-else)

```
원래:
  S → iEtS | iEtSeS | a
  E → b

변환 후:
  S  → iEtSS' | a
  S' → eS | λ
  E  → b
```
여기서 α = iEtS, β₁ = eS, β₂ = λ

### 예시 2 (stmt-sequence)

```
원래:
  stmt-sequence → stmt ; stmt-sequence | stmt

변환 후:
  stmt-sequence → stmt stmt-seq'
  stmt-seq'     → ; stmt-sequence | λ
```

### 예시 3 (exp)

```
원래:
  exp → term + exp | term

변환 후:
  exp  → term exp'
  exp' → + exp | λ
```

### 예시 4 (statement with identifier prefix)

```
원래:
  statement  → assign-stmt | call-stmt | other
  assign-stmt → identifier := exp
  call-stmt   → identifier ( exp-list )

합치면:
  statement → identifier := exp
            | identifier ( exp-list )
            | other

변환 후:
  statement  → identifier statement' | other
  statement' → := exp | ( exp-list )
```

---

## 9. Left Recursion 제거 (Eliminate Left Recursion)

### 좌 순환이 문제인 이유
- 좌 순환 규칙은 Nonterminal이 RHS의 **맨 앞**에 위치
- Top-Down 파싱 시 terminal과 matching 없이 **불필요한 유도가 무한 반복**됨
- 예: `E → E + T | T` 로 `id + id * id` 유도 시
  ```
  E ⇒ E + T ⇒ E + T + T ⇒ E + T + T + T ⇒ ...  (무한 반복)
  ```

### 변환 공식

```
좌 순환 (Left Recursion):
  A → A α | β

우 순환으로 변환 (새로운 Nonterminal A' 추가):
  A  → β A'
  A' → α A' | ε

생성 언어: L(A) = { b, bα, bαα, bααα, ... }
```

```
좌 순환 트리:              우 순환 트리:
      A                         A
     /|\                       / \
    A  α                      β   A'
   /|\                            / \
  A  α              →            α   A'
  |                                   / \
  β                                  α   A'
                                          |
                                          ε
```

### 예시 1 (단순 좌 순환)

```
원래:
  E → E + E | id

(A = E, α = +E, β = id)

변환 후:
  E  → id E'
  E' → + E E' | ε
```

### 예시 2 (immediate left recursion - addop)

```
원래:
  exp → exp addop term | term

변환 후:
  exp  → term exp'
  exp' → addop term exp' | λ
```

### 예시 3 (general immediate left recursion - 여러 α)

```
원래:
  exp → exp + term | exp - term | term

변환 후:
  exp  → term exp'
  exp' → + term exp' | - term exp' | λ
```

### 예시 4 (간접 좌 순환 - Indirect Left Recursion)

```
원래:
  S → Aa | b
  A → Ac | Sd | e
```

**주의:** A의 규칙에 S가 있으므로 단순 변환 불가

**풀이 방법:**
1. A의 생성 규칙에 S의 생성 규칙을 대입:
   ```
   A → Ac | Sd | e
   S → Aa | b 이므로 Sd → Aad | bd
   → A → Ac | Aad | bd | e
   ```
2. A의 좌 순환을 우 순환으로 변환 (α₁=c, α₂=ad, β₁=bd, β₂=e):
   ```
   S  → Aa | b
   A  → bdA' | eA'
   A' → cA' | adA' | ε
   ```

### 예시 5 (종합: exp/term/factor 문법)

```
원래:
  exp    → exp addop term | term
  addop  → + | -
  term   → term multop factor | factor
  multop → *
  factor → ( exp ) | number

변환 후:
  exp    → term exp'
  exp'   → addop term exp' | λ
  addop  → + | -
  term   → factor term'
  term'  → multop factor term' | λ
  multop → *
  factor → ( exp ) | number
```

---

## 10. Pushdown Automata (PDA, 푸시다운 오토마타)

### 정의

PDA는 다음 7-tuple로 정의:

```
M = (Q, Σ, T, δ, q₀, z₀, F)
```

| 구성요소 | 의미 |
|:--------:|:-----|
| Q | 상태 집합 |
| Σ | 입력 기호 집합 |
| **T** | **stack 기호 집합** |
| δ | 전이 함수 (mapping function) |
| q₀ ∈ Q | 시작 상태 |
| **z₀ ∈ T** | **stack의 시작 기호** |
| F ⊆ Q | 종결 상태 집합 |

### 구조 다이어그램

```
  ┌─────────────────────────┐
  │  a₁ a₂ ... aₙ  (입력 테이프) │
  └──────────┬──────────────┘
             ↓
  ┌──────────────────┐        ┌─────────┐
  │   유한 상태 제어  │ ←───→  │  Stack  │
  └──────────────────┘        │   z₁    │
                               │   z₂    │
                               │   ...   │
                               │   zₙ    │
                               └─────────┘
```

### 전이 함수 δ

```
δ : Q × (Σ ∪ {ε}) × T → Q × T*
```

**δ(q, a, z) = {(p, r)} 의 의미:**
- 현재 상태: q
- 현재 입력 기호: a
- 스택 top 기호: z

**→ 동작:**
1. 상태 **p**로 전이
2. 스택 top의 기호 z를 **pop**하고, 기호 r을 **push**
   - r = ε 이면 pop만 수행 (스택에서 제거만)
3. 입력 테이프 포인터를 **오른쪽으로 한 칸** 이동 (다음 입력 문자 읽기)

> ε-전이: 입력 기호 없이 스택 연산만 수행하는 전이  
> CFL(문맥 자유 언어)을 인식하는 오토마타
