# Architecture Decision Records

설계 판단을 기록합니다. 판단의 **근거와 대안**을 남기는 것이 목적이며, 결론만 적는 문서가 아닙니다.

## 규칙

- 파일명은 `NNNN-<주제>.md` 형식입니다. **제목에 결론을 담지 않습니다.**
  - 나쁜 예: `0001-no-rtos-on-edge-nodes.md`
  - 좋은 예: `0001-scheduling-model-for-edge-nodes.md`
- 측정이 필요한 판단은 측정 전에 결론을 쓰지 않습니다. `Status: Proposed`로 두고 데이터가 나온 뒤 `Accepted`로 바꿉니다.
- 나중에 판단이 뒤집히면 기존 ADR을 지우지 않고 `Superseded by NNNN`으로 표시합니다.

## 구조

```markdown
# NNNN. <주제>

- Status: Proposed | Accepted | Superseded by NNNN
- Date: YYYY-MM-DD
- Phase: <해당 Phase>

## Context
무엇을 결정해야 하는가, 왜 지금인가

## Measurement
측정 방법, 조건, 원시 데이터 위치. 측정이 불필요하면 "N/A — 사유" 명시

## Analysis
데이터가 무엇을 의미하는가

## Options
검토한 대안과 각각의 trade-off

## Decision
선택과 그 이유

## Consequences
받아들이는 비용, 남는 위험, 재검토 조건
```

## 목록

| ADR | 주제 | Status |
|---|---|---|
| [0001](0001-scheduling-model-for-edge-nodes.md) | Edge ECU scheduling model (superloop vs RTOS) | Proposed |
