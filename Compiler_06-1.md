# Chapter 5. 문맥 자유 문법(CFG)과 Pushdown Automata — Part I

---

## 1. Parsing Process

### Parsing이란?
- **Parsing (= Syntax Analysis)** : 입력 문장의 구성 요소를 낱낱이 분석하는 과정
- 문장 구조(구문, Syntax)는 **Grammar(문법)** 에 정의되어 있음
- 파싱 결과로 **Parse Tree(구문 트리)** 를 생성함

### 동작 방식
```
입력 문장 → 어떤 문장인가 판단(구문 선택) → 선택된 구문과 비교 → Parse Tree 생성
예) a = b + c;  →  token1 = token2 + token3;  →  LHS = RHS ;
```

### 예: `9 - 5 + 2` 파싱
- 토큰: `9`, `5`, `2` (숫자), `+`, `-` (연산자)
- 문법(Grammar):
```
list → list + digit
list → list – digit
list → digit
digit → 0 | 1 | 2 | … | 9
```
- `9-5+2`가 `list`인지 확인하는 과정:
  1. `9-5` → list 이고, `2` → digit 이면 `list – digit` 규칙 적용 가능 ✅
  2. `9` → list 이고, `5+2` → digit? → digit은 한 자리 수만 가능 ❌

> 하나의 production rule은 nonterminal의 가능한 구조 하나를 정의한다.  
> `list → list + digit` : list는 `list + digit` 형태로 대체될 수 있다는 의미.

---

## 2. Context Free Grammar (CFG)

### 자연 언어 vs. 형식 언어
| 구분 | 자연 언어 (Natural Language) | 형식 언어 (Formal Language) |
|------|----------------------------|-----------------------------|
| 특징 | 비형식적, 정보가 불명확함 | 5W1H 기반, 규칙의 집합 |
| 예 | "넘어져서 다리를 다쳤어" | 프로그래밍 언어 |
| 문제 | Ambiguous (이중 의미 가능) | Well-Formed Formula 요구 |

> 프로그래밍 언어 = 형식 언어 = 규칙(문법)의 집합

### CFG의 4가지 구성 요소
CFG는 `G = {T, N, S, P}` 로 정의된다.

| 기호 | 이름 | 설명 | 예 |
|------|------|------|----|
| **T** | Terminals (단말 기호) | 언어의 원자 기호, 더 이상 쪼개지지 않음 | `(`, `)`, `+`, `-`, `*`, `number` |
| **N** | Nonterminals (비단말 기호) | 언어 구조를 나타내는 변수 | `exp`, `op` |
| **S** | Start Symbol (시작 기호) | 언어의 최상위 구조를 나타내는 nonterminal | `exp` |
| **P** | Productions (생성 규칙) | nonterminal을 terminal/nonterminal 문자열로 대체하는 규칙 | `exp → exp op exp` |

### Nonterminal vs. Terminal
- **Nonterminal** : 가능한 내용을 대표하는 상징적 기호 (예: `사람`, `exp`)
- **Terminal** : 구체적 사실, 바뀔 수 없음 (예: `홍길동`, `number`)
- **Sentential form** : nonterminal을 포함한 불완전한 문장
- **Sentence** : terminal만으로 이루어진 완전한 문장

### CFG 예시
```
exp → exp op exp | ( exp ) | number
op  → + | - | *
```
- `G = {T, N, S, P}`
  - T = `{(, ), +, -, *, number}`
  - N = `{exp, op}`
  - S = `exp`

### BNF (Backus-Naur Form)
CFG를 표현하는 또 다른 표기법:
```
<exp> ::= <exp> <op> <exp> | (<exp>) | NUMBER
<op>  ::= + | - | *
```
> BNF와 CFG 표기는 동일한 내용을 다르게 쓴 것. 시험에서는 둘 다 나올 수 있음.

