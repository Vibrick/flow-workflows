# Flow App Onboarding Runbook

새 Flow 시리즈 앱을 만들 때 반복해야 하는 작업을 한 곳에 모은 런북이다.
목표는 새 앱 이름과 기본 정보만 정하면 Notion, GitHub, VSCode task, TaskFlow, release 자동화까지 빠짐없이 붙이는 것이다.

## 사용 방법

새 앱을 만들 때 Codex나 작업자에게 아래처럼 요청한다.

```text
/Users/bourne/Projects/flow-workflows/FLOW_APP_ONBOARDING.md 를 기준으로
새 앱 <AppName> 온보딩을 진행해줘.

앱 정보:
- 표시명:
- repo 이름:
- source repo 필요 여부:
- Notion Apps 페이지:
- 앱 설명:
- 런타임: GAS+PWA / Expo / 기타
- TaskFlow 카테고리명:
```

## 원칙

- Flow 시리즈의 기본 구조는 `app repo`와 필요 시 `source repo`를 분리한다.
- `source repo`는 GAS 또는 백엔드 소스의 작업 공간이다.
- `app repo`는 사용자가 접하는 PWA, Expo 앱, 또는 equivalent release 단위다.
- `version.json`, GitHub Release, release workflow는 `app repo`에서만 관리한다.
- `source repo`에는 `version.json`이나 release workflow를 두지 않는다.
- TaskFlow는 단순 sync 대상이 아니라 Flow 시리즈 작업 허브다. 새 앱은 TaskFlow의 정식 카테고리로 추가한다.

## 새 앱 변수

아래 값을 먼저 확정한다.

| 항목 | 값 |
|---|---|
| 앱 표시명 | 예: StockFlow |
| 내부 category key | 예: StockFlow |
| app repo | 예: `klausjg/stockflow` |
| source repo | 예: `klausjg/stockflow-source` |
| 로컬 app 경로 | 예: `/Users/bourne/Projects/stockflow/stockflow-app` |
| 로컬 source 경로 | 예: `/Users/bourne/Projects/stockflow/stockflow-source` |
| Notion Apps page id |  |
| Notion Releases DB id | 보통 기존 공통 DB |
| TaskFlow 카테고리명 | 보통 앱 표시명과 동일 |
| release workflow name | 예: `StockFlow Release` |
| app description | 패치노트 AI가 참고할 사용자 관점 설명 |
| fallback title | 예: `StockFlow 업데이트` |

## 1. Notion

### Apps DB 페이지

- 다른 Flow 시리즈 앱과 같은 구조로 Apps DB에 새 앱 페이지를 만든다.
- 페이지 제목, 설명, 상태, 관련 링크, repo 링크를 채운다.
- 기존 앱 페이지의 섹션 구조를 복제한다.
- Apps page id를 온보딩 변수에 기록한다.

### Releases relation

- 새 앱의 GitHub Release가 Notion Releases DB에 기록될 수 있어야 한다.
- release workflow의 `app_page_id`가 새 Apps page id를 가리키는지 확인한다.
- 첫 release 후 Releases DB row와 앱 페이지 relation이 연결됐는지 확인한다.

## 2. GitHub Repos

### App repo

- repo를 생성한다.
- PWA 또는 app-equivalent 산출물을 둔다.
- `version.json`을 app repo에 둔다.
- `.github/workflows/bump-version.yml` wrapper를 추가한다.
- 필요하면 `.github/workflows/notify-notion.yml` wrapper를 추가한다.

### Source repo

source repo가 필요한 앱이면 다음을 준비한다.

- `appsscript.json` 또는 해당 런타임 설정
- `.clasp.json`, `.claspignore` if GAS
- `scripts/release-auto.sh`
- `scripts/sync-claude.sh`
- `.vscode/tasks.json`
- `CLAUDE.md`
- `README.md`

주의:

- source repo에는 `version.json`을 만들지 않는다.
- source repo에는 app release workflow를 두지 않는다.

## 3. GitHub Secrets And PAT

app repo의 Actions secrets를 등록한다.

| Secret | 용도 |
|---|---|
| `RELEASE_PAT` | release workflow가 commit/tag/push할 때 사용 |
| `NOTION_TOKEN` | Notion Releases 기록 |
| `NOTION_RELEASES_DB_ID` | 공통 Releases DB |
| `GAS_RELEASE_WEBHOOK_URL` | TaskFlow/Backlog 후처리 webhook |
| `GAS_RELEASE_SECRET` | webhook 인증 |

체크:

- `RELEASE_PAT`이 새 repo에 접근 가능한지 확인한다.
- PAT이 selected repositories 방식이면 새 repo를 명시적으로 추가한다.
- workflow가 `contents: write`, `models: read` 권한을 갖는지 확인한다.

## 4. Shared Release Workflow

app repo의 workflow wrapper는 `flow-workflows` reusable workflow를 호출한다.

필수 입력:

- `app_name`
- `app_description`
- `fallback_title`
- `source_repo` if source repo exists

Notion notify wrapper 필수 입력:

- `app_name`
- `app_page_id`

첫 실행 확인:

- `version.json` rev 증가
- `rev-N` tag 생성
- GitHub Release 생성
- Notion Releases DB 기록
- 앱 페이지 relation 연결
- TaskFlow Backlog 후처리 성공

## 5. VSCode Task

새 앱을 반복 운영할 수 있도록 source repo 또는 app repo에 VSCode task를 둔다.

필수 task:

- release task: `scripts/release-auto.sh` 실행
- 필요 시 clasp push task
- 필요 시 dev/test task

검토할 단축키:

