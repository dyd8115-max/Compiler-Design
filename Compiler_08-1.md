# Chapter 7 — 상향식 구문분석 (Bottom-up Parsing) Part I

---

## 1. Overview: LR 파서 종류 비교

| 파서 종류 | 풀네임 | 특징 |
|-----------|--------|------|
| **LR(0)** | — | lookahead 없음. 파싱 테이블 가장 작음 |
| **SLR(1)** | Simple LR(1) | lookahead 1개 사용. LR(0)보다 강력 |
| **LALR(1)** | Lookahead LR(1) | SLR(1)보다 강력, LR(1)보다 단순 |
| **LR(1)** | Canonical LR(1) | 가장 강력. 파싱 테이블 가장 큼 |

**포함 관계 (언어 수용 능력):**

```
LR(0) ⊂ SLR(1) ⊂ LALR(1) ⊂ LR(1)
```

파싱 테이블 크기: LR(0) < SLR(1) < LALR(1) < LR(1)

> **실용적 선택**: 실제 컴파일러에서는 대부분 **LALR(1)**을 사용한다. yacc/bison 도구가 LALR(1) 기반이다.

---

## 2. LR 파서 기본 개념

### 2.1 Shift-Reduce Parsing

LR 파서는 **Bottom-Up** 방식으로 동작한다.

- **R**: Rightmost derivation (우측 유도)의 역순으로 파싱
- 빈 스택에서 시작
- 두 가지 핵심 액션:

| 액션 | 설명 |
|------|------|
| **Shift** | 입력 버퍼의 기호를 스택에 push |
| **Reduce** | 스택 top의 기호열 α를 nonterminal A로 교체 (pop α → push A) |

파싱이 성공하면 스택에 **start symbol S만 남는다**.

---

### 2.2 Bottom-Up Parsing 예시

#### 예 1 (괄호 문법)

```
확장 문법: S' → S
S → (S)S | ε
```

| Parsing Stack | Input | Action |
|---------------|-------|--------|
| `$` | `()$` | shift |
| `$(` | `)$` | reduce S → ε |
| `$(S` | `)$` | shift |
| `$(S)` | `$` | reduce S → ε |
| `$(S)S` | `$` | reduce S → (S)S |
| `$S` | `$` | reduce S' → S |
| `$S'` | `$` | **accept** |

> 같은 S이지만 각각 다른 생성 규칙으로 reduce함.  
> 어떤 rule을 적용할지 판단하기 위해 **스택에 저장된 기호(stack look-ahead)를 참조**한다.

---

#### 예 2 (덧셈 문법)

```
확장 문법: E' → E
E → E + n | n
```

| Step | Parsing Stack | Input | Action |
|------|---------------|-------|--------|
| 1 | `$` | `n+n$` | shift |
| 2 | `$n` | `+n$` | reduce E → n |
| 3 | `$E` | `+n$` | shift |
| 4 | `$E+` | `n$` | shift |
| 5 | `$E+n` | `$` | reduce E → E + n |
| 6 | `$E` | `$` | reduce E' → E |
| 7 | `$E'` | `$` | **accept** |

> **중요 관찰**: 단계 3과 6에서 스택에 똑같이 `E`가 있지만 전혀 다른 action이 발생한다.  
> 이유: **input lookahead**가 다르기 때문 — `+` (단계 3) vs `$` (단계 6)

---

## 3. General Observations

### 3.1 Right Sentential Form (우 문장 형태)

> **우 문장 형태**: 우측 유도(rightmost derivation) 과정에서 만들어지는 모든 중간 문자열

- shift-reduce 파서는 우측 유도의 **역순**으로 입력을 추적한다.

**예 1 역순 유도:**
```
S' ⇒ S ⇒ (S)S ⇒ (S) ⇒ ()
```

**예 2 역순 유도:**
```
E' ⇒ E ⇒ E + n ⇒ n + n
```

우 문장 형태는 파싱 스택의 문자열과 입력 버퍼에 남은 문자열로 구분할 수 있다. (`‖` 기호로 구분)

- `E ‖ +n` (step 3)
- `E+ ‖ n` (step 4)

---

### 3.2 Viable Prefix & Handle

| 개념 | 정의 |
|------|------|
| **Handle (핸들)** | reduce의 대상이 되는 스택 top의 기호열. 생성 규칙의 RHS에 해당하는 문자열 α |
| **Viable Prefix** | 우 문장 형태의 handle을 넘어서지 않는 prefix. 즉, 파싱 스택에 있는 문자열 |

