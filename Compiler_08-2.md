# Chapter 7 - 상향식 구문분석 (Bottom-up Parsing) Part II

---

## 1. SLR(1) Parsing

### 개요

**SLR(1) = Simple LR(1)**

LR(0) Parsing의 업그레이드 버전. LR(0)의 DFA를 그대로 사용하되, **다음 입력 토큰(lookahead)** 을 참고해서 액션을 결정한다.

| 항목 | LR(0) | SLR(1) |
|------|-------|--------|
| DFA 기반 | LR(0) items | LR(0) items (동일) |
| Shift 결정 | transition 존재 여부만 확인 | 입력 토큰이 실제로 X인지 확인 |
| Reduce 결정 | complete item이 있으면 무조건 reduce | 입력 토큰 ∈ **FOLLOW(A)** 인 경우에만 reduce |
| 파워 | 약함 | **LR(0)에 비해 엄청난 power-up!** |

### SLR(1) Parsing 알고리즘

#### Case 1: Shift Action

현재 상태 `n`에 **shift item** `A → α . X β` 가 있을 때:

- `X`가 terminal이고, 입력에서 읽어 온 token도 `X`이면 → **shift action**
- `X`를 stack에 push하고, `A → αX . β`를 포함하는 상태 `m`을 push

```
Parsing Stack        Input
$ ... n     X ... $
$ ... n X m ... $
```

```
A → α . X   →[X]→   A → αX .
     n                    m
```

#### Case 2: Reduce Action

현재 상태 `n`에 **complete item** `A → α .` 가 있을 때:

- 입력에서 읽어 온 token이 **FOLLOW(A)** 에 속하면 → **reduce action** (by `A → α`)
- stack에서 `α`와 상태번호를 함께 pop
- nonterminal `A`를 stack에 push
- `B → β . A γ` (reduction 직후의 item)를 포함하는 상태 번호 push

```
B → β . A   →[A]→   B → βA .
A → α .
     n                   m
```

#### Case 3: Accept

현재 상태에 complete item `S' → S .` 가 있고, 입력 버퍼가 비어 있으면(`LOOKAHEAD = '$'`)

→ **Parsing 성공! (The input sentence is syntactically correct)**

---

### SLR(1) Grammar 조건

문법이 SLR(1)이 되려면 어떤 상태 `s`에서도 다음 두 조건을 만족해야 한다:

#### 조건 1: No shift-reduce conflict

어떤 상태 `s`에서 shift item `A → α . X β` (`X`는 terminal)가 있을 때,  
같은 상태에 complete item `B → γ .` 가 존재하면서 **X ∈ FOLLOW(B)** 이면 안 된다.

#### 조건 2: No reduce-reduce conflict

어떤 상태 `s`에 두 개의 complete item `A → α .` 와 `B → β .` 가 있을 때,  
**FOLLOW(A) ∩ FOLLOW(B) = ∅** 이어야 한다.

---

### LR(0)과 SLR(1)의 차이 예시

#### 예시 1: LR(0)이면서 SLR(1)인 문법

```
A' → A
A  → (A) | a
```

이 문법의 DFA는 conflict 없이 LR(0)으로 파싱 가능 → **LR(0) ✓**

#### 예시 2: LR(0)은 아니지만 SLR(1)인 문법

```
E' → E
E  → E + n | n
```

**상태 1의 DFA:**
```
[E' → E . ]    ← complete item (reduce)
[E  → E . + n] ← shift item (shift on '+')
```

- LR(0): 상태 1에 reduce item과 shift item이 공존 → **shift-reduce conflict → LR(0) ✗**
- SLR(1): FOLLOW(E') = {$}, shift는 '+' 일 때만
  - '+' 토큰이면 → shift (E → E.+n 이 있으니까)
  - '$' 토큰이면 → reduce by E' → E (FOLLOW(E') = {$})
  - conflict 없음 → **SLR(1) ✓**

---

## 2. SLR(1) Parsing Table 구성

### 테이블 구조