### CFG의 구조 정의 방식
1. **Hierarchical structure (계층 구조)**: 문장 안에 문장을 중첩
   ```
   Stmt → if ( Expr ) Stmt else Stmt
   ```
2. **Recursion (순환 정의)**: 자기 자신을 참조하는 규칙
   ```
   list → list + digit | digit
   ```

---

## 3. Derivation (유도)

### 개념
- **Derivation** : 시작 기호(Start Symbol)에서 시작하여 생성 규칙을 반복 적용해 문장(sentence)을 생성하는 과정
- 표기: `⇒` (한 단계 유도), `⇒*` (0단계 이상 유도)

```
exp ⇒* s     (exp에서 sentence s를 유도)
L(G) = { s | exp ⇒* s }   (문법 G가 생성하는 언어)
```

### 유도 예시: `(number - number) * number`
```
exp ⇒ exp op exp          [exp → exp op exp]
    ⇒ exp op number        [오른쪽 exp → number]
    ⇒ exp * number         [op → *]
    ⇒ ( exp ) * number     [exp → ( exp )]
    ⇒ ( exp op exp ) * number
    ⇒ ( exp op number ) * number
    ⇒ ( exp - number ) * number
    ⇒ ( number - number ) * number  ✅
```

### ε-production (엡실론 생성 규칙)
- 오른쪽이 비어있는 규칙: `A → ε`
- 예:
  ```
  A → A a | ε
  A ⇒ A a ⇒ A aa ⇒ ε aa ⇒ aa
  ```

### Left vs. Right Recursive Rules
| 종류 | 일반형 | 예 | 유도 방향 |
|------|--------|----|-----------|
| **Left Recursive** | `A → Aα \| β` | `A → A a \| b` | `b → ba → baa → ...` |
| **Right Recursive** | `A → αA \| β` | `A → a A \| b` | `b → ab → aab → ...` |

### 좌단 유도 vs. 우단 유도
| 종류 | 규칙 선택 기준 | 파싱 방향 | 파싱 방법 |
|------|----------------|-----------|-----------|
| **Leftmost derivation (좌단 유도)** | 가장 왼쪽 nonterminal 먼저 대체 | Top-Down | LL Parsing |
| **Rightmost derivation (우단 유도)** | 가장 오른쪽 nonterminal 먼저 대체 | Bottom-Up | LR Parsing |

### 좌단 유도 예: `(34-3)*42`
```
exp ⇒ exp op exp
    ⇒ ( exp ) op exp           [왼쪽 exp 먼저]
    ⇒ ( exp op exp ) op exp
    ⇒ ( number op exp ) op exp
    ⇒ ( number - exp ) op exp
    ⇒ ( number - number ) op exp
    ⇒ ( number - number ) * exp
    ⇒ ( number - number ) * number
```

### 우단 유도 예: `(34-3)*42`
```
exp ⇒ exp op exp
    ⇒ exp op number            [오른쪽 exp 먼저]
    ⇒ exp * number
    ⇒ ( exp ) * number
    ⇒ ( exp op exp ) * number
    ⇒ ( exp op number ) * number
    ⇒ ( exp - number ) * number
    ⇒ ( number - number ) * number
```

### 언어 정의 예제 모음

| 문법 | 생성되는 언어 L(G) | 비고 |
|------|-------------------|------|
| `E → (E) \| a` | `{ a, (a), ((a)), ... }` = `{(ⁿ a )ⁿ \| n ≥ 0}` | 균형 괄호 + a |
| `E → (E)` | `{ }` (공집합) | terminal 도달 불가 |
| `E → E + a \| a` | `{ a, a+a, a+a+a, ... }` | a를 +로 구분 |
| `A → (A)A \| ε` | 모든 균형 잡힌 괄호 문자열 | balanced parentheses |
| `stmt-seq → stmt ; stmt-seq \| stmt` | `{ s, s;s, s;s;s, ... }` | `;`는 구분자(separator) |
| `stmt-seq → stmt ; stmt-seq \| ε` | `{ ε, s;, s;s;, ... }` | `;`는 종결자(terminator) |

