# ac언어 구문 분석기(Parser) 구현

---

## 1. 전체 구조 개요

```
소스코드 (istream)
    ↓
Scanner (어휘 분석기, lexer.py)
    ↓ 토큰(Token) 한 개씩 반환
Parser (구문 분석기)
    ↓ 문법에 맞는지 검사
구문 분석 완료
```

- **Scanner** : `lexer.py` 로 분리된 어휘 분석기. `from lexer import Scanner` 로 가져옴
- **Parser** : 순환 하강(Recursive Descent Parsing) 방식으로 구현

---

## 2. 순환 하강 파싱(Recursive Descent Parsing) 구현 원칙

- 파싱 알고리즘 중 **성능이 가장 낮지만 구현이 빠름**
- 구현 규칙:
  - 생성 규칙 왼쪽(LHS)의 모든 **비단말기호(Nonterminal)** 마다 **개별 함수** 생성
  - 오른쪽(RHS)에 **비단말기호** 있으면 → 해당 **함수 호출**
  - 오른쪽(RHS)에 **단말기호(Terminal)** 있으면 → **match()** 함수로 비교

```
생성 규칙 예시:
Stmt → id assign Val Expr

→ Stmt 함수 안에서:
    match('ID')      ← 단말기호 id
    match('ASSIGN')  ← 단말기호 assign
    Val()            ← 비단말기호 Val → 함수 호출
    Expr()           ← 비단말기호 Expr → 함수 호출
```

---

## 3. Parser 클래스 전체 코드

```python
class Parser:
    def __init__(self, src_code):       
        self.source_code = src_code  # 소스 코드
        self.tok = None 
        self.idx = 0         
```

- `source_code` : 분석할 소스코드 문자열
- `tok` : 현재 읽어온 토큰
- `idx` : 현재 읽고 있는 소스코드 위치(인덱스)

---

## 4. 각 비단말기호(Nonterminal) 함수 상세 설명

### 5-1. Prog() — `Prog → Dcls Stmts $`

```python
def Prog(self):
    """Prog -> Dcls Stmts $"""
    self.idx, self.tok = Scanner(self.idx, self.source_code)  # 첫 토큰 읽어옴
    if (self.tok.type == 'FLTDCL' or self.tok.type == 'INTDCL' or self.tok.type == 'ID' 
        or self.tok.type == 'PRINT' or self.tok.type == 'EOF'):
        self.Dcls()       # 변수 타입 선언 처리
        self.Stmts()      # 문장 처리
        self.match('EOF') # 입력 끝 확인
    else:
        raise SyntaxError(f"expected floatdcl, intdcl, id, print, or eof {tok.type}")            
```

- **시작 기호(Start Symbol)** 이라서 가장 먼저 호출됨
- `Prog()` 에서만 `Scanner` 를 처음 호출해 첫 토큰을 읽어옴
- 이후 `match()` 를 호출할 때마다 다음 토큰을 가져옴
- if 조건: Prog 다음에 올 수 있는 모든 토큰 확인

> **왜 EOF도 조건에 포함?** : 선언부(Dcls)와 문장부(Stmts)가 모두 λ(빈 생성)일 수 있어서 빈 프로그램도 허용해야 하기 때문

---

### 5-2. Dcls() — `Dcls → Dcl Dcls | λ`

```python
def Dcls(self):
    """Dcls -> Dcl Dcls | empty"""
    print(f"Dcls, {self.idx}, {repr(self.tok)}")  
    
    if (self.tok.type == 'FLTDCL' or self.tok.type == 'INTDCL'):
        self.Dcl()   # 선언 하나 처리
        self.Dcls()  # 나머지 선언 처리 (순환 호출)
        
    elif (self.tok.type == 'ID' or self.tok.type == 'PRINT' or self.tok.type == 'EOF'):
        pass  # λ-생성: 선언부 없음, 아무것도 안 함
    else:
        raise SyntaxError("expected floatdcl, intdcl, id, print, or eof")
```

- `FLTDCL` 또는 `INTDCL` 이면 → 선언이 있다는 뜻 → `Dcl()` + `Dcls()` 순환 호출
- `ID`, `PRINT`, `EOF` 이면 → 선언부 끝, 문장부 시작 → λ-생성으로 처리 (`pass`)

> **λ-생성(empty production)** : 아무것도 없어도 문법적으로 올바름. Python에서는 `pass` 로 표현

---

### 5-3. Dcl() — `Dcl → floatdcl id | intdcl id`