| 상태 | ACTION 표 (VT ∪ {$}) | GOTO 표 (VN) |
|------|----------------------|--------------|
| 0    | shift                | 상태 번호     |
| 1    | reduce               | 상태 번호     |
| 2    | accept               |               |
| :    | error                |               |

### 4가지 Semantic Action

1. **Shift**: `ACTION[Sm, ai] = shift S` → 다음 상태 S로 이동
2. **Reduce**: `ACTION[Sm, ai] = reduce by A → α`
3. **Accept**: `ACTION[Sm, ai] = accept`
4. **Syntax error**: `ACTION[Sm, ai] = error` (항목이 비어 있음)

### SLR(1) Parsing Table 구성 규칙

문법의 DFA of LR(0) items를 구성한 후:

1. 상태 `i`에 `A → α . a β` (shift item, `a`는 terminal)가 있고 `GOTO(i, a) = j` 이면
   - `ACTION[i, a] = shift j`

2. 상태 `i`에 `A → α .` (complete item)가 있으면
   - FOLLOW(A)에 속하는 모든 terminal `a`에 대해
   - `ACTION[i, a] = reduce by A → α`

3. 상태 `i`에 `S' → S .` 가 있으면
   - `ACTION[i, $] = accept`

4. 상태 `i`에서 nonterminal `A`에 대해 `GOTO(i, A) = j` 이면
   - `GOTO[i, A] = j`

5. 나머지 항목은 모두 **error**

---

## 3. SLR(1) 예제

### 예제 10: `E' → E`, `E → E + n | n`

#### DFA of LR(0) items

```
→ [E' → .E      ]  --E-->  [E' → E.   ]  (상태 1)
   [E  → .E + n  ]          [E  → E.+n ]
   [E  → .n      ]
   (상태 0)
        |
        n
        ↓
   [E → n.]  (상태 2)
   
   상태 1 --+-->  [E → E+.n ]  (상태 3)
                       |
                       n
                       ↓
                  [E → E+n.] (상태 4)
```

#### FOLLOW 집합

- FOLLOW(E') = {$}
- FOLLOW(E) = {+, $}

#### SLR(1) Parsing Table

| State | n  | +  | $      | E |
|-------|----|----|--------|---|
| 0     | s2 |    |        | 1 |
| 1     |    | s3 | accept |   |
| 2     |    | r(E→n) | r(E→n) | |
| 3     | s4 |    |        |   |
| 4     |    | r(E→E+n) | r(E→E+n) | |

> 💡 **포인트**: 상태 2에서 reduce action을 FOLLOW(E) = {+, $} 에 해당하는 열에만 채운다.  
> 상태 1에서 shift(+)와 accept($)가 서로 다른 열에 있으므로 conflict 없음 → **SLR(1) ✓**

#### Parsing Actions: `n+n+n$` 파싱

| 단계 | Parsing Stack | Input    | Action           |
|------|---------------|----------|------------------|
| 1    | $0            | n+n+n$   | shift 2          |
| 2    | $0n2          | +n+n$    | reduce by E→n    |
| 3    | $0E1          | +n+n$    | shift 3          |
| 4    | $0E1+3        | n+n$     | shift 4          |
| 5    | $0E1+3n4      | +n$      | reduce E→E+n     |
| 6    | $0E1          | +n$      | shift 3          |
| 7    | $0E1+3        | n$       | shift 4          |
| 8    | $0E1+3n4      | $        | reduce by E→E+n  |
| 9    | $0E1          | $        | accept           |

---

### 예제 7: `S' → S`, `S → (S)S | ε`

#### FOLLOW 집합

- FOLLOW(S') = {$}
- FOLLOW(S) = {), $}

#### SLR(1) Parsing Table

| State | (   | )           | $           | S |
|-------|-----|-------------|-------------|---|
| 0     | s2  | r(S→ε)      | r(S→ε)      | 1 |
| 1     |     |             | accept      |   |
| 2     | s2  | r(S→ε)      | r(S→ε)      | 3 |
| 3     |     | s4          |             |   |
| 4     | s2  | r(S→ε)      | r(S→ε)      | 5 |
| 5     |     | r(S→(S)S)   | r(S→(S)S)   |   |

> 💡 `S → ε` (ε-production)의 경우: **모든 상태에서** FOLLOW(S) = {), $}에 해당하는 열에 reduce를 채운다.

