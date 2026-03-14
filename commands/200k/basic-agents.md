> context_tier: 200k

멀티 에이전트 시스템을 실행합니다. 배치 계획을 받아서 실행만 합니다.

프로젝트에 `docs/agent-system/system-spec.md` 또는 유사한 에이전트 명세 파일이 있으면 해당 프로토콜을 따릅니다. 없으면 아래 기본 프로토콜을 적용합니다.

사전 조건: `/basic-review-issues`로 배치 구성 + 리뷰 레벨이 확정된 상태.

## Step 0: 잔여 Worktree 복구 탐지

이전 실행이 중단된 경우를 감지한다:

```bash
git worktree list --porcelain | grep "^worktree" | grep "batch-" | while read _ path; do
  branch=$(git -C "$path" rev-parse --abbrev-ref HEAD)
  commits=$(git -C "$path" log --oneline origin/main.. 2>/dev/null | wc -l)
  pr=$(gh pr list --head "$branch" --json number --jq '.[0].number' 2>/dev/null)

  if [ "$commits" -gt 0 ] && [ -z "$pr" ]; then
    echo "RECOVERY: $path — $commits commits, PR 없음"
  fi
done
```

| 상태 | 조치 |
|------|------|
| 커밋 있음 + PR 없음 | push → PR 생성 → CI 확인 |
| 커밋 있음 + PR 있음 (draft/open) | PR 상태 확인 → 리뷰어 검증으로 이동 |
| 커밋 없음 | worktree 제거 후 새로 생성 |

복구할 배치가 없으면 Step 1로 진행.

## Step 1: Worktree 설정

배치별 worktree를 **순차적으로** 생성한다 (`isolation: "worktree"` 사용 금지 — 동시 생성 시 경합으로 main 오염 위험):

```bash
git worktree add .claude/worktrees/batch-a -b fix/batch-a
git worktree add .claude/worktrees/batch-b -b fix/batch-b
# 배치 수에 따라 추가
```

**격리 검증** — 생성 직후 반드시 확인:
```bash
git worktree list
# main 외 모든 배치가 .claude/worktrees/ 아래에 있는지 확인
# main repo가 fix/* 브랜치에 있으면 즉시 중단
```

logs/, proposals/, reviews/ 디렉토리 생성:
```bash
mkdir -p logs proposals reviews
```

**`logs/status.json` 초기화** — 배치 계획을 파일로 기록:

```json
{
  "startedAt": "2026-03-13T10:00:00+09:00",
  "batches": {
    "batch-a": {
      "agentId": null,
      "status": "pending",
      "unplannedWrites": [],
      "workers": {
        "#263": { "status": "pending", "pr": null, "review": null },
        "#264": { "status": "pending", "pr": null, "review": null }
      }
    },
    "batch-b": {
      "agentId": null,
      "status": "pending",
      "unplannedWrites": [],
      "workers": {
        "#295": { "status": "pending", "pr": null, "review": null },
        "#296": { "status": "pending", "pr": null, "review": null }
      }
    }
  },
  "mergeQueue": []
}
```

워커 상태값: `pending` → `in_progress` → `done` → `pr-created` → `reviewing` → `merged` / `BLOCKED` / `failed`.
세션이 중단되어도 이 파일로 현재 상태를 복원할 수 있다.

## Step 2: 오케스트레이터 프롬프트 생성

프로젝트에 오케스트레이터/워커/리뷰어 템플릿이 있으면 사용하고, 없으면 직접 작성한다.

각 오케스트레이터 프롬프트에 포함할 내용:
- {BATCH_NAME}: 배치 이름
- {BRANCH}: 브랜치명
- {WORKTREE_PATH}: worktree 절대경로
- {WORKERS}: 워커 목록 (이슈 번호 + Write 허용 파일 + 리뷰 레벨)
- {MERGE_PRIORITY}: 머지 순서 기준

오케스트레이터에게 **워커→리뷰어 1:1 페어 실행** 지시를 포함한다.

## Step 3: 백그라운드 병렬 실행

오케스트레이터를 동시에 백그라운드로 실행한다.

Agent tool 설정:
- subagent_type: "general-purpose"
- model: "opus"
- run_in_background: true

실행 후 agent ID를 배치별로 기록한다. `logs/status.json`의 각 배치에 `agentId`와 `"status": "running"` 업데이트.

## Step 4: 모니터링 안내

사용자에게 안내 후 **대기** (sleep/polling 금지 — task-notification 자동 알림 대기):
```
N개 배치가 백그라운드에서 실행 중입니다.

agent ID:
  Batch A: agent_xxxxx
  Batch B: agent_yyyyy

진행 확인: 아무 메시지나 보내주시면 현황 보고드립니다.
BLOCKED 발생 시 즉시 알려드립니다.
```

**금지**: sleep + check 루프. output 파일은 완료 전까지 항상 0 bytes이므로 polling 무의미.

## Step 5: 완료 처리 + 리뷰어 검증

오케스트레이터 완료 알림 수신 시:

### 5-1. 리뷰어 검증

