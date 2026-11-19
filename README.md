# 환불 요청 중계 서버 — 모범 답안

> 환불 요청 중계 서버 과제의 **모범 답안**입니다.
> 정답은 하나가 아니며, 이 코드는 좋은 구현의 한 예시입니다. 과제를 풀어 본 뒤 설계와 비교해 보거나, 평가에 참고할 수 있습니다.
> 평가 기준은 `평가기준표.md`에 있습니다.

과제(지원자용) 저장소: [refund-relay-assignment](https://github.com/DongHyunKIM-Hi/refund-relay-assignment)

| | |
|---|---|
| 평가 역량 | 설계력 |
| 주요 과제 | 느리거나 실패하는 외부 결제대행사 연동, 이중 환불 방지 |
| 제공 가상 서버 | 결제대행사 `ghcr.io/donghyunkim-hi/pg-mock:1.1.1` (포트 9090) |
| 실행에 필요한 것 | JDK 17, Docker (결제대행사) |

## 단계별 구현 (커밋으로 구분)

정답 코드는 저장소 루트의 Gradle 프로젝트 하나입니다. 한 번에 완성하지 않고 **세 단계로 나눠** 만들어 가는 과정이 커밋으로 남아 있어, 각 단계에서 무엇이 되고 무엇이 안 되는지 눈으로 볼 수 있습니다.

| 단계 | 태그 | 내용 |
|---|---|---|
| 1 | `step1` | 정상 경로 — 접수 API와 워커가 분리되어 돌아가는 것까지. 검증도 재시도도 없음 |
| 2 | `step2` | 예외 처리 — 입력 검증, 에러 응답, 재시도, 시간 초과. 아직 동시성과 순서는 없음 |
| 3 | `step3` | 락의 범위와 처리 단위 설계 (최종, `main`과 같음) |

- 단계 이동: `git checkout step1` / `step2` / `step3` (`step3` = `main` = 최종)
- 단계마다 무엇이 추가되는지: `git diff step1 step2`, `git diff step2 step3`
- 단계별 설명과 실측 결과: `docs/stages.md`

각 단계에서 `bash check-scenarios.sh`를 돌려 보면 1단계는 4/13, 2단계는 10/13(동시성 S4, S6, S11이 깨짐), 3단계는 13/13으로 나옵니다. 돌아가는 것과 요구사항을 만족하는 것이 어떻게 다른지 숫자로 보입니다.

| 파일 | 내용 |
|---|---|
| `평가기준표.md` | 배점, 수준별 기준, 확인 방법, 치명 오류, 기록 양식 (100점 만점) |
| `check-scenarios.sh` | 시나리오 S1~S13을 한 번에 자동 확인하는 스크립트 |
| `docs/design.md` | 정답 코드의 설계 문서 |
| `docs/stages.md` | 단계별 구현 설명과 단계마다 실제 실행해 측정한 결과 |

## 실행

```bash
# 1) 가상 결제대행사 (포트 9090)
docker run -p 9090:9090 ghcr.io/donghyunkim-hi/pg-mock:1.1.1

# 2) 환불 서버 (포트 8080)
./gradlew bootRun

# 3) 테스트 9개 (접수 규칙 4, 처리 규칙 4, 실행 확인 1)
./gradlew test

# 4) 시나리오 S1~S13 자동 확인 → 13/13
bash check-scenarios.sh
```

- 결제대행사 주소는 환경변수 `PG_BASE_URL`로 바꿀 수 있습니다. (기본값 `http://localhost:9090`)

## 주요 설계 결정

| 주제 | 결정 |
|---|---|
| 중복 요청과 동시 요청 방지 | 주문(`orderId`) 단위 락 안에서 중복 확인, 한도 확인, 저장을 한 번에 처리합니다. `refundId`의 DB 유일 제약은 마지막 방어 수단입니다. |
| 접수와 처리 분리 | 접수 API는 저장 후 바로 202를 응답합니다. 결제대행사 호출은 별도 워커(스레드 풀)가 담당합니다. |
| 순서 처리 | 같은 주문은 앞 건이 끝나야 다음 건을 결제대행사에 보냅니다. 다른 주문은 동시에 처리합니다. |
| 재시도 | 실패하면 0.5초, 1초, 2초 간격으로 최대 3회 재시도합니다. (최대 총 4회 시도) 재시도할 때도 **항상 같은 `refundId`를 사용합니다.** |
| 시간 초과 | 결제대행사 응답을 5초까지 기다립니다. 5초가 지나면 `PG_TIMEOUT`으로 보고 같은 ID로 재시도합니다. 결제대행사는 같은 ID를 한 번만 처리하므로 이중 환불이 생기지 않습니다. |

- 설정값은 `src/main/resources/application.yml`의 `pg.*`, `worker.*`에 모여 있습니다.

## 검증 결과

정답 서버와 가상 결제대행사를 함께 실행하고 `bash check-scenarios.sh`로 확인했습니다. 시나리오 S1~S13 전부 통과(13/13)합니다. 주요 결과는 아래와 같습니다.

| 시나리오 | 결과 |
|---|---|
| S1 정상 접수 | 202, 0.08초 안에 응답 (결제대행사를 기다리지 않음) |
| S2 같은 ID, 같은 내용으로 재요청 | 200, `duplicate: true` |
| S3 같은 ID, 다른 내용 | 409 `REFUND_ID_CONFLICT` |
| S5, S6 한도 초과 | 422 `REFUND_LIMIT_EXCEEDED`, `REJECTED` 상태로 저장 |
| S7 입력 오류 | 400 `INVALID_REQUEST` |
| S9 재시도 (FAIL, FAIL, SUCCESS 순서로 응답) | `attemptCount: 3`, `COMPLETED` |
| S11 같은 주문 순차 처리 | 두 번째 건은 첫 번째 건의 응답을 받은 뒤에만 결제대행사로 전달됨 |
| S11 다른 주문 병렬 처리 | 세 주문의 요청이 같은 시각에 결제대행사로 전달됨 |
| S12 시간 초과 후 실제 처리됨 (처리에 12초 걸림, 5초에 시간 초과 후 재시도) | `attemptCount: 2`로 `COMPLETED`, 결제대행사 처리 기록 **정확히 1건** (이중 환불 없음) |
| S13 목록 조회 | 최신순 정렬 |

## 패키지 구조

```
com.practicefintech.refundrelay
├── RefundRelayApplication.java
├── common/
│   ├── entity/        # Refund (JPA 엔티티)
│   ├── enums/          # RefundStatus, FailureReason, PgCallResult
│   ├── exception/      # ErrorResponse, ApiExceptionHandler
│   ├── config/         # PgClientProperties, WorkerProperties
│   └── utils/          # PgClient (가상 결제대행사 호출)
└── domain/
    └── refund/
        ├── controller/
        ├── service/     # RefundService(접수, 중복 확인, 한도 확인), RefundWorker(처리 워커)
        ├── repository/
        └── model/
            ├── request/
            └── response/
```

- DTO는 record 대신 Lombok(`@Getter` 등)을 사용한 일반 클래스로 작성했습니다.

## 참고 사항

- 테스트는 9개입니다. 접수 규칙(중복, 충돌, 한도, 거절 건 재요청)은 `RefundServiceTest`, 처리 규칙(순서 보장, 재시도 횟수, 최종 실패, 같은 ID 재사용)은 `RefundWorkerTest`에서 스레드나 대기 없이 결정적으로 확인합니다. 평가 기준표 D4(테스트 코드)의 상 수준 예시를 겸합니다.
- 위 검증 결과는 서버를 직접 실행해 확인한 것입니다.

---

## 평가자 참고

이 아래는 평가에 참고하는 내용입니다.

- **정답은 하나가 아닙니다.** 평가 기준표는 결과뿐 아니라 설계를 선택한 근거도 평가합니다.
- 판정이 애매하면 낮은 점수를 주지 말고 **판정 보류**로 기록한 뒤 이유를 남깁니다.
- 기능 확인은 지원자 서버를 직접 실행하고 요청을 보내는 방식으로 진행합니다. `bash check-scenarios.sh`로 S1~S13을 한 번에 확인할 수 있습니다. 기대 동작이 헷갈리면 이 정답 코드를 실행해 같은 요청을 보내 보세요.
- 지원자가 바뀔 때마다 지원자 서버를 **새로 실행**합니다. 재시작하지 않으면 이전 확인의 데이터가 남습니다.
- 가상 서버의 관리자 기능을 켜려면 `docker run -p 9090:9090 -e ADMIN_ENABLED=true ghcr.io/donghyunkim-hi/pg-mock:1.1.1`로 실행합니다. 지원자에게 제공한 환경에서는 꺼져 있습니다.
