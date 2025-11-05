**C++ — 템플릿과 메타프로그래밍 II: 강의록**

---

### 0. 강의 목표

* 클래스 템플릿, 부분/전면 특수화의 정확한 규칙을 이해한다. 
* CTAD(생성자 기반 자동 추론)·deduction guide를 안정적으로 설계한다. 
* `typename`/`template`/`using`이 필요한 자리와 이유를 직관적으로 파악한다. 
* SFINAE와 C++20 Concepts를 비교하여 제약(Constraints)을 표현한다. 
* 가변 템플릿과 폴딩 표현식으로 반복·축약을 깔끔히 구현한다. 
* 템플릿 디버깅 옵션과 실전 팁으로 빌드 피드백 루프를 빠르게 만든다. 

---

## 1. 클래스 템플릿의 뼈대와 특수화

**핵심 아이디어**
함수 템플릿과 달리, **클래스 템플릿은 “부분 특수화”**가 가능하다. 각 특수화는 **완전히 별개 클래스**로 간주된다.

```
A<T,R>  (일반)
 ├─ A<T,int>      (부분 특수화)
 └─ A<float,int>  (전면 특수화)
```

* “별개 클래스”라는 관점이 중요하다: 일반 버전의 멤버가 부분 특수화에 자동 상속되지 않는다. p.6–9 

**간단 예시 (type trait 직접 구현)**

```cpp
template<class T, class U> struct is_same { static constexpr bool value = false; };
template<class T> struct is_same<T,T> { static constexpr bool value = true; };
static_assert(!is_same<int,char>::value);
static_assert( is_same<float,float>::value);
```

p.10 

**포인터가 `const`를 가리키는지 판정**

```cpp
template<class T> struct is_pointer_to_const : std::false_type {};
template<class U> struct is_pointer_to_const<const U*> : std::true_type {};
static_assert(!is_pointer_to_const<int*>::value);
static_assert( is_pointer_to_const<const int*>::value);
```

p.11 

**클래스 템플릿 생성자 표기 축약**
클래스 내부에서는 기본 템플릿 인자를 반복하지 않는다(암시적). p.13 

---

## 2. CTAD와 Deduction Guide

**문제의식**
C++17부터 **생성자 호출만으로** 템플릿 인자를 추론한다(CTAD). 다만 **원하지 않는 타입**으로 추론될 수 있기에 **deduction guide**로 지도한다. p.15–22 

**핵심 패턴**

```cpp
// 1) 간단한 매핑
template<class T> struct MyString { MyString(T) {} };
MyString(const char*) -> MyString<std::string>;   // 가이드
MyString s{"abc"}; // -> MyString<std::string>
```

```cpp
// 2) 배열형 집계 (C++20부터는 aggregates에서 종종 불필요)
template<class T> struct A { T x, y; };
template<class T> A(T,T) -> A<T>;
A a{1,3}; // A<int>
```

```cpp
// 3) 독립 인자: sizeof를 템플릿 인자에 연결
template<int I> struct A { template<class T> A(T) {} };
template<class T> A(T) -> A<sizeof(T)>;
A x{1}; // A<sizeof(int)>
```

```cpp
// 4) 보편 참조에서 참조 제거
template<class T> struct A { template<class R> A(R&&) {} };
template<class R> A(R&&) -> A<std::remove_reference_t<R>>;
int v; A a{v}; // A<int>
```

```cpp
// 5) 반복자 구간에서 value_type 유도
template<class T> struct Container { template<class It> Container(It,It) {} };
template<class It>
Container(It,It) -> Container<typename std::iterator_traits<It>::value_type>;
std::vector v{1,2,3};
Container c{v.begin(), v.end()}; // Container<int>
```

**제한**
클래스 내부에서는 deduction guide가 작동하지 않는다. 필요하면 **팩토리 함수**로 우회한다. p.22 

---

## 3. 고급 규칙: 의존 이름, 상속, friend, 템플릿-템플릿 인자

**클래스·멤버 템플릿 특수화 규칙**

* 멤버 함수 템플릿은 **부분 특수화가 불가**.
* “클래스가 특정 인자로 **완전 특수화**된 경우”에만 해당 멤버 함수 템플릿을 **전면 특수화**할 수 있다. p.24–26 

**`typename`과 `template` 키워드**

* 의존 컨텍스트에서 `A<R>::type`이 **타입임을** 컴파일러에 명시: `using X = typename A<R>::type;` p.27–28 
* 의존 객체의 **멤버 템플릿 호출**에는 `a.template g<int>()`처럼 `template` 키워드가 필요할 수 있다. p.29 

**상속과 `using`**
`B<T> : A<T>`에서 기반 멤버를 가시화하려면 `using A<T>::x; using A<T>::f;`를 쓰거나 `this->x`로 접근한다. p.30 