**예시:**
```
E → E + n | n
우 문장 형태: E + n
viable prefix: ε, E, E+, E+n
```

- `E, E+, E+n`은 우 문장 형태 `E + n`의 viable prefix
- `n+`은 viable prefix인가? → **아니다** (handle을 넘어서는 prefix)

> viable: 실행 가능한, 생명력이 있는

---

### 3.3 예 3: 우단 유도와 handle 찾기

**입력 문장 `id + id * id`에 대한 우단 유도와 핸들:**

```
생성 규칙:
① E → E + T    ② E → T       ③ T → T * F
④ T → F        ⑤ F → (E)     ⑥ F → id
```

| 우문장 형태 | 핸들 | 감축에 사용되는 생성 규칙 |
|------------|------|--------------------------|
| `id1 + id2 * id3` | `id1` | F → id |
| `F + id2 * id3` | `F` | T → F |
| `T + id2 * id3` | `T` | E → T |
| `E + id2 * id3` | `id2` | F → id |
| `E + F * id3` | `F` | T → F |
| `E + T * id3` | `id3` | F → id |
| `E + T * F` | `T * F` | T → T * F |
| `E + T` | `E + T` | E → E + T |
| `E` | — | (완료) |

---

## 4. 전처리: 확장 문법 (Augmented Grammar)

LR 파싱 전에 반드시 필요한 전처리.

**왜 필요한가?**

- 원래 시작 기호 S가 여러 production을 가질 경우 (`S → α | β`), 시작 상태를 하나로 정할 수 없다.
- `S' → S` 라는 단일 production을 추가하면 시작 상태가 `S' → .S` 하나로 확정된다.

**방법:**

```
원래 문법에 S' → S 를 추가
→ Start symbol이 S에서 S'으로 변경
→ 이를 확장 문법(Augmented Grammar)이라 부름
```

**예시:**

```
원래: S → (S)S | ε
확장: S' → S
      S  → (S)S | ε
```

---

## 5. LR 파싱 알고리즘 개요

**핵심 질문들과 답변:**

**Q1: handle을 어떻게 찾는가?**

→ 생성 규칙의 RHS(Right Hand Side)에 해당하는 문자열 α를 찾는다.

**Q2: shift할지 reduce할지 어떻게 구분하는가?**

→ RHS에서 일부만 찾았으면 **shift**, 다 찾은 경우에만 **reduce**한다.

```
A → X₁X₂ ... Xᵢ • Xᵢ₊₁ ... Xₙ   ← 아직 못 찾은 것 있음 → shift
A → X₁X₂ ... Xₙ •                ← 전부 찾음 → reduce
```

**Q3: 어디까지 찾았는지 어떻게 아는가?**

→ **dot(.)** 으로 marking한다. (LR(0) Item)

**Q4: 모든 생성 규칙에 대해 일일이 marking하면 너무 많지 않은가?**

→ 비슷한 성질을 갖는 item들을 묶어서 **state(상태)** 로 관리한다.

---

## 6. LR(0) Items

### 6.1 정의 및 종류

> **LR(0) Item**: 생성 규칙의 RHS에 **dot(.)** 을 삽입한 것

- 숫자 '0'의 의미: **입력(lookahead)을 참조하지 않음**

`A → XYZ`의 생성 가능한 LR(0) item 4개:

```
A → . X Y Z    ← initial item
A → X . Y Z
A → X Y . Z
A → X Y Z .    ← complete (reduce) item
```

---

**Item의 dot 의미:**

```
A → α . β
```

| 위치 | 의미 |
|------|------|
| dot 앞 (α) | 현재 파싱 스택에 저장되어 있음. reduce 시킬 대상 |
| dot 뒤 (β) | 입력 버퍼에 아직 남아 있음. shift 시킬 대상 |

---

**Item 종류 정리:**

| 종류 | 형태 | 설명 |
|------|------|------|
| **Initial item** (= Closure item의 씨앗) | `A → . α` | dot이 맨 앞. 아직 아무것도 인식 안 함 |
| **mark symbol** | `A → α . X β` 에서 X | dot 바로 다음에 나오는 기호 |
| **Complete item** (= Reduce item) | `A → α .` | dot이 맨 끝. RHS α를 전부 찾은 상태 → reduce 가능 |

---

