> context_tier: 200k

이슈 트리아지부터 에이전트 실행까지 전체 파이프라인을 자동으로 진행합니다.

인자: `$ARGUMENTS` (선택)
- 미지정: 승인 게이트 없이 끝까지 자동 실행
- `--confirm`: 2개 승인 게이트에서 멈춤

## 중단 후 재실행

컨텍스트/토큰 소진으로 중단된 경우, 새 세션에서 `/basic-worktree-clean` → `/basic-pipeline` 재실행.
파이프라인은 멱등(idempotent)하게 설계되어 있다:
- 닫힌 이슈는 트리아지에서 제외
- 명세 있는 이슈는 spec에서 건너뜀
- MIS는 현재 open 이슈 기준으로 재계산

**기존 PR 감지**: Phase 1에서 `gh pr list --state open`으로 이전 실행의 draft/open PR을 확인한다. 해당 이슈는 "PR 진행 중"으로 표시하고 중복 실행을 방지한다.

## 파이프라인 흐름

```
/basic-nextplan (트리아지)
  ↓
[--confirm 시 승인 게이트 1] "이 이슈들 진행?"
  ↓
/basic-spec (명세 작성)
  ↓
/basic-review-issues (계획 수립)
  ↓
[--confirm 시 승인 게이트 2] "이 배치로 실행?"
  ↓
  ├─ 이슈 ≤3개 → 직접 구현 → PR → CI 확인 → 머지 (Phase 4-B)
  └─ 이슈 >3개 → /basic-agents (Phase 4)
  ↓
/basic-worktree-clean (정리)
```

## 실행 방식

### 이슈 수 경고

| 워커가능 이슈 수 | 조치 |
|-----------------|------|
| ~100개 | 정상 진행 |
| 100~150개 | ⚠️ 주의: 컨텍스트 소모 증가. spec 완료 후 새 세션에서 review-issues 이후 진행 권장 |
| 150개+ | 🔴 경고: 세션 분할 강력 권장. spec을 그룹별 분할 실행 |

### Phase 1: 트리아지 (/basic-nextplan)

1. open 이슈 전체 조회 (200개)
2. 중복 탐지 → 자동 닫기 (--confirm 시 확인 요청)
3. 카테고리 분류 (워커가능/대규모/외부의존/완료)
4. 우선순위 태깅 (🔴/🟡/🟢) — 선정하지 않음, 태깅만
5. 워커 실행 가능 이슈 **전체** 목록 확정 (걸러내지 않음)

--confirm 모드: 이슈 목록을 보여주고 승인 대기.
자동 모드: 워커 실행 가능 이슈 전부 다음 단계로.

### Phase 2: 명세 (/basic-spec)

1. **워커가능 전체** 이슈 대상 (명세 없는 것만)
2. 이미 명세 있는 이슈는 건너뜀 (멱등)
3. 코드 분석 → GitHub 이슈 본문에 명세 기록
4. 대량이면 Opus Agent 그룹 병렬 (4개씩)

**핵심:** triage가 선정하지 않으므로 워커가능 전체를 spec한다. spec은 멱등 + 재사용 가능 → 초기 투자 후 이후 pipeline에서 Phase 2 스킵.

### Phase 3: 계획 (/basic-review-issues)

1. 명세 있는 이슈의 Write 파일 매핑 (파싱)
2. **워커가능 전체 풀**에서 충돌 분석 + MIS 계산 → 독립 배치 최대화
3. 실행 방식 결정 (직접 vs agents)
4. 배치 구성 (배치당 상한 5~6개) + 리뷰 레벨 지정 (L1/L2)

직접 처리 판정 시 (이슈 3개 이하):
  → Phase 4 건너뛰고 직접 구현 시작.
  → 구현 → PR 생성 → CI 확인 → 머지/보고 (Phase 4-B).

--confirm 모드: 배치 구성을 보여주고 승인 대기.
자동 모드: 바로 Phase 4로.

### Phase 4: 실행 (/basic-agents)

**`/basic-agents` 커맨드를 그대로 실행한다. 자체 구현 금지.**
Phase 3의 배치 구성 + 리뷰 레벨을 `/basic-agents`에 전달하면 된다.

`/basic-agents`가 수행하는 것:
1. Worktree 사전 생성 + 격리 검증
2. 오케스트레이터 프롬프트 생성
3. 백그라운드 병렬 실행 (polling 금지 — 알림 대기)
4. 리뷰어 검증 (PASS/FAIL/BLOCKED)
5. PR 생성 + CI 확인
6. 머지 (사용자 명시 지시 시에만)
7. 작업 보고서 작성

### Phase 4-B: 직접 처리 후속 (직접 처리 경로)

Phase 3에서 직접 처리로 판정된 경우, 구현 완료 후 실행:

1. **L1 리뷰 수행 (권장)**: PR diff와 이슈 명세를 대조하여 명세 충족 여부, 누락/과잉 구현, 부작용 확인.
   - 직접 처리 경로도 basic-agents와 동일한 검증 기준을 적용한다.
   - 이슈가 보안/아키텍처 관련이면 L2 수준 검토 권장.
2. `gh pr list`로 생성된 PR들의 CI 상태 확인
3. CI + Vercel 전부 통과 확인
4. 자동 모드: `gh pr merge --squash`로 즉시 머지 (`--delete-branch` 미사용)
5. --confirm 모드: CI 결과 보고 후 사용자 승인 대기 → 승인 시 머지
6. `git pull origin main` + 로컬 브랜치 정리

### Phase 5: 정리 (/basic-worktree-clean)

basic-agents 완료 후 (성공/실패 무관) 잔여 worktree를 정리한다.
- `git worktree list`로 확인
- main 외 worktree 제거
- `.claude/worktrees/` 잔여 디렉토리 삭제
- 머지 완료된 feature 브랜치 정리

### 완료 보고

```
## Pipeline 완료

Phase 1 (트리아지): N개 open → K개 워커 실행 가능
Phase 2 (명세): K개 중 M개 신규 명세 작성
Phase 3 (계획): MIS P개, Q개 배치
Phase 4 (실행): R개 PR 생성

결과:
- ✅ 머지 완료: PR #A, #B, #C
- ❌ BLOCKED: #D (사유)
- ⚠️ CI 실패: #E (로그 URL)

보류 이슈: #X, #Y (대규모/외부의존)
```