- 현재 일부 문서에는 `Cmd+Shift+R`, 사용자 운영 표현에는 `Cmd+Shift+B`가 섞여 있을 수 있다.
- 새 앱 추가 시 실제 VSCode keybinding과 task label을 통일한다.
- 문서에는 실제 사용하는 단축키 하나만 적는다.

주의:

- fresh branch에서는 `git push`가 upstream 없이 실패할 수 있다.
- 최초 1회 `git push --set-upstream origin <branch>`가 필요한지 확인한다.

## 6. TaskFlow Integration

새 앱은 TaskFlow의 정식 카테고리로 추가한다.

### UI 카테고리

- TaskFlow 앱 선택 목록에 새 앱을 추가한다.
- 필터, 색상, 아이콘, 표시명, 정렬 위치가 있다면 함께 반영한다.
- 오늘 보기, backlog 보기, 완료 보기 등 category 기반 화면에 새 앱이 보이는지 확인한다.

### Notion 매핑

- 실제 버튼-driven create path에 새 앱 page id를 추가한다.
- 보조 sync 파일만 수정해서는 부족하다.
- `createTaskInSheet()` 또는 동급 UI 진입점에서 사용하는 매핑을 확인한다.
- `APP_PAGE_IDS_TF` 같은 TaskFlow 전용 mapping에도 추가한다.

### E2E 검증

아래 흐름을 실제로 확인한다.

- 업무 추가
- 오늘 추가
- 바로 시작
- 시작 전 -> 진행 중
- 진행 중 -> 완료
- Notion Backlog row 생성
- Notion Backlog의 앱/category/page relation이 새 앱을 가리킴

실패 패턴:

- 보조 sync 파일에는 새 앱이 있지만 UI 버튼 경로의 mapping에는 없어 create sync가 조용히 스킵된다.
- Notion select/status option에는 값이 있지만 relation page id가 없어 relation 연결이 빠진다.

## 7. Runtime Specific Setup

### GAS 앱

- `.clasp.json` script id 확인
- `appsscript.json` 확인
- `.claspignore` 확인
- `clasp push` 가능 여부 확인
- production deployment id 관리
- `clasp version` + `clasp deploy`가 release task에서 실행되는지 확인

주의:

- `clasp push`만으로 production URL이 최신 버전을 서빙하지 않을 수 있다.
- release task가 새 version을 만들고 active deployment를 swap하는지 확인한다.

### PWA 앱

- `index.html`
- `manifest.json`
- `sw.js`
- `icon.png`
- `favicon.png`
- `version.json`
- GitHub Pages 설정
- 캐시 busting 전략

### Expo 앱

- `app.json` 또는 `app.config.js`
- bundle id / package name
- EAS project 설정
- 필요한 secrets
- EAS Update 또는 EAS Build 적용 시점
- native module 변경이면 OTA가 아니라 새 build 필요

## 8. Local Workspace

- `/Users/bourne/Projects` 아래 위치를 정한다.
- app repo와 source repo가 같은 부모 아래 있으면 운영이 쉽다.
- 필요하면 `_flow_repos` 또는 workspace 파일에 추가한다.
- README와 `CLAUDE.md`에 로컬 경로, repo, release 방법을 적는다.

## 9. Smoke Test

온보딩 완료 판단은 아래가 모두 통과했을 때다.

- 앱 repo clone/pull 가능
- source repo clone/pull 가능 if exists
- VSCode release task가 실행됨
- GitHub workflow가 성공함
- GitHub Release가 생성됨
- `version.json`이 갱신됨
- Notion Releases DB에 row가 생김
- Notion 앱 페이지 relation이 연결됨
- TaskFlow에서 새 앱 카테고리로 업무를 만들 수 있음
- TaskFlow 업무가 Notion Backlog에 생성됨
- TaskFlow 상태 변경이 Notion에 반영됨

## 10. 유지보수

새 앱을 추가하면서 빠진 단계가 발견되면 이 문서에 바로 반영한다.

업데이트할 때는 다음을 남긴다.

- 추가한 단계
- 발견한 실패 패턴
- 어느 앱 온보딩에서 발견했는지
- 재발 방지 체크 항목

## Codex 실행 체크리스트

새 앱 온보딩 작업을 맡은 Codex는 다음 순서로 움직인다.

1. 이 문서를 읽고 새 앱 변수를 표로 정리한다.
2. 기존 Flow 앱 중 가장 비슷한 앱을 하나 고른다.
3. repo 구조와 release 구조를 비교한다.
4. Notion Apps page id를 확보한다.
5. app repo workflow wrapper를 만든다.
6. source repo task/release script를 만든다.
7. GitHub secrets와 PAT 접근 권한을 확인한다.
8. TaskFlow UI 카테고리와 실제 button-driven mapping을 추가한다.
9. release smoke test를 실행한다.
10. TaskFlow -> Notion Backlog E2E를 실행한다.
11. 발견한 예외를 이 문서에 업데이트한다.

## Known Pitfalls

- `source repo`에 `version.json`을 만들어 release 기준이 흔들림
- `APP_PAGE_ID`를 workflow에는 넣었지만 TaskFlow mapping에는 안 넣음
- TaskFlow 보조 sync 파일만 고치고 실제 UI create path를 놓침
- GitHub PAT selected repository 목록에 새 repo를 추가하지 않음
- GitHub secret은 등록했지만 workflow permissions가 부족함
- `clasp push`는 됐지만 production deployment가 옛 버전을 계속 서빙함
- 새 branch에서 upstream이 없어 release script가 `git push`에서 실패함
- Notion Releases row는 생겼지만 앱 페이지 relation이 비어 있음
- PWA service worker가 오래된 rev를 계속 잡고 있음
- Expo OTA로 해결할 수 없는 native 변경을 OTA로 배포하려고 함
