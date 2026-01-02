---
title: "WebClient Connection Reset by Peer 에러 해결기"
excerpt: "외부 API 호출 시 간헐적으로 발생하는 Connection reset by peer 에러의 원인을 분석하고, Connection Pool의 maxIdleTime 설정으로 해결한 과정"
permalink: /troubleshooting/webclient-connection-reset-by-peer/
categories: [Troubleshooting]
tags: [WebClient, Netty, Connection Pool, Reactor]
image:
  path: /assets/covers/problem_solving.png
toc: true
---

## 이슈

```text
2025-10-19T22:58:18.303Z DEBUG 1 --- [or-http-epoll-3] o.s.w.r.f.client.ExchangeFunctions       : [279d9bb] HTTP GET <https://ugt.upbit.com/biz/api/v1/members>
2025-10-19T22:58:18.304Z DEBUG 1 --- [or-http-epoll-1] r.netty.http.client.HttpClientConnect    : [035e7304-2, L:/10.201.7.11:34250 - R:ugt.upbit.com/104.18.217.119:443] Handler is being applied: {uri=https://ugt.upbit.com/biz/api/v1/members, method=GET}
2025-10-19T22:58:18.304Z DEBUG 1 --- [or-http-epoll-1] r.n.http.client.HttpClientOperations     : [035e7304-2, L:/10.201.7.11:34250 - R:ugt.upbit.com/104.18.217.119:443] No sendHeaders() called before complete, sending zero-length header
2025-10-19T22:58:18.305Z  WARN 1 --- [or-http-epoll-1] r.netty.http.client.HttpClientConnect    : [035e7304-2, L:/10.201.7.11:34250 - R:ugt.upbit.com/104.18.217.119:443] The connection observed an error, the request cannot be retried as the headers/body were sent
io.netty.channel.unix.Errors$NativeIoException: recvAddress(..) failed: Connection reset by peer
2025-10-19T22:58:18.308Z ERROR 1 --- [or-http-epoll-1] k.c.f.cointradeapi.client.UpbitClient    : 회원 정보 조회 중 예외 발생

...
Caused by: org.springframework.web.reactive.function.client.WebClientRequestException: recvAddress(..) failed: Connection reset by peer
	at org.springframework.web.reactive.function.client.ExchangeFunctions$DefaultExchangeFunction.lambda$wrapException$9(ExchangeFunctions.java:137)
	Suppressed: The stacktrace has been enhanced by Reactor, refer to additional information below:
Error has been observed at the following site(s):
	*__checkpoint ⇢ Request to GET <https://ugt.upbit.com/biz/api/v1/members> [DefaultWebClient]
Original Stack Trace:
```

- 업비트를 통해 회원 정보 불러오는 API에서 간헐적으로 connection reset by peer 에러가 발생하였습니다.

## 분석

### 1단계: 가능한 원인 분류

네트워크 요청 실패가 발생할 수 있는 지점들을 카테고리로 분류했습니다.

1. **방화벽/보안 장비** 보안 정책에 의해 요청이 차단되는 경우입니다.
2. **중간 네트워크 장비** 로드밸런서의 타임아웃, 프록시 서버 설정 문제, NAT 타임아웃 등 클라이언트와 서버 사이의 네트워크 장비에서 발생하는 문제입니다.
3. **서버 애플리케이션** 서버가 요청을 거부하거나(403, 429 등), 과부하 상태이거나, 재시작 중인 경우입니다.
4. **클라이언트** Connection Pool 관리 문제

### 2단계: 관련 팀 로그 확인

각 지점별로 문제가 있었는지 관련 팀에 협조를 요청했습니다.

- **업비트 팀 확인 결과** 해당 시간대 서버 로그에서 아무런 기록을 찾지 못했습니다. 만약 서버에서 에러가 발생했다면 5xx 응답을 보냈을 것이므로, 서버는 요청 자체를 받지 못한 것으로 판단됩니다.
- **보안팀 확인 결과** 해당 시간대 차단 로그가 없었습니다. 업비트 도메인은 화이트리스트에 등록되어 있으며, 만약 차단이 발생했다면 명시적인 차단 로그가 남았을 것입니다.

