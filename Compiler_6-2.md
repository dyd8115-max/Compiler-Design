# Chapter 5. 문맥 자유 문법(CFG)과 Pushdown Automata — Part II


---

## 1. Abstract Syntax Tree (AST)

### Parse Tree의 문제점
- **Parse Tree(= Concrete Syntax Tree)** 는 지나치게 많은 정보를 포함함
  - 트리 탐색에 시간이 많이 걸림
  - 동적 구조를 구현하기 위해 많은 메모리가 필요

### AST(추상 구문 트리)란?
- 번역(Translation)에 **필요한 정보만** 포함하는 트리
- Parse Tree보다 **더 효율적인 형태**로 표현
- 중간 코드 생성(Semantic Analysis 이후) 단계에서 사용

### Parse Tree vs. AST 비교

`3 + 4` 예시:

```
[Parse Tree (Concrete)]          [AST (Abstract)]

        exp                              +
       / | \                           /   \
     exp  op  exp          →          3     4
      |    |    |
   number  +  number
     (3)       (4)
```

- Parse Tree: `exp`, `op` 등 **nonterminal 노드 존재**
- AST: **연산자가 루트**, 피연산자가 자식. 불필요한 노드 제거

### `(34 - 3) * 42` AST 예시

```
[Parse Tree]                      [AST]

      exp (1)                          *
     / | \                           /   \
 exp(4) op(3) exp(2)               -      42
  /|\     |      |               /   \
 ( exp(5) ) *  number           34    3
   / | \
exp(8) op(7) exp(6)
  |      |      |
number   -   number
```

> **단점**: 원래 source code sequence로 되돌릴 수 없음 → **reverse compile 불가능**

---

## 2. Ambiguous Grammar (모호한 문법)

### 정의
> **하나의 입력 문장에 대해 두 개 이상의 Parse Tree가 생성**될 수 있으면, 그 문법 G는 **모호하다(ambiguous)**.

### 모호성의 문제
- 같은 코드인데도 **전혀 다른 machine code** 생성 가능
- 실행 결과가 달라질 수 있음 → 컴파일러에서는 치명적

### 모호한 문법 예시

```
E → E + E | E - E | E * E | E / E | ( E ) | 0 | 1 | ... | 9
```

입력 `3 + 4 * 5`에 대해 두 가지 유도 존재:

**유도 1** (E → E + E 먼저 적용):
```
E ⇒ E + E ⇒ 3 + E ⇒ 3 + E * E ⇒ 3 + 4 * E ⇒ 3 + 4 * 5
```
```
Parse Tree 1:
      E
    / | \
   E  +   E
   |     / | \
   3    E  *  E
        |     |
        4     5
결과: 3 + (4 * 5) = 23
```

**유도 2** (E → E * E 먼저 적용):
```
E ⇒ E * E ⇒ E + E * E ⇒ 3 + E * E ⇒ 3 + 4 * E ⇒ 3 + 4 * 5
```
```
Parse Tree 2:
      E
    / | \
   E  *  E
  /|\    |
 E + E   5
 |   |
 3   4
결과: (3 + 4) * 5 = 35
```

→ **같은 문장, 다른 Parse Tree** → 모호한 문법!

---

### 모호성 해결 방법

#### 방법 1: 문법 변환 — 우선순위와 결합 법칙을 규칙에 반영

```
E → E + T | E - T | T         ← +, - 는 낮은 우선순위 (root에 가까움)
T → T * F | T / F | F         ← *, / 는 높은 우선순위
F → ( E ) | 0 | 1 | ... | 9  ← 괄호 및 숫자
```

**설계 원칙:**
- 우선순위가 같은 연산자는 **같은 rule에** 묶기
- 우선순위가 **낮을수록 parse tree의 root에 가깝게** 배치
- **좌결합 법칙** → left-recursive rule (`E → E + T`)
- **우결합 법칙** → right-recursive rule (`F → P ^ F`)

**좌결합 법칙 예시** (`9 + 5 + 2` = `(9+5)+2`):
```
List → List + Digit | Digit
Digit → 0 | 1 | ... | 9

Parse Tree:
       List
      /    \
   List   Digit
   /   \    |
 List  Digit 2
  |      |
Digit    5
  |
  9
→ 왼쪽부터 묶임 = 좌결합
```

#### 방법 2: 모호하지 않은 문법으로 변환 불가능한 경우 — 규칙 적용

- 적용 가능한 생성 규칙 중 **일정한 기준에 따라 하나를 선택**
- 대표 예: **Dangling Else 문제**

---

### Dangling Else 문제

```
statement  → if-stmt | other
if-stmt    → if ( exp ) statement
           | if ( exp ) statement else statement
exp        → 0 | 1
```

