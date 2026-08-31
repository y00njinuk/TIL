# Off-by-one 에러

## 1. 정의
"Off-by-one 에러"는 "값이나 범위를 1만큼 잘못 계산해서 생기는 버그"를 말합니다. 프로그래밍에서 가장 흔하게 발생하는 논리 오류 중 하나로, 반복문, 배열 인덱스, 범위 검사 등에서 자주 발생합니다.

## 2. 발생 원인

### 2.1. 인덱싱 방식의 차이
대부분의 프로그래밍 언어는 0-based 인덱싱을 사용하지만, 사람의 직관은 1-based 카운팅에 익숙합니다.

**예시:**
```
배열: [A, B, C, D, E]
인덱스: 0  1  2  3  4  (컴퓨터)
순서:   1  2  3  4  5  (사람)
```

### 2.2. 경계 조건(Boundary Condition) 실수
범위의 시작과 끝을 처리할 때 `<` 와 `<=`, `>` 와 `>=` 를 혼동하여 발생합니다.

### 2.3. 루프 카운터 초기화 오류
반복문의 시작값이나 종료 조건을 잘못 설정하는 경우입니다.

## 3. 대표적인 발생 사례

### 3.1. 배열 인덱스 오류

**잘못된 코드 (C/C++):**
```cpp
int arr[5] = {1, 2, 3, 4, 5};
for (int i = 0; i <= 5; i++) {  // ❌ i <= 5는 잘못됨
    cout << arr[i] << endl;
}
// arr[5]는 배열 범위를 벗어남 (인덱스는 0~4만 유효)
```

**올바른 코드:**
```cpp
int arr[5] = {1, 2, 3, 4, 5};
for (int i = 0; i < 5; i++) {  // ✅ i < 5가 올바름
    cout << arr[i] << endl;
}
```

### 3.2. 문자열 처리

**잘못된 코드 (C):**
```c
char str[10] = "Hello";
int len = strlen(str);  // len = 5
for (int i = 0; i <= len; i++) {  // ❌ null terminator 포함
    printf("%c", str[i]);
}
// str[5]는 '\0'이므로 출력되지 않음
```

**올바른 코드:**
```c
char str[10] = "Hello";
int len = strlen(str);
for (int i = 0; i < len; i++) {  // ✅ null terminator 제외
    printf("%c", str[i]);
}
```

### 3.3. 버퍼 복사

**잘못된 코드:**
```cpp
void copyBuffer(char* dest, const char* src, int size) {
    for (int i = 0; i <= size; i++) {  // ❌ 1바이트 초과 복사
        dest[i] = src[i];
    }
}

char buffer[10];
copyBuffer(buffer, "Hello", 5);  // buffer[5]까지 복사 (6바이트)
```

**올바른 코드:**
```cpp
void copyBuffer(char* dest, const char* src, int size) {
    for (int i = 0; i < size; i++) {  // ✅ 정확히 size 바이트만 복사
        dest[i] = src[i];
    }
}
```

### 3.4. 범위 검사

**잘못된 코드:**
```python
def is_valid_index(index, list_size):
    return index >= 0 and index <= list_size  # ❌ list_size는 유효하지 않음

my_list = [1, 2, 3, 4, 5]
if is_valid_index(5, len(my_list)):  # len(my_list) = 5
    print(my_list[5])  # IndexError!
```

**올바른 코드:**
```python
def is_valid_index(index, list_size):
    return index >= 0 and index < list_size  # ✅ list_size - 1까지가 유효

my_list = [1, 2, 3, 4, 5]
if is_valid_index(4, len(my_list)):
    print(my_list[4])  # 5 출력
```

### 3.5. 범위 분할

**잘못된 코드:**
```java
// 배열을 두 부분으로 나누기
int[] arr = {1, 2, 3, 4, 5, 6};
int mid = arr.length / 2;  // mid = 3

// 첫 번째 부분: 0 ~ mid
// 두 번째 부분: mid ~ arr.length
// 문제: arr[3]이 중복됨! ❌
```

**올바른 코드:**
```java
int[] arr = {1, 2, 3, 4, 5, 6};
int mid = arr.length / 2;  // mid = 3

// 첫 번째 부분: 0 ~ mid-1 (또는 0 ~ mid 미만)
// 두 번째 부분: mid ~ arr.length-1 ✅
```

### 3.6. 이진 탐색

**잘못된 코드:**
```cpp
int binarySearch(int arr[], int size, int target) {
    int left = 0, right = size;  // ❌ right = size는 범위 초과
    while (left < right) {
        int mid = (left + right) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid;
    }
    return -1;
}
```