### 6.2 예 4: LR(0) Items 목록

#### 예 4(a)

```
S' → S
S  → (S)S | ε
```

| Item | 의미 |
|------|------|
| `S' → . S` | initial item |
| `S' → S .` | complete item |
| `S → . (S)S` | initial item |
| `S → ( . S)S` | |
| `S → (S . )S` | |
| `S → (S) . S` | |
| `S → (S)S .` | complete item |
| `S → .` | complete item (ε production) |

---

#### 예 4(b)

```
E' → E
E  → E + n | n
```

| Item |
|------|
| `E' → . E` |
| `E' → E .` |
| `E → . E + n` |
| `E → E . + n` |
| `E → E + . n` |
| `E → E + n .` |
| `E → . n` |
| `E → n .` |

---

## 7. Finite Automata of LR(0) Items

### 7.1 NFA 상태 천이 규칙

**state = LR(0) items들의 집합**

#### 케이스 1: X가 terminal (token)

```
A → α . X η  --X-->  A → α X . η
```

X를 읽으면 스택에 push하고 dot을 오른쪽으로 이동.

---

#### 케이스 2: X가 nonterminal

X를 스택에 push하려면 먼저 X를 인식(reduce)해야 한다.

```
A → α . X η  --ε-->  X → . β   --β-->  X → β .
                                            ↓ (reduce by X → β)
A → α . X η  --X-->  A → α X . η
```

- `A → α . X η` 에서 X가 nonterminal이면 → ε-transition으로 `X → . β` 추가
- `X → β .` (complete item)에서 reduce → X를 스택에 push
- 결과적으로 `A → α X . η` 로 천이

---

#### 시작 상태

```
시작 상태: S' → . S
```

**확장 문법이 필요한 이유:**
- `S → α | β`인 경우 `S → .α` 와 `S → .β` 중 어느 것을 시작 상태로 할지 불분명
- `S' → S`는 single production이므로 시작 상태를 `S' → .S` 하나로 확정 가능

**No accepting states:**
- 오토마타는 상태 천이로 파싱 과정을 추적할 뿐, 인식 여부는 **파싱 알고리즘**에서 결정

---

### 7.2 DFA 변환: CLOSURE와 GOTO

#### CLOSURE(I)

> CLOSURE(I): kernel item I로부터 ε-closure를 구해 상태를 완성하는 함수

**규칙:**

1. kernel item I는 CLOSURE(I)에 포함
2. `[A → α • B β] ∈ CLOSURE(I)` 이고, `B → γ ∈ P` 이면 → `[B → . γ]` 를 CLOSURE(I)에 추가
3. 더 이상 새로운 item이 추가되지 않을 때까지 반복

**수식:**

```
CLOSURE(I) = CLOSURE(I) ∪ { [B → .γ] | [A → α.Bβ] ∈ CLOSURE(I), B → γ ∈ P }
```

---

**예 8 — CLOSURE 계산 예시:**

```
E' → E      E → E + T | T
T  → T * F | F    F → (E) | id
```

```
CLOSURE([E' → • E]) =
  { [E' → •E],
    [E → •E+T],
    [E → •T],
    [T → •T*F],
    [T → •F],
    [F → •(E)],
    [F → •id] }
```

```
CLOSURE([E → E+ • T]) =
  { [E → E+•T],
    [T → •T*F],
    [T → •F],
    [F → •(E)],
    [F → •id] }
```

---

#### GOTO(I, X) 함수

> 현재 상태 I에서 기호 X를 읽었을 때 이동하는 다음 상태

**수식:**

```
GOTO(I, X) = CLOSURE({ [A → αX.β] | [A → α.Xβ] ∈ I })
```

즉, I 안의 item 중 mark symbol이 X인 것들을 골라서, dot을 X 오른쪽으로 옮긴 뒤 CLOSURE를 구한다.

**예 9:**

```
I = { [E' → E.], [E → E.+T] }
GOTO(I, +) = CLOSURE([E → E+.T])
           = { [E → E+.T], [T → .T*F], [T → .F], [F → .(E)], [F → .id] }
```

---

#### DFA 상태 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Kernel item** | 상태 천이 과정에서 직접 생성된 item. 상태를 정의하고 천이를 결정하는 주체 |
| **Closure item** | ε-transition(CLOSURE)을 통해 추가된 item. kernel item에 의해 자연스럽게 추가됨 |

---

### 7.3 DFA 구성 방법