입력: `if (0) if (1) other else other`

두 가지 해석이 가능:
```
해석 1: if (0) { if (1) other else other }   ← else가 안쪽 if와 짝
해석 2: if (0) { if (1) other } else other   ← else가 바깥쪽 if와 짝
```

**해결책: Most Closely Nested Rule**
> **else는 가장 가까이 있는 if와 짝**을 이룬다 → 해석 1 채택

```c
// else가 안쪽 if와 짝 → 해석 1 (대부분의 언어 표준)
if (x != 0)
    if (y == 1/x) ok = TRUE;
    else x = 1/x;

// 바깥쪽 if와 짝을 원하면 명시적 괄호 사용
if (x != 0) {
    if (y == 1/x) ok = TRUE;
} else x = 1/x;
```

**Ada/Pascal 스타일 해결책: `end if` 키워드 사용**
```ada
if x/=0 then
    if y=1/x then ok := true;
    end if;
else z := 1/x;
end if;
```

```
if-stmt → if ( exp ) statement endif
        | if ( exp ) statement else statement endif
```

---

### 예 12: `3+4*5`, `3-4-5` 구문 분석

```
E → E + T | E - T | T
T → T * F | T / F | F
F → ( E ) | 0 | 1 | ... | 9
```

**`3 + 4 * 5` Parse Tree:**
```
        E
       / \
      E   T        ← E + T (+ 는 낮은 우선순위, 루트 근처)
      |  / | \
      T  T  *  F   ← T * F (* 는 높은 우선순위, 아래에 위치)
      |  |     |
      F  F     5
      |  |
      3  4
→ 3 + (4 * 5) = 23  ✅ 우선순위 올바르게 반영
```

**`3 - 4 - 5` Parse Tree:**
```
           E
          / \
         E   T         ← E - T (좌결합: 왼쪽부터)
        / \  |
       E   T F
      / \  | |
     ... ...F 5
             |
             4
→ (3 - 4) - 5 = -6  ✅ 좌결합 올바르게 반영
```

---

### Quiz #4 분석

```
E → E + T | E - T | T
T → T * F | T / F | F
F → P ^ F | P              ← ^ : 거듭제곱
P → -P | H                 ← - : 단항 마이너스
H → ( E ) | 0 | 1 | ... | 9
```

**a. 연산자 우선순위 (낮음 → 높음):**

| 우선순위 | 연산자 | 위치 |
|---------|--------|------|
| 낮음 (1) | `+`, `-` (이항) | E 레벨 |
| (2) | `*`, `/` | T 레벨 |
| (3) | `^` (거듭제곱) | F 레벨 |
| 높음 (4) | `-` (단항) | P 레벨 |

**b. 결합 법칙:**
- `+`, `-` (이항): **좌결합** (E → E + T : left recursive)
- `*`, `/`: **좌결합** (T → T * F : left recursive)
- `^`: **우결합** (F → P ^ F : right recursive) → `2^3^2 = 2^(3^2) = 512`
- `-` (단항): **우결합** (P → -P : right recursive)

**c. `-2 + 3 * (4 ^ 5)` 유도 요약:**
```
E ⇒ E + T
  ⇒ T + T
  ⇒ F + T
  ⇒ P + T                    [F → P]
  ⇒ -P + T                   [P → -P]
  ⇒ -H + T                   [P → H]
  ⇒ -2 + T                   [H → 2]
  ⇒ -2 + T * F
  ⇒ -2 + F * F
  ⇒ -2 + P * F
  ⇒ -2 + H * F
  ⇒ -2 + 3 * F
  ⇒ -2 + 3 * P ^ F → ... → -2 + 3 * (4 ^ 5)
```

---

## 3. EBNF (Extended BNF)

### BNF의 한계와 EBNF
- BNF는 반복과 선택을 표현하기 위해 recursion을 사용해야 함
- EBNF는 `{ }`, `[ ]` 기호로 반복과 선택을 **간결하게** 표현

### EBNF 기호

| 기호 | 의미 | 대응 BNF 패턴 |
|------|------|---------------|
| `{ X }` | X를 0번 이상 반복 | `A → Aα \| ε` 또는 `A → αA \| ε` |
| `[ X ]` | X는 선택적 (0 또는 1번) | `A → α \| ε` |
| `( X \| Y )` | X 또는 Y 중 선택 | `A → X \| Y` |

---

### EBNF: `{ }` — 반복

**Left Recursive → `{ }` 변환:**
```
A → Aα | β          (left recursive)
A → β { α }         (EBNF)
```

**Right Recursive → `{ }` 변환:**
```
A → αA | β          (right recursive)
A → { α } β         (EBNF)
```

