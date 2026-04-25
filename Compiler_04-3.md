# Chapter 3-2. 유한 오토마타 (Finite Automata) Part I

---

## 1. 유한 오토마타란?

### 직관적 이해 - 커피 자판기

> 커피 자판기는 **상태(state)** 를 갖고 있고, 동전이라는 **입력(input)** 에 따라 상태가 바뀐다.

| 입력 | 상태 변화 |
|------|-----------|
| 10원 투입 | "140원이 더 필요" 상태로 |
| 50원 투입 | "100원이 더 필요" 상태로 |
| 100원 투입 | "50원이 더 필요" 상태로 |
| 500원 투입 | "Press the button!" 상태로 |

- **시작 상태**: 0원 투입된 상태
- **종결 상태(accepting state)**: 150원 이상 누적된 상태 → 커피 제공
- 상태 수 = 16개: S₀(0원), S₁(10원), ..., S₁₅(150원 이상)

### Finite State Automata 정의

**FA (Finite Automata)** = 유한 개의 상태를 갖는 가상 기계(Hypothetical Machine)

- 유한(finite)개의 **상태(state)** 를 가짐
- **입력**에 따라 상태가 바뀜 (ex: 윷판의 말이 윷 결과에 따라 움직이듯)
- **시작 상태**와 **종결 상태**가 존재
- 정규 표현(RE)은 유한 오토마타(recognizer)로 변환 가능

> **Automata**는 복수, **Automaton**은 단수

---

## 2. DFA (결정적 유한 오토마타)

### 공식 정의

$$DFA\ M = (\Sigma,\ S,\ T,\ s_0,\ A)$$

| 구성요소 | 의미 |
|----------|------|
| **Σ** (Sigma) | 입력 알파벳 |
| **S** | 유한 상태 집합 |
| **s₀** | 초기(시작) 상태, s₀ ∈ S |
| **A** | 수용(종결) 상태 집합, A ⊂ S |
| **T (또는 δ)** | 상태 전이 함수 |

### 상태 전이 함수 (State Transition Function)

$$T: S \times \Sigma \rightarrow S$$

$$T(S_i, a) = S_j \quad (S_i \in S,\ S_j \in S,\ a \in \Sigma)$$

- `T(현재 상태, 입력 기호) = 다음 상태`
- **(ex)** T(S₁, 100원) = S₁₁ → 10원 상태에서 100원 투입 시 110원 상태로 이동

### DFA의 핵심 특징

DFA는 아래 **두 조건을 모두** 만족해야 함:

1. **ε-transition이 없다** (ε = 빈 문자열에 의한 상태 전이 없음)
2. **현재 상태에서 입력 기호에 대한 다음 상태는 정확히 하나** (unique)

$$\delta: S \times \Sigma \rightarrow S$$

> ⚠️ 위 조건 중 하나라도 만족하지 않으면 → **NFA**

### DFA 표현 방법

**① 상태 전이 다이어그램 (State Transition Diagram)**

- 원(○) = 상태
- 이중 원(◎) = 수용 상태 (accepting/final state)
- 화살표(→) = 전이, 레이블 = 입력 기호
- 시작 상태에는 별도 화살표 표시

**② 상태 전이 테이블 (State Transition Table)**

`M = ({a, b}, {q0, q1, q2}, T, q0, {q2})` 예시:

| 상태 | a | b |
|------|---|---|
| q0 | q1 | q2 |
| q1 | q2 | q0 |
| **q2*** | q0 | q1 |

> `*` 표시 = 수용 상태 / 시작 상태(q0)는 별도 표기

**다이어그램:**
```
     a         a         a
q0 ←──→ q1 ←──→ q2 ←──→ q0
 ↑   b    ↑   b    ↑   b
 └────────┘   └────────┘
```

실제로는 q0, q1, q2 세 상태가 a/b 입력으로 서로 순환하는 구조:
- q0 -a→ q1 -a→ q2 -a→ q0 (사이클)
- q0 -b→ q2 -b→ q1 -b→ q0 (반대 방향 사이클)

