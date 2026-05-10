# Study Archive

Obsidian + GitHub 기반으로 관리하는 개발자 학습 아카이브입니다.

이 저장소는 단순 TIL 모음이 아니라, `지식 베이스 + 학습 로그` 구조를 목표로 합니다.  
Daily 문서는 학습 흐름을 남기고, Topic 문서는 기술과 개념을 누적형으로 정리합니다.

## 아카이브 원칙

| 항목 | 원칙 |
| --- | --- |
| 기록 방식 | Daily는 짧게, Topic은 누적형으로 깊게 |
| 연결 방식 | `[[WikiLink]]` 기반으로 Daily와 Topic 연결 |
| 문서 초점 | 무엇인지보다 왜 필요한지, 언제 쓰는지, 트레이드오프 중심 |
| 활용 목적 | 복습, 면접, 포트폴리오, 실무 회고 |
| 문서 스타일 | 핵심 위주, 비교 중심, 표 적극 활용 |

## 저장소 구조

```text
study-archive/
├── README.md
├── daily/
│   ├── README.md
│   └── YYYY-MM/
│       └── YYYY-MM-DD.md
├── cs/
│   ├── README.md
│   ├── network/
│   ├── os/
│   ├── database/
│   │   └── sqlp/
│   └── data-structure/
├── backend/
│   ├── README.md
│   ├── spring/
│   ├── jpa/
│   ├── redis/
│   ├── kafka/
│   ├── msa/
│   └── kubernetes/
├── infra/
│   └── README.md
├── android/
│   ├── README.md
│   ├── kotlin/
│   └── compose/
├── ai/
│   └── README.md
└── projects/
    └── README.md
```

## 문서 작성 규칙

### 1. Daily 문서

- 하루 학습 내용을 기록한다.
- 설명은 짧게 쓰고, Topic 문서로 연결한다.
- 회고는 한두 줄이면 충분하다.

형식:

```md
# YYYY-MM-DD

## 공부한 것
- 
- 

## 정리 링크
- [[문서명]]
- [[문서명]]

## 느낀 점
- 
```

### 2. Topic 문서

- 특정 기술이나 개념을 깊게 정리한다.
- 한 번 쓰고 끝내는 문서가 아니라, 나중에 계속 덧붙이는 누적형 문서로 관리한다.
- 실무 관점, 주의할 점, 자주 하는 실수까지 포함한다.

기본 형식:

```md
# 주제명

## 개념
## 왜 사용하는가
## 문제 상황
## 해결 방식
## 장단점
## 트레이드오프
## 실무에서의 사용
## 주의할 점
## 자주 하는 실수
## 관련 개념
## 참고 링크
```

### 3. 항상 포함할 관점

- 왜 등장했는가
- 어떤 문제를 해결하는가
- 기존 방식과 어떤 차이가 있는가
- 성능과 확장성에 어떤 영향을 주는가
- 실무에서는 어떤 상황에서 쓰는가
- 언제 쓰면 안 되는가

### 4. 네이밍 규칙

- 파일명은 `소문자-kebab-case.md`
- 예시:
  - `transaction-template.md`
  - `redis-zset-ranking.md`
  - `spring-transaction-propagation.md`

## 카테고리 목차

### Daily

- [daily guide](daily/README.md)
- [daily template](daily/daily-guide.md)
- [2026-05-09](daily/2026-05/2026-05-09.md)

### CS

- [cs index](cs/README.md)
- `cs/network/`
- `cs/os/`
- `cs/database/`
- `cs/database/sqlp/`
- `cs/data-structure/`

### Backend

- [backend index](backend/README.md)
- `backend/spring/`
- `backend/jpa/`
- `backend/redis/`
- `backend/kafka/`
- `backend/msa/`
- `backend/kubernetes/`

### Android

- [android index](android/README.md)
- `android/kotlin/`
- `android/compose/`

### Infra / AI / Projects

- [infra index](infra/README.md)
- [ai index](ai/README.md)
- [projects index](projects/README.md)

## 작성 흐름

1. Daily에 그날 공부한 내용을 짧게 적는다.
2. 핵심 개념은 적절한 카테고리의 Topic 문서로 분리한다.
3. Daily에서 Topic으로 `[[WikiLink]]`를 건다.
4. Topic 문서에서 관련 Topic을 다시 서로 연결한다.

## 템플릿 문서

- [topic template](topic-template.md)
- [daily guide](daily/daily-guide.md)

## 작성 기준

좋은 문서는 다음 조건을 만족해야 한다.

- 정의만 적지 않고, 왜 이 개념이 필요한지 설명한다.
- 실무에서 마주치는 문제 상황과 연결한다.
- 기존 방식과의 차이를 비교한다.
- 장점만 쓰지 않고, 비용과 한계도 같이 적는다.
- 다시 읽었을 때 빠르게 복습할 수 있게 구조화한다.
