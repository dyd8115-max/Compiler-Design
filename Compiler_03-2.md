# ac(adding calculator) 컴파일러 프론트엔드 정리

---

## 1. ac 언어란?

**ac (adding calculator)** 는 덧셈/뺄셈 계산기 수준의 미니 언어다.  
컴파일러 프론트엔드(어휘 분석 → 구문 분석)를 학습하기 위한 예제용 언어.

### ac 언어 문법 규칙

```
Prog  → Dcls Stmts $
Dcls  → Dcl Dcls | ε
Dcl   → floatdcl id | intdcl id
Stmts → Stmt Stmts | ε
Stmt  → id assign Val Expr | print id
Expr  → plus Val Expr | minus Val Expr | ε
Val   → id | inum | fnum
```

### ac 언어 예시 입력

```
f b   i a   a = 5   b = a + 3.2   p b
```

| 토큰 | 의미 |
|------|------|
| `f b` | float 타입으로 변수 b 선언 |
| `i a` | int 타입으로 변수 a 선언 |
| `a = 5` | a에 정수 5 할당 |
| `b = a + 3.2` | b에 a + 3.2 할당 |
| `p b` | b 출력 |

### 토큰 종류

| 토큰 타입 | 의미 | 예시 |
|----------|------|------|
| `FLTDCL` | float 선언 키워드 | `f` |
| `INTDCL` | int 선언 키워드 | `i` |
| `PRINT` | 출력 키워드 | `p` |
| `ID` | 변수 이름 | `a`, `b`, `c`, ... |
| `ASSIGN` | 대입 연산자 | `=` |
| `PLUS` | 덧셈 | `+` |
| `MINUS` | 뺄셈 | `-` |
| `INUM` | 정수 리터럴 | `5`, `007` |
| `FNUM` | 실수 리터럴 | `3.2`, `32.` |
| `EOF` | 입력 끝 | (문자열 끝) |
| `ERROR` | 인식 불가 문자 | `?`, `@` 등 |

> `f`, `i`, `p`는 키워드라서 변수 이름(ID)으로 사용 불가.  
> 나머지 알파벳 소문자 23개(`a`~`z` 중 f, i, p 제외)는 모두 변수로 사용 가능.

---

## 2. Token 클래스

```python
class Token:
    def __init__(self, typ, val):
        self.type = typ    # 토큰 종류 (ex: 'ID', 'PLUS', ...)
        self.value = val   # 토큰 값   (ex: 'a', None, ...)
        
    def __repr__(self):
        return f"Token({self.type}, {repr(self.value)})"
```

### 해설

- 토큰은 **type**과 **value** 2개 속성을 가진다.
- `type` : 토큰이 어떤 종류인지 → `'ID'`, `'PLUS'`, `'EOF'` 등
- `value` : 토큰의 실제 값 → `'a'`, `'5'`, `None` (값이 없는 경우)
- `__repr__` : `print()` 또는 `repr()`로 출력할 때 사용되는 형식 정의

```python
# 출력 예시
tok = Token('ID', 'a')
print(tok)  # Token(ID, 'a')

tok = Token('PLUS', None)
print(tok)  # Token(PLUS, None)
```

---

## 3. 심볼 테이블 (Symbol Table)

```python
import string

alphabet = string.ascii_lowercase  # 'abcdefghijklmnopqrstuvwxyz'
symbol_table = {}

for i in range(len(alphabet)):
    symbol_table[alphabet[i]] = 0  # 초기값 0

# 키워드는 변수로 사용 불가 → None으로 표시
symbol_table['f'] = symbol_table['i'] = symbol_table['p'] = None
```

### 해설

- **심볼 테이블** : 변수 이름과 그 값을 저장하는 딕셔너리(dict)
- ac 언어에서 변수는 알파벳 소문자 1글자만 사용 가능
- `f`, `i`, `p`는 키워드이므로 `None`으로 설정해 구분
- 컴파일러에서 심볼 테이블은 **어휘 분석 단계부터 코드 생성까지** 전 단계에서 공유됨

```python
# 확인
print(symbol_table['a'])  # 0
print(symbol_table['f'])  # None
```