**예시:**
```
stmt-sequence → stmt-sequence ; stmt | stmt    (left recursive)
stmt-sequence → stmt { ; stmt }                (EBNF)

stmt-sequence → stmt ; stmt-sequence | stmt    (right recursive)
stmt-sequence → { stmt ; } stmt                (EBNF)

exp → exp addop term | term
exp → term { addop term }                      (EBNF)

exp → term addop exp | term
exp → { term addop } term                      (EBNF)
```

> **주의**: EBNF로 변환하면 **문장 구조의 특징(좌/우순환)이 사라짐**  
> → Parse Tree가 어떻게 만들어졌는지 알기 어려워짐

---

### EBNF: `[ ]` — 선택적

**기본 패턴:**
```
A → α | ε           (BNF)
A → [ α ]           (EBNF)
```

**if-else 예시:**
```
if-stmt → if ( exp ) statement
        | if ( exp ) statement else statement   (BNF)

if-stmt → if ( exp ) statement [ else statement ]  (EBNF)
```

**우순환 규칙을 `[ ]`로 표기:**
```
stmt-sequence → stmt ; stmt-sequence | stmt    (BNF, right recursive)
stmt-sequence → stmt [ ; stmt-sequence ]       (EBNF) ← 순환이 그대로 살아있음!

exp → term addop exp | term                    (BNF)
exp → term [ addop exp ]                       (EBNF)
```

> **`{ }` vs `[ ]` 비교**:  
> - `{ }` : 반복을 없애고 iterative하게 표현 (순환 제거)  
> - `[ ]` : 선택적 표현, **순환(recursion)은 그대로 유지됨**

---

### 예 13: 문법의 EBNF 변환

**원래 BNF:**
```
exp   → exp addop term | term
addop → + | -
term  → term mulop factor | factor
mulop → *
factor → ( exp ) | number
```

**EBNF 변환:**
```
exp    → term { addop term }
addop  → + | -
term   → factor { mulop factor }
mulop  → *
factor → ( exp ) | number
```

---

## 4. Syntax Diagram

### 개념
- 문법 규칙을 **그래픽(흐름도)** 형태로 표현하는 방법
- 파스칼(Pascal), Ada 언어 명세에서 많이 사용

### 기호 표현

| 도형 | 의미 |
|------|------|
| **원, 타원** | Terminal 기호 |
| **사각형** | Nonterminal 기호 |
| **화살표** | 흐름 방향 |
| **분기** | `\|` (선택) |
| **되돌아오는 화살표** | `{ }` (반복) |
| **위로 우회하는 경로** | `[ ]` (선택적) |

### `factor → ( exp ) | number` Syntax Diagram (텍스트 표현)

```
              ┌──(oval: "(")──[rect: exp]──(oval: ")")──┐
factor ───────┤                                          ├──→
              └──────────(oval: number)──────────────────┘
```

### `A → { B }` (반복) Syntax Diagram

```
         ┌────────────────────────────────┐
A ───────┤→──────────────────────────────→┤──→
         └──← [rect: B] ←──┘
```
> B를 거치는 루프 경로 존재 → 0번 이상 반복

### `A → [ B ]` (선택적) Syntax Diagram

```
         ┌──→──────────────────────────→──┐
A ───────┤                                ├──→
         └──→──[rect: B]──────────────→──┘
```
> 위 경로(B 건너뜀) 또는 아래 경로(B 통과) 선택 가능

### 예 13 전체 Syntax Diagram (텍스트 표현)

```
exp:
         ┌────────────────────────────────────────┐
exp ─→ [term] ──→                           ──────→
              └──← [term] ←── [addop] ←──┘

addop:
addop ──┬──( + )──┬──→
        └──( - )──┘

term:
          ┌──────────────────────────────────────────┐
term ─→ [factor] ──→                           ───────→
               └──← [factor] ←── [mulop] ←──┘

mulop:
mulop ──( * )──→

factor:
                ┌──( ( )──[exp]──( ) )──┐
factor ─────────┤                       ├──→
                └────── (number) ────────┘
```

### 예 13: if-stmt Syntax Diagram

```
statement:
            ┌──[if-stmt]──┐
statement ──┤             ├──→
            └──(other)────┘

if-stmt:
                                             ┌────────────────────────────┐
if-stmt ─( if )─( ( )─[exp]─( ) )─[statement]┤                            ├──→
                                             └─( else )─[statement]────────┘
```

---

## 5. Chomsky 문법 분류

### CFG의 형식적 정의
```
G = (T, N, P, S)
  T : terminals (T ∩ N = ∅)
  N : nonterminals
  P : productions  (형태: A → α, A ∈ N, α ∈ (T ∪ N)*)
  S : start symbol (S ∈ N)
```

### Chomsky 계층 구조 (Chomsky Hierarchy)

