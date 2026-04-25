# Chapter 4. 어휘 분석 (Lexical Analysis)

---

## 01. 어휘분석 개요

### 어휘분석기(Scanner)란?

소스 프로그램에서 **토큰(Token)** 을 찾아내는 컴파일러의 첫 번째 단계.

> 핵심: Scanner는 토큰을 한꺼번에 다 찾는 게 아니라, **Parser가 요청할 때마다** 하나씩 찾아 전달한다.

```
소스 프로그램 ──→ [어휘 분석기] ──토큰──→ [구문 분석기] ──→ ...
                       ↑  ↕ 다음 토큰 요구
                      [기호표]
```

- Parser가 `getToken()` 호출 → Scanner가 토큰 1개 반환
- 반환값: `TokenType getToken(void);`
  - Token의 **type** (ex: `ID`, `NUM`, `+` ...)
  - Token의 **value** (ex: `"a"`, `42` ...)

(ex: `a[index] = 4 + 2` 파싱 중 첫 호출 → `return ID("a")`)

---

### 어휘분석기의 기능

| 기능 | 설명 |
|---|---|
| 토큰 전달 | Parser 요청 시 토큰을 찾아 반환 |
| 공백/주석 제거 | white space, comment 무시 |
| 기호표 구성 | symbol table에 식별자 등록 |

---

### 용어 정리

| 용어 | 영어 | 설명 | 예시 |
|---|---|---|---|
| **토큰** | Token | 문법적으로 의미있는 최소 단위 (terminal 기호로 구성) | `if`, `123`, `+`, `myVar` |
| **패턴** | Pattern | 토큰의 특징을 표현하는 규칙 → **정규 표현(RE)** 으로 표현 | `letter(letter\|digit)*` |
| **렉심** | Lexeme | 패턴과 매칭되는 실제 문자열 | `myVar`, `123` |

> **비유**: 패턴은 "식별자란 알파벳으로 시작하는 문자열"이라는 **규칙**이고, 렉심은 그 규칙을 만족하는 실제 소스코드 속 **문자열** (`count`, `total` 등)이다.

---

## 02. 토큰 인식

### Delimiter (구분기호)

**정의**: 현재 토큰 문자열이 어디까지인지 알려주는 경계 기호.

- Delimiter는 현재 토큰에 **포함되지 않음**
- (ex: `xtemp=ytemp` → `=`이 구분기호, `xtemp`가 토큰)
- (ex: `while x` → 공백(blank)이 구분기호)
- (ex: `do/* */if` → 주석(comment)이 구분기호)

```
whitespace = ( newline | blank | tab | comment )+
```

---

### Lookahead (미리 보기)

**문제 상황**: Delimiter가 다음 토큰의 일부일 수 있다.

| 예 | delimiter | 다음 토큰 |
|---|---|---|
| `xtemp=ytemp` | `=` | `=` 자체가 다음 토큰 |
| `x+=2;` | `+` | `+=`가 다음 토큰 |

**해결**: 읽은 문자를 버리지 말고 **먼저 살펴보고(lookahead)**:
- 현재 토큰의 일부 → 지금 처리
- 아니면 (delimiter) → 입력 버퍼에 돌려놓고, 다음 토큰 찾을 때 사용

> **비유**: 신호등에서 "노란불"을 보고 지금 건너야 하는지, 멈춰야 하는지 판단하는 것과 같다. 읽긴 했지만 아직 처리할지 결정 안 한 상태.

---

## 03. 형식 문법 및 어휘분석기 설계

### 정규 표현 = 유한 오토마타 = 정규 문법 (삼위일체)

```
Regular Expression  ←→  Finite Automata  ←→  Regular Grammar
```

이 세 가지는 **표현하는 언어가 동일**하다. 서로 변환 가능.

- **어휘 분석기 설계 흐름**:
  ```
  토큰 패턴 → 정규 표현(RE) → NFA → DFA → 최소화된 DFA
                    ↕
                 정규 문법
  ```

---

### Chomsky 형식 문법 분류

