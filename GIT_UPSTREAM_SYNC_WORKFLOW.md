# Git Upstream Sync Workflow

이 문서는 `moonid-lab/MoneyBall` 저장소에서 포크한 원본 저장소의 `main` 브랜치 변경사항을 주기적으로 동기화하기 위한 브랜치 전략과 GitHub Actions 운영 방식을 정리합니다.

## 목적

이 저장소는 포크 기반으로 운영되며, 다음 두 가지 요구를 동시에 만족해야 합니다.

1. 원본 저장소(`upstream`)의 최신 변경사항을 지속적으로 추적한다.
2. 포크 저장소의 `main` 브랜치에서는 장기적인 커스텀 개발을 안정적으로 유지한다.

이 문서의 전략은 `main` 브랜치를 직접 upstream 동기화 브랜치로 사용하지 않고, 별도의 추적 브랜치 `upstream-main`을 두어 역할을 분리하는 방식입니다.

---

## 저장소 원격(remote) 구성

- `origin`: `git@github.com:moonid-lab/MoneyBall.git`
- `upstream`: `https://github.com/666ghj/MiroFish.git`

의미:
- `origin`은 내 포크 저장소입니다.
- `upstream`은 원본 저장소입니다.

---

## 브랜치 전략

### 브랜치 역할

- `main`
  - 내 저장소의 장기 커스텀 개발 메인 브랜치
  - 제품 운영 및 통합 기준 브랜치
- `upstream-main`
  - `upstream/main`을 추적하는 동기화 전용 브랜치
  - 원본 최신 상태를 반영하는 미러 브랜치

### 왜 `main`을 동기화 브랜치로 쓰지 않는가

`main` 브랜치가 장기적으로 내 커스텀 개발의 중심이 되면, 여기에 upstream 동기화 역할까지 동시에 부여하는 것은 비추천입니다.

문제점:
- upstream 추적 이력과 내 커스텀 개발 이력이 섞입니다.
- 충돌 관리가 어려워집니다.
- 원본 변경과 내 변경의 구분이 어려워집니다.
- 장기 유지보수와 검토가 복잡해집니다.

따라서 다음처럼 역할을 분리합니다.

- `upstream/main` → 원본 최신 상태
- `upstream-main` → 원본 추적용 브랜치
- `main` → 내 개발 메인 브랜치

---

## 동기화 운영 원칙

이 저장소에서는 `main` 브랜치에 직접 자동 push 하지 않는 방식을 채택합니다.

즉:
- GitHub Actions가 `upstream/main`의 변경을 감지합니다.
- 로컬/원격의 `upstream-main` 브랜치를 최신으로 갱신합니다.
- `upstream-main`에서 `main`으로 향하는 Pull Request를 자동 생성합니다.
- 최종 반영은 사람이 검토 후 수동 merge 합니다.

이 방식의 장점:
- `main` 브랜치 보호 정책과 잘 맞습니다.
- 자동 반영으로 인한 예기치 않은 문제를 줄일 수 있습니다.
- 충돌이 있어도 PR 단계에서 안전하게 검토할 수 있습니다.
- 내 커스텀 개발 브랜치를 안정적으로 유지할 수 있습니다.

---

## GitHub Actions 트리거 전략