관련 팀들의 답변을 종합하면 "요청이 서버까지 도달하지 못했으며 중간에 막히지는 않았다."

### 3. 로그 심층 분석

```kotlin
2025-10-19T22:58:18.304Z DEBUG 1 --- [or-http-epoll-1] r.netty.http.client.HttpClientConnect    : 
[035e7304-2, L:/10.201.7.11:34250 - R:ugt.upbit.com/104.18.217.119:443] 
Handler is being applied: {uri=https://ugt.upbit.com/biz/api/v1/members, method=GET}
```

- `035e7304-2`: 커넥션 ID에 `2`가 붙음 → **재사용된 커넥션!**

왜 클라이언트는 커넥션이 끊어진 걸 몰랐을까 ?
TCP 커넥션은 idel상태에서 서버가 일방적으로 끊으면:
1. 서버는 FIN 패킷 전송
2. 클라이언트가 해당 소켓을 읽지 않으면 FIN 패킷이 커널 버퍼에만 쌓임
3. Connection Pool은 커넥션이 살아있다고 착각함.
4. 실제 요청 시도할 때 비로소 끊어진 것을 발견 -> Connection rest 에러


### 4. 재현 테스트
가설을 검증하기 위해 서버의 keep-alive timeout을 초과하는 시나리오를 재현했습니다:

```kotlin
@Test
fun `Connection reset 재현 테스트`() = runBlocking {
    val client = WebClient.builder()
        .clientConnector(ReactorClientHttpConnector(HttpClient.create()))
        .baseUrl("<https://ugt.upbit.com>")
        .build()

    // 1. 첫 요청 (커넥션 생성)
    client.get().uri("/biz/api/v1/members")
        .retrieve()
        .bodyToMono<String>()
        .block()
    
    println("첫 요청 성공 - 커넥션이 풀에 반환됨")
    
    // 2. 70초 대기 (서버 timeout 초과 예상)
    delay(70_000)
    
    // 3. 재시도 (stale 커넥션 사용 시도)
    try {
        client.get().uri("/biz/api/v1/members")
            .retrieve()
            .bodyToMono<String>()
            .block()
        println("성공 - 커넥션이 아직 살아있음")
    } catch (e: WebClientRequestException) {
        println("실패! Connection reset by peer")
        println("원인: 서버가 이미 커넥션을 끊었음")
    }
}
```

```kotlin
첫 요청 성공 - 커넥션이 풀에 반환됨
(70초 대기...)
실패! Connection reset by peer
````

점진적으로 대기 시간을 줄여가며 테스트한 결과, **60초 이상**에서 동일한 에러 재현을 확인했습니다.

근본 원인: 클라이언트의 Connection Pool idle timeout이 서버의 keep-alive timeout보다 길어서, 서버는 이미 끊었지만 클라이언트는 여전히 유효하다고 판단하는 **타이밍 불일치** 문제

## 해결

maxIdelTime을 서버 keep-alive timeout보다 안전 마진을 둬서 짧게 설정
```
@Bean
fun webClient(): WebClient {
    val httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .responseTimeout(Duration.ofSeconds(10))
        .option(ChannelOption.SO_KEEPALIVE, true)
		...
        .connectionProvider(
            ConnectionProvider.builder("upbit-pool")
                .maxConnections(100)
                .pendingAcquireMaxCount(200)
                .maxIdleTime(Duration.ofSeconds(40))
                .maxLifeTime(Duration.ofMinutes(5))
                .evictInBackground(Duration.ofSeconds(30))
                .build()
        )
	...
}
```

## 검증
배포 후 1주일간 모니터링 결과 `Connection reset by peer` 에러 **0건** 발생 