---

## 3. DFA 인식 과정

### L(M): DFA M이 인식하는 언어

DFA M이 문자열 $w = c_1 c_2 \cdots c_n$ 을 인식하려면:

$$s_1 = T(s_0, c_1),\quad s_2 = T(s_1, c_2),\quad \cdots,\quad s_n = T(s_{n-1}, c_n)$$

- 상태 전이: $s_0 \rightarrow s_1 \rightarrow s_2 \rightarrow \cdots \rightarrow s_n$
- **조건**: $s_n \in A$ (마지막 상태가 수용 상태여야 함)

### 인식 예시 (M = ({a,b}, {q0,q1,q2}, T, q0, {q2}))

**① 입력 "aba" → 인식 실패**

$$T(q0, a) = q1 \rightarrow T(q1, b) = q0 \rightarrow T(q0, a) = q1$$

- 최종 상태 q1 ∉ A → **인식 안 됨**

**② 입력 "ababb" → 인식 성공**

$$T(q0, a) = q1 \rightarrow T(q1, b) = q0 \rightarrow T(q0, a) = q1 \rightarrow T(q1, b) = q0 \rightarrow T(q0, b) = q2$$

- 최종 상태 q2 ∈ A → **인식됨** ✓

> 💡 이 DFA가 인식하는 패턴: `a`와 `b`의 **개수가 3의 배수만큼 차이나는 문자열**  
> (더 정확히는, (a개수 - b개수) mod 3 == 2 인 문자열)

---

## 4. DFA 예제 모음

### Practice #1: 이진 문자열 DFA

다이어그램 (Σ = {0, 1}):
```
    1(self)        1           0(self)
→ [s0] ──0──→ (s1) ──1──→ ((s2))
              ←──0──        ←──1──
```
- s0: 시작 상태, self-loop on 1
- s1: self-loop on 0
- s2: **수용 상태**, self-loop 없음
- s0 ↔ s2: 입력 1로 s0→s2 연결 (arc)

| 상태 | 0 | 1 |
|------|---|---|
| s0 | s1 | s0 |
| s1 | s1 | s2 |
| **s2*** | s1 | s0 |

**인식 예시:**
- "011": s0 -0→ s1 -1→ s2 -1→ s0 → **인식 안 됨** (s0 ∉ A)
- "0111": s0 -0→ s1 -1→ s2 -1→ s0 -1→ s0 → **인식 안 됨**

> 💡 이 DFA가 인식하는 언어: **1의 개수가 정확히 2 mod 3 인 문자열** (즉 "11" 포함, 3개 단위로 리셋)

---

### Practice #4: Dead State (Trap State)

어떤 DFA는 특정 입력에 대한 전이가 정의되지 않을 수 있다 → **DFA 정의 위반**  
해결: **Dead State(죽은 상태)** 추가

```
상태 전이 테이블 (dead state D 추가):

      0    1
A     D    B
B     C    B
C*    C    B
D     D    D    ← 어떤 입력이 와도 D에서 탈출 불가 (trap)
```

- **Dead State**: 한번 들어오면 나갈 수 없는 상태
- 수용 상태가 아님 → 인식 실패를 의미

**인식 예시:**
- "1100": A -1→ B -1→ B -0→ C -0→ C → **인식됨** ✓ (C ∈ A)
- "10001": A -1→ B -0→ C -0→ C -0→ C -1→ B → **인식 안 됨**
- "0001": A -0→ D -0→ D -0→ D -1→ D → **인식 안 됨** (trap)

---

### Practice #5: Even Parity Checker (짝수 패리티 검사기)

| 상태 | 0 | 1 |
|------|---|---|
| **A*** | A | B |
| B | B | A |

```
A ←─0─→ A
A ←─1─→ B ←─1─→ A
         B ←─0─→ B
```