---

## 4. 어휘 분석기 (Lexer / Scanner)

어휘 분석기는 소스 코드 문자열을 읽어 **토큰 하나씩** 반환한다.  
호출할 때마다 토큰 1개를 리턴하는 구조.

### 4-1. peek() — 현재 문자 읽기

```python
def peek(i, src_code):
    return src_code[i]
```

- 인덱스 `i`가 가리키는 문자 **1개**를 읽어옴
- 인덱스는 움직이지 않음 (그냥 들여다보기만)

### 4-2. advance() — 다음 문자로 이동

```python
def advance(i, lim, src_code):
    i += 1
    if (i < lim):
        s = src_code[i]  
    else: 
        s = None
    return i, s
```

- 인덱스를 1 증가시키고 다음 문자를 반환
- 문자열 끝을 넘어가면 `None` 반환 (범위 초과 방지)
- Python 함수는 여러 값을 반환할 수 있어서 `return i, s` 형태로 작성

### 4-3. ScanDigit() — 숫자 토큰 인식

```python
def ScanDigit(idx, src_code):
    val = ""           # 숫자 문자를 쌓아갈 빈 문자열
    s = peek(idx, src_code)
    limit = len(src_code)
    
    while s.isdigit():          # 숫자인 동안 계속 읽기
        val += s
        idx, s = advance(idx, limit, src_code)
        if s == None:
            return idx, Token('INUM', val)   # 문자열 끝 → 정수
    
    if (idx < limit and s != '.'):
        type = 'INUM'           # 소숫점 없음 → 정수
    else:    
        type = 'FNUM'           # 소숫점 있음 → 실수
        val += s                # '.' 추가
        idx, s = advance(idx, limit, src_code)     
        if s == None:
            return idx, Token(type, val)
        
        while s.isdigit():      # 소숫점 이후 숫자 계속 읽기
            val += s  
            idx, s = advance(idx, limit, src_code)                     
            if s == None:
                break   
                    
    return idx, Token(type, val)
```

#### 해설

- **정수(INUM)** vs **실수(FNUM)** 구분 기준 = 소숫점(`.`) 유무
- 숫자 문자를 `val`에 하나씩 이어붙여 문자열로 만든 후 Token 생성
- `while s.isdigit()` 로 숫자가 끝날 때까지 계속 읽음

```python
# 동작 예시
ScanDigit(0, "32.572")  # → Token(FNUM, '32.572')
ScanDigit(0, "32.")     # → Token(FNUM, '32.')
ScanDigit(0, "32")      # → Token(INUM, '32')
ScanDigit(0, "007")     # → Token(INUM, '007')  ← 앞의 0도 그냥 포함됨
```

### 4-4. representativeChar() — ID vs 키워드 구분

```python
def representativeChar(c):
    if c.isalpha():
        if c not in ['f', 'i', 'p']:
            return True
    return False
```

- 알파벳이면서 `f`, `i`, `p`가 아니면 → **변수 이름(ID)**으로 인식
- `f`, `i`, `p`는 키워드라서 별도 처리

### 4-5. Scanner() — 메인 어휘 분석기

```python
def Scanner(idx, src_code):
    limit = len(src_code)  
    val = ""    
    ans = Token('EOF', None)     # 기본값은 EOF
    
    if idx >= limit-1:           # 문자열 끝이면 EOF 반환
        return idx, ans
    
    s = peek(idx, src_code)
    
    # 공백(space), 탭(\t), 줄바꿈(\n) 건너뛰기
    while (s == ' ' or s == '\t' or s == '\n'):
        idx, s = advance(idx, limit, src_code)     
    
    if s != None:
        if s.isdigit():              # 숫자이면
            idx, ans = ScanDigit(idx, src_code)
        else:                        # 문자이면
            if representativeChar(s):
                ans = Token('ID', s)       # 변수 이름
            elif s == 'f':
                ans = Token('FLTDCL', None)
            elif s == 'i':
                ans = Token('INTDCL', None)
            elif s == 'p':
                ans = Token('PRINT', None)
            elif s == '=':
                ans = Token('ASSIGN', None)
            elif s == '+':
                ans = Token('PLUS', None)
            elif s == '-':
                ans = Token('MINUS', None)   
            else:
                ans = Token('ERROR', s)    # 인식 불가 문자
    
    return idx, ans
```