**올바른 코드 (방법 1):**
```cpp
int binarySearch(int arr[], int size, int target) {
    int left = 0, right = size - 1;  // ✅ right = size - 1
    while (left <= right) {
        int mid = (left + right) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**올바른 코드 (방법 2 - Half-open interval):**
```cpp
int binarySearch(int arr[], int size, int target) {
    int left = 0, right = size;  // right는 exclusive
    while (left < right) {  // left < right (등호 없음)
        int mid = (left + right) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid;  // mid도 exclusive
    }
    return -1;
}
```

## 4. 보안 취약점으로 이어지는 경우

### 4.1. 버퍼 오버플로우

**취약한 코드:**
```c
char buffer[100];
int len = getUserInputLength();  // 사용자가 100을 입력

for (int i = 0; i <= len; i++) {  // ❌ buffer[100] 접근 (오버플로우)
    buffer[i] = getUserInputChar(i);
}
```

**결과:**
- 스택 메모리 손상
- Return address 덮어쓰기
- 임의 코드 실행 가능 (RCE)

### 4.2. 힙 오버플로우

**취약한 코드:**
```cpp
void processData(uint8_t* data, size_t size) {
    uint8_t* buffer = new uint8_t[size];
    for (size_t i = 0; i <= size; i++) {  // ❌ 1바이트 초과
        buffer[i] = data[i] ^ 0xFF;
    }
    delete[] buffer;
}
```

**결과:**
- 힙 메타데이터 손상
- Use-after-free 취약점 유발
- 메모리 손상 공격

### 4.3. 정수 오버플로우와 결합

**취약한 코드:**
```c
void allocateBuffer(unsigned int count) {
    if (count > MAX_COUNT) return;  // 범위 검사
    
    // ❌ count + 1이 오버플로우 가능
    char* buffer = malloc(count + 1);
    
    for (unsigned int i = 0; i <= count; i++) {  // ❌ off-by-one
        buffer[i] = 0;
    }
}
```

## 5. 언어별 특성과 Off-by-one

### 5.1. C/C++
- 가장 취약: 직접적인 메모리 접근
- 컴파일러가 경계 검사를 하지 않음
- 버퍼 오버플로우로 직접 연결

### 5.2. Java
```java
int[] arr = new int[5];
arr[5] = 10;  // ArrayIndexOutOfBoundsException 발생
```
- Runtime에 자동으로 경계 검사
- 예외 발생으로 크래시 방지
- 성능 오버헤드 존재

### 5.3. Python
```python
lst = [1, 2, 3, 4, 5]
print(lst[5])  # IndexError: list index out of range
```
- 자동 경계 검사
- 음수 인덱스 지원으로 혼란 가능
- Slicing에서 off-by-one이 덜 치명적

### 5.4. Rust
```rust
let arr = [1, 2, 3, 4, 5];
let x = arr[5];  // panic! at runtime
```
- 컴파일 타임 + 런타임 검사
- `get()` 메서드로 안전한 접근 제공
- 소유권 시스템으로 메모리 안전성 보장

## 6. 예방 방법

### 6.1. 코딩 습관

**1. 반복문은 일관된 패턴 사용**
```cpp
// 추천: 0부터 시작, n 미만
for (int i = 0; i < n; i++) { }

// 추천: 범위 기반 for문 (C++11+)
for (auto& item : container) { }
```

**2. 경계값 테스트**
- 빈 배열 (크기 0)
- 1개 요소
- 최대 크기
- 최소/최대 인덱스

**3. 명확한 변수명**
```cpp
// 나쁨
int n = array.size();
for (int i = 0; i <= n; i++) { }  // n이 크기인지 마지막 인덱스인지 헷갈림

// 좋음
int size = array.size();
int lastIndex = array.size() - 1;
for (int i = 0; i < size; i++) { }  // 명확함
```

### 6.2. 안전한 API 사용

**C++ STL:**
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// 위험
vec[5];  // undefined behavior

// 안전
try {
    vec.at(5);  // throws std::out_of_range
} catch (const std::out_of_range& e) {
    // 처리
}
```

**Java Collections:**
```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

// 안전한 접근
if (index >= 0 && index < list.size()) {
    int value = list.get(index);
}
```

### 6.3. 정적 분석 도구

**C/C++:**
- Clang Static Analyzer
- Coverity
- AddressSanitizer (ASan)
- Valgrind