```
무제약 문법 (type 0)
  └─ 문맥인식 문법 (type 1) : CSG → CSL
       └─ 문맥자유 문법 (type 2) : CFG → CFL
            └─ 정규 문법 (type 3) : RG → RL
```

| type | 문법 | 언어 | 인식기 |
|---|---|---|---|
| 3 | 정규 문법 (RG) | 정규 언어 (RL) | 유한 오토마타 (FA) |
| 2 | 문맥자유 문법 (CFG) | 문맥자유 언어 (CFL) | 푸시다운 오토마타 |
| 1 | 문맥인식 문법 (CSG) | 문맥인식 언어 (CSL) | 선형 유계 오토마타 |
| 0 | 무제약 문법 | 재귀열거 가능 언어 | 튜링 기계 |

> **어휘분석은 type 3 (정규 문법/FA)** 으로 처리. 구문분석은 type 2 (CFG/PDA).

---

### 자주 나오는 언어 예시

| 언어 | 형태 | 분류 |
|---|---|---|
| 단순 매칭 | $a^n b^n$ | CFL (type 2) |
| 중복 매칭 | $a^n b^n c^n$ | CSL (type 1) |
| 좌우 대칭 | $ww^R$ | CFL (type 2) |
| 회문 | $w = w^R$ | CFL (type 2) |
| 괄호 | balanced parenthesis | CFL (type 2) |

> $a^n b^n$이 CFL인 이유: a를 스택에 쌓고 b를 만날 때마다 pop하면 인식 가능 → 푸시다운 오토마타 필요 → FA로는 불가.

---

## 04. FA ↔ 정규 문법 변환

### FA → 정규 문법 변환 방법

**입력**: FA $M = (Q, \Sigma, \delta, q_0, F)$  
**출력**: 정규 문법 $G = (V_N, V_T, P, S)$

**변환 규칙**:
- $V_N = Q$ (상태 → Nonterminal)
- $V_T = \Sigma$ (입력 알파벳 → Terminal)  
- $S = q_0$ (시작 상태 → 시작 기호)
- 전이 함수 → 생성 규칙:
  - $\delta(q, a) = r$ 이면 → $q \rightarrow ar$
  - $q \in F$ (종결 상태) 이면 → $q \rightarrow \varepsilon$

> 이렇게 만들어지는 문법은 **Right-linear Grammar** (우선형 문법): $A \rightarrow aB$ 형태

---

### 예제: FA → 정규 문법 → 정규 표현

**FA 구조** (슬라이드 예):
```
start → A ---(a|b)--→ A
        A ---a------→ B
        B ---b------→ C
        C ---b------→ D  (D는 종결 상태)
```

**정규 문법 변환**:
```
A → aA | bA | aB
B → bC
C → bD
D → ε
```

**정규 표현 변환**:
```
D = ε
C = bD = b
B = bC = bb
A = aA + bA + aB
  = (a+b)A + abb
  → A = (a+b)* abb    [∵ X = αX + β → X = α*β]

∴ L(M) = (a+b)*abb
```

---

### 정규 표현 방정식의 해

> $\alpha$가 $\varepsilon$을 포함하지 않을 때:
> $$X = \alpha X + \beta \Rightarrow X = \alpha^* \beta$$

**증명**:
$$X = \alpha(\alpha^*\beta) + \beta = \alpha^+\beta + \beta = (\alpha^+ + \varepsilon)\beta = \alpha^*\beta$$

이 공식이 핵심이다. 정규 문법 → 정규 표현 변환의 거의 모든 문제가 이 꼴로 귀결된다.

---

### 정규 표현 대수 법칙

| 법칙 | 식 |
|---|---|
| 결합법칙 | $(\alpha+\beta)+\gamma = \alpha+(\beta+\gamma)$ |
| 교환법칙 | $\alpha + \beta = \beta + \alpha$ |
| 분배법칙 | $\alpha(\beta+\gamma) = \alpha\beta + \alpha\gamma$ |
| 항등 | $\alpha + \alpha = \alpha$, $\varepsilon\alpha = \alpha$ |
| 클리니 | $\varepsilon + \alpha\alpha^* = \alpha^*$ |
| 이중 클리니 | $(\alpha^*)^* = \alpha^*$ |