```
1. 시작 상태 = CLOSURE([S' → .S])

2. 현재 상태 I에 속한 각 LR(0) item [A → α.Xβ]에서 mark symbol X를 찾는다.

3. 중복되지 않은 mark symbol 개수만큼 상태 천이 발생

4. 다음 상태 J = GOTO(I, X)
   → dot을 X 오른쪽으로 옮기고 CLOSURE 계산

5. reduce item(complete item)이면 다음 상태 없음

6. 더 이상 새로운 상태를 구할 수 없으면 종료
```

---

### 7.4 예 7 / 예 10: DFA 구성 예시

#### 예 7 — `S' → S`, `S → (S)S | ε`

**DFA 상태 목록:**

| 상태 | Kernel item | Closure items |
|------|-------------|---------------|
| **0** | `S' → .S` | `S → .(S)S`, `S → .` |
| **1** | `S' → S.` | — |
| **2** | `S → (.S)S` | `S → .(S)S`, `S → .` |
| **3** | `S → (S.)S` | — |
| **4** | `S → (S).S` | `S → .(S)S`, `S → .` |
| **5** | `S → (S)S.` | — |

**상태 천이:**

```
상태 0 --S--> 상태 1
상태 0 --(--> 상태 2
상태 2 --(--> 상태 2  (루프: 중첩 괄호 처리)
상태 2 --S--> 상태 3
상태 3 --)--> 상태 4
상태 4 --(--> 상태 2
상태 4 --S--> 상태 5
```

---

#### 예 10 — `E' → E`, `E → E + n | n`

**DFA 상태 목록:**

| 상태 | 포함 items |
|------|-----------|
| **0** | `E' → .E` (kernel), `E → .E+n`, `E → .n` (closure) |
| **1** | `E' → E.`, `E → E.+n` |
| **2** | `E → n.` |
| **3** | `E → E+.n` |
| **4** | `E → E+n.` |

**상태 천이:**

```
상태 0 --E--> 상태 1
상태 0 --n--> 상태 2
상태 1 --+--> 상태 3
상태 3 --n--> 상태 4
```

**Kernel vs Closure 구분 (상태 0):**

```
[kernel]  E' → .E      ← 출발점. 상태 천이의 주체
[closure] E  → .E+n    ← E가 nonterminal이므로 CLOSURE로 추가
[closure] E  → .n      ← 마찬가지
```

---

## 8. LR(0) 파싱 알고리즘

### 8.1 알고리즘 설명

파싱 알고리즘 = **DFA의 상태 추적**

- 스택에는 **(symbol, state 번호)** 쌍을 함께 저장
- **현재 상태는 항상 스택의 top에 위치**
- LR(0)은 **현재 상태만 보고** shift/reduce를 결정 (lookahead 없음)

---

#### Case 1: Shift Action

현재 상태 n이 `A → α . X β` 형태의 item을 갖고, **X가 terminal**인 경우:

```
→ shift action: 입력 lookahead X를 스택에 push
→ 다음 상태 m = GOTO(n, X) (A → αX.β를 포함하는 상태)
```

```
Parsing Stack: $ ... n           Input: X ...
After shift:   $ ... n X m       Input: ...
```

---

#### Case 2: Reduce Action

현재 상태 n이 `A → α .` (complete item)을 갖는 경우:

```
→ reduce action: reduction by A → α
→ 스택에서 α에 해당하는 기호와 상태 번호를 pop
→ nonterminal A를 스택에 push
→ GOTO(이전 상태, A)로 이동한 상태 번호를 push
```

**Accept 조건:**  
`S' → S.` 이고 입력 버퍼가 비어 있으면 (lookahead = `$`) → **accept**

---

### 8.2 Stepwise Execution 예시

**문법:** `A' → A`, `A → (A) | a`

**DFA:**

```
상태 0: A' → .A, A → .(A), A → .a  (kernel: A' → .A)
상태 1: A' → A.
상태 2: A → a.
상태 3: A → (.A), A → .(A), A → .a  (kernel: A → (.A))
상태 4: A → (A.)
상태 5: A → (A).
```

**입력: `((a))$`**