- **A**: 1의 개수가 짝수 (수용 상태, 시작 상태)
- **B**: 1의 개수가 홀수

**인식 예시:**
- "1001": A -1→ B -0→ B -0→ B -1→ A → **인식됨** ✓ (1이 2개, 짝수)
- "10100 1": A -1→ B -0→ B -1→ A -0→ A -0→ A -1→ B → **인식 안 됨**
- "0011": A -0→ A -0→ A -1→ B -1→ A → **인식됨** ✓

> 💡 **짝수 패리티**: 1의 개수가 짝수인 문자열만 인식  
> **홀수 패리티 검사기**: A, B의 수용 상태를 반전시키면 됨 (B가 수용 상태)

> **Q: 빈 문자열 ε 인식 가능?**  
> → 시작 상태 A = 수용 상태이므로 **ε 인식 가능** ✓ (1이 0개 = 짝수)

---

### Practice #6: "aabb" 부분문자열 포함하는 문자열 인식

Σ = {a, b}

```
상태 흐름: A -a→ B -a→ C -b→ D -b→ E(수용)
                          ↑              ↑
           나머지는 루프    a,b로 자기 유지
```

| 상태 | a | b |
|------|---|---|
| A | B | A |
| B | C | A |
| C | C | D |
| D | B | E |
| **E*** | E | E |

- A: 초기, aabb 시작 전
- B: a 1개 읽음
- C: aa 읽음
- D: aab 읽음
- E: **aabb 완성** (수용 상태, self-loop)

---

### Practice #7: "aabb" 포함하지 않는 문자열 인식

> 💡 **보수 DFA (Complement DFA)**: 수용 상태 ↔ 비수용 상태를 뒤집으면 됨

Practice #6의 DFA에서:
- **final state → non-final state**
- **non-final state → final state**

| 상태 | a | b |
|------|---|---|
| **A*** | B | A |
| **B*** | C | A |
| **C*** | C | D |
| **D*** | B | E |
| E | E | E |

---

### 숫자 인식 DFA

#### signedNat (부호 있는 자연수)
```
digit = [0-9]
nat = digit+
signedNat = (+|-)? nat
```

구조: `[시작] -+/-- [부호상태] -digit→ [digit루프, 수용]`  
또는: `[시작] -digit→ [digit루프, 수용]` (부호 생략 경우)

#### number (실수)
```
number = signedNat ("." nat)?
```

구조: signedNat DFA에 `.` → nat 루프 추가

#### floating-point number (부동소수점)
```
number = signedNat ("." nat)? (E signedNat)?
```

구조: number DFA에 `E` → signedNat 루프 추가

---

### 주석(Comment) 인식 DFA

**① 중괄호로 감싸인 주석 `{...}`**
```
[시작] -'{'→ [중괄호 내부, other self-loop] -'}'→ ((수용))
```

**② `/* ... */` 스타일 주석**
```
1 -'/'→ 2 -'*'→ 3 ←─other─┐
                  └─'*'→ 4 ─┘
                         4 -'/'→ ((5))
                         4 ←─other─ 3
```

| 상태 | 의미 |
|------|------|
| 1 | 시작 |
| 2 | `/` 읽음 |
| 3 | 주석 내부 |
| 4 | `*` 읽음 (닫기 후보) |
| **5** | 주석 완성 (수용) |

---

## 5. NFA (비결정적 유한 오토마타)

### NFA가 필요한 이유

여러 token(`:=`, `<=`, `=` 등)에 대한 DFA를 **하나의 DFA로 합치기가 어렵다**
- DFA: 입력 기호에 대해 다음 상태가 **unique**해야 함
- 여러 DFA를 합치면 같은 입력에 여러 상태가 가능 → DFA 위반

→ 해결책: **NFA** (DFA 정의를 확장)

### NFA의 특징 (DFA와의 차이)