```python
def Dcl(self):
    """Dcl -> floatdcl id | intdcl id"""
    print(f"Dcl, {self.idx}, {repr(self.tok)}")  
    
    if self.tok.type == 'FLTDCL':
        self.match('FLTDCL')  # f 읽고 다음 토큰 가져옴
        self.match('ID')      # 변수명 읽고 다음 토큰 가져옴
    elif self.tok.type == 'INTDCL':
        self.match('INTDCL')  # i 읽고 다음 토큰 가져옴
        self.match('ID')      # 변수명 읽고 다음 토큰 가져옴
    else:
        raise SyntaxError("expected float or int declaration")
```

- 선언 하나를 처리하는 함수
- `f b` → `match('FLTDCL')` + `match('ID')`
- `i a` → `match('INTDCL')` + `match('ID')`

---

### 5-4. Stmts() — `Stmts → Stmt Stmts | λ`

```python
def Stmts(self):
    """Stmts -> Stmt Stmts | empty"""
    print(f"Stmts, {self.idx}, {repr(self.tok)}") 
    
    if (self.tok.type == 'ID' or self.tok.type == 'PRINT'):
        self.Stmt()   # 문장 하나 처리
        self.Stmts()  # 나머지 문장 처리 (순환 호출)
    elif self.tok.type == 'EOF':
        pass  # λ-생성: 더 이상 문장 없음
    else:
        raise SyntaxError("expected id, print, or eof")          
```

- `ID` 또는 `PRINT` 이면 → 문장이 있다는 뜻 → `Stmt()` + `Stmts()` 순환 호출
- `EOF` 이면 → 모든 문장 처리 완료 → λ-생성으로 처리 (`pass`)

---

### 5-5. Stmt() — `Stmt → id assign Val Expr | print id`

```python
def Stmt(self):
    """Stmt -> id assign Val Expr | print id"""
    print(f"Stmt, {self.idx}, {repr(self.tok)}")     
    
    if self.tok.type == 'ID':
        self.match('ID')      # 변수명
        self.match('ASSIGN')  # =
        self.Val()            # 값
        self.Expr()           # 연산식
    elif self.tok.type == 'PRINT':
        self.match('PRINT')   # p
        self.match('ID')      # 변수명
    else:
        raise SyntaxError("expected id or print")               
```

- `a = 5` → `match('ID')` + `match('ASSIGN')` + `Val()` + `Expr()`
- `p b` → `match('PRINT')` + `match('ID')`

---

### 5-6. Expr() — `Expr → plus Val Expr | minus Val Expr | λ`

```python
def Expr(self):
    """Expr → plus Val Expr | minus Val Expr | empty"""
    print(f"Expr, {self.idx}, {repr(self.tok)}")   
    
    if self.tok.type == 'PLUS':
        self.match('PLUS')  # +
        self.Val()          # 값
        self.Expr()         # 나머지 연산 (순환 호출)
    elif self.tok.type == 'MINUS':
        self.match('MINUS') # -
        self.Val()          # 값
        self.Expr()         # 나머지 연산 (순환 호출)
    elif (self.tok.type == 'ID' or self.tok.type == 'PRINT' or self.tok.type == 'EOF'): 
        pass  # λ-생성: 연산식 없음
    else:
        raise SyntaxError("expected plus, minus, id, print, or eof")              
```

- `+ 3.2` → `match('PLUS')` + `Val()` + `Expr()`
- 연산 없으면 → λ-생성 (`pass`)
- `ID`, `PRINT`, `EOF` 가 오면 → 다음 문장이 시작된다는 뜻 → 연산식 종료

> **λ-생성 조건에 ID, PRINT, EOF가 포함된 이유**: 연산식이 끝나면 다음에 새 문장(ID, PRINT)이 오거나 입력이 끝나기(EOF) 때문

---

### 5-7. Val() — `Val → id | inum | fnum`

```python
def Val(self):
    """Val -> id | inum | fnum"""
    print(f"Val, {self.idx}, {repr(self.tok)}")  
    
    if self.tok.type == 'ID':
        self.match('ID')    # 변수
    elif self.tok.type == 'INUM':
        self.match('INUM')  # 정수 상수
    elif self.tok.type == 'FNUM':
        self.match('FNUM')  # 실수 상수
    else:            
        raise SyntaxError("expected id, inum, or fnum")            
```

- 값 하나를 처리하는 함수
- 변수(`ID`), 정수(`INUM`), 실수(`FNUM`) 세 가지 중 하나

---

### 5-8. match() — 단말기호(Terminal) 비교