| Step | Parsing Stack | Input | Action |
|------|---------------|-------|--------|
| 1 | `$0` | `((a))$` | shift → state 3 |
| 2 | `$0(3` | `(a))$` | shift → state 3 |
| 3 | `$0(3(3` | `a))$` | shift → state 2 |
| 4 | `$0(3(3a2` | `))$` | reduce A → a (pop a2, push A) |
| 5 | `$0(3(3A4` | `))$` | shift `)` → state 5 |
| 6 | `$0(3(3A4)5` | `)$` | reduce A → (A) (pop (3A4)5, push A) |
| 7 | `$0(3A4` | `)$` | shift `)` → state 5 |
| 8 | `$0(3A4)5` | `$` | reduce A → (A) (pop (3A4)5, push A) |
| 9 | `$0A1` | `$` | **accept** |

**reduce A → (A) 수행 시 동작 상세 (step 6):**
- pop: `)5`, `A4`, `(3` → 스택 top에 상태 0 노출
- push: `A` (nonterminal)
- GOTO(0, A) = 1 → push 상태 1 ... 이 아니라 GOTO(3, A) = 4 → push 상태 4
  - 스택 top이 3이므로 GOTO(3, A) = 4

---

## 9. LR(0) Grammar 조건 및 Conflict

### 9.1 LR(0) 문법 조건

> **LR(0) Grammar**: 어떤 상태도 conflict가 없어야 한다.

각 상태는 다음 중 하나여야 한다:

| 조건 | 내용 |
|------|------|
| **Shift state** | shift item(dot이 맨 끝이 아닌 item)만 포함 |
| **Reduce state** | **단 하나의** complete item만 포함 |

> shift면 shift, reduce면 reduce — **하나로 통일된 item만** 있어야 함

---

### 9.2 Conflict 종류

#### Shift-Reduce Conflict

한 상태에 shift item과 reduce item이 **동시에** 존재:

```
상태 내에:
  A → α .        ← reduce item (reduce A → α)
  B → β . X γ    ← shift item (shift X)
```

→ shift해야 할지 reduce해야 할지 결정 불가

---

#### Reduce-Reduce Conflict

한 상태에 complete item이 **두 개 이상** 존재:

```
상태 내에:
  A → α .    ← A → α로 reduce?
  B → β .    ← B → β로 reduce?
```

→ 어떤 production으로 reduce해야 할지 결정 불가

---

**LR(0)가 아닌 문법 예시 (shift-reduce conflict):**

```
S' → S
S  → (S)S | ε
```

→ 상태 0, 2, 4에서 `S → .` (reduce) 와 `S → .(S)S` (shift) 가 **동시에 존재** → conflict!

> SLR(1)은 이 conflict를 FOLLOW 집합을 이용해 해결한다.

---

### 9.3 예 11: LR(0) 문법 파싱 전체 예시

**문법:** `A' → A`, `A → (A) | a`

이 문법은 **LR(0) Grammar**이다. (conflict 없음)

#### DFA 및 Parsing Table

| State | Action | Rule | `(` | `a` | `)` | **A** (Goto) |
|-------|--------|------|-----|-----|-----|-------------|
| 0 | shift | | 3 | 2 | | 1 |
| 1 | reduce | A' → A | | | | |
| 2 | reduce | A → a | | | | |
| 3 | shift | | 3 | 2 | | 4 |
| 4 | shift | | | | 5 | |
| 5 | reduce | A → (A) | | | | |

#### DFA 상태 설명

```
State 0 (start):
  [kernel]  A' → . A
  [closure] A  → . (A)
  [closure] A  → . a

State 1: A' → A.     ← accept state

State 2: A → a.      ← reduce state (A → a)

State 3:
  [kernel]  A → ( . A)
  [closure] A → . (A)
  [closure] A → . a

State 4: A → (A . )  ← shift ) 필요

State 5: A → (A) .   ← reduce state (A → (A))
```

#### Parsing 과정 `((a))$`

| Step | Parsing Stack | Input | Action |
|------|---------------|-------|--------|
| 1 | `$0` | `((a))$` | shift → state 3 |
| 2 | `$0(3` | `(a))$` | shift → state 3 |
| 3 | `$0(3(3` | `a))$` | shift → state 2 |
| 4 | `$0(3(3a2` | `))$` | reduce A → a |
| 5 | `$0(3(3A4` | `))$` | shift `)` → state 5 |
| 6 | `$0(3(3A4)5` | `)$` | reduce A → (A) |
| 7 | `$0(3A4` | `)$` | shift `)` → state 5 |
| 8 | `$0(3A4)5` | `$` | reduce A → (A) |
| 9 | `$0A1` | `$` | **accept** |

