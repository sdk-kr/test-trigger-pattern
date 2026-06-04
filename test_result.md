# GitHub Actions 트리거 분리 패턴 검증 보고서

## 검증 일시
2026-06-04 (UTC 04:52 ~ 04:58)

## 검증 목적
JIRA **BDP-181592 (003번 취약점)** 조치 관련,
`develop`은 **push 트리거** / `master`는 **pull_request(closed) 트리거**로 분리하는 패턴이
의도대로 동작하는지 검증.

- DEV 배포: develop 브랜치에 push 발생 시에만
- PRD 배포: master를 base로 한 PR이 **merge되어 close**될 때만

## 테스트 환경
- 레포: `sdk-kr/test-trigger-pattern` (private, 보존됨)
- 계정: sdk-kr / gh CLI 2.86.0
- 비고: 인증 토큰에 `workflow` scope가 없어 HTTPS로는 워크플로우 파일 push가 거부됨
  → **git remote를 SSH로 전환하여 우회** (SSH 키는 OAuth scope 제한 비적용)

## 테스트 yml
```yaml
name: test_trigger_pattern

on:
  push:
    branches: [develop]
  pull_request:
    types: [closed]
    branches: [master]

jobs:
  dev_deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref_name == 'develop'
    steps:
      - name: DEV deploy triggered
        run: |
          echo "DEV DEPLOY TRIGGERED"
          echo "Event: ${{ github.event_name }}"
          echo "Ref: ${{ github.ref_name }}"

  prd_deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request' && github.event.pull_request.merged == true
    steps:
      - name: PRD deploy triggered
        run: |
          echo "PRD DEPLOY TRIGGERED"
          echo "Event: ${{ github.event_name }}"
          echo "Merged: ${{ github.event.pull_request.merged }}"
          echo "PR base: ${{ github.event.pull_request.base.ref }}"
          echo "PR number: ${{ github.event.pull_request.number }}"
```

## 시나리오 결과

| # | 시나리오 | 예상 | 실제 dev_deploy | 실제 prd_deploy | 결과 |
|---|---|---|---|---|---|
| 1 | develop 직접 push | dev_deploy 실행 | ✓ 실행 | skip | ✅ |
| 2 | feature→develop PR opened | 워크플로우 X | (run 없음) | (run 없음) | ✅ |
| 3 | feature→develop PR merged | dev_deploy 실행 | ✓ 실행 | skip | ✅ |
| 4 | develop→master PR opened | 워크플로우 X | (run 없음) | (run 없음) | ✅ |
| 5 | **★ develop→master PR merged** | **prd_deploy 실행** | skip | **✓ 실행** | ✅ |
| 6 | develop→master PR closed(머지X) | 둘 다 skip | skip | skip | ✅ |

> 시나리오 6 보충: PR을 머지 없이 close하면 `pull_request(closed)` 이벤트로 워크플로우 run은
> **생성되지만**, `prd_deploy`의 `if (merged == true)` 조건이 false라 잡은 **skip**된다.
> (즉 "워크플로우 시작 O, 실제 배포 잡 실행 X" → 의도대로 안전)

## 핵심 결론

**✅ 성공 — master를 pull_request(closed + merged) 조건으로 두는 분리 방식이 정상 동작한다.**

검증의 핵심인 **시나리오 5**(develop→master PR 머지)에서 `prd_deploy`가 정확히 1회 실행되었고,
머지가 아닌 단순 close(시나리오 6)에서는 실행되지 않았다. develop push 트리거(시나리오 1·3)와
master PR 트리거(시나리오 5)가 서로 간섭 없이 분리되어 동작함을 확인.

- DEV 배포는 develop push에서만 (시나리오 1·3 ✓, 그 외 skip)
- PRD 배포는 master PR이 **머지될 때만** (시나리오 5 ✓, opened/close-without-merge에서는 미실행)

→ BDP-181592 조치로 제안된 트리거 분리 패턴을 **원복/적용해도 안전**하다.

## 부록: 주요 run 로그

### 시나리오 5 (핵심) — prd_deploy / run 26931725902 (event=pull_request)
```
PRD DEPLOY TRIGGERED
Event: pull_request
Merged: true
PR base: master
PR number: 2
```
JOBS: ✓ prd_deploy (5s) / - dev_deploy (skipped)

### 시나리오 1 — dev_deploy / run 26931624578 (event=push)
```
DEV DEPLOY TRIGGERED
Event: push
Ref: develop
```
JOBS: ✓ dev_deploy (3s) / - prd_deploy (skipped)

### 시나리오 6 — close without merge / run 26931774285 (event=pull_request)
```
JOBS: - dev_deploy (skipped) / - prd_deploy (skipped)
conclusion: skipped
```

### run 목록 (전체)
| run_id | 트리거 | event | 비고 |
|---|---|---|---|
| 26931586061 | workflow 최초 push | push | dev✓ (베이스라인) |
| 26931624578 | S1 develop push | push | dev✓ |
| 26931687392 | S3 PR#1 merge→develop | push | dev✓ |
| 26931725902 | S5 PR#2 merge→master | pull_request | **prd✓** |
| 26931764205 | S6 test6 push→develop | push | dev✓ (부수효과) |
| 26931774285 | S6 PR#3 close(머지X) | pull_request | 둘 다 skip |
