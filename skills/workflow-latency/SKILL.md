---
name: workflow-latency
description: Workflow 도구용 스크립트를 쓰기 직전에 workflow-authoring과 함께 호출한다. 에이전트 프롬프트의 실행 방식, 슬라이스 정의, 120초 넘는 명령, 검증 루프처럼 워크플로를 느리게 만드는 요인을 줄이는 절차다.
---

# Workflow 지연 방지 체크리스트

번들 `workflow-authoring`(스크립트 API, 품질 패턴)을 보완한다. 스크립트를 쓰는 동안 아래 여섯 항목을 점검한다.

측정 근거: 2026-10-27 하루, Windows PC 1대, 워크플로 13개·에이전트 178개. 에이전트 시간의 약 60%가 도구 실행이었고 대부분 Bash/PowerShell이었다. Bash 호출 중앙값은 약 17초, Read는 0.3초였다. 프로세스 하나를 띄우는 데 1~3초가 들었고, 에이전트 21~40개가 동시에 돌 때 셸 호출 중앙값은 약 30초였다. 느린 에이전트 63개 중 34개가 한 워크플로에서 나왔으므로 숫자는 재측정이 필요하다.

## 1. 실행 방식을 프롬프트 상수로 고정한다

모든 `agent()` 프롬프트 앞에 같은 문구를 붙인다. 기계적 단계(목록, 추출, 분류)는 `sonnet` + `effort: 'low'`로 두고, 판정·적대적 검증 단계는 세션 모델과 높은 effort를 유지한다. 측정한 실행에서는 리뷰 에이전트 36개가 모두 effort high로 돌아 중앙값 35분이 걸렸다.

```js
const EXEC = '실행 방식: 파일 읽기와 검색은 Read·Grep·Glob으로 한다. Bash는 실행이 필요할 때만 쓴다. '
  + '셸 for 루프로 외부 명령을 반복하지 말고 python 한 번으로 묶는다. '
  + '120초를 넘길 수 있는 명령은 run_in_background로 돌리고 Monitor로 확인한다. 타임아웃 뒤 sleep 폴링을 하지 않는다.\n\n'
const ask = (prompt, opts) => agent(EXEC + prompt, opts)
const CHEAP = { model: 'sonnet', effort: 'low' }
```

## 2. 120초를 넘길 수 있는 명령은 처음부터 백그라운드로

전체 테스트, playwright, SSH 로그 스캔, 대용량 zgrep이 해당한다. 타임아웃 → 백그라운드 전환 → sleep 폴링 순서로 흘러간 사례가 36건이었고 에이전트당 24~27분을 잃었다. 과업 문장에도 "처음부터 run_in_background + Monitor"를 적는다.

## 3. 슬라이스는 열거된 목록으로만 정의한다

"그 외 전부" 슬라이스는 같은 단계의 다른 슬라이스 중앙값의 6.8배, 5.2배가 걸렸다. 목록이 없으면 싼 에이전트로 먼저 목록을 만들고 그 목록으로 쪼갠다.

```js
const LIST = { type: 'object', properties: { items: { type: 'array', items: { type: 'string' } } }, required: ['items'] }
const listed = await ask('스키마의 모든 테이블을 항목 하나에 테이블 하나로 나열하라.', { ...CHEAP, schema: LIST })
const results = await pipeline(listed.items, item => ask(`테이블 ${item}을 분석하라.`, { label: item, schema: RESULT }))
```

## 4. 입력 자료는 항목별 파일로 나눠 준다

에이전트가 Read 한두 번으로 읽을 수 있는 크기여야 한다. 0.8~2.3MB 단일 마크다운을 sed로 약 1,600번 조회한 에이전트 36개는 중앙값 35분이 걸렸다. 300초 제한이 있는 쿼리는 날짜나 경로로 미리 나누고 과업에 그 분할을 적는다.

## 5. 검증 → 수정 → 재검증은 blocker·major가 남은 동안만 반복한다

반복 횟수에 상한을 두지 않는다. minor만 남으면 끝낸다. 수정 단계도 구현 단계와 같은 파일 소유 분할로 병렬화하고, 재검증은 수정한 파일과 회귀 표본으로 한정한다. 측정한 85분짜리 루프는 개인정보 노출 blocker 1건을 포함해 결함 43건을 실제로 잡았다.

```js
const open = fs => fs.filter(f => f.severity === 'blocker' || f.severity === 'major')
let found = open(await verifyAll())
for (let round = 1; found.length; round++) {
  log(`${round}회차: blocker/major ${found.length}건`)
  const byOwner = {}
  found.forEach(f => { (byOwner[f.owner] = byOwner[f.owner] || []).push(f) })
  await parallel(Object.entries(byOwner).map(([owner, fs]) => () =>
    ask(`${owner}이 소유한 파일만 고친다: ${JSON.stringify(fs)}`, { label: `fix:${owner}` })))
  found = open(await reverify(Object.keys(byOwner)))
}
```

## 6. 쪼개기는 직렬 구간이 클 때만 효과가 있다

동시 실행 상한은 min(16, CPU 수 - 2)다. 이미 16개를 동시에 돌리는 fan-out은 더 쪼개도 벽시계 시간이 줄지 않는다. 단계 사이 대기나 에이전트 한두 개가 지배하는 구간을 먼저 찾는다.
