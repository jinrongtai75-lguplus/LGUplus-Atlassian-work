  # LGUplus Atlassian Work
  
  LG U+ Atlassian(Confluence/Jira) 자동화 작업 저장소.

  ## Confluence 자동 동기화
  
  이 저장소의 `README.md`를 수정해서 `main` 브랜치에 푸시하면 GitHub Actions가 자동으로 Confluence 페이지를 생성/갱신합니다.

  워크플로우: [.github/workflows/update-confluence.yml](.github/workflows/update-confluence.yml)

  2단계: 워크플로우 파일 추가
  
  🔗 https://github.com/jinrongtai75-lguplus/LGUplus-Atlassian-work/new/main?filename=.github%2Fworkflows%2Fupdate-confluence.yml

  위 링크 열고 → 아래 내용 붙여넣기 → "Commit changes" 클릭

  name: Update Confluence on README Change
  
  on:
    push:
      branches: [main]
      paths:
        - 'README.md'
    workflow_dispatch:
  
  jobs:
    update-confluence:
      runs-on: ubuntu-latest

      steps:
        - name: Checkout repository
          uses: actions/checkout@v4
  
        - name: Set up Python
          uses: actions/setup-python@v5
          with:
            python-version: '3.11'

        - name: Install dependencies
          run: pip install atlassian-python-api markdown
  
        - name: Upload README to Confluence
          env:
            CONFLUENCE_BASE_URL: ${{ secrets.CONFLUENCE_BASE_URL }}
            CONFLUENCE_EMAIL: ${{ secrets.CONFLUENCE_EMAIL }}
            CONFLUENCE_API_TOKEN: ${{ secrets.CONFLUENCE_API_TOKEN }}
            CONFLUENCE_SPACE: ${{ vars.CONFLUENCE_SPACE }}
            CONFLUENCE_PARENT_ID: ${{ vars.CONFLUENCE_PARENT_ID }}
          run: |
            python - <<'EOF'
            import os
            import markdown
            from atlassian import Confluence

            confluence = Confluence(
                url=os.environ['CONFLUENCE_BASE_URL'],
                username=os.environ['CONFLUENCE_EMAIL'],
                password=os.environ['CONFLUENCE_API_TOKEN'],
                cloud=True
            )

            with open('README.md', 'r', encoding='utf-8') as f:
                md_content = f.read()

            # README 첫 번째 H1을 페이지 제목으로 사용
            title = 'README'
            for line in md_content.splitlines():
                if line.startswith('# '):
                    title = line[2:].strip()
                    break

            # Markdown -> HTML (Confluence storage format)
            html_content = markdown.markdown(
                md_content,
                extensions=['tables', 'fenced_code', 'toc']
            )

            space = os.environ['CONFLUENCE_SPACE']
            parent_id = os.environ.get('CONFLUENCE_PARENT_ID') or None

            # 기존 페이지 존재 여부 확인
            page = confluence.get_page_by_title(space=space, title=title)

            if page:
                page_id = page['id']
                confluence.update_page(
                    page_id=page_id,
                    title=title,
                    body=html_content
                )
                print(f"[Updated] {title} (ID: {page_id})")
            else:
                kwargs = dict(space=space, title=title, body=html_content)
                if parent_id:
                    kwargs['parent_id'] = parent_id
                result = confluence.create_page(**kwargs)
                print(f"[Created] {title} (ID: {result['id']})")
            EOF

  3단계: Secrets 설정

  🔗 https://github.com/jinrongtai75-lguplus/LGUplus-Atlassian-work/settings/secrets/actions

  "New repository secret" 클릭, 3개 차례로 추가:

  ┌──────────────────────┬───────────────────────────────┐
  │         Name         │             Value             │
  ├──────────────────────┼───────────────────────────────┤
  │ CONFLUENCE_BASE_URL  │ https://lgucorp.atlassian.net │
  ├──────────────────────┼───────────────────────────────┤
  │ CONFLUENCE_EMAIL     │ 본인 회사 이메일                  │
  ├──────────────────────┼───────────────────────────────┤
  │ CONFLUENCE_API_TOKEN │ 본인 Atlassian API 토큰          │
  └──────────────────────┴───────────────────────────────┘

  4단계: Variables 설정
  
  같은 페이지 상단의 "Variables" 탭 → "New repository variable" 클릭

  ┌──────────────────────┬───────────────────────────┐
  │         Name         │           Value           │
  ├──────────────────────┼───────────────────────────┤
  │ CONFLUENCE_SPACE     │ ~61e11a6449f1950069c13850 │
  ├──────────────────────┼───────────────────────────┤
  │ CONFLUENCE_PARENT_ID │ (생략 가능)                  │
  └──────────────────────┴───────────────────────────┘

  5단계: 실행 테스트
  
  🔗 https://github.com/jinrongtai75-lguplus/LGUplus-Atlassian-work/actions

  "Update Confluence on README Change" 클릭 → "Run workflow" 버튼 → "Run workflow" 확정
