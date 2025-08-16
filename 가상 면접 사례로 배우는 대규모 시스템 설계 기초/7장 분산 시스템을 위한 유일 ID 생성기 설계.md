## 분산 시스템을 위한 유일 ID 생성기 설계

분산 환경에서는 auto increment를 쓰기 어렵다. 유일하며 정렬 가능한 숫자 ID를 (+초당 1만개의 ID 생성이 가능) 어떻게 만들까?

- 다중 마스터 복제

- UUID

> 크기가 128비트로 길며, 숫자가 아닌 값이 포함되며 시간순으로 정렬할 수 없다.

- 티켓 서버

> 중앙집중형 데이터베이스 서버에서 auto increment를 쓰는 것. SPOF를 피하려면 새로운 문제를 해결해야 한다.

- 트위터 스노플레이크 접근법

> 64비트를 sign 비트, 41비트의 타임스탬프, 5비트의 데이터센터 ID, 5비트의 서버 ID, 12비트의 일련번호로 쪼갠다. 분산 환경을 구분하면서도 시간순 정렬이 가능하며, 64비트의 길이로 너무 길지 않다.

#### 애플리케이션에서, 데이터 센터 ID와 서버 ID를 하드코딩 하지 않고 외부에서 주입하는 편리한 방법?

Spring Boot를 예로 든다면,

```yml

    # 기본값 설정 (실행 시점에 덮어써야 함)
    snowflake:
    datacenter-id: 0
    worker-id: 0

```

```java

    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;

    @Configuration
    public class AppConfig {

        @Value("${snowflake.datacenter-id}")
        private long datacenterId;

        @Value("${snowflake.worker-id}")
        private long workerId;

        @Bean
        public SnowflakeIdGenerator snowflakeIdGenerator() {
            return new SnowflakeIdGenerator(datacenterId, workerId);
        }
    }

```

요렇게 자동구성에서 값 받아오고

```bash

    java -jar -Dsnowflake.datacenter-id=0 -Dsnowflake.worker-id=2 my-application.jar

```

실행 시 환경에 맞게 변수를 주입하는 스크립트를 구성해두면 된다. 도커나 쿠버네티스를 사용한다면 컨테이너 환경에서 환경 변수를 주입해 편리하게 배포가 가능할 것.

<br>

궁금해서 Gemini로 생성해 본 실제 구현 코드

```java

    import org.springframework.stereotype.Component;

    @Component
    public class SnowflakeIdGenerator {

        private final long datacenterId;
        private final long workerId;
        private long sequence = 0L;
        private long lastTimestamp = -1L;

        // 비트 수 정의
        private static final long DATACENTER_ID_BITS = 5L;
        private static final long WORKER_ID_BITS = 5L;
        private static final long SEQUENCE_BITS = 12L;

        // 최대값 정의
        private static final long MAX_DATACENTER_ID = (1L << DATACENTER_ID_BITS) - 1;
        private static final long MAX_WORKER_ID = (1L << WORKER_ID_BITS) - 1;

        // 비트 시프트 위치 정의
        private static final long WORKER_ID_SHIFT = SEQUENCE_BITS;
        private static final long DATACENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;
        private static final long TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS;

        // 시작 시점 (2025-01-01)
        private static final long EPOCH = 1735689600000L;

        // 생성자: ID들을 받아와 유효성 검사
        public SnowflakeIdGenerator(long datacenterId, long workerId) {
            if (datacenterId > MAX_DATACENTER_ID || datacenterId < 0) {
                throw new IllegalArgumentException("Datacenter ID can't be greater than " + MAX_DATACENTER_ID + " or less than 0");
            }
            if (workerId > MAX_WORKER_ID || workerId < 0) {
                throw new IllegalArgumentException("Worker ID can't be greater than " + MAX_WORKER_ID + " or less than 0");
            }
            this.datacenterId = datacenterId;
            this.workerId = workerId;
        }

        // ID 생성 메서드 (스레드 안전을 위해 synchronized 사용)
        public synchronized long nextId() {
            long timestamp = System.currentTimeMillis();

            if (timestamp < lastTimestamp) {
                throw new RuntimeException("Clock moved backwards. Refusing to generate id for " + (lastTimestamp - timestamp) + " milliseconds");
            }

            if (lastTimestamp == timestamp) {
                sequence = (sequence + 1) & ((1L << SEQUENCE_BITS) - 1);
                if (sequence == 0) {
                    timestamp = tilNextMillis(lastTimestamp);
                }
            } else {
                sequence = 0L;
            }

            lastTimestamp = timestamp;

            // 비트 연산을 통해 64비트 ID 조합
            return ((timestamp - EPOCH) << TIMESTAMP_LEFT_SHIFT) |
                (datacenterId << DATACENTER_ID_SHIFT) |
                (workerId << WORKER_ID_SHIFT) |
                sequence;
        }

        private long tilNextMillis(long lastTimestamp) {
            long timestamp = System.currentTimeMillis();
            while (timestamp <= lastTimestamp) {
                timestamp = System.currentTimeMillis();
            }
            return timestamp;
        }
}

```