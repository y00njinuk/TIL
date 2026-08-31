# Kafka Streams suppress 옵션

## 1. 개요
Kafka Streams의 `suppress()` 연산자는 스트림 처리에서 중간 결과의 방출을 제어하는 중요한 기능입니다. 윈도우 집계나 상태 기반 연산에서 최종 결과만을 downstream으로 전달하고 싶을 때 사용됩니다.

참고: https://kafka.apache.org/28/streams/developer-guide/memory-mgmt/?utm_source=chatgpt.com

## 2. suppress 옵션의 동작 원리

### 2.1. 기본 개념
`suppress()`는 Kafka Streams의 KTable 연산에서 사용되며, 레코드의 방출(emit)을 지연시키거나 제어하는 역할을 합니다.

**주요 목적:**
- 윈도우 집계의 중간 결과를 억제하고 최종 결과만 방출
- downstream 처리 부하 감소
- 불필요한 업데이트 제거

### 2.2. 사용 위치
`suppress()`는 주로 다음과 같은 위치에서 사용됩니다:

```java
KStream<String, Long> stream = builder.stream("input-topic");

stream
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count()
    .suppress(Suppressed.untilWindowCloses(unbounded()))  // 여기서 사용
    .toStream()
    .to("output-topic");
```

### 2.3. 동작 방식

**윈도우 기반 suppress:**
- 윈도우가 완전히 닫힐 때까지 결과를 버퍼에 보관
- 윈도우 종료 시점에만 최종 결과를 방출
- 워터마크(watermark)와 grace period를 활용하여 윈도우 종료 시점 결정

**예시 타임라인:**
```
시간 T0-T5: 윈도우 내 이벤트 발생
  - 기존(suppress 없음): T1, T2, T3, T4, T5 각 시점마다 중간 결과 방출
  - suppress 사용: T5 (윈도우 종료) 시점에만 최종 결과 방출
```

### 2.4. Suppressed 정책 옵션

**1. untilWindowCloses()**
```java
Suppressed.untilWindowCloses(BufferConfig.unbounded())
```
- 윈도우가 완전히 닫힐 때까지 결과를 억제
- 가장 일반적으로 사용되는 옵션
- 늦게 도착하는 데이터(late arrival)를 고려한 grace period 지원

**2. untilTimeLimit()**
```java
Suppressed.untilTimeLimit(Duration.ofSeconds(30), 
    BufferConfig.maxRecords(1000))
```
- 지정된 시간 동안만 결과를 억제
- 시간 제한에 도달하면 중간 결과도 방출

**3. BufferConfig 옵션**
- `unbounded()`: 메모리 제한 없이 모든 결과 버퍼링 (주의 필요)
- `maxRecords(n)`: 최대 n개의 레코드만 버퍼링
- `maxBytes(n)`: 최대 n 바이트까지만 버퍼링
- `emitEarlyWhenFull()`: 버퍼가 가득 차면 조기 방출

## 3. 캐시 정책과의 차이점

### 3.1. Kafka Streams 캐싱
Kafka Streams는 내부적으로 레코드 캐싱 메커니즘을 가지고 있습니다.

**캐싱의 특징:**
- 상태 스토어(State Store)의 읽기/쓰기 성능 최적화
- commit interval 동안 변경사항을 메모리에 누적
- 주로 성능 향상을 위한 메커니즘
- `cache.max.bytes.buffering` 설정으로 제어

**캐싱 동작:**
```java
// 캐시 설정 예시
Properties props = new Properties();
props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024); // 10MB
```

### 3.2. suppress vs 캐싱 비교표

| 특성 | suppress() | 캐싱(Caching) |
|-----|-----------|-------------|
| **주 목적** | 의미론적 제어 (중간 결과 억제) | 성능 최적화 |
| **제어 수준** | 애플리케이션 레벨 (명시적) | 시스템 레벨 (암묵적) |
| **방출 타이밍** | 윈도우 종료, 시간 제한 등 | commit interval 기반 |
| **사용 위치** | 특정 연산에 명시적 적용 | 전역적으로 적용 |
| **설정 방법** | API 호출 (suppress()) | 설정 파일 (config) |
| **메모리 관리** | BufferConfig로 세밀한 제어 | cache.max.bytes.buffering |
| **의미론** | 정확히 1회 최종 결과 보장 | 성능을 위한 배칭 |

### 3.3. 주요 차이점 상세 설명

**1. 목적의 차이:**
- **suppress**: 비즈니스 로직에 따른 결과 제어가 목적
  - 예: "윈도우가 완전히 닫힐 때까지 결과를 보내지 마"
- **캐싱**: 시스템 성능 향상이 목적
  - 예: "상태 스토어 접근을 줄이기 위해 메모리에 버퍼링"

**2. 제어 방식:**
- **suppress**: 개발자가 명시적으로 코드에서 제어
  - 특정 연산 체인에만 적용
  - 세밀한 버퍼 관리 옵션 제공
- **캐싱**: 설정 파일을 통한 전역 제어
  - 모든 상태 스토어에 자동 적용
  - 단순한 크기 제한만 가능