**가상함수와 템플릿은 함께 못 쓴다**
가상 디스패치는 런타임, 템플릿 인스턴스는 컴파일타임. 무한히 많은 경우의 수에 대한 가상 테이블 생성을 언어 차원에서 금한다. p.31 

**friend와 템플릿**

* 특정 인스턴스만 friend 가능: `friend void f<int>();`
* 모든 템플릿 전반을 friend로: `template<class T> friend void f();`
* **부분 특수화**를 friend로 선언하는 것은 허용되지 않는다. p.32 

**템플릿-템플릿 인자**

```cpp
template<class T> struct A {};
template< template<class> class R >
struct B { R<int> x; R<float> y; };

B<A> b; // class/typename 키워드는 C++17부터 상호교환 가능
```

p.33 

---

## 4. 템플릿 메타프로그래밍(TMP)

**개념 요약**

* 컴파일타임에 계산한다(런타임은 0).
* 표현력은 충분하지만(튜링 완전), **컴파일 시간과 복잡도**를 높인다. p.35–36 

**재귀 예: 팩토리얼**

```cpp
template<int N> struct Fact { static constexpr int value = N * Fact<N-1>::value; };
template<> struct Fact<0> { static constexpr int value = 1; };
static_assert(Fact<5>::value == 120);
```

C++14+에선 `constexpr` 함수가 더 간명·유연·빠르다. p.37–38 

**로그 예**

* `Log2<N>`과 일반 `Log<N,BASE>`에서 `Max<1, N/BASE>`로 0 분기 회피. p.39–40 

**언롤 패턴(컴파일타임 × 런타임 혼합)**

```cpp
template<int N, int I=0> struct Unroll {
  template<class Op> static void run(Op op){ op(I); Unroll<N, I+1>::run(op); }
};
template<int N> struct Unroll<N,N>{ template<class Op> static void run(Op){} };
```

p.41 

---

## 5. SFINAE: 대체 실패는 에러가 아니다

**직관 모델**

```
후보 집합 { f<T>... } 
   └─ 템플릿 인자 대체(substitution) 시도
        ├─ 실패 → 후보에서 조용히 제외
        └─ 성공 → 최종 오버로드 해석 대상
```

p.43 

**동기 예시**
부호 있는/없는 정수에 대해 `ceil_div`를 안전히 오버로드하려면 타입 군별 처리가 필요하다. p.44 

**`std::enable_if` 4가지 배치법**

1. **반환 타입** 제어
2. **매개변수 타입** 자체를 제약
3. **숨은 기본 인자**(디폴트 인자)로 제약
4. **숨은 템플릿 인자**로 제약
   각각 장단이 있으므로 팀 규약을 정해 일관되게 쓰자. p.46–49 

**`decltype`로 표현 가능성 시험**

```cpp
template<class T, class U>
decltype(T{} + U{}) add(T a, U b) { return a + b; } // '+' 가능한 경우만 남는다
```

p.50 

**배열 vs 포인터 오버로드 함정**
배열은 인수 전달 시 포인터로 decay된다. 배열 참조 시그니처를 따로 두고, 포인터 전용 템플릿엔 `enable_if_t<is_pointer_v<T>>`로 제약하여 해소한다. p.51 

**나쁜 패턴**
같은 자리의 익명 템플릿 매개변수로 enable_if를 중복 선언해 **재정의 충돌**을 일으키지 말 것. `auto` 반환과 `enable_if_t`를 섞는 것도 금지. p.52 

**클래스 SFINAE & 멤버 존재 검사**

* 부분 특수화의 두 갈래 버전으로 타입군을 분기. p.53 
* 데이터 멤버/형식 멤버/스트림 연산자 지원 여부를 `declval`, `void_t`, `decltype`로 감지. p.54–57 

> p.58의 만화는 불길 속에서 “THIS IS SFINAE”라 말한다. 어렵고 미묘하니, 테스트를 늘리고 제약을 **최소 필요**로만 적자. 

---

## 6. 가변 템플릿과 폴딩 표현식

**파라미터 팩과 확장**

```cpp
template<class... Ts> void f(Ts... xs) {
  int v[] = { (xs)... };
  static_assert(sizeof...(xs) > 0);
}
```

p.60–61 

**재귀적 축약 vs C++17 폴딩**

```cpp
// 재귀
template<class T, class... Ts> auto add(T a, Ts... xs){ return a + add(xs...); }
template<class T, class U>    auto add(T a, U b){ return a + b; }

// 폴딩
template<class... Ts> auto add_fold(Ts... xs){ return (... + xs); }
```

p.62–65, 70–72 

**마지막 인자 뽑기(콤마 연산자)**

```cpp
template<class... Ts> constexpr auto last(Ts... xs){ return (xs, ...); }
```

p.71 

**동형 제약의 예(모두 `int`)**

```cpp
template<class... Ts>
std::enable_if_t<(std::is_same_v<Ts,int> && ...)> f(Ts...);
```

p.73 

**클래스에서의 가변성**

