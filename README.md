# flow-workflows

Flow 시리즈 앱 레포에서 공유하는 재사용 가능한 GitHub Actions 워크플로.

## 포함 워크플로

- `bump-version.yml` — version.json 갱신, 태그 생성, GitHub Release 발행
- `notify-notion.yml` — Release 발행 시 Notion DB에 자동 기록

## 호출 방법

각 앱 레포의 `.github/workflows/`에 얇은 래퍼를 두고 아래처럼 호출한다.
`source_repo`가 있으면 app repo 변경과 source repo 변경을 함께 AI 입력으로 사용하고,
없으면 app repo 변경만 사용한다.

```yaml
jobs:
  bump:
    uses: klausjg/flow-workflows/.github/workflows/bump-version.yml@main
    with:
      rev: ${{ inputs.rev }}
      title: ${{ inputs.title }}
      notes: ${{ inputs.notes }}
      app_name: "MyFlow"
      app_description: "사용자에게 보이는 앱 설명"
      fallback_title: "MyFlow 업데이트"
      source_repo: klausjg/myflow-source # app-only flow면 생략
    secrets:
      RELEASE_PAT: ${{ secrets.RELEASE_PAT }}
```

Notion/Backlog 기록은 release 발행 이벤트를 받아 별도 래퍼로 호출한다.

```yaml
jobs:
  notify:
    uses: klausjg/flow-workflows/.github/workflows/notify-notion.yml@main
    with:
      app_name: "MyFlow"
      app_page_id: "notion-page-id"
    secrets:
      NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
      NOTION_RELEASES_DB_ID: ${{ secrets.NOTION_RELEASES_DB_ID }}
      GAS_RELEASE_WEBHOOK_URL: ${{ secrets.GAS_RELEASE_WEBHOOK_URL }}
      GAS_RELEASE_SECRET: ${{ secrets.GAS_RELEASE_SECRET }}
```

## 사용 중인 앱 레포

- taskflow
- bodyflow
- coflow
- assetflow
- estateflow
- stockflow
