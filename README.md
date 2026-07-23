# Gitkeeper

PR 단계에서 시크릿 유출과 코드 취약점을 자동으로 탐지하고, 위험도에 따라 다르게 대응하는 GitHub 보안 파이프라인입니다.

## 왜 만들었나

AI가 코드를 대신 짜주는 시대에는 "일단 동작만 하게" 짜인 코드에 하드코딩된 크리덴셜, 인증 우회 코드, SQL Injection 같은 위험이 쉽게 숨어듭니다. Gitkeeper는 커밋 단계와 PR 단계에서 이런 위험을 자동으로 걸러내고, 진짜 위험한 것만 담당자에게 빠르게 알리는 걸 목표로 합니다.

## 현재 구현된 기능

- GitHub Actions 기반 PR 테스트 파이프라인 (`.github/workflows/test.yml`)
- pytest 실행 → 실패 시 로그를 AI(Groq API)로 분석해서 원인 요약 + 심각도(`[심각]/[주의]/[사소]`) 분류
- 분석 결과를 Slack으로 자동 알림

## 설계 중인 아키텍처

### 방어선 2단계

| 단계 | 위치 | 역할 | 강제력 |
|---|---|---|---|
| 1차 | Pre-commit (로컬) | 빠른 1차 경고, 라이브 검증 없음 | 없음 (우회 가능) |
| 2차 | CI/CD (GitHub Actions) | 최종 관문, 라이브 검증 포함 | 있음 (required check) |

### 탐지 엔진 2개, 병렬 실행

**Track A — 시크릿 탐지** (TruffleHog 스타일)
정규식 + 엔트로피로 하드코딩된 키/토큰 후보를 찾고, 발견 시 해당 서비스 API에 직접 요청해 실제로 살아있는 크리덴셜인지 라이브 검증합니다.

```
탐지 → merge 임시 차단(in_progress) → 비동기 라이브 검증
   ├─ 확인된 무효 → 차단 해제 (경고 코멘트만 남김)
   ├─ 확인된 유효 → 계속 차단 + 긴급 알림 (즉시 키 회전 요청)
   └─ 검증 불가  → 계속 차단, 사람이 직접 확인
```

**Track B — 코드 취약점 탐지** (Semgrep/Bandit 스타일)
AST를 파싱해서 SQL/Command Injection, 안전하지 않은 역직렬화, 취약한 해싱 등 구조적 패턴을 탐지합니다. 라이브 검증이 필요 없어 즉시 결과가 나옵니다.

```
탐지 → 즉시 실패 처리 → PR diff에 인라인 코멘트
   └─ 오탐으로 판단되면 → 보안 담당자가 승인 라벨 부착
        → 별도 워크플로우가 Checks API로 체크 상태를 success로 갱신
```

## 핵심 설계 원칙

1. **목표는 "코드에서 지우는 것"이 아니라 "크리덴셜을 무효화하는 것"입니다.** 시크릿은 push된 순간 이미 노출된 것으로 간주하고, merge 차단보다 즉시 알림(키 회전 요청)을 우선합니다.
2. **애매하면 막습니다 (fail-closed).** 라이브 검증이 실패하거나 결론이 안 나면 통과시키지 않고 사람이 확인하게 합니다.
3. **Pre-commit은 예방 보조 수단이지 강제 수단이 아닙니다.** `--no-verify`로 우회 가능하므로, 실질적인 강제력은 CI의 required check에만 있습니다.

## 역할 분담

| 담당 | 영역 |
|---|---|
| 친구 | Track A/B 탐지 로직 코어, 검수 항목 Tier 분류 |
| 나 | GitHub Actions 연동, Checks API 상태 관리, 알림 발송 |

## 참고 자료

- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — 시크릿 탐지 + 라이브 검증 방식 참고
- [Semgrep](https://semgrep.dev) — AST 기반 커스텀 룰 작성 방식 참고
- [Bandit](https://bandit.readthedocs.io) — Python 정적 분석 룰 구조 참고
- [GitHub Push Protection](https://docs.github.com/en/code-security/secret-scanning) — 예방/탐지 단계 구분 참고

## 로드맵

- [ ] 탐지 로직을 `scanner secrets` / `scanner injection` 서브커맨드를 가진 CLI로 통합
- [ ] `.pre-commit-config.yaml` 작성 및 팀 배포
- [ ] Track A/B를 CI에서 별도 job으로 분리 + Checks API 연동
- [ ] 라이브 검증 모듈 구현 (AWS, GitHub, Slack 등)
- [ ] 라벨 트리거 기반 오탐 예외 처리 워크플로우