**사용 예시:**
```bash
# AddressSanitizer로 컴파일
g++ -fsanitize=address -g program.cpp -o program

# Valgrind로 실행
valgrind --leak-check=full ./program
```

### 6.4. 단위 테스트

**경계값 테스트 예시 (Python):**
```python
import unittest

class TestArrayAccess(unittest.TestCase):
    def test_empty_array(self):
        arr = []
        with self.assertRaises(IndexError):
            _ = arr[0]
    
    def test_single_element(self):
        arr = [1]
        self.assertEqual(arr[0], 1)
        with self.assertRaises(IndexError):
            _ = arr[1]
    
    def test_last_element(self):
        arr = [1, 2, 3, 4, 5]
        self.assertEqual(arr[4], 5)  # 마지막 요소
        with self.assertRaises(IndexError):
            _ = arr[5]  # 범위 초과
```

### 6.5. 코드 리뷰 체크리스트

✅ 반복문의 종료 조건이 `<` 인가 `<=` 인가?
✅ 배열/리스트의 크기가 n이면, 유효한 인덱스는 0~n-1인가?
✅ 문자열 끝에 null terminator를 고려했는가?
✅ 버퍼 크기와 복사할 데이터 크기를 정확히 계산했는가?
✅ 경계값(0, 1, max-1, max)에서 테스트했는가?

## 7. 실전 디버깅

### 7.1. 증상 인식

**메모리 손상 증상:**
- Segmentation Fault (SIGSEGV)
- 예측 불가능한 동작
- 랜덤 크래시
- 다른 변수 값이 변경됨

**예시:**
```cpp
int arr[5] = {1, 2, 3, 4, 5};
int important_var = 42;

for (int i = 0; i <= 5; i++) {  // ❌ off-by-one
    arr[i] = 0;
}

// important_var의 값이 0으로 변경될 수 있음 (스택 레이아웃에 따라)
```

### 7.2. 디버깅 기법

**1. 디버거 사용:**
```bash
gdb ./program
(gdb) run
# Segmentation fault 발생
(gdb) backtrace
(gdb) print i
(gdb) print arr
```

**2. 로깅 추가:**
```cpp
for (int i = 0; i < n; i++) {
    printf("Accessing index %d (size: %d)\n", i, n);  // 디버그 출력
    arr[i] = 0;
}
```

**3. Assert 사용:**
```cpp
#include <cassert>

void accessArray(int* arr, int size, int index) {
    assert(index >= 0 && index < size);  // 경계 검사
    arr[index] = 0;
}
```

### 7.3. 자동화된 검출

**AddressSanitizer 출력 예시:**
```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60300000eff4 at pc 0x000000400b5c
WRITE of size 4 at 0x60300000eff4 thread T0
    #0 0x400b5b in main example.cpp:10
    #1 0x7ffff7a05b96 in __libc_start_main
Address 0x60300000eff4 is located 0 bytes to the right of 20-byte region
```

## 8. 역사적 사례

### 8.1. 심각한 보안 취약점

**OpenSSL Heartbleed (2014):**
```c
// 취약한 코드 (단순화)
unsigned char* buffer = malloc(payload_length + padding);
memcpy(buffer, payload, payload_length + padding);  // ❌ off-by-one과 유사
```
- Off-by-one 유사 버그
- 메모리 정보 누출
- 전 세계 서버 영향

**Windows 메타파일 취약점 (MS06-001):**
- WMF 파싱 중 off-by-one 에러
- 원격 코드 실행 가능
- Critical 등급 취약점

### 8.2. 소프트웨어 버그

**Mars Climate Orbiter (1999):**
- 단위 변환 오류 (유사한 계산 실수)
- 궤도 진입 실패
- $327M 손실

## 9. 결론

**Off-by-one 에러는:**
- 가장 흔한 프로그래밍 버그 중 하나
- 단순해 보이지만 심각한 결과 초래 가능
- 메모리 안전성과 직결
- 보안 취약점의 원인이 됨

**예방의 핵심:**
1. **일관된 코딩 스타일** - 반복문은 항상 `< n` 사용
2. **경계값 테스트** - 0, 1, n-1, n 테스트
3. **안전한 API 사용** - 경계 검사가 있는 함수 선호
4. **자동화 도구** - 정적 분석, 동적 분석 활용
5. **코드 리뷰** - 동료의 눈으로 검증

**기억할 점:**
- 배열 크기가 n이면, 유효 인덱스는 0부터 n-1까지
- `<` 와 `<=` 의 차이를 항상 의식하기
- 경계 조건은 두 번 확인하기
- 테스트는 경계값에 집중하기
