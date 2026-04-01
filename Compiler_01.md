# 컴파일러 설계 (Compiler Design)

## 1. 기초 개념

**컴파일러 (Compiler)**
> 특정 고급 프로그래밍 언어로 작성된 프로그램을 특정 대상 컴퓨터의 실행 가능한 코드로 번역하는 컴퓨터 프로그램

**컴파일 (Compile)**
> 사람이 쓴 코드(소스)를 컴퓨터가 이해할 수 있는 언어(기계어)로 번역하는 과정
> 의미(semantics)는 보존하고 표현(syntax)만 바꿈

**프로그래밍 언어 (Programming Language)**
> 알고리즘과 계산을 형식적이고 모호하지 않게 표현하여 컴퓨터가 실행할 수 있도록 하는 인공 언어

**인간 언어 (Human Language)**
> 표기 방식과 구조가 언어마다 다양한 자연 발생 언어
> 모호성이 존재하기 때문에 컴퓨터용으로는 형식적인 프로그래밍 언어가 필요

---

## 2. 컴파일러 구조

### 추상적 구조 (Abstract View)

```
Source Program
      ↓
 [Front-End]   ← language-dependent (언어에 종속)
      ↓
 IC (Intermediate Code)
      ↓
 [Back-End]    ← machine-dependent (기계에 종속)
      ↓
Target Program
```

### 상세 구조 (Detailed View)

```
Source Program
      ↓
┌──────────────────────────────────────┐
│              Front-End               │  Analysis (분석) — Theoretic
│                                      │
│  1. Lexical Analysis  (Scanner)      │→ Token Stream
│  2. Syntax Analysis   (Parser)       │→ Parse Tree → AST
│  3. Semantic Analysis                │→ Annotated Tree
│                                      │
└──────────────────────────────────────┘
      ↓
  IC (Intermediate Code)                → 3-address code
      ↓
┌──────────────────────────────────────┐
│              Back-End                │  Synthesis (합성) — Rule-of-Thumb
│                                      │
│  4. Source Code Optimizer            │→ Constant Folding 등
│  5. Code Generator                   │→ Assembly 코드
│  6. Target Code Optimizer            │→ 명령어 교체/최적화
│                                      │
└──────────────────────────────────────┘
      ↓
Target Program

※ 전 단계 공통 참조
   ├── Symbol Table   (식별자 정보)
   ├── Literal Table  (상수값 정보)
   └── Error Handler  (오류 감지/보고/복구)
```

---

## 3. 주요 용어

**Source Program**
> 사람이 고급 언어로 작성한 원본 코드 (컴파일러의 입력)

**Target Program**
> 컴파일러가 생성한 특정 기계에서 실행 가능한 코드 (컴파일러의 출력)

**IC (Intermediate Code)**
> Front-end가 생성한 AST를 기계어로 바로 변환하는 대신 만드는 기계 독립적 중간 표현
> 언어별 Front-end만 바꾸면 같은 Back-end로 여러 기계 지원 가능 (예: 3-address code)

**Token**
> 소스 코드를 구성하는 의미 있는 최소 단위 (변수명, 연산자, 상수 등)

**Parse Tree**
> 토큰 스트림을 문법 규칙에 따라 트리 형태로 나타낸 것
> 모든 문법 기호 포함

**AST (Abstract Syntax Tree)**
> Parse Tree에서 코드 생성에 불필요한 문법 기호(`[`, `]`, `;` 등)를 제거한 트리
> "Abstract" = 불필요한 요소를 추상화(제거)했다는 의미

**Annotated Tree**
> Semantic Analysis 단계에서 AST에 타입 정보를 추가한 트리

---

## 4. 보조 테이블

| 테이블 | 저장 대상 | 예시 |
|--------|---------|------|
| Symbol Table | 식별자 (변수, 함수명) | `position`, `rate` |
| Literal Table | 상수값 | `"Park"`, `3.14`, `60` |
| Error Handler | 각 단계별 오류 감지/보고/복구 | 타입 불일치, 구문 오류 |

---

## 5. Analysis vs Synthesis

| | Analysis (분석) | Synthesis (합성) |
|--|---------------|----------------|
| 위치 | Front-End | Back-End |
| 입력/출력 | Source → IC | IC → Target |
| 방식 | Theoretic (이론/규칙 기반) | Rule-of-Thumb (경험 기반) |
| 종속성 | 언어에 종속 | 기계에 종속 |