| 구분 | DFA | NFA |
|------|-----|-----|
| 다음 상태 수 | 정확히 1개 | 0개 이상 (집합) |
| ε-transition | ❌ 없음 | ✅ 있을 수 있음 |
| 전이 함수 | T: S × Σ → S | T: S × (Σ ∪ {ε}) → P(S) |

### NFA 공식 정의

$$NFA\ M = (\Sigma,\ S,\ T,\ s_0,\ A)$$

상태 전이 함수:

$$T: S \times (\Sigma \cup \{\varepsilon\}) \rightarrow P(S)$$

$$T(S_i, a) = S' \quad \text{단, } S' \subseteq S,\quad a \in \Sigma \cup \{\varepsilon\}$$

- **P(S)**: S의 **Power Set** (S의 모든 부분집합의 집합)
  - (ex) S = {A, B} → P(S) = {∅, {A}, {B}, {A,B}} = 2² = 4개
  - (ex) S = {A, B, C} → P(S) = 2³ = 8개

### ε-transition (엡실론 전이)

- **입력 기호 없이** 자동으로 발생하는 상태 전이
- Lookahead 없음, 입력 문자열 변경 없음
- "자발적으로(spontaneously)" 발생

```
(상태1) ──ε──→ (상태2)   ← 입력 소비 없이 전이
```

### NFA 예시: Practice #8

Σ = {0, 1}

**L1: 1로 끝나는 모든 문자열**
```
→ [A] ──0,1──→ [A] ──1──→ ((B))
```
| 상태 | 0 | 1 |
|------|---|---|
| A | A | A, B |
| **B*** | - | - |

**L2: 0을 포함하는 모든 문자열**
```
→ [A] ──0──→ ((B)) ──0,1──→ ((B))
   └──0,1──┘
```

**L3: 10으로 시작하는 모든 문자열**
```
→ [A] ──1──→ [B] ──0──→ ((C)) ──0,1──→ ((C))
```

---

## 6. NFA 인식 과정

### L(M): NFA M이 인식하는 언어

NFA가 문자열 $c_1 c_2 \cdots c_n$ 을 인식하려면:  
- 가능한 상태 전이 경로 중 **적어도 하나**가 수용 상태로 끝나야 함
- $c_i \in \Sigma \cup \{\varepsilon\}$ (ε 포함)

### NFA 문자열 인식 예시 (1)

NFA: start → q0 -a→ q1 / q0 -a,b→ q0 / q1 -b→ q2 -b→ q3(수용)

입력: **"baabb"**

$$\delta(q0, baabb) = \delta(q0, aabb)$$
$$= \delta(q0, abb) \cup \delta(q1, abb)$$
$$= \{δ(q0, bb) \cup \delta(q1, bb)\} \cup \emptyset$$
$$= \delta(q0, b) \cup \delta(q2, b)$$
$$= \{q0\} \cup \{q3\}$$
$$= \{q0, q3\}$$

→ {q0, q3} ∩ A = **{q3}** ≠ ∅ → **인식됨** ✓

### ε-closure를 이용한 NFA 인식

NFA에서 ε-transition이 있으면:

1. **ε-closure** 계산: 현재 상태에서 ε으로 도달 가능한 모든 상태 집합
2. 각 입력 기호마다 → 도달 가능한 모든 상태에 전이 적용 → 다시 ε-closure

**(ex)** 상태 1에서 시작, 1 -ε→ {2, 4}, 1 -a→ {2, 3}, 3 -ε→ 4

입력 **"abb"** 처리:
```
시작: ε-closure(1) = {1, 2, 4}

'a' 처리:
  1 -a→ {2, 3} → ε-closure = {2, 3, 4}
  2 -a→ {}
  4 -a→ {}
  결과: {2, 3, 4}

'b' 처리:
  2 -b→ {4} (수용 상태 도달!)
  ...
```

---

## 7. 왜 NFA가 필요한가?

### 컴파일러에서의 역할

```
정규 표현(RE) ──→ NFA ──→ DFA ──→ 프로그램(Lexer)
```