#### Parsing Actions: `()()$` 파싱

| 단계 | Parsing Stack       | Input  | Action        |
|------|---------------------|--------|---------------|
| 1    | $0                  | ()()$  | shift 2       |
| 2    | $0(2               | )()$   | reduce S→ε    |
| 3    | $0(2S3             | )()$   | shift 4       |
| 4    | $0(2S3)4           | ()$    | shift 2       |
| 5    | $0(2S3)4(2         | )$     | reduce S→ε    |
| 6    | $0(2S3)4(2S3       | )$     | shift 4       |
| 7    | $0(2S3)4(2S3)4     | $      | reduce S→ε    |
| 8    | $0(2S3)4(2S3)4S5   | $      | reduce S→(S)S |
| 9    | $0(2S3)4S5         | $      | reduce S→(S)S |
| 10   | $0S1               | $      | accept        |

---

### 예제 12: SLR(1) - 산술식 문법

```
0. S' → E
1. E  → E + T
2. E  → T
3. T  → T * F
4. T  → F
5. F  → (E)
6. F  → id
```

#### FOLLOW 집합

- FOLLOW(E) = {$, +, )}
- FOLLOW(T) = {*, +, ), $}
- FOLLOW(F) = {*, +, ), $}

#### LR(0) items - C₀ (Canonical Collection)

```
I0 = CLOSURE([S' → •E])
   = { [S' → •E],
       [E  → •E+T],
       [E  → •T],
       [T  → •T*F],
       [T  → •F],
       [F  → •(E)],
       [F  → •id] }

GOTO(I0, E)  = I1  = { [S' → E•], [E → E•+T] }
GOTO(I0, T)  = I2  = { [E → T•], [T → T•*F] }
GOTO(I0, F)  = I3  = { [T → F•] }
GOTO(I0, ()  = I4  = { [F → (•E)], [E → •E+T], [E → •T],
                        [T → •T*F], [T → •F], [F → •(E)], [F → •id] }
GOTO(I0, id) = I5  = { [F → id•] }

GOTO(I1, +)  = I6  = { [E → E+•T], [T → •T*F], [T → •F],
                        [F → •(E)], [F → •id] }
GOTO(I2, *)  = I7  = { [T → T*•F], [F → •(E)], [F → •id] }
GOTO(I4, E)  = I8  = { [F → (E•)], [E → E•+T] }
GOTO(I6, T)  = I9  = { [E → E+T•], [T → T•*F] }
GOTO(I7, F)  = I10 = { [T → T*F•] }
GOTO(I8, ))  = I11 = { [F → (E)•] }
```

#### SLR(1) Parsing Table

| 상태 | id | + | * | ( | ) | $ | E | T | F |
|------|----|---|---|---|---|---|---|---|---|
| 0    | s5 |   |   | s4 |  |   | 1 | 2 | 3 |
| 1    |    | s6 |  |   |   | acc |  |  |   |
| 2    |    | r2 | s7 |  | r2 | r2 |  |  |   |
| 3    |    | r4 | r4 |  | r4 | r4 |  |  |   |
| 4    | s5 |   |   | s4 |   |   | 8 | 2 | 3 |
| 5    |    | r6 | r6 |  | r6 | r6 |  |  |   |
| 6    | s5 |   |   | s4 |   |   |   | 9 | 3 |
| 7    | s5 |   |   | s4 |   |   |   |   | 10|
| 8    |    | s6 |   |   | s11 |  |   |  |   |
| 9    |    | r1 | s7 |  | r1 | r1 |  |  |   |
| 10   |    | r3 | r3 |  | r3 | r3 |  |  |   |
| 11   |    | r5 | r5 |  | r5 | r5 |  |  |   |