### 사용 트리거

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * *'
```

의미:
- `workflow_dispatch`: 필요 시 수동 실행 가능
- `schedule`: 매일 UTC 02:00 자동 실행

참고:
- GitHub Actions의 cron은 UTC 기준입니다.
- `0 2 * * *`는 한국 시간(KST) 기준 오전 11시입니다.

### 왜 upstream push 이벤트를 직접 받지 않는가

GitHub Actions는 기본적으로 **해당 저장소 내부 이벤트**를 기준으로 동작합니다.
따라서 외부 저장소인 `upstream`의 `push` 이벤트를 내 포크 저장소에서 직접 트리거로 받을 수 없습니다.

그래서 이 문서에서는 **주기적 polling 방식(`schedule`)**을 사용합니다.

---

## 최종 워크플로우 동작 방식

워크플로우 파일:
- `.github/workflows/sync-upstream-main.yml`

동작 순서:

1. `origin/main`과 `upstream/main`을 fetch 합니다.
2. `upstream/main`의 최신 SHA를 읽습니다.
3. `origin/upstream-main`의 현재 SHA를 읽습니다.
4. 두 SHA가 같으면 종료합니다.
5. 다르면 로컬 `upstream-main` 브랜치를 `upstream/main`과 동일하게 맞춥니다.
6. `origin/upstream-main`에 강제 동기화 push 합니다.
7. `upstream-main -> main` Pull Request가 이미 열려 있는지 확인합니다.
8. 기존 PR이 없으면 새 PR을 생성합니다.
9. 이후 검토자가 충돌 해결 또는 검토 후 수동으로 merge 합니다.

핵심 포인트:
- `main`에 직접 push 하지 않습니다.
- 항상 PR 기반으로 반영합니다.
- 중복 PR 생성은 방지합니다.

---

## 최종 워크플로우 파일

아래는 현재 기준 최종 워크플로우입니다.

````yaml
name: Sync upstream main by pull request

on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * *'

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: sync-upstream-main-pr
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: main

      - name: Configure git identity
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Configure upstream remote
        run: |
          if git remote get-url upstream >/dev/null 2>&1; then
            git remote set-url upstream https://github.com/666ghj/MiroFish.git
          else
            git remote add upstream https://github.com/666ghj/MiroFish.git
          fi

      - name: Fetch branches
        run: |
          git fetch origin main
          git fetch upstream main

      - name: Resolve upstream SHA
        id: upstream_sha
        run: |
          echo "sha=$(git rev-parse upstream/main)" >> "$GITHUB_OUTPUT"

      - name: Resolve origin upstream-main SHA
        id: current_sync_sha
        run: |
          if git ls-remote --exit-code origin refs/heads/upstream-main >/dev/null 2>&1; then
            echo "sha=$(git ls-remote origin refs/heads/upstream-main | awk '{print $1}')" >> "$GITHUB_OUTPUT"
          else
            echo "sha=" >> "$GITHUB_OUTPUT"
          fi

      - name: Stop if no upstream changes
        if: steps.upstream_sha.outputs.sha == steps.current_sync_sha.outputs.sha
        run: echo "No upstream/main changes detected."

      - name: Update local upstream-main branch
        if: steps.upstream_sha.outputs.sha != steps.current_sync_sha.outputs.sha
        run: |
          if git show-ref --verify --quiet refs/heads/upstream-main; then
            git checkout upstream-main
          else
            git checkout -b upstream-main upstream/main
          fi
          git reset --hard upstream/main

      - name: Push upstream-main to origin
        if: steps.upstream_sha.outputs.sha != steps.current_sync_sha.outputs.sha
        run: |
          git push origin upstream-main --force-with-lease

      - name: Create or reuse pull request
        if: steps.upstream_sha.outputs.sha != steps.current_sync_sha.outputs.sha
        uses: actions/github-script@v7
        with:
          script: |
            const owner = context.repo.owner;
            const repo = context.repo.repo;
            const head = `${owner}:upstream-main`;
            const base = 'main';
            const upstreamSha = '${{ steps.upstream_sha.outputs.sha }}';

            const existing = await github.rest.pulls.list({
              owner,
              repo,
              state: 'open',
              head,
              base
            });

            if (existing.data.length > 0) {
              core.info(`PR already exists: ${existing.data[0].html_url}`);
              return;
            }

            const pr = await github.rest.pulls.create({
              owner,
              repo,
              title: `Sync upstream/main into main (${upstreamSha.slice(0, 7)})`,
              head: 'upstream-main',
              base: 'main',
              body: [
                'This pull request updates `upstream-main` from the upstream repository and proposes merging it into `main`.',
                '',
                `- Upstream repository: https://github.com/666ghj/MiroFish`,
                `- Upstream branch: \`main\``,
                `- Upstream commit: \`${upstreamSha}\``,
                '',
                'Review the changes and merge this PR manually.'
              ].join('\n')
            });

            core.info(`Created PR: ${pr.data.html_url}`);
````

---

## 브랜치 보호 전략

### 권장 사항

이 저장소에서는 `main` 브랜치를 장기 커스텀 브랜치로 사용하므로, **브랜치 보호를 유지하는 것을 권장**합니다.

추천 설정 방향:
- `main` 직접 push 제한 유지
- Pull Request 기반 merge 유지
- 필요 시 status check 적용

### 이유

이 워크플로우는 `main`에 직접 push 하지 않기 때문에, 브랜치 보호 정책과 충돌하지 않습니다.
오히려 보호 설정이 활성화되어 있어야 실수로 직접 반영되는 위험을 줄일 수 있습니다.

---

## 브랜치 보호 설정 위치

### Branch protection rule 또는 Rulesets 확인 위치

1. 저장소로 이동
2. **Settings**
3. 다음 중 하나로 이동
   - **Branches**
   - 또는 **Rules → Rulesets**

확인할 항목 예시:
- Require a pull request before merging
- Require status checks to pass before merging
- Restrict who can push to matching branches
- Do not allow bypassing the above settings

현재 전략에서는 위 항목들을 해제할 필요가 없습니다.
오히려 `main`은 보호된 상태로 유지하는 것이 적합합니다.

---

## 저장소 설정 권장값

### Actions 권한 설정

경로:
- **Settings → Actions → General**

권장:
- **Workflow permissions**: `Read and write permissions`

이 권한이 있어야 workflow가 다음 작업을 수행할 수 있습니다.
- `upstream-main` 브랜치 push
- Pull Request 생성

---

## 운영 예시

### 시나리오 1: upstream 변경 없음

- workflow 실행
- `upstream/main`과 `origin/upstream-main` SHA 비교
- 동일하면 종료

### 시나리오 2: upstream 변경 있음

- `upstream-main` 갱신
- `origin/upstream-main` push
- 기존 PR 확인
- PR이 없으면 `upstream-main -> main` PR 생성

### 시나리오 3: 이미 PR이 열려 있음

- `upstream-main`은 최신으로 갱신됨
- 기존 open PR 재사용
- 새 PR은 생성하지 않음

---

## 수동 검토 및 반영 절차

1. 자동 생성된 PR을 확인합니다.
2. 변경 내용을 검토합니다.
3. 충돌이 있다면 PR 또는 로컬에서 해결합니다.
4. 테스트 후 `main`에 merge 합니다.

이 절차를 통해 upstream 변경을 안전하게 내 커스텀 `main` 브랜치에 반영할 수 있습니다.

---

## 요약

이 문서의 최종 전략은 다음과 같습니다.

- `main`은 내 커스텀 메인 브랜치로 유지한다.
- `upstream/main`은 `upstream-main` 브랜치로 추적한다.
- GitHub Actions는 일정 주기로 upstream 변경을 확인한다.
- 변경이 있으면 `upstream-main`을 업데이트한다.
- `main`에 직접 push 하지 않고, 항상 PR을 생성한다.
- 최종 반영은 사람이 검토 후 수동 merge 한다.

이 전략은 장기적으로 포크 저장소를 안정적으로 운영하면서도 upstream 변경을 놓치지 않도록 하기 위한 실용적인 운영 방식입니다.