| 단계 | 특징 |
|------|------|
| RE → NFA | **변환이 쉬움** (Thompson's construction 알고리즘) |
| NFA → DFA | **Subset Construction** 알고리즘으로 변환 |
| DFA → 코드 | **코딩이 쉬움** → 빠른 인식 속도 |

- RE는 항상 FA로 나타낼 수 있다
- NFA는 반드시 동등한 기능의 DFA로 변환 가능
- DFA가 실제 구현에 사용됨 (fast recognition time)

### PL Token 인식 NFA 예시

프로그래밍 언어의 토큰들 (`:=`, `<=`, `=`):

```
각각 별도 NFA:
→ [  ] -':'→ [  ] -'='→ (( )) return ASSIGN
→ [  ] -'<'→ [  ] -'='→ (( )) return LE
→ [  ] -'='→ (( ))           return EQ

ε-transition으로 하나로 합친 NFA:
         ε → ASSIGN DFA
→ [시작] ε → LE DFA
         ε → EQ DFA
```

---

## 8. 보충: Pumping Lemma

### Regular Set (정규 집합)

- **정규 집합**: 정규 표현(RE)으로 나타낼 수 있는 문자열 집합
- 유한 오토마타는 **메모리가 없음** → 개수를 셀 수 없음

### Non-Regular Language (비정규 언어)

(ex) Σ = {a, b}, b 하나를 같은 수의 a로 둘러싼 문자열:

$$L = \{b,\ aba,\ aabaa,\ aabaaa,\ \cdots\} = \{a^n b a^n\ |\ n \neq 0\}$$

- `a*ba*` 로는 표현 불가 → a의 개수가 같다는 조건을 정규 표현으로 표현할 수 없음
- **"Regular expression can't count"**

### Pumping Lemma

> 어떤 언어가 **정규 언어가 아닌지** 판별하는 도구

**핵심 아이디어:**

- 유한 오토마타 상태 수 = k (finite)
- 상태 수 k보다 긴 문자열 N (|N| > k)이 정규 언어라면
- 반드시 **루프를 갖는 상태**가 존재 → 해당 부분을 **"pumping"** (반복) 가능

**Pumping Lemma 공식 진술:**

L이 정규 언어라면, pumping 상수 P가 존재하여:  
L에 속한 문자열 w (|w| ≥ P)를 w = xyz로 나눌 때:

1. |y| ≥ 1 (y는 빈 문자열이 아님)
2. |xy| ≤ P
3. 모든 i ≥ 0에 대해 xy^i z ∈ L

**만약 위 조건을 만족하지 않는 i가 존재한다면** → L은 정규 언어가 아님 (모순)

**(ex)** L = {aⁿbаⁿ | n ≠ 0}이 비정규 언어임을 증명:

1. L이 정규 언어라 가정, pumping 상수 P 존재
2. w = aᴾbaᴾ (|w| = 2P+1 > P)
3. w = xyz로 나눌 때, |xy| ≤ P이므로 y는 a들로만 구성
4. i = 0이면: xz = aᴾ⁻ʸbaᴾ → 앞뒤 a 개수가 다름 → L에 속하지 않음
5. **모순** → L은 정규 언어가 아님 ∎

---

## 요약 비교표

| 구분 | DFA | NFA |
|------|-----|-----|
| 다음 상태 | 정확히 1개 | 0개 이상 (집합) |
| ε-transition | ❌ | ✅ |
| 전이 함수 | S × Σ → S | S × (Σ∪{ε}) → P(S) |
| 표현력 | 정규 언어 | 정규 언어 (동등) |
| 구현 용이성 | ✅ 쉬움 | 어려움 |
| 설계 용이성 | 어려움 | ✅ 쉬움 |
| 변환 관계 | NFA를 DFA로 항상 변환 가능 | RE → NFA 변환이 쉬움 |

> **핵심**: RE ─쉽게→ NFA ─항상 변환→ DFA ─구현→ Lexer(어휘 분석기)