```
┌─────────────────────────────────┐
│  Type 0: Unrestricted Grammar   │
│  ┌───────────────────────────┐  │
│  │  Type 1: CSG              │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  Type 2: CFG        │  │  │
│  │  │  ┌───────────────┐  │  │  │
│  │  │  │  Type 3:      │  │  │  │
│  │  │  │  Regular      │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 4가지 문법 종류 비교

| Type | 이름 | 규칙 형태 | 제약 조건 | 예 |
|------|------|-----------|-----------|-----|
| **0** | Unrestricted | 제한 없음 | 없음 | `SaB → cS` |
| **1** | Context-Sensitive (CSG) | `αAγ → αβγ` | `\|LHS\| ≤ \|RHS\|` | `SaB → caB` |
| **2** | Context-Free (CFG) | `A → α` | LHS에 Nonterminal **1개만** | `A → aABb` |
| **3** | Regular Grammar | `A → aB` 또는 `A → a` | RHS에 NT 최대 1개, 위치 고정 | `A → aB`, `A → Ba` |

### Regular Grammar 세부 분류

```
Right Linear Regular Grammar:
  A → aB   (terminal 다음에 nonterminal)
  A → a    (terminal만)

Left Linear Regular Grammar:
  A → Ba   (nonterminal 다음에 terminal)
  A → a    (terminal만)
```

> **주의**: Right Linear과 Left Linear을 **혼용하면 안 됨**. 하나의 문법 내에서 통일해야 함.

### 정규 표현 ↔ 정규 문법 변환

```
[정규 표현]
digit  = 0 | 1 | 2 | ... | 9
number = digit digit*

[정규 문법 (Regular Grammar)]
digit  → 0 | 1 | 2 | ... | 9
number → number digit | digit    (A → Aa | a, Left Linear)
```

> 정규 표현(Regular Expression)은 항상 정규 문법(Regular Grammar)으로 변환 가능하고, 그 반대도 성립.

### 예 14: CSG이지만 CFG가 아닌 문법

```
L(G) = { aⁿbⁿcⁿ | n ≥ 1 }  ← CFG로 표현 불가능

생성 규칙:
1. S  → aSBC
2. S  → aBC
3. aB → ab
4. bB → bb
5. bC → bc  (교재: C → c)
6. CB → CX  ← 문맥 감지: C 오른쪽에 B가 있을 때만
7. CX → BX
8. BX → BC

유도 예: S ⇒ aSBC ⇒ aaSBCBC ⇒ aaBCBC ⇒ ... ⇒ aabbcc
```

> CFG는 LHS에 Nonterminal 1개만 가능하므로 `CB → CX` 같은 규칙은 CFG 불가 → CSG

### 예 15: 문법 종류 판별

| 규칙 | 판단 | 이유 |
|------|------|------|
| `aSb → aAcBb` | **Type 1 (CSG)** | LHS에 NT 포함된 문자열, `\|LHS\| ≤ \|RHS\|` |
| `B → aA` | **Type 3 (Right Linear)** | `A → aB` 형태 |
| `a → ABC` | **Type 0 (Unrestricted)** | LHS가 terminal — 제약 없음 |
| `S → aBc` | **Type 2 (CFG)** | LHS에 NT 1개, RHS 자유 |
| `Ab → b` | **Type 1 (CSG)** | LHS에 NT+T 조합, `\|LHS\| = \|RHS\|` |
| `AB → BA` | **Type 0 (Unrestricted)** | LHS에 NT 2개 — CFG 이상 |

---

## 핵심 요약

| 개념 | 내용 |
|------|------|
| **AST** | Parse Tree에서 불필요한 노드 제거. 번역에 필요한 정보만 유지. Reverse compile 불가 |
| **Ambiguous Grammar** | 하나의 입력 → 2개 이상 Parse Tree → 문법 변환으로 해결 |
| **우선순위 반영** | 낮은 우선순위 연산자 → root 가까이 (상위 규칙) |
| **결합 법칙 반영** | 좌결합 → left recursive, 우결합 → right recursive |
| **Dangling Else** | Most Closely Nested Rule: else는 가장 가까운 if와 짝 |
| **EBNF `{ }`** | 반복 (0번 이상). 순환 제거 |
| **EBNF `[ ]`** | 선택적 (0 또는 1번). 순환 유지 |
| **Syntax Diagram** | 원/타원 = terminal, 사각형 = nonterminal |
| **Chomsky Type 0** | 제한 없음 |
| **Chomsky Type 1** | CSG: `αAγ → αβγ`, `\|LHS\| ≤ \|RHS\|` |
| **Chomsky Type 2** | CFG: LHS = NT 1개 |
| **Chomsky Type 3** | Regular Grammar: RHS에 NT 최대 1개, 위치 고정 |