> **separator vs. terminator 구분 중요!**  
> - separator: `s;s;s` (마지막에 `;` 없음)  
> - terminator: `s;s;s;` (마지막에 `;` 있음)

---

## 4. Parse Tree (파스 트리)

### 개념
- **Parse Tree** : 유도 과정을 그래픽으로 표현한 트리
- 유도 순서(좌단/우단)는 달라도 **같은 문장이면 같은 parse tree** 를 가짐

### 트리 구조
```
        root (= start symbol)
       /     |     \
  내부 노드        내부 노드      ← Nonterminals
   (non-terminal)
      |
   leaf 노드  ← Terminals (token 또는 ε)
```

- **root** ↔ Start Symbol
- **leaf** ↔ Terminal 또는 ε
- **interior node** ↔ Nonterminal

### Parse Tree 예: `number + number`
```
문법: exp → exp op exp | ( exp ) | number
      op  → + | - | *

유도:
(1) exp ⇒ exp op exp
(2)     ⇒ number op exp   [좌단: 왼쪽 exp → number]
(3)     ⇒ number + exp    [op → +]
(4)     ⇒ number + number

트리 (텍스트 표현):
        exp
       / | \
     exp  op  exp
      |    |    |
   number  +  number
```
> Leftmost derivation → 트리 노드를 **preorder(전위순회)** 로 번호 매김  
> Rightmost derivation → 트리 노드를 **postorder(후위순회)** 로 번호 매김

### Parse Tree 예: `(34-3)*42`
```
우단 유도 결과:

              exp (1)
             / | \
         exp(4) op(3) exp(2)
          /|\     |      |
         ( exp(5) ) *  number
           / | \
        exp(8) op(7) exp(6)
          |      |      |
        number   -   number

잎 노드(왼→오): number - number * number
→ ( number - number ) * number  ✅
```

---

## Quiz 풀이 가이드

### Quiz #1
```
S → 0 A S | 0
A → S 1 A | S S | 1 0
입력: 0 0 1 1 0 0
```
- **좌단 유도**: 항상 가장 왼쪽 nonterminal을 먼저 치환
- **우단 유도**: 항상 가장 오른쪽 nonterminal을 먼저 치환

### Quiz #2
```
E → E + T | T | E - T
T → T * F | F | T / F
F → (E) | id | -E | num
```
- `num + num * num` 파스 트리: `*`가 `+`보다 먼저 묶임 (연산자 우선순위 반영됨)
  ```
        E
       /|\
      E  +  T
      |    /|\
      T   T  *  F
      |   |     |
      F   F    num
      |   |
     num num
  ```
- `- id + num` 파스 트리: `-E` 규칙 적용 (단항 마이너스)

### Quiz #3
```
S → S S + | S S * | a
입력: a a + a *
```
- 이 문법은 **후위 표기법(postfix notation)** 을 생성
- `a a + a *` = `(a + a) * a` in infix
- L(G) = postfix 표현식들의 집합

---

## 핵심 요약

| 개념 | 내용 |
|------|------|
| CFG | G = {T, N, S, P} |
| Terminal | 더 이상 치환 불가. 실제 토큰 |
| Nonterminal | 치환 가능한 변수. 언어 구조 표현 |
| Derivation | 생성 규칙을 반복 적용해 sentence 생성 |
| Sentential form | nonterminal 포함 (불완전) |
| Sentence | terminal만 포함 (완전한 문장) |
| Leftmost | 왼쪽 NT 먼저 → Top-Down → LL |
| Rightmost | 오른쪽 NT 먼저 → Bottom-Up → LR |
| Parse Tree | 유도 과정의 그래픽 표현 |
| ε-production | 오른쪽이 비어있는 규칙 |