> 기호 정리: `+` = OR(선택), `·` = 연결(concatenate), `*` = 0회 이상 반복

---

### 예제 모음

#### 예제 1 (정규 문법 → RE)
```
S → aA | bS
A → aS | bB
B → aB | bB | ε
```
풀이:
```
B = (a+b)B + ε  →  B = (a+b)*
A = aS + b(a+b)*
S = aA + bS = a(aS + b(a+b)*) + bS
  = aaS + ab(a+b)* + bS
  = (aa+b)S + ab(a+b)*
  → S = (aa+b)* ab(a+b)*

∴ L(G) = (aa+b)*ab(a+b)*
```

#### 예제 2 (정규 문법 → RE)
```
S → aA | bS
A → aA | bA | b
```
풀이:
```
A = (a+b)A + b  →  A = (a+b)*b
S = a(a+b)*b + bS = bS + a(a+b)*b
  → S = b*a(a+b)*b

∴ L(G) = b*a(a+b)*b
```

#### 예제 3 (FA → RE, 슬라이드 p.18~19)
```
G: A → aA | bA | aB,  B → bC,  C → bD,  D → ε
```
풀이:
```
D = ε,  C = b,  B = bb
A = (a+b)A + abb
  → A = (a+b)*abb

∴ L(G) = (a+b)*abb
```

#### 예제 4 (Unreachable state 제거)
```
A → aB | bC | ε
B → bA          (aD 전이는 도달 불가 → 제거)
C → aA          (bD 전이는 도달 불가 → 제거)
```
풀이:
```
B = bA,  C = aA
A = aB + bC + ε
  = a(bA) + b(aA) + ε
  = abA + baA + ε
  = (ab+ba)A + ε
  → A = (ab+ba)*

∴ L(G) = (ab+ba)*
```

---

## 05. 토큰 인식 예제

### 정수(Integer) 인식

**DFA**:
```
start → S --d-→ [C] --d-→ (자기 루프)
```

**정규 문법**:
```
S → dC
C → dC | ε
```

**정규 표현**:
```
C = dC + ε  →  C = d*
S = dC = dd* = d+

∴ L(G) = d+  (한 자리 이상의 정수)
```

---

### 식별자(Identifier) 인식

**정규 문법 정의**:
```
<ident>  ::= (letter | _) { letter | digit | _ }
<letter> ::= a | b | ... | z | A | ... | Z
<digit>  ::= 0 | 1 | ... | 9
```
(편의상 letter=l, digit=d로 축약)

**DFA**:
```
start → [1] --(l|_)-→ [2] --(l|d|_)-→ (자기 루프, 종결상태)
```

**정규 문법**:
```
S → lA | _A
A → lA | dA | _A | ε    (종결 상태이므로 ε 추가)
```

**정규 표현**:
```
A = (l+d+_)A + ε  →  A = (l+d+_)*
S = (l+_)A = (l+_)(l+d+_)*

∴ L(G) = (l+_)(l+d+_)*
```

> 즉, 식별자는 **알파벳 또는 언더스코어로 시작**하고, 이후 **알파벳/숫자/언더스코어**가 0회 이상 반복. (`_count`, `myVar2` 등)

---

## 정리: 변환 흐름 요약

```
소스코드
  ↓
[어휘분석기 (Scanner)]
  - 정규 표현(RE)으로 토큰 패턴 정의
  - FA(유한 오토마타)로 인식
  - getToken() 호출 시 토큰 1개 반환
  ↓
토큰 스트림 → [구문분석기 (Parser)]
```

**설계 순서**:
```
토큰 패턴 → 정규 문법 → 정규 표현 → NFA → DFA → 최소화 DFA → 코드
```

**핵심 공식**:
$$X = \alpha X + \beta \Rightarrow X = \alpha^*\beta$$