```python
def match(self, t):     
    """match a token of the expected terminal."""
    if self.tok.type != t:
        raise SyntaxError(f"syntax error {t}, {self.tok.type}")
        exit()
    else:
        if self.idx-1 < len(self.source_code):             
            self.idx += 1
            self.idx, self.tok = Scanner(self.idx, self.source_code)  # 다음 토큰 읽어옴
        else:
            exit()            
```

- 현재 토큰 타입이 기대한 단말기호와 **일치하는지** 확인
- 일치하면 → 다음 토큰 읽어옴 (`Scanner` 호출)
- 불일치하면 → `SyntaxError` 발생

> `match()` 안의 `print` 문은 주석 처리되어 있음. 필요하면 주석 해제해서 토큰 흐름 확인 가능

---

## 5. 실행 예시

```python
istream = "f b   i a   a = 5   b = a + 3.2   p b"

p = Parser(istream)
p.Prog()  # 시작 기호(Start Symbol)인 Prog() 호출
```

- `Parser` 객체 생성 후 반드시 **`Prog()`** 를 호출해야 함
  - `Prog` 가 ac 문법의 시작 기호이기 때문
- `Prog()` 내부에서 `Scanner` 를 처음 호출해 첫 토큰을 읽어옴

**실행 흐름 (f b / i a / a = 5 / b = a + 3.2 / p b)**

```
Prog()
    ↓ Scanner 호출 → 첫 토큰 'FLTDCL' 읽어옴
    Dcls()
        Dcl()
            match('FLTDCL')  ← f
            match('ID')      ← b
        Dcls()
            Dcl()
                match('INTDCL') ← i
                match('ID')     ← a
            Dcls()
                pass (λ, ID 확인)
    Stmts()
        Stmt()
            match('ID')      ← a
            match('ASSIGN')  ← =
            Val()
                match('INUM') ← 5
            Expr()
                pass (λ, ID 확인)
        Stmts()
            Stmt()
                match('ID')      ← b
                match('ASSIGN')  ← =
                Val()
                    match('ID')  ← a
                Expr()
                    match('PLUS') ← +
                    Val()
                        match('FNUM') ← 3.2
                    Expr()
                        pass (λ, PRINT 확인)
            Stmts()
                Stmt()
                    match('PRINT') ← p
                    match('ID')    ← b
                Stmts()
                    pass (λ, EOF 확인)
    match('EOF')
```

---

## 6. 토큰 타입 매핑

코드에서 사용하는 토큰 타입 이름과 ac 언어 기호 대응표:

| 코드 토큰 타입 | ac 단말기호 | 실제 입력 |
|--------------|------------|---------|
| `FLTDCL` | floatdcl | `f` |
| `INTDCL` | intdcl | `i` |
| `ID` | id | `a`, `b`, ... |
| `ASSIGN` | assign | `=` |
| `PRINT` | print | `p` |
| `PLUS` | plus | `+` |
| `MINUS` | minus | `-` |
| `INUM` | inum | `5`, `123`, ... |
| `FNUM` | fnum | `3.2`, `0.5`, ... |
| `EOF` | $ | 입력 끝 |

---

## 7. 핵심 포인트 정리

**순환 하강 파싱 규칙 요약**

| 생성 규칙 오른쪽(RHS) | 코드에서 처리 방법 |
|---------------------|-----------------|
| 비단말기호(Nonterminal) | 해당 함수 호출 |
| 단말기호(Terminal) | `match()` 함수로 비교 |
| λ (빈 생성) | `pass` 로 처리 |

**λ-생성 조건 판단 방법**

각 함수에서 λ-생성을 적용하는 조건은 **"다음에 올 수 있는 토큰"** 을 보고 판단:
- `Dcls()` λ : `ID`, `PRINT`, `EOF` → 선언 끝나고 문장 시작
- `Stmts()` λ : `EOF` → 모든 문장 처리 완료
- `Expr()` λ : `ID`, `PRINT`, `EOF` → 연산식 끝나고 다음 문장 시작 또는 입력 끝

**에러 처리**

- 문법에 맞지 않는 토큰이 오면 `SyntaxError` 발생
- `match()` : 예상한 토큰과 다르면 구문 오류
- 각 함수의 `else` : 어떤 생성 규칙도 적용 안 되면 구문 오류

---

> 📁 참고: `ac언어-구문분석기(배포용).ipynb` — 홍윤식 교수  
> 코드의 `print` 문은 디버깅용. 주석 해제하면 각 함수 호출 흐름 확인 가능
