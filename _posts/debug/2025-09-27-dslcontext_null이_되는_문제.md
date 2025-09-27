---
title: Kotlin에서 JOOQ 사용시 `dslContext`가 null이 되는 문제 해결기
excerpt: Kotlin 클래스가 기본적으로 final이라 Spring AOP 프록시가 생성되지 못하면서 발생한 `dslContext` null 문제를 분석하고, `open` 키워드 및 `kotlin-spring` 플러그인을 통한 해결 방법을 정리했습니다.
permalink: /posts/kotlin-spring-jooq-dslcontext-null/
categories: [Kotlin, Spring, JOOQ, Troubleshooting]
tags: [Kotlin, Spring Boot, JOOQ, DSLContext, AOP, Proxy, kotlin-spring]
image:
  path: /assets/covers/ks_template.png
toc: true
---

## 문제 상황

테스트 코드 실행 시 `ActorRepository` 내부에서 `dslContext`가 `null`이 되어 `NullPointerException` 문제가 발생하였습니다.

``` kotlin

@Repository

class ActorRepository(

    private val dslContext: DSLContext

) {
    fun findById(id: Long): Actor {
        return dslContext.select(ACTOR) // NullPointerException 발생
            .from(ACTOR)
            .where(ACTOR.ACTOR_ID.eq(id))
            .fetchOneInto(Actor::class.java)
            ?: throw IllegalArgumentException("Actor not found")
    }

}

```
-   `DSLContext` 빈은 정상적으로 생성됨을 확인하였습니다. (JOOQ 로고 출력 확인)
-   데이터베이스 연결도 정상입니다. (HikariCP 풀 정상 시작)

BUT !! `ActorRepository` 생성자 주입이 실패 → 런타임에 `dslContext == null`


------------------------------------------------------------------------

## 원인 분석
### Kotlin 클래스의 기본 제약
-   Kotlin 클래스와 메서드는 기본적으로 `final`.
-   Spring Boot 3.x는 기본적으로 CGLIB 기반 프록시를 사용합니다.
-   하지만 CGLIB은 `final` 클래스/메서드에 프록시를 만들 수 없습니다.

### `@Repository`와 프록시
-   `@Repository`는 예외 변환 AOP가 적용되는데 이 과정에서 프록시 생성이 필요합니다.
-   클래스나 메서드가 `final`이면 프록시를 만들지 못해 Bean 주입 과정이 꼬임 → 결과적으로 `dslContext`가 null이 되었습니다.


------------------------------------------------------------------------


## 해결 과정

### 시도 1: `lateinit var` 사용 (실패)

``` kotlin

@Repository

class ActorRepository {
    @Autowired
    private lateinit var dslContext: DSLContext
}

```
왜 lateinit만으로는 해결되지 않나? lateinit은 단지 **"null이 허용되지 않는 프로퍼티를 나중에 주입하겠다"*** 라는 Kotlin 문법 val 대신 var에 lateinit을 붙이면 생성자 파라미터 없이도 DI 가능합니다. 하지만 주입 자체가 정상적으로 일어나야만 값이 채워집니다.
문제의 본질은 주입 시점에 프록시가 생성되지 않아 Repository Bean이 제대로 등록되지 않았기 때문입니다. 
프록시 생성이 실패하면 Spring이 Bean 인스턴스를 만들 수 없고, 따라서 DI도 수행되지 않습니다. 
그 결과 lateinit 필드가 끝내 초기화되지 않은 채 접근되어 UninitializedPropertyAccessException 또는 null 문제가 발생합니다.



### 시도 2: 클래스만 `open` 처리 (불완전)

``` kotlin

@Repository
open class ActorRepository(
    private val dslContext: DSLContext
)
```
-   클래스만 열어도 일부 경우 여전히 프록시 적용이 안 되었습니다..
-   특히 메서드가 `final`이면 AOP 포인트컷이 감싸지지 않았습니다..

  	CGLIB Proxy는 실제 클래스를 상속해서 서브클래스를 런타임에 생성합니다. 이 서브클래스에서 원본 메서드를 오버라이드 -> advice 코드 삽입합니다.  그런데 원본 메서드가 final이면 오버라이드 불가하기 때문에 AOP도 적용이 불가능합니다.



### 최종 해결: 클래스 + 메서드 모두 `open`

``` kotlin

@Repository

open class ActorRepository(
    private val dslContext: DSLContext
) {

    open fun findById(id: Long): Actor {
        return dslContext.select(ACTOR)
            .from(ACTOR)
            .where(ACTOR.ACTOR_ID.eq(id))
            .fetchOneInto(Actor::class.java)
            ?: throw IllegalArgumentException("Actor not found")
    }

}

```
-   클래스와 메서드를 모두 `open` 처리하니 프록시 생성 정상화 → `dslContext` 주입 성공하였습니다.

### 근본적인 해결: kotlin-spring 플러그인

Gradle 플러그인에 다음 추가:
``` gradle

plugins {
    id("org.jetbrains.kotlin.plugin.spring") version "2.1.21"
}

```

-   `kotlin-spring`은 `@Repository`, `@Service`, `@Configuration`,`@Transactional` 등이 붙은 ****클래스와 메서드를 자동으로 open 처리****.
-   일일이 `open` 키워드를 붙일 필요가 없어집니다.
-   결과적으로 Spring AOP와 Kotlin의 `final` 제약이 충돌하지 않습니다.


------------------------------------------------------------------------
## 결론

-   `dslContext`가 null이 된 원인은 ****프록시 생성 실패****.
-   ****Kotlin 클래스/메서드가 기본적으로 final → Spring AOP가 못 감쌈****
-   임시 해결은 직접 `open` 지정
-   ****근본 해결은** `**kotlin-spring**` **플러그인 적용****.
