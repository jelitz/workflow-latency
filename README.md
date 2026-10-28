# workflow-latency

Claude Code의 Workflow 도구로 스크립트를 쓸 때 지연을 키우는 요인을 줄이는 체크리스트 skill입니다. 번들 `workflow-authoring` 스킬은 스크립트 API와 품질 패턴을 다루고, 이 skill은 그 위에서 에이전트 프롬프트, 슬라이스, 긴 명령, 검증 루프를 점검합니다.

## 설치

```
/plugin marketplace add jelitz/workflow-latency
/plugin install workflow-latency@workflow-latency
```

## 사용

Workflow 스크립트를 쓰기 전에 `workflow-authoring`을 로드한 다음 이 skill을 호출합니다. 번들 스킬은 이름으로만 로드되도록 지정되어 있어서 이 skill은 자동으로 따라오지 않습니다. 항상 호출하려면 CLAUDE.md에 한 줄을 넣어 두세요.

```
- Workflow 스크립트를 쓰기 전에 `workflow-authoring`과 함께 `workflow-latency:workflow-latency` skill을 호출한다
```

## 담긴 내용

1. 에이전트 프롬프트의 실행 방식(Read·Grep·Glob 우선, Bash는 필요할 때만)을 상수로 고정
2. 120초를 넘길 수 있는 명령은 처음부터 `run_in_background`와 Monitor로 실행
3. 슬라이스는 열거된 목록으로만 정의하고, "그 외 전부" 슬라이스를 만들지 않음
4. 입력 자료를 에이전트가 Read 한두 번에 읽을 크기의 파일로 분할
5. 검증 → 수정 → 재검증은 blocker·major가 남은 동안만 반복
6. 동시 실행 상한 16에서는 더 쪼개도 줄지 않으므로 직렬 구간부터 확인

각 항목의 근거 수치와 코드 예시는 [SKILL.md](skills/workflow-latency/SKILL.md)에 있습니다.

## 근거와 한계

수치는 2026-10-27 하루, Windows PC 1대에서 측정한 워크플로 13개·에이전트 178개의 기록입니다. 느린 에이전트 63개 중 34개가 한 워크플로에서 나왔고, 호출 수와 소요 시간은 서로를 거의 그대로 반영하므로 원인으로 읽기는 어렵습니다. 셸 호출 비용은 PC와 동시 부하에 따라 달라집니다. 다른 환경에서는 직접 다시 재 보세요.

## 라이선스

MIT