#### 해설

- **공백 처리** : `while` 루프로 공백/탭/줄바꿈을 먼저 건너뜀 (white space 무시)
- **숫자** → `ScanDigit()` 호출
- **문자** → `representativeChar()`로 ID/키워드 구분 후 각 토큰 반환
- **모르는 문자** → `ERROR` 토큰 반환
- 호출할 때마다 **토큰 1개**만 반환

### 4-6. Scanner 전체 실행 예시

```python
istream = "f b   i a   a = 5   b = a + 3.2   p b ?"
limit = len(istream)
index = 0

while index < limit:
    index, tok = Scanner(index, istream)
    if tok.type == 'ERROR':
        raise SyntaxError(f"ERROR >>> unexpected char {tok.value}")
        break
    print(index, tok)
    index, s = advance(index, limit, istream)
```

```
# 출력 결과 (? 직전까지)
0 Token(FLTDCL, None)
2 Token(ID, 'b')
6 Token(INTDCL, None)
8 Token(ID, 'a')
...
SyntaxError: ERROR >>> unexpected char ?   ← ?는 ERROR 처리
```

> `raise SyntaxError` : `print()`로 에러를 알리는 대신 예외를 던지는 방식.  
> Java의 `throw` 문과 같은 개념.

---

## 5. 구문 분석기 (Parser)

어휘 분석기(Scanner)가 만든 토큰들을 받아 **문법에 맞는 구조인지 검사**한다.

### 5-1. Recursive Descent Parsing이란?

> 문법의 각 **비단말 기호(Nonterminal)마다 함수 하나씩** 만들고,  
> 재귀 호출로 파싱하는 방식.

| 문법 요소 | 코드 대응 |
|----------|----------|
| 비단말 기호 (Nonterminal) | 함수 정의 |
| 우변의 비단말 | 해당 함수 호출 |
| 우변의 단말 (Terminal) | `match()` 호출 |
| 빈 문자열 (ε) | `pass` |

**장점** : 문법 → 코드로 바로 변환 가능, 구현 빠름  
**단점** : 파싱 알고리즘 중 성능이 가장 나쁨

### 5-2. Parser 클래스 전체

```python
from lexer import Scanner

class Parser:
    def __init__(self, src_code):       
        self.source_code = src_code  # 소스 코드 저장
        self.tok = None              # 현재 토큰
        self.idx = 0                 # 현재 인덱스
```

### 5-3. Prog() — 시작 기호

```python
def Prog(self):
    """Prog -> Dcls Stmts $"""
    self.idx, self.tok = Scanner(self.idx, self.source_code)  # 첫 토큰 읽기
    if (self.tok.type == 'FLTDCL' or self.tok.type == 'INTDCL' or self.tok.type == 'ID' 
        or self.tok.type == 'PRINT' or self.tok.type == 'EOF'):
        self.Dcls()        # 변수 선언 파싱
        self.Stmts()       # 문장 파싱
        self.match('EOF')  # 입력 끝 확인
    else:
        raise SyntaxError(f"expected floatdcl, intdcl, id, print, or eof")
```

#### 해설

- `Prog`가 **시작 기호(Start Symbol)** → 가장 먼저 호출하는 함수
- 파싱 시작 시 **첫 토큰을 Scanner로 읽어옴**
- 문법 `Prog → Dcls Stmts $` 그대로 함수 호출 순서로 표현
- `if` 조건 : 이 규칙이 적용될 수 있는 토큰인지 확인

### 5-4. Dcls() — 변수 선언 목록

```python
def Dcls(self):
    """Dcls -> Dcl Dcls | empty"""
    print(f"Dcls, {self.idx}, {repr(self.tok)}")  
    
    if (self.tok.type == 'FLTDCL' or self.tok.type == 'INTDCL'):
        self.Dcl()    # 선언 하나 파싱
        self.Dcls()   # 재귀 호출 → 또 다른 선언이 있을 수 있음
        
    elif (self.tok.type == 'ID' or self.tok.type == 'PRINT' or self.tok.type == 'EOF'):
        pass          # ε 생성 → 선언 없음, 아무것도 안 함
    else:
        raise SyntaxError("expected floatdcl, intdcl, id, print, or eof")
```