* `Add<N1,N2,...>` 같은 **비형식(non-type) 인자 팩**으로 컴파일타임 누적.
* 재귀 `Tuple<T,Ts...>`로 간단한 튜플 구현. p.74–75 

**함수/람다의 인자 개수(arity) 추론**

* 함수 포인터·참조·멤버 호출 연산자 `operator()` 특수화로 `sizeof...(Args)`를 얻는다. p.76–78 

---

## 7. C++20 Concepts: SFINAE의 문장력

**동기**
“산술형만 더하자”를 SFINAE로 쓰면 장황하다. Concept은 **짧고 읽히며**, **에러 메시지가 선명**하다. p.79–83 

**핵심 문법**

```cpp
template<class T> concept Arithmetic = std::is_arithmetic_v<T>;

template<Arithmetic T> T add(T a, T b) { return a + b; }
// 또는
auto add(Arithmetic auto a, Arithmetic auto b) { return a + b; }
```

* `requires` **절(clause)**: 선언부 앞/뒤에서 불리언 제약을 건다.
* `requires` **표현식(expression)**: “가능해야 하는 연산”을 컴파일타임에 기술한다.
* 절과 표현식은 함께 쓸 수 있고, `constexpr` 분기와도 잘 맞물린다. p.83–90 

**예: 연산 가능성과 결과 타입까지 제약**

```cpp
#include <concepts>
template<class T>
concept SquarableToInt = requires(T a) { { a * a } -> std::same_as<int>; };
```

p.86 

---

## 8. 템플릿 디버깅 옵션(현업 생존 키트)

* `-ftemplate-backtrace-limit=N` : 인스턴스 추적 길이 제어(최근 원인만 보려면 1).
* `-ftemplate-depth=N` : 재귀 인스턴스 깊이(기본 900).
* `-Wfatal-errors` : 첫 에러에서 중단, 잡음 감소.
* `-fdiagnostics-show-template-tree` : 중첩 템플릿을 트리로 출력. p.91–93 

---

## 9. 개념 다지기 — 요점 카드

* **부분 특수화는 새 클래스**다: 일반 버전 멤버가 자동 제공되지 않는다. p.6–9 
* **CTAD**는 강력하지만 **유도 가이드**로 안전 레일을 깔아라. p.15–22 
* 의존 이름엔 `typename`/`template`가 필요할 수 있다. p.27–29 
* **멤버 템플릿 부분 특수화 금지** 규칙을 기억하라. p.26 
* **SFINAE**는 후보에서 “조용히 제거”, **Concepts**는 “명시적 계약”. p.43, 79–83 
* **폴딩 표현식**으로 재귀를 지워라. p.70–72 

---

## 10. 실습 과제

1. **`is_all_integral<Ts...>`**: 모든 타입이 정수형이면 `true`.

   * SFINAE 버전과 Concepts 버전 두 가지로 구현. p.73, 82 

2. **`span_like` 생성자 CTAD**

   * 임의의 컨테이너 `c`의 `begin`/`end` 쌍으로 `View<T>`를 만들고, `value_type`을 deduction guide로 유도. p.20 

3. **`Unroll<N>`로 벡터 합산**

   * `Unroll` 템플릿을 이용해 `N`번 인덱스를 방문하며 합을 누적. p.41 

4. **배열 vs 포인터 f()**

   * 배열 참조, 포인터 인자 두 오버로드가 올바로 선택되도록 SFINAE 설계. p.51 

5. **`arity_of<F>`**

   * 함수·람다·펑터 모두의 인자 개수를 구하는 trait 완성. p.76–78 

---

## 11. 안전·성능 메모(현업용)

* **컴파일 시간 예산**: TMP 남용은 빌드 병목. 폴딩/`constexpr`로 단순화할 수 있으면 그렇게 하라. p.35–38, 70–72 
* **에러 메시지 품질**: “사용자 정의 concept + `requires` 문장”은 팀 온보딩 비용을 줄인다. p.82–88 
* **API 표면 축소**: `template<template<class> class>` 같은 고차 템플릿은 강력하지만 과용 시 사용성을 급격히 낮춘다. p.33 
* **가상 + 템플릿 불가**: 다형성은 런타임, 템플릿은 컴파일타임. 필요하면 **type erasure**(본 강의 범위 밖) 고려. p.31 

---

### 부록 A. 빠른 레퍼런스(문법 스니펫)

**의존 이름**

```cpp
using X = typename A<T>::type;
a.template g<int>();
```

p.27–29 

**Concepts**

```cpp
template<class T>
concept C = requires(T x){ x+x; } && (sizeof(T) >= 4);

template<class T> requires C<T>
void f(T);

template<class T>
void g(T) requires C<T>;
```

p.83–88 

**폴딩**

```cpp
template<class... Ts> auto sum(Ts... xs){ return (... + xs); }
template<class... Ts> auto last(Ts... xs){ return (xs, ...); }
```

p.70–72