#### Parsing Actions: `id * (id * id)$` 파싱

| 단계 | 스택           | 입력 기호      | 구문 분석 내용 |
|------|----------------|----------------|---------------|
| 0    | 0              | id*(id*id)$    | 이동 5        |
| 1    | 0id5           | *(id*id)$      | 감축 6        |
| 2    | 0F             | *(id*id)$      | GOTO 3        |
| 3    | 0F3            | *(id*id)$      | 감축 4        |
| 4    | 0T             | *(id*id)$      | GOTO 2        |
| 5    | 0T2            | *(id*id)$      | 이동 7        |
| 6    | 0T2*7          | (id*id)$       | 이동 4        |
| 7    | 0T2*7(4        | id*id)$        | 이동 5        |
| 8    | 0T2*7(4id5     | *id)$          | 감축 6        |
| 9    | 0T2*7(4F       | *id)$          | GOTO 3        |
| 10   | 0T2*7(4F3      | *id)$          | 감축 4        |
| 11   | 0T2*7(4T       | *id)$          | GOTO 2        |
| 12   | 0T2*7(4T2      | *id)$          | 이동 7        |
| 13   | 0T2*7(4T2*7    | id)$           | 이동 5        |
| 14   | 0T2*7(4T2*7id5 | )$             | 감축 6        |
| 15   | 0T2*7(4T2*7F   | )$             | GOTO 10       |
| 16   | 0T2*7(4T2*7F10 | )$             | 감축 3        |
| 17   | 0T2*7(4T       | )$             | GOTO 2        |
| 18   | 0T2*7(4T2      | )$             | 감축 2        |
| 19   | 0T2*7(4E       | )$             | GOTO 8        |
| 20   | 0T2*7(4E8      | )$             | 이동 11       |
| 21   | 0T2*7(4E8)11   | $              | 감축 5        |
| 22   | 0T2*7F         | $              | GOTO 10       |
| 23   | 0T2*7F10       | $              | 감축 3        |
| 24   | 0T             | $              | GOTO 2        |
| 25   | 0T2            | $              | 감축 2        |
| 26   | 0E             | $              | GOTO 1        |
| 27   | 0E1            | $              | 수락          |

---

### Disambiguating Rules (모호성 해결)

#### Shift-Reduce Conflict 해결

**원칙: 항상 shift를 reduce보다 우선한다 (prefer shift over reduce)**

→ **dangling else** 문제에 적용:

```
I → if S | if S else S
```

상태에서 `I → if S .` (reduce) 와 `I → if S . else S` (shift on else) 가 공존할 때:
- 다음 토큰이 `else`이면 → **shift 선택**
- 결과: `else`는 **가장 가까운 if**와 짝을 이룸 (most closely nested rule)

```
if (c1) if (c2) stmt1 else stmt2
→ if (c1) { if (c2) stmt1 else stmt2 }  ← shift 우선
→ (if (c2) stmt1과 else stmt2가 짝을 이룸)
```

#### Reduce-Reduce Conflict 해결

대부분 **문법 자체에 오류**가 있는 경우 발생. 문법을 수정해야 한다.

#### 예제 13: Dangling-else 파싱 테이블

문법:
```
S → I | other
I → if S | if S else S
```

파싱 테이블 (규칙 번호: (1)S→I, (2)S→other, (3)I→if S, (4)I→if S else S):

| State | if | else | other | $  | S | I |
|-------|----|------|-------|----|---|---|
| 0     | s4 |      | s3    |    | 1 | 2 |
| 1     |    |      |       | acc |  |   |
| 2     |    | r1   |       | r1  |  |   |
| 3     |    | r2   |       | r2  |  |   |
| 4     | s4 |      | s3    |    | 5 | 2 |
| **5** |    | **s6** |     | **r3** | | |
| 6     | s4 |      | s3    |    | 7 | 2 |
| 7     |    | r4   |       | r4  |  |   |