#### 해설

- `Dcls → Dcl Dcls | ε` 문법을 그대로 if-elif로 표현
- `FLTDCL` or `INTDCL` → 선언이 있으면 `Dcl()` 호출 후 **자기 자신 재귀 호출** (Dcls → Dcl Dcls)
- `ID`, `PRINT`, `EOF` → 더 이상 선언이 없음 → `pass` (ε 생성)
- **`pass`** : Python에서 아무 동작도 하지 않는 문장 (ε 표현)

### 5-5. Dcl() — 변수 선언 하나

```python
def Dcl(self):
    """Dcl -> floatdcl id | intdcl id"""
    print(f"Dcl, {self.idx}, {repr(self.tok)}")  
    
    if self.tok.type == 'FLTDCL':
        self.match('FLTDCL')   # 'f' 확인
        self.match('ID')       # 변수 이름 확인
    elif self.tok.type == 'INTDCL':
        self.match('INTDCL')   # 'i' 확인
        self.match('ID')       # 변수 이름 확인
    else:
        raise SyntaxError("expected float or int declaration")
```

#### 해설

- `Dcl → floatdcl id | intdcl id` 문법 표현
- 단말 기호는 `match()`로 처리 → 현재 토큰이 기대한 토큰인지 확인

### 5-6. Stmts() — 문장 목록

```python
def Stmts(self):
    """Stmts -> Stmt Stmts | empty"""
    print(f"Stmts, {self.idx}, {repr(self.tok)}") 
    
    if (self.tok.type == 'ID' or self.tok.type == 'PRINT'):
        self.Stmt()    # 문장 하나 파싱
        self.Stmts()   # 재귀 호출
    elif self.tok.type == 'EOF':
        pass           # ε 생성
    else:
        raise SyntaxError("expected id, print, or eof")
```

### 5-7. Stmt() — 문장 하나

```python
def Stmt(self):
    """Stmt -> id assign Val Expr | print id"""
    print(f"Stmt, {self.idx}, {repr(self.tok)}")     
    
    if self.tok.type == 'ID':
        self.match('ID')       # 변수 이름
        self.match('ASSIGN')   # '='
        self.Val()             # 값
        self.Expr()            # 추가 연산 (있으면)
    elif self.tok.type == 'PRINT':
        self.match('PRINT')    # 'p'
        self.match('ID')       # 출력할 변수
    else:
        raise SyntaxError("expected id or print")
```

#### 해설

- 문장은 두 가지 형태: 대입문(`a = 5`) 또는 출력문(`p b`)
- `a = 5 + 3.2` 처럼 추가 연산이 있을 수 있어 `Expr()` 호출

### 5-8. Expr() — 추가 연산

```python
def Expr(self):
    """Expr -> plus Val Expr | minus Val Expr | empty"""
    print(f"Expr, {self.idx}, {repr(self.tok)}")   
    
    if self.tok.type == 'PLUS':
        self.match('PLUS')   # '+' 확인
        self.Val()           # 피연산자
        self.Expr()          # 또 연산이 있을 수 있음 (재귀)
    elif self.tok.type == 'MINUS':
        self.match('MINUS')
        self.Val()
        self.Expr()
    elif (self.tok.type == 'ID' or self.tok.type == 'PRINT' or self.tok.type == 'EOF'): 
        pass                 # ε 생성 → 연산 끝
    else:
        raise SyntaxError("expected plus, minus, id, print, or eof")
```

#### 해설

- `a + b + c` 처럼 연산이 여러 개일 수 있어서 재귀 호출
- `Expr → plus Val Expr` → `+` 다음 값, 다음 또 Expr (재귀)
- `ID`, `PRINT`, `EOF` 나오면 → 연산 끝, `pass`

### 5-9. Val() — 값 (피연산자)