각 워커 완료 후, 오케스트레이터가 리뷰어를 1:1로 실행한다.

**리뷰어 실행 흐름**:
```
Worker (Sonnet) → Draft PR 생성
  ↓
Reviewer (L1→Sonnet / L2→Opus) → 명세 검증
  ├── PASS → gh pr ready (Draft 해제) → 머지 대기열
  ├── FAIL (1회차) → 수정 지시서 → Worker 재실행 → Reviewer 재실행
  └── FAIL (2회차) → BLOCKED → 본체 에스컬레이션
```

**리뷰어 입력**:
- GitHub 이슈 본문 (명세)
- PR diff (`gh pr diff`)

**검증 항목**:

| # | 체크 | 설명 |
|---|---|---|
| 1 | 명세 충족 | 이슈 체크리스트가 코드에 반영됐는가 |
| 2 | 누락 확인 | 명세에 있는데 구현 안 된 항목 |
| 3 | 과잉 구현 | 명세에 없는데 추가된 코드 |
| 4 | 부작용 | 기존 기능에 영향 주는 변경 |
| 5 | (L2만) 아키텍처 정합성 | 프로젝트 패턴/구조와 일치하는가 |

**판정**:

| 결과 | 후속 |
|------|------|
| PASS | 머지 대기열에 추가 |
| FAIL (1회차) | 수정 지시서 작성 → 워커 재실행 |
| FAIL (2회차) | BLOCKED → 본체 에스컬레이션 |

**리뷰 출력물**: `reviews/{issue}-{timestamp}.md`

### 5-1b. 명세 외 Write 추적

각 워커 완료 후, 오케스트레이터는 실제 변경 파일과 명세의 Write 파일을 비교한다:

```bash
# 워커가 실제로 변경한 파일
git diff --name-only main..
# vs 명세의 Write 파일 목록
```

명세에 없던 파일이 변경/생성된 경우, `logs/status.json`의 해당 배치 `unplannedWrites` 배열에 기록:

```json
"unplannedWrites": ["src/components/layout/price-refresh-button.tsx"]
```

이 정보는 충돌 발생 시 원인 추적용이다. 자동 차단하지 않음.

### 5-2. PR 처리 및 CI 확인

1. 각 오케스트레이터 완료 리포트 확인
2. Proposal 목록 검토 및 처리 결정
3. 테스트 통과 여부 확인
4. Worktree 정리:
   ```bash
   git worktree remove .claude/worktrees/batch-a
   git worktree remove .claude/worktrees/batch-b
   ```
5. CI/CD 배포 상태 확인 (프로젝트 CI에 맞게 — GitHub Actions, Vercel 등):
   ```bash
   gh pr checks <PR_NUMBER> --watch --interval 30
   ```
   - ✅ 통과: 머지 준비 완료
   - ❌ 실패: 실패한 체크 이름 + 로그 URL 함께 보고
6. 사용자에게 최종 보고:
   ```
   PR 목록 및 CI 결과:
   - PR #N (Batch A): ✅ 리뷰 PASS + CI 통과 — 머지 가능
   - PR #M (Batch B): ❌ 리뷰 FAIL (BLOCKED) — 수동 확인 필요
   - PR #K (Batch B): ⚠️ 리뷰 PASS + CI 실패 — <체크 이름> 실패, 로그: <URL>

   머지 순서: #N → #K (의존성 순)
   ```

**기본값: Claude는 머지하지 않습니다. 사용자가 명시적으로 "머지해줘" 지시 시에만 의존성 순서대로 `gh pr merge --squash` 실행.** (`--delete-branch` 미사용 — 브랜치 삭제는 Phase 5 worktree-clean에서 일괄 처리.)

PR 생성/머지 시 `logs/status.json` 업데이트: 해당 워커의 `"status": "pr-created"/"merged"`, `"pr": <PR번호>`, `"review": "PASS"/"FAIL"`. 머지된 워커는 `mergeQueue`에 추가.

## Step 6: 작업 보고서 작성

머지 완료 후 (또는 사용자가 머지 확인 후) `docs/reports/YYYY-MM-DD-{주제}.md` 작성 (경로가 없으면 프로젝트 루트에 생성):

```
# 작업 보고서: {주제}

날짜 / 처리 이슈 / PR / 실행 방식

## 워크플로우 의사결정 (이번 세션에서 새로 결정된 것)
- 사용자가 결정한 것: 어떤 맥락에서 왜 결정했는지
- Claude가 제안하고 사용자가 승인한 것

## 이슈별 상세
각 이슈마다: 문제 → 의사결정 근거 → 변경 파일 → 리뷰 결과 (L1/L2, PASS/FAIL) → 추후 주의사항

## 진행 과정
배치 구성 / 테스트 결과 / 리뷰 결과 / CI 결과 / 머지 과정 (충돌·문제 포함)

## 잔여 위험 및 후속 과제
```

작성 후 main에 직접 push (docs 변경):
```bash
git add docs/reports/YYYY-MM-DD-{주제}.md
git commit -m "docs: YYYY-MM-DD {주제} 작업 보고서 추가"
git push
```
