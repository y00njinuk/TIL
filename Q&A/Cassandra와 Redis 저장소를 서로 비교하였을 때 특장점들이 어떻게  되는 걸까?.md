# Cassandra와 Redis 비교

## 1. 개요
Cassandra와 Redis는 모두 분산 저장소이지만, 서로 다른 목적과 아키텍처를 가진 데이터베이스 시스템입니다. 각각의 특장점을 이해하고 적절한 상황에서 활용하면 시스템의 성능과 안정성을 크게 향상시킬 수 있습니다.

## 2. Cassandra의 특징

### 2.1. 아키텍처
- **분산 NoSQL 데이터베이스**: P2P(Peer-to-Peer) 아키텍처를 기반으로 모든 노드가 동등한 역할을 수행
- **Wide-Column Store**: 컬럼 패밀리 기반의 데이터 모델로 유연한 스키마 지원
- **Masterless 구조**: 단일 장애점(SPOF)이 없어 고가용성 보장
- **분산 해시링(Consistent Hashing)**: 데이터를 여러 노드에 균등하게 분산

### 2.2. 장점
- **선형적 확장성(Linear Scalability)**: 노드를 추가하면 처리량이 선형적으로 증가
- **높은 가용성**: 복제 팩터(Replication Factor) 설정을 통한 데이터 복제
- **대용량 데이터 처리**: 페타바이트급 데이터를 처리할 수 있는 능력
- **쓰기 최적화**: LSM Tree 구조로 빠른 쓰기 성능
- **지리적 분산**: 데이터센터 간 복제를 통한 글로벌 분산 지원
- **튜닝 가능한 일관성**: Eventual Consistency부터 Strong Consistency까지 조정 가능

### 2.3. 단점
- **복잡한 운영**: 클러스터 관리, 튜닝, 모니터링이 복잡
- **읽기 성능**: 쓰기에 비해 읽기 성능이 상대적으로 느림 (특히 랜덤 읽기)
- **메모리 사용량**: 효율적인 성능을 위해 많은 메모리 필요
- **학습 곡선**: CQL(Cassandra Query Language) 및 데이터 모델링 학습 필요

## 3. Redis의 특징

### 3.1. 아키텍처
- **인메모리 Key-Value Store**: 모든 데이터를 메모리에 저장하여 초고속 접근
- **단일 스레드 모델**: 명령어 처리는 단일 스레드로 수행 (I/O는 멀티플렉싱)
- **다양한 데이터 구조**: String, Hash, List, Set, Sorted Set, Bitmap, HyperLogLog 등 지원
- **Pub/Sub 메시징**: 실시간 메시징 기능 제공

### 3.2. 장점
- **매우 빠른 성능**: 메모리 기반으로 마이크로초 단위의 지연시간
- **다양한 데이터 타입**: 복잡한 자료구조를 네이티브로 지원
- **간단한 사용법**: 직관적인 명령어와 API
- **원자적 연산**: 모든 명령어가 원자적으로 실행
- **TTL 지원**: 키에 만료 시간 설정 가능
- **Lua 스크립트**: 복잡한 로직을 서버 사이드에서 실행 가능

### 3.3. 단점
- **메모리 제약**: 모든 데이터가 메모리에 있어야 하므로 용량 제한
- **영속성 트레이드오프**: RDB/AOF 방식의 영속성은 성능과 내구성 사이의 균형 필요
- **단일 스레드**: CPU 집약적 작업 시 전체 성능에 영향
- **확장성 제한**: 클러스터 모드를 사용해도 Cassandra만큼의 확장성은 없음

## 4. 비교표

| 특성 | Cassandra | Redis |
|-----|----------|-------|
| 저장 방식 | 디스크 기반 (메모리 캐시 활용) | 인메모리 (선택적 디스크 영속화) |
| 데이터 모델 | Wide-Column Store | Key-Value + 다양한 자료구조 |
| 성능 | 높은 쓰기 처리량, 중간 수준 읽기 | 매우 빠른 읽기/쓰기 (마이크로초) |
| 확장성 | 페타바이트급 (수평 확장 우수) | 기가~테라바이트급 (제한적) |
| 일관성 | 튜닝 가능 (Eventual ~ Strong) | Strong Consistency (단일 인스턴스) |
| 복제 | 다중 데이터센터 복제 지원 | Master-Slave 복제, Sentinel, Cluster |
| 쿼리 언어 | CQL (SQL-like) | 명령어 기반 |
| 주 사용 목적 | 대규모 쓰기 중심 워크로드 | 캐싱, 세션 관리, 실시간 분석 |

