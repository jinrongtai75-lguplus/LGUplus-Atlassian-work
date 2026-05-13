# Claude Code - Atlassian(confluence) 연동 소개

사내망에서 Atlassian Cloud API가 차단되어 있어도, Claude Code에서 자연어로 **Confluence**(검색/조회/업로드)와 **Jira**(검색/조회/생성/댓글/상태 전환)를 사용할 수 있게 만드는 가이드입니다.

## 목차

### Part 1 — Confluence

1. [배경: 왜 이런 방식이 필요한가](#1-배경-왜-이런-방식이-필요한가)
2. [전체 구조 한눈에 보기](#2-전체-구조-한눈에-보기)
3. [사전 준비물](#3-사전-준비물)
4. [Step 1: 회사 GitHub 레포 만들기](#step-1-회사-github-레포-만들기)
5. [Step 2: gh CLI 설치](#step-2-gh-cli-설치)
6. [Step 3: gh CLI 인증 (1회만)](#step-3-gh-cli-인증-1회만)
7. [Step 4: 워크플로우 파일 작성 및 push](#step-4-워크플로우-파일-작성-및-push)
8. [Step 5: GitHub Secrets / Variables 설정](#step-5-github-secrets--variables-설정)
9. [Step 6: 워크플로우 동작 테스트](#step-6-워크플로우-동작-테스트)
10. [Step 7: Claude Code 스킬 작성](#step-7-claude-code-스킬-작성)
11. [Step 8: 자연어로 사용해보기](#step-8-자연어로-사용해보기)

### Part 2 — Jira (Confluence 위에 얹기)

12. [Part 2: Jira도 같이 연동하기](#part-2-jira도-같이-연동하기)

### 공통

13. [트러블슈팅](#트러블슈팅)
14. [안전 수칙](#안전-수칙)
15. [사고 사례에서 배운 점](#사고-사례에서-배운-점)

---

## 1. 배경: 왜 이런 방식이 필요한가

LG U+ 사내망에서는 `*.atlassian.net` / `api.atlassian.com` 같은 Atlassian Cloud REST API 호출이 차단되어 있습니다. 동일한 사용자 계정과 API 토큰이라도:

- ❌ 사내망 로컬 PC에서 직접 호출 → `403 Forbidden`
- ❌ Claude Code의 `mcp-atlassian` MCP 서버 → `403 Forbidden`
- ❌ Atlassian 공식 Remote MCP의 OAuth flow → 조직 admin allowlist 차단
- ✅ **GitHub Actions에서 호출** → 정상 작동 (외부 IP)

브라우저는 SSO를 거치는 별도 경로로 동작해서 위키 페이지는 잘 보이지만, API 직호출은 IP 기반으로 거부됩니다.

따라서 **GitHub Actions를 우회 경로**로 사용하고, Claude Code의 **스킬**로 감싸서 자연어로 호출 가능하게 만듭니다.

---

## 2. 전체 구조 한눈에 보기

```
[사용자: Claude Code 자연어 요청]
        ↓
[Claude Code Skills]
   ~/.claude/skills/confluence/  (SKILL.md + search/fetch/upload.sh)
   ~/.claude/skills/jira/        (SKILL.md + search/fetch/update.sh)
        ↓ (gh CLI로 워크플로우 트리거)
[GitHub Actions (회사 계정 repo)]
   Confluence: search-confluence.yml / fetch-confluence-page.yml / update-confluence.yml
   Jira      : search-jira.yml       / fetch-jira-issue.yml      / update-jira-issue.yml
        ↓ (API 호출, 외부 IP)
[Atlassian Cloud (lgucorp.atlassian.net)]
   /wiki/...     ← Confluence
   /rest/api/... ← Jira
```

각 단계가 분리되어 있어 어디서 막혀도 디버깅이 쉽고, 보안상 자격증명은 GitHub Secrets 한 곳에만 저장됩니다. **Jira는 Confluence와 같은 사이트라서 같은 토큰을 그대로 재사용**합니다 — 별도 secret 추가 없이 동작합니다.

---

## 3. 사전 준비물

### 계정/도구

- **회사 GitHub 계정** (예: `yourname-lguplus`) — 개인 계정이 아니어야 함. 회사 자산을 다루기 때문.
- **Atlassian 계정** — 회사 이메일로 로그인 가능한 lgucorp.atlassian.net 계정
- **Confluence 라이센스** — 본인 계정에 개인 스페이스가 있어야 함 (없으면 IT에 요청)
- **macOS** (이 가이드 기준; 다른 OS도 가능하지만 명령어 조정 필요)

### Atlassian API 토큰 발급

1. https://id.atlassian.com/manage-profile/security/api-tokens
2. 회사 계정으로 로그인된 상태 확인
3. **"API 토큰 만들기"** 클릭 (Classic — "범위를 포함하여..." 가 아닌 일반)
4. Label 입력 (예: `claude-code-confluence`) → Create
5. 토큰 복사해서 안전한 곳에 보관 (한 번만 표시됨)

> 🔍 **확인 팁:** 토큰이 정상 작동하는지 한 번 테스트:
> ```bash
> # 이건 사내망에서는 403이 정상. 우리는 GitHub Actions에서만 쓸 거라 괜찮음.
> curl -sI -u "본인이메일:토큰" "https://lgucorp.atlassian.net/wiki/rest/api/space?limit=1"
> ```

---

## Step 1: 회사 GitHub 레포 만들기

회사 GitHub 계정으로 로그인해서:

1. **https://github.com/new** 접속
2. Owner: 본인 회사 계정 선택 (개인 계정 X)
3. Repository name: 예) `LGUplus-Atlassian-work`
4. Visibility: **Private** (회사 정보 보호)
5. **"Add a README file"** 체크 해제 (나중에 직접 추가)
6. **"Create repository"** 클릭

빈 repo가 생성됩니다. URL은 `https://github.com/<회사계정>/LGUplus-Atlassian-work` 형식.

---

## Step 2: gh CLI 설치

`gh` (GitHub CLI)를 설치하면 PAT를 매번 발급하지 않아도 됩니다. macOS에서 `brew` 없이 설치하는 방법:

```bash
# 최신 버전 자동 다운로드 (Apple Silicon 기준)
cd ~/Downloads
LATEST=$(curl -s https://api.github.com/repos/cli/cli/releases/latest | grep '"tag_name"' | head -1 | cut -d '"' -f 4)
VERSION=${LATEST#v}
ARCH=$(uname -m)
curl -L -o gh.zip "https://github.com/cli/cli/releases/download/${LATEST}/gh_${VERSION}_macOS_${ARCH}.zip"
unzip -q -o gh.zip
mkdir -p ~/.local/bin
cp "gh_${VERSION}_macOS_${ARCH}/bin/gh" ~/.local/bin/gh
chmod +x ~/.local/bin/gh

# PATH에 추가 (zsh)
grep -q 'HOME/.local/bin' ~/.zshrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 확인
gh --version
```

`gh version 2.x.x` 표시되면 성공.

> Intel Mac이면 `uname -m`이 `x86_64`로 나옵니다. zip 파일명 `amd64`로 검색이 안 되니 위 스크립트는 Apple Silicon만 자동. Intel은 `ARCH=x86_64` 대신 manual 다운로드 필요.

---

## Step 3: gh CLI 인증 (1회만)

```bash
gh auth login
```

대화형 프롬프트에서 다음과 같이 선택:

1. `Where do you use GitHub?` → **GitHub.com**
2. `What is your preferred protocol for Git operations?` → **HTTPS**
3. `Authenticate Git with your GitHub credentials?` → **Yes**
4. `How would you like to authenticate?` → **Login with a web browser**
5. 터미널에 표시되는 **8자리 device code** (예: `XXXX-XXXX`) 복사
6. Enter 누르면 브라우저 자동 열림
7. 브라우저에서 device code 입력 → 회사 계정으로 로그인 → (필요시 2FA) → **Authorize**
8. 터미널에 "Authentication complete!" 표시되면 성공

확인:
```bash
gh auth status
# → ✓ Logged in to github.com account <yourname-lguplus> (keyring)
```

**이 인증은 macOS 키체인에 영구 저장됩니다. 다시 할 필요 없음.**

---

## Step 4: 워크플로우 파일 작성 및 push

로컬에 작업 폴더와 워크플로우 파일을 만듭니다.

```bash
mkdir -p ~/Projects/LGUplus-Atlassian-work/.github/workflows
cd ~/Projects/LGUplus-Atlassian-work
```

### 4-1. README.md 생성

```bash
cat > README.md <<'EOF'
# LGUplus Atlassian Work

LG U+ Atlassian(Confluence/Jira) 자동화 작업 저장소.
EOF
```

### 4-2. 검색 워크플로우 (`search-confluence.yml`)

```bash
cat > .github/workflows/search-confluence.yml <<'EOF'
name: Search Confluence

on:
  workflow_dispatch:
    inputs:
      space:
        description: 'Space key (e.g. M2M, DEV)'
        required: true
        default: 'M2M'
      query:
        description: 'Search keyword'
        required: true
      limit:
        description: 'Max results per search type'
        required: false
        default: '15'

jobs:
  search:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install atlassian-python-api
      - name: Search
        env:
          CONFLUENCE_BASE_URL: ${{ secrets.CONFLUENCE_BASE_URL }}
          CONFLUENCE_EMAIL: ${{ secrets.CONFLUENCE_EMAIL }}
          CONFLUENCE_API_TOKEN: ${{ secrets.CONFLUENCE_API_TOKEN }}
          SPACE: ${{ inputs.space }}
          QUERY: ${{ inputs.query }}
          LIMIT: ${{ inputs.limit }}
        run: |
          python - <<'PYEOF'
          import os
          from atlassian import Confluence
          base = os.environ['CONFLUENCE_BASE_URL'].rstrip('/')
          c = Confluence(url=base, username=os.environ['CONFLUENCE_EMAIL'],
                         password=os.environ['CONFLUENCE_API_TOKEN'], cloud=True)
          space = os.environ['SPACE']; query = os.environ['QUERY']
          limit = int(os.environ['LIMIT'])
          def hits(label, cql):
              print(f"\n{'='*70}\n{label}\nCQL: {cql}\n{'='*70}")
              for r in (c.cql(cql, limit=limit, expand='content.version').get('results') or []):
                  ct = r.get('content') or r
                  when = r.get('friendlyLastModified') or (ct.get('version') or {}).get('when') or ''
                  print(f"\n  • [{when}] {ct.get('title','')}")
                  print(f"    {base}/wiki/spaces/{space}/pages/{ct.get('id','')}")
                  ex = (r.get('excerpt') or '').replace('\n',' ').strip()[:240]
                  if ex: print(f"    > {ex}")
          hits(f"TITLE — {query}", f'space = "{space}" AND title ~ "{query}" AND type = "page" ORDER BY lastModified DESC')
          hits(f"BODY  — {query}", f'space = "{space}" AND text ~ "{query}"  AND type = "page" ORDER BY lastModified DESC')
          hits(f"RECENT — {space}", f'space = "{space}" AND type = "page" ORDER BY lastModified DESC')
          PYEOF
EOF
```

### 4-3. 업로드 워크플로우 (`update-confluence.yml`)

핵심: **다른 스페이스 페이지를 절대 건드리지 않는 안전장치 포함.**

```bash
cat > .github/workflows/update-confluence.yml <<'EOF'
name: Update Confluence

on:
  workflow_dispatch:
    inputs:
      space:
        description: 'Space key'
        required: true
      parent_id:
        description: 'Parent page ID (optional)'
        required: false
        default: ''
      title:
        description: 'Page title'
        required: true
  push:
    branches: [main]
    paths: ['README.md']

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install atlassian-python-api markdown
      - name: Upload
        env:
          CONFLUENCE_BASE_URL: ${{ secrets.CONFLUENCE_BASE_URL }}
          CONFLUENCE_EMAIL: ${{ secrets.CONFLUENCE_EMAIL }}
          CONFLUENCE_API_TOKEN: ${{ secrets.CONFLUENCE_API_TOKEN }}
          IN_SPACE: ${{ inputs.space }}
          IN_PARENT: ${{ inputs.parent_id }}
          IN_TITLE: ${{ inputs.title }}
          DEFAULT_SPACE: ${{ vars.CONFLUENCE_SPACE }}
        run: |
          python - <<'PYEOF'
          import os, markdown
          from atlassian import Confluence
          c = Confluence(url=os.environ['CONFLUENCE_BASE_URL'],
                         username=os.environ['CONFLUENCE_EMAIL'],
                         password=os.environ['CONFLUENCE_API_TOKEN'], cloud=True)
          with open('README.md','r',encoding='utf-8') as f: md = f.read()
          # Title: workflow input 우선, 없으면 README의 첫 H1, 그것도 없으면 'README'
          title = os.environ.get('IN_TITLE') or ''
          if not title:
              for line in md.splitlines():
                  if line.startswith('# '): title = line[2:].strip(); break
              if not title: title = 'README'
          html = markdown.markdown(md, extensions=['tables','fenced_code','toc'])
          space = os.environ.get('IN_SPACE') or os.environ.get('DEFAULT_SPACE')
          parent_id = os.environ.get('IN_PARENT') or None

          # CQL로 같은 스페이스 내에서만 같은 제목 페이지 찾기
          found = None
          for r in (c.cql(f'space = "{space}" AND title = "{title}" AND type = "page"', limit=5).get('results') or []):
              ct = r.get('content') or r
              if ct.get('title') == title: found = ct; break

          # 안전장치: 찾은 페이지의 실제 스페이스 키가 일치하는지 한 번 더 검증
          if found:
              verify = c.get_page_by_id(found['id'], expand='space')
              if verify.get('space',{}).get('key') != space:
                  print(f"[Safety] '{title}' found in space '{verify.get('space',{}).get('key')}', NOT '{space}'. Skipping update.")
                  found = None

          if found:
              c.update_page(page_id=found['id'], title=title, body=html)
              print(f"[Updated] {title} (id={found['id']}) in {space}")
          else:
              kw = dict(space=space, title=title, body=html)
              if parent_id: kw['parent_id'] = parent_id
              r = c.create_page(**kw)
              print(f"[Created] {title} (id={r['id']}) in {space}")
          PYEOF
EOF
```

### 4-4. (선택) 페이지 조회 워크플로우 (`fetch-confluence-page.yml`)

이건 위 두 워크플로우보다 길어서 별도 파일에 작성 권장. 본문/첨부/댓글/하위/라벨을 가져옵니다.
참고로만 두고 필요할 때 작성하세요. 예시는 본인 repo의 `.github/workflows/` 결과물 참고.

### 4-5. GitHub repo에 push

```bash
cd ~/Projects/LGUplus-Atlassian-work
git init -b main
git add README.md .github/
git config user.email "본인회사이메일"
git config user.name "본인회사계정"
git commit -m "Initial: search + update workflows"
git remote add origin https://github.com/<회사계정>/LGUplus-Atlassian-work.git
git push -u origin main   # gh auth 덕분에 인증 자동
```

---

## Step 5: GitHub Secrets / Variables 설정

### Secrets (3개)

🔗 `https://github.com/<회사계정>/LGUplus-Atlassian-work/settings/secrets/actions`

**New repository secret** 클릭해서 차례로 추가:

| Name | Value |
|---|---|
| `CONFLUENCE_BASE_URL` | `https://lgucorp.atlassian.net` |
| `CONFLUENCE_EMAIL` | 본인 회사 이메일 |
| `CONFLUENCE_API_TOKEN` | Step 3에서 발급받은 토큰 |

### Variables (1개)

같은 페이지 상단의 **Variables** 탭으로 이동.

| Name | Value |
|---|---|
| `CONFLUENCE_SPACE` | 본인 개인 스페이스 키 또는 자주 쓰는 팀 스페이스 키 |

스페이스 키는 Confluence URL에서 추출: `https://lgucorp.atlassian.net/wiki/spaces/M2M/...` → `M2M`. 개인 스페이스는 `~XXXX...` 형식.

`CONFLUENCE_PARENT_ID`는 추가하지 마세요 (UI가 빈 값을 막아서 차라리 변수 자체를 안 만드는 게 깔끔).

---

## Step 6: 워크플로우 동작 테스트

검색 워크플로우부터 한 번 수동으로 돌려봅니다.

```bash
gh -R <회사계정>/LGUplus-Atlassian-work workflow run search-confluence.yml \
  -f space=M2M -f query="주간업무" -f limit=10

# 가장 최근 run 추적
gh -R <회사계정>/LGUplus-Atlassian-work run watch
```

`✓ Run completed` 메시지 + 로그에 검색 결과가 나오면 성공입니다.

문제가 생기면 [트러블슈팅](#트러블슈팅) 섹션 참고.

---

## Step 7: Claude Code 스킬 작성

### 7-1. 스킬 폴더 생성

```bash
mkdir -p ~/.claude/skills/confluence
```

### 7-2. 헬퍼 스크립트 — 검색

```bash
cat > ~/.claude/skills/confluence/search.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
REPO="<회사계정>/LGUplus-Atlassian-work"   # ← 본인 repo로 변경
GH="${HOME}/.local/bin/gh"; [ -x "$GH" ] || GH="gh"
SPACE="${1:?space required}"; QUERY="${2:?query required}"; LIMIT="${3:-15}"
"$GH" -R "$REPO" workflow run search-confluence.yml \
  -f space="$SPACE" -f query="$QUERY" -f limit="$LIMIT" >/dev/null
sleep 3
RID=$("$GH" -R "$REPO" run list --workflow=search-confluence.yml --limit=1 --json databaseId --jq '.[0].databaseId')
echo ">>> Run: https://github.com/$REPO/actions/runs/$RID"
"$GH" -R "$REPO" run watch "$RID" --exit-status >/dev/null 2>&1 || true
"$GH" -R "$REPO" run view "$RID" --log 2>/dev/null
EOF
chmod +x ~/.claude/skills/confluence/search.sh
```

### 7-3. 헬퍼 스크립트 — 업로드

```bash
cat > ~/.claude/skills/confluence/upload.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
REPO="<회사계정>/LGUplus-Atlassian-work"   # ← 본인 repo로 변경
GH="${HOME}/.local/bin/gh"; [ -x "$GH" ] || GH="gh"
SPACE="${1:?space required}"; TITLE="${2:?title required}"
SRC="${3:?markdown path or -}"; PARENT_ID="${4:-}"

# Body 준비
if [ "$SRC" = "-" ]; then BODY="$(cat)"; else BODY="$(cat "$SRC")"; fi
TMP=$(mktemp); { echo "# $TITLE"; echo ""; echo "$BODY"; } > "$TMP"

# README 갱신
SHA=$("$GH" api "/repos/$REPO/contents/README.md" --jq '.sha' 2>/dev/null || true)
B64=$(base64 -i "$TMP" | tr -d '\n')
if [ -n "$SHA" ]; then
  "$GH" api -X PUT "/repos/$REPO/contents/README.md" \
    -f message="Upload: $TITLE" -f content="$B64" -f sha="$SHA" -f branch="main" >/dev/null
else
  "$GH" api -X PUT "/repos/$REPO/contents/README.md" \
    -f message="Upload: $TITLE" -f content="$B64" -f branch="main" >/dev/null
fi
rm -f "$TMP"

# 워크플로우 dispatch
"$GH" -R "$REPO" workflow run update-confluence.yml \
  -f space="$SPACE" -f parent_id="$PARENT_ID" -f title="$TITLE" >/dev/null
sleep 3
RID=$("$GH" -R "$REPO" run list --workflow=update-confluence.yml --limit=1 --json databaseId --jq '.[0].databaseId')
echo ">>> Run: https://github.com/$REPO/actions/runs/$RID"
"$GH" -R "$REPO" run watch "$RID" --exit-status >/dev/null 2>&1 || true
"$GH" -R "$REPO" run view "$RID" --log 2>/dev/null | grep -E "\[(Created|Updated|Safety)\]" | head -5 \
  || "$GH" -R "$REPO" run view "$RID" --log 2>/dev/null | tail -40
EOF
chmod +x ~/.claude/skills/confluence/upload.sh
```

### 7-4. SKILL.md (Claude가 읽는 가이드)

```bash
cat > ~/.claude/skills/confluence/SKILL.md <<'EOF'
---
name: confluence
description: Search or upload to Confluence on lgucorp.atlassian.net. Use whenever the user wants to find or post content on Confluence/위키 — including phrases like "스페이스에서 검색", "주간업무 찾아줘", "이 내용 컨플에 올려줘". Bypasses corporate network API blocking by routing through GitHub Actions.
---

# Confluence Helper

## When to invoke
Use this skill when the user wants to:
- Search Confluence pages in a space (CQL-driven)
- Upload markdown content as a new or updated Confluence page

## How to search
Run `~/.claude/skills/confluence/search.sh <space> <query> [limit]`. Parse the log,
present each hit (title, last-modified, URL) concisely. If space not given, ask once.

## How to upload
Run `~/.claude/skills/confluence/upload.sh <space> <title> <markdown-or-> [parent-id]`.
- Confirm space with user if not obvious.
- Warn that same-title page in the same space will be UPDATED (body only; parent
  unchanged on update). Different-space same-title pages are safely skipped.

## Auth
`~/.local/bin/gh` is authenticated to GitHub via keyring. If `gh auth status` fails,
ask the user to run `gh auth login` once — do not auto-fix.
EOF
```

> `<회사계정>` 부분을 본인 GitHub 계정명으로 바꿔주세요.

---

## Step 8: 자연어로 사용해보기

**Claude Code 세션을 새로 시작**합니다 (기존 세션은 새 스킬을 인식 못함).

새 세션에서 자연어로 시도:

```
M2M 스페이스에서 주간업무 검색해줘
```

또는 슬래시 명령어:

```
/confluence M2M에서 주간보고 찾아줘
```

Claude가 자동으로 `search.sh`를 호출하고, 20~40초 후 결과를 정리해서 보여줍니다.

업로드도 동일:

```
오늘 회의록을 M2M 스페이스에 "회의록 2026-05-12"로 올려줘:

## 참석자
...
```

---

## Part 2: Jira도 같이 연동하기

Part 1에서 만든 인프라(회사 GitHub repo + gh CLI + GitHub Secrets)를 그대로 재사용해서, **Jira용 워크플로우 3개**와 **Jira 스킬**만 추가하면 됩니다. 새로 발급할 토큰도, 새로 등록할 secret도 없습니다.

### 9. 왜 추가 secret이 필요 없는가

Atlassian Cloud의 API 토큰은 **사이트 단위**(lgucorp.atlassian.net)로 발급되기 때문에, 같은 사이트의 Confluence와 Jira에 동일하게 동작합니다. URL만 다릅니다:

- Confluence: `https://lgucorp.atlassian.net/wiki`
- Jira: `https://lgucorp.atlassian.net` (루트, /wiki 없음)

그래서 Jira 워크플로우 안에서 Confluence secret을 가져다 쓰고, 코드 안에서 `/wiki`만 떼어내면 됩니다:

```python
base = os.environ['JIRA_BASE_URL'].rstrip('/').replace('/wiki', '')
```

(이름이 헷갈리는 게 싫으면 별도 `JIRA_BASE_URL` / `JIRA_EMAIL` / `JIRA_API_TOKEN` secret을 같은 값으로 추가해도 됩니다 — 동작은 같음.)

---

### 10. Jira 워크플로우 3개 작성

`~/Projects/LGUplus-Atlassian-work/.github/workflows/` 아래에 3개 파일을 만듭니다.

#### 10-1. `search-jira.yml` (JQL 검색)

```yaml
name: Search Jira

on:
  workflow_dispatch:
    inputs:
      jql:
        description: 'JQL query (e.g. project = PROJ AND assignee = currentUser())'
        required: true
      limit:
        description: 'Max results'
        required: false
        default: '20'
      fields:
        description: 'Comma-separated fields to fetch'
        required: false
        default: 'summary,status,assignee,reporter,priority,issuetype,updated,labels'

jobs:
  search:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install atlassian-python-api
      - name: Search
        env:
          JIRA_BASE_URL: ${{ secrets.CONFLUENCE_BASE_URL }}
          JIRA_EMAIL: ${{ secrets.CONFLUENCE_EMAIL }}
          JIRA_API_TOKEN: ${{ secrets.CONFLUENCE_API_TOKEN }}
          JQL: ${{ inputs.jql }}
          LIMIT: ${{ inputs.limit }}
          FIELDS: ${{ inputs.fields }}
        run: |
          python - <<'PYEOF'
          import os
          from atlassian import Jira
          base = os.environ['JIRA_BASE_URL'].rstrip('/').replace('/wiki', '')
          jira = Jira(url=base, username=os.environ['JIRA_EMAIL'],
                      password=os.environ['JIRA_API_TOKEN'], cloud=True)
          jql = os.environ['JQL']; limit = int(os.environ['LIMIT'])
          fields = os.environ['FIELDS']
          print(f"{'='*70}\nJQL: {jql}\nLimit: {limit}  Fields: {fields}\n{'='*70}\n")
          result = jira.jql(jql, limit=limit, fields=fields)
          issues = result.get('issues') or []
          total = result.get('total', len(issues))
          print(f"Total matches: {total}  (showing {len(issues)})\n")
          if not issues:
              print("(no results)"); raise SystemExit(0)
          for i in issues:
              k = i.get('key',''); f = i.get('fields',{}) or {}
              s = f.get('summary',''); st = ((f.get('status') or {}).get('name')) or ''
              a = ((f.get('assignee') or {}).get('displayName')) or '(unassigned)'
              r = ((f.get('reporter') or {}).get('displayName')) or ''
              p = ((f.get('priority') or {}).get('name')) or ''
              t = ((f.get('issuetype') or {}).get('name')) or ''
              u = f.get('updated','') or ''
              labels = ','.join(f.get('labels') or []) or '-'
              print(f"• {k}  [{st}]  ({t}, {p})\n  {s}")
              print(f"  assignee: {a}  reporter: {r}  labels: {labels}")
              print(f"  updated: {u}\n  {base}/browse/{k}\n")
          PYEOF
```

#### 10-2. `fetch-jira-issue.yml` (이슈 상세 조회)

이슈 키나 URL을 받아서 본문/댓글/첨부/링크/트랜지션/이력을 가져옵니다. 길어서 핵심 input만 표시 — **전체 파일은 본 repo의 `.github/workflows/fetch-jira-issue.yml` 참조**:

```yaml
on:
  workflow_dispatch:
    inputs:
      key:
        description: 'Issue key (e.g. PROJ-123) or full URL'
        required: true
      include:
        description: 'Comma-separated: details, comments, attachments, links, transitions, history, all'
        required: false
        default: 'details,comments,transitions'
```

내부적으로:
1. URL에서 정규식으로 이슈 키 추출 (`[A-Z][A-Z0-9_]+-\d+`)
2. `atlassian.Jira.issue(key, expand='changelog,renderedFields')` 호출
3. include 옵션에 따라 섹션별로 출력 (DETAILS / COMMENTS / ATTACHMENTS / LINKS / TRANSITIONS / HISTORY)

#### 10-3. `update-jira-issue.yml` (생성/댓글/트랜지션/수정)

`action` input으로 4가지 동작을 분기합니다. **전체 파일은 본 repo의 `.github/workflows/update-jira-issue.yml` 참조**. 핵심 인터페이스:

```yaml
on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Operation: create | comment | transition | update'
        required: true
        type: choice
        options: [create, comment, transition, update]
      key:           # comment/transition/update 시 필요
      project:       # create 시 필요
      summary:       # create 시 필요, update 시 선택
      issuetype:     # create 시 (Task/Bug/Story/Epic 등)
      description:   # create/update 시
      comment:       # action=comment 시
      transition:    # action=transition 시 (이름 또는 id)
      assignee:      # accountId 또는 email
      priority:      # High/Medium/Low 등
      labels:        # 콤마 구분
      fields_json:   # 고급 필드 JSON 직접 입력
```

각 action별 동작:
- **create**: `jira.issue_create(fields=...)`
- **comment**: `jira.issue_add_comment(key, body)`
- **transition**: 이름이면 `jira.issue_transition(key, name)`, 숫자면 REST API로 id 호출
- **update**: `jira.update_issue_field(key, fields)`

---

### 11. 3개 파일 GitHub repo에 push

이전과 같이 `gh api`로 한 번에:

```bash
REPO="<회사계정>/LGUplus-Atlassian-work"
DIR="$HOME/Projects/LGUplus-Atlassian-work/.github/workflows"
for f in search-jira.yml fetch-jira-issue.yml update-jira-issue.yml; do
  gh api -X PUT "/repos/$REPO/contents/.github/workflows/$f" \
    -f message="Add $f" \
    -f content="$(base64 -i "$DIR/$f")" \
    -f branch="main"
done
```

`OK: ... @ <sha>` 가 세 번 나오면 완료.

---

### 12. Jira 스킬 작성

```bash
mkdir -p ~/.claude/skills/jira
```

#### 12-1. `search.sh`

```bash
cat > ~/.claude/skills/jira/search.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
REPO="<회사계정>/LGUplus-Atlassian-work"
GH="${HOME}/.local/bin/gh"; [ -x "$GH" ] || GH="gh"
JQL="${1:?JQL required}"; LIMIT="${2:-20}"
FIELDS="${3:-summary,status,assignee,reporter,priority,issuetype,updated,labels}"
"$GH" -R "$REPO" workflow run search-jira.yml -f jql="$JQL" -f limit="$LIMIT" -f fields="$FIELDS" >/dev/null
sleep 3
RID=$("$GH" -R "$REPO" run list --workflow=search-jira.yml --limit=1 --json databaseId --jq '.[0].databaseId')
echo ">>> Run: https://github.com/$REPO/actions/runs/$RID"
"$GH" -R "$REPO" run watch "$RID" --exit-status >/dev/null 2>&1 || true
"$GH" -R "$REPO" run view "$RID" --log 2>/dev/null
EOF
chmod +x ~/.claude/skills/jira/search.sh
```

#### 12-2. `fetch.sh`

```bash
cat > ~/.claude/skills/jira/fetch.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
REPO="<회사계정>/LGUplus-Atlassian-work"
GH="${HOME}/.local/bin/gh"; [ -x "$GH" ] || GH="gh"
KEY="${1:?key or URL required}"; INCLUDE="${2:-details,comments,transitions}"
"$GH" -R "$REPO" workflow run fetch-jira-issue.yml -f key="$KEY" -f include="$INCLUDE" >/dev/null
sleep 3
RID=$("$GH" -R "$REPO" run list --workflow=fetch-jira-issue.yml --limit=1 --json databaseId --jq '.[0].databaseId')
echo ">>> Run: https://github.com/$REPO/actions/runs/$RID"
"$GH" -R "$REPO" run watch "$RID" --exit-status >/dev/null 2>&1 || true
"$GH" -R "$REPO" run view "$RID" --log 2>/dev/null
EOF
chmod +x ~/.claude/skills/jira/fetch.sh
```

#### 12-3. `update.sh`

`key=value` 형식으로 인자 받아 update-jira-issue.yml 호출:

```bash
cat > ~/.claude/skills/jira/update.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
REPO="<회사계정>/LGUplus-Atlassian-work"
GH="${HOME}/.local/bin/gh"; [ -x "$GH" ] || GH="gh"
ACTION="${1:?action required: create|comment|transition|update}"; shift
declare -A P
for arg in "$@"; do k="${arg%%=*}"; v="${arg#*=}"; P[$k]="$v"; done
FLAGS=(-f "action=$ACTION")
for k in key project summary issuetype description comment transition assignee priority labels fields_json; do
  FLAGS+=(-f "$k=${P[$k]:-}")
done
"$GH" -R "$REPO" workflow run update-jira-issue.yml "${FLAGS[@]}" >/dev/null
sleep 3
RID=$("$GH" -R "$REPO" run list --workflow=update-jira-issue.yml --limit=1 --json databaseId --jq '.[0].databaseId')
echo ">>> Run: https://github.com/$REPO/actions/runs/$RID"
"$GH" -R "$REPO" run watch "$RID" --exit-status >/dev/null 2>&1 || true
"$GH" -R "$REPO" run view "$RID" --log 2>/dev/null | grep -E "\[(Created|Commented|Transitioned|Updated)\]|ERROR" | head -5 \
  || "$GH" -R "$REPO" run view "$RID" --log 2>/dev/null | tail -50
EOF
chmod +x ~/.claude/skills/jira/update.sh
```

#### 12-4. `SKILL.md` (Claude가 읽는 가이드)

```bash
cat > ~/.claude/skills/jira/SKILL.md <<'EOF'
---
name: jira
description: Search, view, create, comment on, or transition Jira issues on LG U+ (lgucorp.atlassian.net). Use whenever the user wants to find issues by JQL, look up a ticket, leave a comment, move status, or open a new ticket — including phrases like "내가 담당인 이슈", "PROJ-123 상세 보여줘", "이슈 댓글 달아줘", "In Progress로 옮겨줘", "새 이슈 만들어줘". Bypasses corporate network API blocking by routing through GitHub Actions.
---

# Jira Helper

## How to search
Run `~/.claude/skills/jira/search.sh "<JQL>" [limit] [fields]`.
Translate Korean intents to JQL:
- 내가 담당인 / 내 이슈 → `assignee = currentUser()`
- 미해결 → `resolution = Unresolved`
- 최근 N일 → `updated >= -7d`
- 특정 프로젝트 → `project = PROJ`
- 정렬 → `ORDER BY updated DESC`

## How to fetch
Run `~/.claude/skills/jira/fetch.sh <key-or-url> [include]`.
include = details,comments,attachments,links,transitions,history,all (default: details,comments,transitions)

## How to act on issues
Run `~/.claude/skills/jira/update.sh <action> [key=val ...]`.
action = create|comment|transition|update

Examples:
- `update.sh comment key=PROJ-1 comment="검토 부탁드립니다"`
- `update.sh transition key=PROJ-1 transition="In Progress"`
- `update.sh create project=PROJ summary="버그 수정" issuetype=Bug priority=High`

## Safety
Confirm destructive actions (transition/update/create) with the user before invoking.
If transition name is unclear, fetch transitions first and ask.
EOF
```

> `<회사계정>` 부분을 본인 GitHub 계정명으로 바꿔주세요.

---

### 13. Jira 자연어로 사용해보기

**Claude Code 세션을 새로** 시작한 뒤:

```
내가 담당한 미해결 Jira 이슈 보여줘
PROJ-1234 상세 내용 알려줘
PROJ-1234에 "검토 시작합니다" 댓글 달아줘
PROJ-1234를 "In Progress"로 옮겨줘
PROJ 프로젝트에 Bug 새로 만들어줘: 제목 "로그인 오류", 우선순위 High
```

또는 슬래시:

```
/jira 내 이슈 보여줘
```

Confluence 스킬과 똑같이 워크플로우 → 결과 파싱 → 요약 답변 흐름으로 처리됩니다.

---

## 트러블슈팅

### YAML 들여쓰기가 자꾸 깨질 때 (GitHub 웹 에디터)

GitHub의 웹 에디터(`https://github.com/.../edit/...`)는 paste 시 자동 들여쓰기로 YAML이 깨질 수 있습니다.

**해결:**
- `github.dev`로 편집 (URL의 `github.com`을 `github.dev`로 변경) — VS Code 웹 인터페이스에서 정확하게 편집 가능
- 또는 로컬에서 `cat file.yml | pbcopy`로 클립보드에 정확히 복사 후 paste

### git push 실패 — `denied to <개인계정>`

```
remote: Permission to <회사계정>/repo denied to <개인계정>
```

macOS 키체인의 GitHub credential이 개인 계정으로 저장되어 있을 때 발생. 해결:

```bash
gh auth login   # 회사 계정으로 다시 로그인
gh auth setup-git
```

이후 `git push`는 회사 계정으로 진행됨.

### git push 실패 — `without workflow scope`

```
refusing to allow a Personal Access Token to create or update workflow ... without 'workflow' scope
```

PAT를 직접 만들어서 쓰는 경우, scope에 `workflow`를 빠뜨림. 해결:
- PAT를 만들 때 `repo` + `workflow` 두 가지 모두 체크
- 또는 `gh` CLI 사용 (필요한 scope가 자동으로 부여됨)

### 워크플로우 실행 시 403 "caller cannot access Confluence"

GitHub Secrets에 저장된 API 토큰이 만료되었거나, 본인 계정이 Confluence 권한을 잃었을 때. 해결:
- Atlassian에서 토큰 재발급 → Secrets 업데이트
- IT에 본인 계정의 Confluence 라이센스 확인 요청

### 워크플로우 실행이 30초 이상 걸려서 Claude가 끊김

`gh run watch`는 기본 3분까지 대기. 그 안에 끝나면 OK. 매우 큰 검색이면 limit를 줄여서 재시도.

### Claude Code가 `/confluence` 스킬을 못 찾는다

- 새 Claude Code 세션을 열어야 인식됨 (기존 세션은 시작 시점의 스킬 목록 고정)
- `~/.claude/skills/confluence/SKILL.md` 파일이 존재하는지 확인
- `SKILL.md`의 frontmatter (`name: confluence`)가 정확한지 확인

---

## 안전 수칙

1. **회사 자료는 반드시 회사 GitHub 계정의 Private repo에** — 개인 계정/Public은 절대 안 됨
2. **API 토큰은 Repo Secrets에만** — 코드/스크립트/README에 절대 직접 적지 말 것
3. **업로드 대상 스페이스는 매번 명시적으로 확인** — Claude에게 모호하게 요청하지 말기
4. **첫 업로드 후에는 항상 Confluence에서 결과 확인** — 의도한 위치에 만들어졌는지
5. **같은 제목 페이지 충돌 가능성** — 팀 스페이스에 같은 제목이 있으면 본문이 갱신됨. 본인 페이지임을 식별 가능한 제목 사용 권장 (예: `[김길동] 주간보고 2026-05-12`)

---

## 사고 사례에서 배운 점

### 사례 1: 다른 스페이스의 동일 제목 페이지를 의도치 않게 덮어씀

**원인:** 초기 코드의 `confluence.get_page_by_title(space=..., title=...)`이 space 필터를 안전하게 적용하지 않아, 다른 스페이스에 있던 같은 제목 페이지를 찾아서 갱신함.

**증상:** 본인 계정에 view 권한이 없는 다른 조직 스페이스의 페이지가 본인 README 내용으로 변경됨. API 토큰에는 write 권한이 있어서 거부되지 않음.

**해결:** 워크플로우에 두 단계 안전장치 추가
1. CQL로 명시적 스페이스 필터: `space = "KEY" AND title = "..." AND type = "page"`
2. 찾은 페이지를 `get_page_by_id(expand='space')`로 다시 조회해서 `space.key`가 일치하는지 검증
3. 불일치 시 `[Safety]` 로그만 남기고 update 건너뛰기

이 패턴은 위 `update-confluence.yml`에 이미 적용되어 있습니다.

### 사례 2: PAT scope 부족으로 workflow 파일 push 거부

**증상:** 워크플로우 파일(`.github/workflows/*.yml`)을 push할 때:
```
refusing to allow a Personal Access Token to create or update workflow ... without 'workflow' scope
```

**해결:** PAT를 만들 때 `repo`뿐만 아니라 `workflow` scope도 체크. 또는 `gh` CLI 사용으로 우회.

### 사례 3: 인증 토큰 종류 혼동 (8자리 vs 6자리)

`gh auth login` 도중:
- **터미널의 8자리 코드** (`XXXX-XXXX`) → 브라우저 `github.com/login/device` 페이지에 입력
- **Authenticator의 6자리 코드** → 그 다음 GitHub 로그인 단계의 2FA 입력

두 코드는 서로 다른 단계에서 사용됩니다.

---

## 정리

이 가이드를 따라 한 번 설정하면, 이후에는:

- 인증/PAT 발급 반복 ❌
- GitHub 웹 UI에서 YAML 편집 ❌
- 워크플로우 실행 결과를 매번 클릭해서 확인 ❌

→ **Claude Code 안에서 자연어 한 마디**로 다음 모두가 가능해집니다:

| Confluence | Jira |
|---|---|
| 스페이스 검색 (CQL) | 이슈 검색 (JQL) |
| 페이지 본문/첨부/댓글/하위 조회 | 이슈 상세/댓글/이력/트랜지션 조회 |
| 페이지 생성/갱신 (안전장치 포함) | 이슈 생성/수정/댓글/상태 전환 |

같은 인프라(회사 GitHub repo + gh CLI + 공유 secrets)를 위에 얹는 구조라, Jira를 추가했을 때 추가 비용은 **워크플로우 파일 3개 + 스크립트 3개 + SKILL.md** 뿐이었습니다. 이후 Bitbucket, Trello 같은 같은 사이트의 다른 Atlassian product도 동일 패턴으로 확장 가능합니다.

향후 사내망 API 차단이 풀리면 `mcp-atlassian` MCP가 직접 작동하므로 이 우회 경로 없이 즉시 응답을 받게 됩니다. 그때는 이 스킬들을 비활성화하거나 그대로 둬도 무방합니다 (백업 경로 역할).