```python
def Val(self):
    """Val -> id | inum | fnum"""
    print(f"Val, {self.idx}, {repr(self.tok)}")  
    
    if self.tok.type == 'ID':
        self.match('ID')
    elif self.tok.type == 'INUM':
        self.match('INUM') 
    elif self.tok.type == 'FNUM':
        self.match('FNUM') 
    else:            
        raise SyntaxError("expected id, inum, or fnum")
```

#### 해설

- 값은 변수(`ID`), 정수(`INUM`), 실수(`FNUM`) 세 가지
- 모두 단말 기호이므로 `match()`로 처리

### 5-10. match() — 토큰 매칭

```python
def match(self, t):     
    """match a token of the expected terminal."""
    
    if self.tok.type != t:
        raise SyntaxError(f"syntax error {t}, {self.tok.type}")
        exit()
    else:
        if self.idx-1 < len(self.source_code):             
            self.idx += 1
            self.idx, self.tok = Scanner(self.idx, self.source_code)  # 다음 토큰 읽기
        else:
            exit()
```

#### 해설

- **현재 토큰이 기대한 토큰인지 확인**하는 함수
- 맞으면 → `Scanner()` 호출해서 **다음 토큰으로 넘어감**
- 틀리면 → `SyntaxError` 발생

```
match('FLTDCL') 호출 시
  → 현재 tok.type이 'FLTDCL'이 맞나? 확인
  → 맞으면 Scanner() 호출해서 다음 토큰 읽어옴
  → 틀리면 SyntaxError
```

### 5-11. Parser 실행

```python
istream = "f b   i a   a = 5   b = a + 3.2   p b"

p = Parser(istream)
p.Prog()   # 시작 기호 함수 호출 → 파싱 시작
```

#### 실행 흐름

```
p.Prog()
  → Scanner() 호출 → 첫 토큰 'FLTDCL' 읽음
  → Dcls() 호출
      → Dcl() 호출
          → match('FLTDCL') → 다음 토큰 'ID(b)' 읽음
          → match('ID')     → 다음 토큰 'INTDCL' 읽음
      → Dcls() 재귀 호출
          → Dcl() 호출
              → match('INTDCL') → ...
              → match('ID')     → ...
          → Dcls() 재귀 호출
              → tok = 'ID' → pass (선언 끝)
  → Stmts() 호출
      → Stmt() 호출 (a = 5)
      → Stmts() 재귀 호출
          → Stmt() 호출 (b = a + 3.2)
          → Stmts() 재귀 호출
              → Stmt() 호출 (p b)
              → Stmts() 재귀 호출
                  → tok = 'EOF' → pass (문장 끝)
  → match('EOF') → 파싱 완료
```

---

## 6. 전체 흐름 요약

```
소스 코드 문자열
      ↓
  Scanner() 호출 (어휘 분석)
      ↓
  토큰 1개 반환
      ↓
  Parser가 토큰 받아서 문법 검사 (구문 분석)
  ├── 비단말 → 함수 호출
  ├── 단말   → match() 호출
  └── ε      → pass
      ↓
  구문 오류 없으면 파싱 완료
  구문 오류 있으면 SyntaxError 발생
```

### 어휘 분석 vs 구문 분석

| | 어휘 분석 (Scanner) | 구문 분석 (Parser) |
|---|---|---|
| 역할 | 문자 → 토큰 변환 | 토큰 → 문법 구조 검사 |
| 단위 | 문자 1개씩 처리 | 토큰 1개씩 처리 |
| 오류 | 인식 불가 문자 (`ERROR`) | 문법 구조 오류 (`SyntaxError`) |
| 구현 | `Scanner()` 함수 | `Parser` 클래스 + 재귀 함수 |

### 핵심 포인트

- `Scanner()` : 호출할 때마다 토큰 **1개** 반환
- `match()` : 토큰 확인 후 **다음 토큰** 읽어옴
- 비단말마다 **함수 1개** = Recursive Descent Parsing의 핵심
- `pass` = ε 생성 규칙 표현 (아무것도 안 함)
- 시작 기호(`Prog`)의 함수를 **제일 먼저** 호출