> 💡 **상태 5**: `I → if S .` (complete, FOLLOW(I) = {else, $}) 와 `I → if S . else S` (shift on else)  
> → else가 들어오면 **shift 우선** → conflict 해결

---

## 4. SLR(1)의 한계 (Limits)

### "Phony" Problem (허구 충돌)

**실제 입력에서 절대 나타날 수 없는 상황** 때문에 conflict가 발생하는 문제.  
SLR(1)의 약점: FOLLOW 집합이 너무 크다.

### 예제: Reduce-Reduce Conflict

```
S → id | V := E
V → id
E → V | n
```

DFA 구성 시 `id`를 읽은 후의 상태:
```
상태 2: S → id .    (FOL(S) = {$})
        V → id .    (FOL(V) = {:=, $})
```

- `S → id .` 는 `$` 일 때 reduce
- `V → id .` 는 `:=` 또는 `$` 일 때 reduce

→ `$`에 대해 두 reduce action 충돌 → **reduce-reduce conflict!**

**실제로는?** 이 문법에서 `V`가 등장하는 문장 구조는 `V := E` 뿐이다.  
즉, `V` 다음에 `$`가 나타나는 right sentential form은 존재하지 않는다.  
→ FOLLOW(V)에 `$`가 포함된 건 **"가짜 lookahead"** → SLR(1)의 한계

### 예제 14: Shift-Reduce Conflict

```
S → L = R | R
L → *R | id
R → L
```

DFA의 상태 I₂:
```
S → L . = R    (shift item, lookahead = {=})
R → L .        (complete item, FOL(R) = {=, $})
```

- `=` 에 대해 shift(S → L.=R) 와 reduce(R → L., FOLLOW(R) = {=,$}) 가 충돌
- 실제로 **R 다음에 `=`이 나타나는 right sentential form은 없다!**
  - `S => L=R`, `S => R => L => *R`, `S => R => L => id` ...
  - 어떤 유도에서도 `R =` 형태가 나타나지 않음
- 하지만 SLR(1)은 FOLLOW 집합 전체를 보기 때문에 `=`가 FOLLOW(R)에 포함되어 conflict 발생

**→ 이 문제를 해결하기 위해 LR(1)이 필요!**

---

## 5. LR Parsing 종류 비교

```
LR(1) ⊃ LALR(1) ⊃ SLR(1)
```

| 방법 | 파싱 테이블 크기 | 인식 능력 | 비고 |
|------|----------------|----------|------|
| SLR(1) | 작음 | 낮음 | LR(0) DFA + FOLLOW 집합 |
| LALR(1) | 중간 | 중간 | Yacc에서 사용 |
| CLR(1) = LR(1) | 큼 | 높음 | LR(1) items DFA |

또한:
- `LL(1) ⊄ SLR(1)` 이지만
- `LL(1) ⊂ LR(1)`

---

## 6. General LR(1) / CLR(1) Parsing

### 핵심 아이디어

**LR(0) item + lookahead = LR(1) item**

SLR(1)은 DFA를 다 구성한 뒤에 lookahead를 사용했지만,  
**LR(1)은 DFA를 구성할 때부터 lookahead를 item에 포함시킨다.**

→ 실제 그 상태에서 등장 가능한 토큰만 lookahead로 사용  
→ "가짜 충돌" 방지

### LR(1) Item

```
[A → α . β, a]
```

- `A → α . β`: LR(0) item
- `a`: lookahead (a ∈ VT ∪ {$})
- 의미: **다음 입력 토큰이 `a`일 때 이 item을 사용할 수 있다**

### Finite Automata of LR(1) Items

#### 시작 상태

```
Start State: [S' → . S, $]
```

#### Transition 규칙

**① Kernel item → kernel item (shift)**

```
[A → α . X β, a]  →[X]→  [A → αX . β, a]
```
(X는 terminal 또는 nonterminal)

**② Kernel item → closure item (ε-transition)**