## 5. Redis를 Cassandra의 보조 저장소로 활용하는 방법

### 5.1. 캐싱 레이어로 활용
Redis를 Cassandra 앞단의 캐시로 배치하여 읽기 성능을 극대화할 수 있습니다.

**아키텍처 패턴:**
```
Application → Redis (Cache) → Cassandra (Primary Store)
```

**구현 방식:**
1. **Cache-Aside Pattern**: 
   - 애플리케이션이 먼저 Redis에서 데이터 조회
   - 캐시 미스 시 Cassandra에서 조회 후 Redis에 저장
   - TTL 설정으로 오래된 데이터 자동 삭제

2. **Write-Through Pattern**:
   - 데이터 쓰기 시 Redis와 Cassandra에 동시에 저장
   - 읽기 시 항상 Redis에서 조회

3. **Write-Behind Pattern**:
   - 데이터를 먼저 Redis에 쓰고 비동기적으로 Cassandra에 저장
   - 높은 쓰기 처리량이 필요한 경우 유용

### 5.2. 세션 스토어로 활용
사용자 세션 데이터를 Redis에 저장하고, 영구 저장이 필요한 데이터만 Cassandra에 저장합니다.

**활용 예시:**
- Redis: 사용자 로그인 세션, 임시 토큰, 실시간 상태
- Cassandra: 사용자 프로필, 이력 데이터, 영구 기록

### 5.3. 실시간 집계 및 분석
Redis의 Sorted Set, HyperLogLog 등을 활용한 실시간 집계와 Cassandra의 장기 저장을 결합합니다.

**활용 예시:**
- Redis: 실시간 순위, 최근 N개 이벤트, 카운터
- Cassandra: 과거 데이터, 배치 집계 결과

### 5.4. 메시지 큐 및 Pub/Sub
Redis의 Pub/Sub, Stream을 활용하여 이벤트 기반 아키텍처를 구성하고, Cassandra에 이벤트를 영구 저장합니다.

**활용 예시:**
- Redis Streams: 이벤트 스트리밍, 실시간 처리
- Cassandra: 이벤트 로그의 영구 저장소

### 5.5. 구현 시 고려사항

**데이터 일관성:**
- Redis와 Cassandra 간 동기화 전략 수립 필요
- 캐시 무효화(Cache Invalidation) 정책 설계
- 장애 발생 시 데이터 불일치 처리 방안

**성능 최적화:**
- Redis 메모리 사용량 모니터링 및 eviction policy 설정
- Cassandra의 읽기 경로 최적화 (Bloom Filter, Compaction Strategy)
- 네트워크 지연을 고려한 타임아웃 설정

**운영 관리:**
- Redis 장애 시 Cassandra로 직접 조회하는 fallback 로직
- Redis 데이터 백업 및 복구 전략
- 모니터링 및 알림 설정

## 6. 결론

Cassandra와 Redis는 서로 보완적인 관계로 활용할 수 있습니다:

- **Cassandra**: 대용량 데이터의 영구 저장소, 높은 쓰기 처리량, 분산 환경
- **Redis**: 초고속 캐시, 실시간 데이터 처리, 세션 관리

두 시스템을 함께 사용하면:
1. 읽기 성능을 Redis로 대폭 향상
2. 대용량 데이터 저장은 Cassandra로 처리
3. 비용 효율적인 메모리 사용 (핫 데이터만 Redis에)
4. 장애 복구력 향상 (Redis 장애 시 Cassandra로 fallback)

적절한 캐싱 전략과 데이터 일관성 관리를 통해 두 시스템의 장점을 최대한 활용할 수 있습니다.
