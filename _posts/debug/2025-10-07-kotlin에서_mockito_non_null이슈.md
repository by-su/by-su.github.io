---
title: Kotlin에서 Mockito 사용시 `any() must not be null` 에러 해결기
excerpt: Java Mockito의 `any()`가 null을 반환하면서 Kotlin의 null safety와 충돌하여 발생한 NullPointerException 문제를 분석하고, mockito-kotlin 라이브러리를 통한 해결 방법과 Mockito의 내부 동작 원리를 정리합니다.
permalink: /posts/kotlin-mockito-any-must-not-be-null/
categories: [Kotlin, Testing, Mockito, Troubleshooting]
tags: [Kotlin, Mockito, mockito-kotlin, Unit Test, Null Safety]
image:
  path: /assets/covers/ks_template.png
toc: true
---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [왜 에러가 발생했을까?](#2-왜-에러가-발생했을까)
3. [Mockito의 내부 동작 원리](#3-mockito의-내부-동작-원리)
4. [해결 방법: mockito-kotlin](#4-해결-방법-mockito-kotlin)
5. [정리 및 베스트 프랙티스](#5-정리-및-베스트-프랙티스)

---  

## 1. 문제 상황

### 🚨 발생한 에러
```  
any(...) must not be null  
java.lang.NullPointerException: any(...) must not be null  
    at ShortenUrlServiceTest.충돌_시_최대_3회_재시도_후_성공(ShortenUrlServiceTest.kt:67)  
```  

### 💻 문제가 된 코드
```kotlin  
@Test  
fun `충돌_시_최대_3회_재시도_후_성공`() {  
    // ❌ Java Mockito 사용  
    `when`(shortUrlWriter.save(any()))  
        .thenThrow(DataIntegrityViolationException("Duplicate entry"))  
        .thenReturn(generateShortUrlWithSlug(THIRD_MOCK_SLUG))  
}  
```  

### 🤔 이상한 점
- `any<ShortUrl>()`로 타입을 명시해도 같은 에러 발생
- Java에서는 잘 동작하는 코드

---  

## 2. 왜 에러가 발생했을까?

### 🔍 근본 원인: Java vs Kotlin의 null 처리 차이

#### Java Mockito의 `any()` 구현
```java  
public static <T> T any() {  
    reportMatcher(new InstanceOfMatcher(Object.class));    return null;  // ⚠️ 항상 null을 반환!  
}  
```  

#### Kotlin의 null safety
```kotlin  
// ShortUrlWriter.kt  
fun save(shortUrl: ShortUrl): ShortUrl {  
    //       ^^^^^^^^    //       non-null 타입! null 허용 안됨  
    return shortUrlRepository.save(shortUrl)}  
```  

### 💥 충돌 발생
```kotlin  
`when`(shortUrlWriter.save(any()))  
//                         ^^^ null을 반환  
//     shortUrl: ShortUrl  
//               ^^^^^^^^ null 불가!  
// 결과: NullPointerException!  
```  

### 📊 비교표

| 항목 | Java | Kotlin |  
|------|------|--------|  
| null 처리 | 모든 타입이 nullable | 명시적으로 `?` 붙여야 nullable |  
| `any()` 반환값 | `null` | `null` (문제 발생!) |  
| 컴파일러 체크 | 런타임에 체크 | 컴파일 타임에 체크 |  
| `String` | nullable | non-nullable |  
| `String?` | - | nullable |  
  
---  

## 3. Mockito의 내부 동작 원리

### 🎯 핵심 질문: Mockito는 어떻게 Matcher를 인식할까?

답: **ThreadLocal 저장소**를 사용합니다!

### 📦 ThreadLocal이란?

각 스레드마다 독립적인 저장 공간을 제공하는 Java 기능

```java  
// 간단한 예시  
ThreadLocal<String> threadLocal = new ThreadLocal<>();  
  
// Thread-1에서  
threadLocal.set("Thread-1의 데이터");  
  
// Thread-2에서  
threadLocal.set("Thread-2의 데이터");  
  
// 각 스레드는 자신의 데이터만 볼 수 있음  
```  

### 🔄 Mockito의 Matcher 동작 과정

```kotlin  
`when`(shortUrlWriter.save(any()))  
    .thenReturn(result)  
```  

#### Step 1: `any()` 호출
```java  
public static <T> T any() {  
    // 1️⃣ Matcher를 ThreadLocal 스택에 저장  
    reportMatcher(new InstanceOfMatcher(Object.class));  
    // 2️⃣ null 반환 (이게 문제의 원인!)  
    return null;}  
  
private static void reportMatcher(ArgumentMatcher<?> matcher) {  
    // ThreadLocal 스택에 push    mockingProgress()        .getArgumentMatcherStorage()        .reportMatcher(matcher);}  
```  

**현재 상태:**
```  
Thread-1의 ThreadLocal Stack┌─────────────────────────┐  
│ InstanceOfMatcher       │ ← push됨!  
└─────────────────────────┘  
```  

#### Step 2: `save(null)` 호출
```java  
// Mockito의 Proxy가 인터셉트  
public Object handle(Invocation invocation) {  
    // 3️⃣ 같은 스레드의 ThreadLocal에서 Matcher 확인  
    List<ArgumentMatcher> matchers =        mockingProgress()            .getArgumentMatcherStorage()            .pullMatchers();  // 꺼내면서 스택 비움  
  
    if (!matchers.isEmpty()) {        // 4️⃣ "아! Matcher가 있었구나!"  
        registerStubbing(invocation, matchers);    }}  
```  

### 📊 전체 흐름도

```  
시간 순서  
═══════════════════════════════════════════════════════  
  
T1: any() 호출  
    │    ├─ reportMatcher() 실행  
    │  └─ ThreadLocal Stack: [InstanceOfMatcher] 저장 ✅  
    │    └─ return null       │       ▼  
T2: save(null) 호출  
    │    ├─ Mockito Proxy가 가로챔  
    │    ├─ ThreadLocal Stack 확인  
    │  └─ [InstanceOfMatcher] 발견! 🎯  
    │    ├─ "Matcher를 사용한 stubbing이구나!"  
    │    ├─ Stubbing 등록  
    │  └─ "save(any) → return result"    │    └─ Stack 비우기: []  
```  

## 4. 해결 방법: mockito-kotlin

### 📦 mockito-kotlin 라이브러리

Kotlin 전용으로 만들어진 Mockito wrapper

#### 설치
```kotlin  
// build.gradle.kts  
dependencies {  
    testImplementation("org.mockito.kotlin:mockito-kotlin:6.1.0")}  
```  

### 🔧 mockito-kotlin의 `any()` 구현

```kotlin  
// Java Mockito  
public static <T> T any() {  
    reportMatcher(new InstanceOfMatcher(Object.class));    return null;  // ❌ null 반환  
}  
  
// mockito-kotlin  
inline fun <reified T : Any> any(): T {  
    // 1. Java Mockito와 호환성 유지  
    reportMatcher(InstanceOfMatcher(T::class.java))  
    // 2. null 대신 mock 객체 반환 ✅  
    return createInstance<T>() ?: null as T}  
```  

**차이점:**
- Java Mockito: 항상 `null` 반환
- mockito-kotlin: **실제 mock 객체** 반환 (Kotlin null safety 만족!)


## 5. 정리 및 베스트 프랙티스

### 🎓 핵심 내용 정리

#### 문제의 원인
1. **Java Mockito의 `any()`는 null을 반환**
2. **Kotlin은 non-null 타입에 null을 허용하지 않음**
3. **결과: NullPointerException 발생**

#### Mockito의 내부 동작
1. **`any()`가 ThreadLocal 스택에 Matcher 저장**
2. **`save(null)` 호출 시 같은 ThreadLocal 확인**
3. **Matcher 발견 → Stubbing 생성**
4. **ThreadLocal로 스레드 안전성 보장**

#### 해결 방법
1. **mockito-kotlin 라이브러리 사용**
2. **null 대신 실제 mock 객체 반환**
3. **Kotlin의 null safety 만족**

### ✅ Kotlin 프로젝트 베스트 프랙티스

#### 1. 항상 mockito-kotlin 사용
```kotlin  
import org.mockito.kotlin.*```  
  
#### 2. whenever 사용  
```kotlin  
// ❌ 백틱이 필요한 `when``when`(mock.method())  
  
// ✅ Kotlin 친화적인 wheneverwhenever(mock.method())  
```  

#### 3. 타입 추론 활용
```kotlin  
// mockito-kotlin은 reified 타입 파라미터 지원  
whenever(mock.save(any()))  // 자동으로 타입 추론!  
  
// 명시적으로 지정도 가능  
whenever(mock.save(any<ShortUrl>()))  
```  

#### 4. argThat 사용 시
```kotlin  
// mockito-kotlin의 argThat은 it 자동 제공  
whenever(mock.save(argThat { slug == "abc" }))  
//                           ^^ non-null 보장!  
  
// Java Mockito는 nullablewhenever(mock.save(argThat { it?.slug == "abc" }))  
//                           ^^^ nullable 체크 필요  
```  

### 참고 자료

- [Mockito 공식 문서](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [mockito-kotlin GitHub](https://github.com/mockito/mockito-kotlin)
