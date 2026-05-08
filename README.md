# flow-workflows

Flow 시리즈 앱 레포에서 공유하는 재사용 가능한 GitHub Actions 워크플로.

## 포함 워크플로

- `bump-version.yml` — version.json 갱신, 태그 생성, GitHub Release 발행
- `notify-notion.yml` — Release 발행 시 Notion DB에 자동 기록

## 호출 방법

각 앱 레포의 `.github/workflows/`에 얇은 래퍼를 두고 아래처럼 호출:

\`\`\`yaml
jobs:
  bump:
    uses: klausjg/flow-workflows/.github/workflows/bump-version.yml@main
    with:
      title: ${{ inputs.title }}
    secrets:
      RELEASE_PAT: ${{ secrets.RELEASE_PAT }}
\`\`\`

## 사용 중인 앱 레포

- taskflow
- bodyflow
- coflow
- assetflow