```
[A → α . B γ, a]  →[ε]→  [B → . β, b]
```
- `B`는 nonterminal
- `b ∈ FIRST(γa)` (γ 다음에 a가 오는 경우의 FIRST)

> 💡 **핵심**: closure item의 lookahead = `FIRST(γa)`  
> - `γ`가 ε이면 → `FIRST(a)` = `{a}`  
> - `γ`가 terminal `t`이면 → `FIRST(ta)` = `{t}`

### LR(1) Grammar 조건

어떤 상태 `s`에서도 다음 두 조건 만족:

1. **No shift-reduce conflict**: 어떤 shift item `[A → α . X, a]`에 대해, 같은 상태에 `[B → γ ., X]` 형태의 complete item이 없어야 한다.

2. **No reduce-reduce conflict**: 같은 상태에 `[A → α ., a]` 와 `[B → β ., a]` 가 동시에 존재하면 안 된다.

---

### General LR(1) Parsing Algorithm

#### Case 1: Shift

상태 `n`에 `[A → α . X β, a]` 가 있을 때:
- `X`가 terminal이고, 입력 token도 `X`이면 → shift

#### Case 2: Reduce

상태 `n`에 complete item `[A → α ., a]` 가 있을 때:
- 입력 token이 **`a`** 이면 → reduce by `A → α`  
  (SLR의 FOLLOW(A) 전체가 아닌, **이 item의 lookahead `a`** 만 확인)

#### Case 3: Accept

상태에 `[S' → S ., $]` 가 있고 입력이 `$`이면 → accept

---

### 예제 15: CLR(1) parsing - `S → CC`, `C → cC | d`

```
0. S' → S
1. S  → CC
2. C  → cC
3. C  → d
```

#### Canonical Collection of LR(1) items (일부)

```
I0 = { [S' → •S, $], [S → •CC, $], [C → •cC, c/d], [C → •d, c/d] }
I1 = GOTO(I0, S) = { [S' → S•, $] }
I2 = GOTO(I0, C) = { [S → C•C, $], [C → •cC, $], [C → •d, $] }
I3 = GOTO(I0, c) = { [C → c•C, c/d], [C → •cC, c/d], [C → •d, c/d] }
I4 = GOTO(I0, d) = { [C → d•, c/d] }
```

> 💡 `I0`에서 `C → •cC`의 lookahead가 `c/d`인 이유:  
> `[S → •CC, $]`에서 ε-transition 으로 `C` 유도  
> `B=C, γ=C, a=$` → `FIRST(C$)` = `FIRST(C)` = `{c, d}` → lookahead = `c/d`

#### CLR(1) Parsing Table

| 상태 | c   | d   | $   | S | C |
|------|-----|-----|-----|---|---|
| 0    | s3  | s4  |     | 1 | 2 |
| 1    |     |     | acc |   |   |
| 2    | s6  | s7  |     |   | 5 |
| 3    | s3  | s4  |     |   | 8 |
| 4    | r3  | r3  |     |   |   |
| 5    |     |     | r1  |   |   |
| 6    | s6  | s7  |     |   | 9 |
| 7    |     |     | r3  |   |   |
| 8    | r2  | r2  |     |   |   |
| 9    |     |     | r2  |   |   |

#### Parsing Actions: `ccdd$`

| 단계 | 스택        | 입력 기호 | 구문 분석 내용 |
|------|-------------|-----------|---------------|
| 0    | 0           | ccdd$     | 이동 3        |
| 1    | 0c3         | cdd$      | 이동 3        |
| 2    | 0c3c3       | dd$       | 이동 4        |
| 3    | 0c3c3d4     | d$        | 감축 3        |
| 4    | 0c3c3C      | d$        | GOTO 8        |
| 5    | 0c3c3C8     | d$        | 감축 2        |
| 6    | 0c3C        | d$        | GOTO 8        |
| 7    | 0c3C8       | d$        | 감축 2        |
| 8    | 0C          | d$        | GOTO 2        |
| 9    | 0C2         | d$        | 이동 7        |
| 10   | 0C2d7       | $         | 감축 3        |
| 11   | 0C2C        | $         | GOTO 5        |
| 12   | 0C2C5       | $         | 감축 1        |
| 13   | 0S          | $         | GOTO 1        |
| 14   | 0S1         | $         | 수락          |