**3. 방출 시점:**
- **suppress**: 
  - 윈도우 종료 시점 (untilWindowCloses)
  - 시간 제한 도달 (untilTimeLimit)
  - 버퍼 가득 참 (emitEarlyWhenFull)
- **캐싱**:
  - commit.interval.ms 도달
  - 캐시 크기 초과
  - 자동 플러시 조건 충족

**4. 메모리 관리:**
- **suppress**:
  - 각 suppress 연산마다 독립적인 버퍼
  - 레코드 수, 바이트 크기 등으로 세밀하게 제어
  - OOM 방지를 위한 조기 방출 옵션
- **캐싱**:
  - 모든 상태 스토어가 공유하는 전역 캐시
  - 총량 제어만 가능

## 4. 실전 사용 예시

### 4.1. 윈도우 집계에서 최종 결과만 방출
```java
KStream<String, PageView> views = builder.stream("page-views");

// suppress 없이 - 중간 결과가 계속 방출됨
views
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count()
    .toStream()
    .to("view-counts");  // 매 이벤트마다 업데이트된 카운트 전송

// suppress 사용 - 윈도우 종료 시에만 최종 결과 방출
views
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count()
    .suppress(Suppressed.untilWindowCloses(
        BufferConfig.maxRecords(10000).emitEarlyWhenFull()
    ))
    .toStream()
    .to("view-counts");  // 5분 윈도우 종료 시에만 1회 전송
```

### 4.2. 실시간 대시보드 업데이트 최적화
```java
// 너무 빈번한 업데이트를 방지
KTable<String, Long> userActivity = stream
    .groupByKey()
    .count()
    .suppress(Suppressed.untilTimeLimit(
        Duration.ofSeconds(10),  // 10초마다 업데이트
        BufferConfig.maxBytes(1024 * 1024)  // 1MB 버퍼
    ));
```

### 4.3. 메모리 제한이 있는 환경
```java
// 메모리 부족 시 조기 방출
stream
    .groupByKey()
    .windowedBy(SessionWindows.with(Duration.ofMinutes(30)))
    .count()
    .suppress(Suppressed.untilWindowCloses(
        BufferConfig
            .maxRecords(5000)
            .shutDownWhenFull()  // 또는 emitEarlyWhenFull()
    ))
    .toStream()
    .to("session-counts");
```

## 5. 성능 및 운영 고려사항

### 5.1. 메모리 사용
- suppress는 결과를 버퍼에 보관하므로 메모리 사용량 증가
- unbounded() 사용 시 OOM 위험 - 프로덕션 환경에서는 제한 설정 권장
- 메모리 모니터링 필수: JVM 힙, RocksDB 상태 크기 등

### 5.2. 지연 시간(Latency)
- suppress 사용 시 결과 방출이 지연됨
- 실시간성이 중요한 경우 untilTimeLimit 사용 고려
- grace period 설정에 따라 추가 지연 발생

### 5.3. Grace Period와의 관계
```java
TimeWindows.of(Duration.ofMinutes(5))
    .grace(Duration.ofSeconds(30))  // 30초의 grace period
```
- suppress와 함께 사용 시, 윈도우 종료 + grace period 후에 결과 방출
- 늦게 도착하는 데이터를 처리하기 위한 추가 대기 시간

### 5.4. 모니터링 지표
- Suppression buffer size
- Suppressed records count
- Late records dropped
- Memory usage per suppress operator

## 6. 언제 suppress를 사용해야 하는가?

### 6.1. 사용하면 좋은 경우
✅ 윈도우 집계의 최종 결과만 필요한 경우
✅ Downstream 시스템의 부하를 줄여야 하는 경우
✅ 데이터베이스 업데이트 빈도를 줄이고 싶은 경우
✅ 정확한 윈도우 결과가 필요한 경우 (중간 결과 불필요)

### 6.2. 사용을 피해야 하는 경우
❌ 실시간 업데이트가 매우 중요한 경우
❌ 메모리가 제한적인 환경에서 대량 데이터 처리 시
❌ 중간 결과도 의미가 있는 경우 (예: 실시간 대시보드의 근사값)
❌ 윈도우가 매우 길고 이벤트가 많은 경우 (메모리 압박)

## 7. 결론

**suppress 옵션:**
- 의미론적 제어를 위한 명시적 API
- 윈도우 집계 결과의 방출 타이밍 제어
- 비즈니스 로직에 따른 세밀한 제어 가능

**캐싱:**
- 성능 최적화를 위한 시스템 레벨 기능
- 자동으로 적용되는 배칭 메커니즘
- 상태 스토어 접근 최적화

두 기능은 **목적과 동작 방식이 다르며**, 함께 사용될 수 있습니다:
- 캐싱: 항상 활성화하여 전반적인 성능 향상
- suppress: 비즈니스 요구사항에 따라 선택적으로 적용

적절한 BufferConfig 설정과 모니터링을 통해 suppress를 효과적으로 활용하면 불필요한 중간 결과를 제거하고 시스템 효율성을 높일 수 있습니다.