---

### 예제 17: CLR(1)이 SLR(1) conflict를 해결하는 사례

```
S → id | V := E
V → id
E → V | n
```

LR(1) DFA의 상태 2 (`id` 입력 후):
```
[S → id ., $]     ← lookahead = $만
[V → id ., :=]    ← lookahead = :=만
```

- `$` 입력 시 → `S → id` 로 reduce  
- `:=` 입력 시 → `V → id` 로 reduce  
- **conflict 없음!**

SLR(1)에서는 `FOLLOW(V) = {:=, $}` 이기 때문에 `$`에서 두 rule이 충돌했지만,  
LR(1)에서는 lookahead가 각 item에 **정확하게** 붙어 있어 해결된다.

---

### 예제 18: CLR(1) - `S → L=R | R`

```
0. S' → S
1. S  → L = R
2. S  → R
3. L  → *R
4. L  → id
5. R  → L
```

예제 14에서 SLR(1)이 실패했던 shift-reduce conflict를 CLR(1)에서는 해결한다.

**상태 I₂ (CLR(1)):**
```
[S → L . = R, $]   ← shift on '='
[R → L .,    $]    ← reduce, lookahead = $만
```

- `=` 입력 → shift (S → L.=R)
- `$` 입력 → reduce (R → L.)
- **conflict 없음!** (SLR(1)에서는 FOLLOW(R) = {=, $}라서 `=` 충돌)

#### CLR(1) Parsing Table (요약)

| 상태 | = | * | id | $ | S | R | L |
|------|---|---|----|---|---|---|---|
| 0    |   | s4 | s5 |   | 1 | 3 | 2 |
| 1    |   |   |    | acc |  |  |   |
| 2    | s6 |  |    | r5 |  |  |   |
| 3    |   |   |    | r2 |  |  |   |
| 4    |   | s4 | s5 |   |   | 7 | 8 |
| 5    | r4 |  |    | r4 |  |  |   |
| 6    |   | s11 | s12 |  |  | 9 | 10 |
| 7    | r3 |  |    | r3 |  |  |   |
| 8    | r5 |  |    | r3 |  |  |   |
| ...  |   |   |    |   |  |  |   |

---

## 7. LALR(1) Parsing

### 왜 LALR(1)이 필요한가?

LR(1)은 정확하지만 **상태 수가 너무 많다**.

LR(0) DFA가 상태 n개라면, LR(1) DFA는 거의 **2n개** 가까이 될 수 있다.  
(같은 LR(0) core에 lookahead만 다른 상태들이 별도로 생성되기 때문)

**해결책**: LR(0) core(= LR(0) item 부분)가 같은 상태들을 **합친다 (merge)**

### LALR(1) 아이디어

```
같은 core를 갖는 LR(1) 상태들을 하나로 merge
→ lookahead를 합집합으로 취함
```

**예시:**
```
I4 = { [C → d•, c/d] }
I7 = { [C → d•, $]   }

→ I47 = { [C → d•, c/d/$] }  ← lookahead가 합집합
```

### LALR(1) vs CLR(1) 상태 수 비교

| 문법 | SLR/LR(0) states | LALR(1) states | CLR(1) states |
|------|-----------------|---------------|--------------|
| `A → (A)\|a` | 6 | 6 | 10 |
| `S → CC, C → cC\|d` | - | 6 | 10 |

> LALR(1)의 상태 수 = LR(0)의 상태 수 (core가 같은 것을 합치므로)

### CLR table → LALR table 변환

**예제 20: `S → CC`, `C → cC | d`**

CLR 상태:
- I3 (lookahead c/d) + I6 (lookahead $) → **I36** = lookahead c/d/$
- I4 (lookahead c/d) + I7 (lookahead $) → **I47** = lookahead c/d/$
- I8 (lookahead c/d) + I9 (lookahead $) → **I89** = lookahead c/d/$

**CLR Parsing Table:**

| 상태 | c | d | $ | S | C |
|------|---|---|---|---|---|
| 0  | s3 | s4 |     | 1 | 2 |
| 1  |    |    | acc |   |   |
| 2  | s6 | s7 |     |   | 5 |
| 3  | s3 | s4 |     |   | 8 |
| 4  | r3 | r3 |     |   |   |
| 5  |    |    | r1  |   |   |
| 6  | s6 | s7 |     |   | 9 |
| 7  |    |    | r3  |   |   |
| 8  | r2 | r2 |     |   |   |
| 9  |    |    | r2  |   |   |

**LALR Parsing Table (merge 후):**

| 상태 | c    | d    | $    | S | C  |
|------|------|------|------|---|----|
| 0    | s36  | s47  |      | 1 | 2  |
| 1    |      |      | acc  |   |    |
| 2    | s36  | s47  |      |   | 5  |
| 36   | s36  | s47  |      |   | 89 |
| 47   | r3   | r3   | r3   |   |    |
| 5    |      |      | r1   |   |    |
| 89   | r2   | r2   | r2   |   |    |

> 💡 **CLR table에서 상태를 합치는 방법:**  
> 1. core(LR(0) item 부분)가 같은 상태 찾기  
> 2. lookahead를 합집합으로 합치기  
> 3. GOTO 열도 merge된 상태 번호로 갱신  
> 4. merge 후 새로운 reduce-reduce conflict가 생기면 → LALR(1) 불가

### LALR(1)에서 새로운 conflict가 생길 수 있는가?

- **Shift-reduce conflict**: merge 전에 conflict 없으면 → merge 후에도 없음
- **Reduce-reduce conflict**: ⚠️ **생길 수 있음!**
  - 두 상태의 lookahead 합집합이 겹치게 되면 conflict 발생
  - 이 경우 그 문법은 LALR(1)이 아님

---

## 8. LR(1) vs LALR(1) 최종 정리

### LR Parsing 방법 총정리

| 방법 | 기반 | Lookahead 결정 방법 | 상태 수 | 인식 능력 |
|------|------|-------------------|---------|----------|
| **LR(0)** | LR(0) DFA | 없음 | 작음 | 가장 낮음 |
| **SLR(1)** | LR(0) DFA | FOLLOW(A) (전역적) | 작음 | 낮음 |
| **LALR(1)** | LR(1) merge | 각 item의 실제 lookahead (merge) | 중간 | 높음 |
| **CLR(1)** | LR(1) DFA | 각 item의 실제 lookahead | 큼 | 가장 높음 |

### 포함 관계

```
SLR(1) ⊂ LALR(1) ⊂ LR(1)

LL(1) ⊄ SLR(1),  LL(1) ⊂ LR(1)
```

### 핵심 차이점 요약

```
SLR(1)  : reduce 여부를 FOLLOW(A) 로 결정 → 가짜 lookahead 문제
LR(1)   : reduce 여부를 item의 정확한 lookahead로 결정 → 상태 수 증가
LALR(1) : LR(1)의 같은 core 상태를 merge → LR(0)과 동일한 상태 수, LR(1) 수준의 정확도
```

### 실무에서는?

- **Yacc, Bison**: LALR(1) 사용
- LALR(1)은 실제 프로그래밍 언어(C, Java 등) 문법을 대부분 처리 가능
- CLR(1)은 파싱 테이블이 너무 커서 실용적이지 않음

---

## 9. 연습 문제

### 문제: LR(0) DFA 구성

다음 문법의 LR(0) items DFA를 구성하라:

```
P → b D ; S e
D → d ; D | d
S → s ; S | s
```

**힌트:**
1. Augmented grammar: `P' → P` 추가
2. Start item: `[P' → . P]`
3. CLOSURE와 GOTO를 반복하여 모든 상태 구성

