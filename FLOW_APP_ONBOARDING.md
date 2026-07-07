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
| 로컬 app 경로 | 예: `/Users/bourne/Projects/stockflow-app` (`~/Projects` 바로 아래) |
| 로컬 source 경로 | 예: `/Users/bourne/Projects/stockflow-source` (`~/Projects` 바로 아래) |
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
- 모든 값은 `gh secret list -R <owner>/<repo> --json name,updatedAt`로 **이름만** 확인한다. secret 원문을 로그에 찍지 않는다.

### 자동 등록 절차

CoFlow app 온보딩에서 아래 절차가 검증됐다. 새 app repo의 secret 세팅은 가능한 한 자동화한다.

1. **값을 유도할 수 있는 secret은 바로 등록한다.**
   - `NOTION_RELEASES_DB_ID`: Notion Releases DB 페이지/관계에서 확인한 DB id.
   - `GAS_RELEASE_WEBHOOK_URL`: TaskFlow GAS deploy id로 `https://script.google.com/macros/s/<deploy-id>/exec` 구성. 보통 `taskflow-source/.clasp-deploy-id`에서 읽는다.

2. **TaskFlow GAS Script Properties에 있는 값은 임시 HEAD route로 옮긴다.**
   - 대상: `NOTION_TOKEN` -> GitHub secret `NOTION_TOKEN`, `RELEASE_WEBHOOK_SECRET` -> `GAS_RELEASE_SECRET`.
   - 실제 repo를 직접 수정하지 말고 `/private/tmp/<taskflow-secret-bridge>` 같은 복사본에서만 임시 파일을 만든다.
   - 임시 route는 긴 nonce로 보호하고, production deployment가 아닌 **HEAD web app deployment**에만 올린다.
   - 호출할 때는 `Authorization: Bearer <clasp access_token>`를 붙인다.
   - 응답값은 화면에 출력하지 말고 바로 `gh secret set` stdin/body로 넘긴다.
   - 완료 즉시 실제 `taskflow-source`의 원래 tracked file set(`appsscript`, `Code`, `Index`, `notion_release_sync`, `notion-backlog-sync`)으로 Apps Script HEAD를 복원한다.
   - 복원 후 Apps Script content API에서 `CodexSecretBridge` 같은 임시 파일명이 없는지 확인한다.

3. **`RELEASE_PAT`는 기존 repo secret에서 복사할 수 없다.**
   - GitHub Actions secret은 원문 조회가 불가능하다.
   - 다른 Flow 앱과 같은 구성으로 맞추려면 현재 `gh` 인증 토큰을 사용할 수 있다: `gh auth token | gh secret set -R <owner>/<repo> RELEASE_PAT`.
   - 실행 전 `gh auth status`에서 `repo`, `workflow` scope가 있는지 확인한다.
   - 기존 모든 repo와 byte-for-byte 같은 PAT가 필요하면, 새 PAT를 발급해 모든 Flow repo의 `RELEASE_PAT`를 함께 회전한다.

4. **`clasp` CLI가 `Premature close`로 실패하면 curl 기반 API로 우회한다.**
   - `~/.clasprc.json`의 토큰이 깨졌거나 비어 있으면 Google OAuth callback을 로컬에서 받고, token endpoint는 `curl`로 교환해 복구할 수 있다.
   - `clasp push/run/deployments`가 Node fetch 문제로 실패해도, 같은 access token으로 Apps Script REST API를 `curl` 호출하면 동작할 수 있다.
   - 이 우회는 secret 값을 출력하지 않는 스크립트 안에서만 사용한다.

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

런타임별 주의:

- **GAS source repo**: clasp push task + `scripts/sync-claude.sh`(clasp push + git) + `release-auto.sh`.
- **Expo app repo (GAS source와 분리된 경우)**: clasp가 없으므로 sync는 **git 전용**으로 만든다 — `scripts/sync-claude.sh`(git add/commit + **`git pull --rebase`** + push). pull --rebase가 필요한 이유: release 워크플로(`bump-version.yml`)가 **app repo 원격에 `version.json`을 commit**하므로, 안 하면 sync 때마다 `non-fast-forward`로 push가 막힌다. `.vscode/tasks.json`엔 🤖 Sync, 🎯 Release Auto(= sync + `eas build`), dev(`expo start`/`--web`), 빌드(`eas build`). release 의미 분리는 §7 Expo 참조.

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
- **release 의미를 둘로 분리한다**:
  - **배포(앱을 사용자에게)** = app repo의 🎯 Release Auto = `scripts/release-auto.sh`(sync `git push` → **`eas build --local`**). 로컬 빌드는 EAS 클라우드 quota를 쓰지 않는다 — 대신 **JDK 17 + Android SDK**(iOS는 Xcode)가 필요. 도구체인 없으면 스크립트가 안내 후 중단하며, 클라우드 빌드는 `EAS_REMOTE=1 bash scripts/release-auto.sh`. OTA(`eas update`)는 `expo-updates` 설치 후 추가(Phase 2.5+).
  - **로컬 빌드 스크립트 표준**(estate/stock/body/co 공통): ① 빌드를 release-auto의 **맨 마지막**(version 기록 후)에 둬 빌드 실패가 기록을 막지 않게 함 ② `brew openjdk@17`는 keg-only라 PATH에 없을 수 있어 `JAVA_HOME`을 스크립트에서 직접 주입(`/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home`), `ANDROID_HOME` 미설정 시 `~/Library/Android/sdk` 폴백 ③ `--output ~/Desktop/<flow>-<profile>-<MMDD>.apk`로 APK 저장 ④ 빌드 후 `scripts/share-apk.sh`로 같은 와이파이 폰에 LAN(URL+QR) 전송(카톡 APK 차단 우회) ⑤ gitignore된 `google-services.json`은 로컬 빌드 아카이브에 안 들어오니 `GOOGLE_SERVICES_JSON`에 로컬 절대경로 주입(또는 Known Pitfalls의 `.easignore`). **monorepo**면 빌드만 `( cd apps/mobile && eas build --local )`.
  - **version 기록(version bump + GH Release + Notion + Backlog)** = source-driven. app repo의 `bump-version.yml`이 `source_repo`를 넘겨 AI 패치노트를 **소스 커밋 로그**에서 생성하므로, version 기록 release는 **source repo의 Release Auto**로 돌린다.
  - 따라서 **app의 release-auto.sh는 `bump-version`을 트리거하지 않는다**(트리거하면 소스 기준 노트가 나옴).
- app repo에도 `CLAUDE.md`를 둔다(§8) — 구조·작업 흐름·빌드/release 방식 기록. Expo 앱은 누락되기 쉽다.
- `.claude-commit-msg`를 app repo `.gitignore`에 등록(GAS의 `.claspignore`는 불필요).
- **dev 실행 셸 단축어**: app 경로가 길어 매번 `cd`가 번거로우므로 `~/.npm-global/bin`에 실행 스크립트 3종을 둔다 — `<flow>-start`(`cd <app경로> && exec npx expo start --port <고정포트> "$@"`), `<flow>-web`(`--web --port <고정포트>`), `<flow>-stop`(해당 포트 Metro kill, 비상용·평소엔 Ctrl-C). 이 폴더는 `.zshrc`에서 이미 PATH라 어느 폴더에서든 작동하고 **셸 프로파일 수정 없이 파일만 추가**하면 된다(자동화가 `.zshrc` 수정을 차단하면 alias 대신 이 방식). 이름은 경로가 아니라 flow 이름 기준(`bodyflow-app/apps/mobile` → `bodyflow-*`). `chmod +x` 필수.
  - **앱별 고정 포트** — 여러 앱을 동시에 띄워 번갈아 확인할 수 있고, `-stop`이 자기 앱만 정확히 종료한다. 고정하지 않으면 둘째 앱이 자동으로 다른 포트(8082…)로 떠서 `-stop`이 못 잡는다. 현재 배정: estateflow `8081`, stockflow `8082`, bodyflow `8083`, coflow `8084`. **새 앱은 마지막 포트+1**(다음은 `8085`). start/web에 `--port <포트>`, stop은 `lsof -ti:<포트>`로 kill.

## 8. Local Workspace

- **`-app`과 `-source` 폴더는 둘 다 `/Users/bourne/Projects` 바로 아래에 둔다** (`~/Projects/<flow>-app`, `~/Projects/<flow>-source`). 부모 폴더로 묶지 않는다 — 셸 단축어·자동화 스크립트가 이 표준 경로를 전제한다. (2026-07-02, stockflow-app이 `stockflow/stockflow-app` 중첩으로 잘못 배치됐던 것을 표준 위치로 이동하며 확정)
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

### 업데이트 로그

- **2026-07-02 · StockFlow (로컬 폴더 표준 위치)**
  - 발견: stockflow-app이 `~/Projects/stockflow/stockflow-app`으로 부모 폴더에 중첩돼 있었음. 표준은 **`-app`/`-source` 모두 `~/Projects` 바로 아래** (taskflow-app, coflow-app 등과 동일).
  - 조치: `~/Projects/stockflow-app`으로 이동, `~/.npm-global/bin`의 stockflow-{start,web,stop,build,ota,update} 경로 수정, 부모 폴더의 `.claude/launch.json` 설정(design-preview)을 앱 쪽으로 병합.
  - 재발 방지: §8과 새 앱 변수표의 경로 예시를 표준 위치로 정정. 새 앱 온보딩 시 부모 폴더를 만들지 말 것.

- **2026-06-09 · EstateFlow (Expo app)**
  - 발견: 이미 온보딩된 앱인데도 `estateflow-app`(Expo)에 §5 VSCode task와 §8 `CLAUDE.md`가 없었음. release는 source-driven인데 app에 Release Auto를 두려다, `bump-version.yml`의 `source_repo: klausjg/estateflow-source` 때문에 소스 기준 패치노트가 생성되는 구조를 확인.
  - 추가한 단계: app repo에 git 전용 `scripts/sync-claude.sh` + `.vscode/tasks.json`(Sync / `expo start`·`--web` / `eas build`), `CLAUDE.md`, `.gitignore`에 `.claude-commit-msg` 등록.
  - 재발 방지: §5 "런타임별 주의", §7 "Expo 앱"의 source-driven release·CLAUDE.md·gitignore 항목, Known Pitfalls에 반영.
  - **운영 중 추가 발견(같은 날)**:
    - sync push가 `non-fast-forward`로 거부 — release 워크플로가 app repo 원격에 `version.json`을 bump하기 때문. → `sync-claude.sh`에 `git pull --rebase origin main` 추가.
    - 첫 sync의 `git add -A`가 `.env.save`(secret 백업)·`.pyc`·`.claude/`까지 커밋. push가 거부돼 **GitHub엔 미유출**, 커밋 amend로 제거 + `.gitignore` 보강(`.env.save`/`.claude/`/`__pycache__`/`*.pyc`).
    - app Release Auto 결정 정정: 처음엔 "app엔 Release Auto 두지 않음"이었으나, 사용자가 app 배포 버튼을 원해 **`release-auto.sh` = sync + `eas build`(배포 전용)** 로 추가. version 기록(bump-version)은 그대로 source-driven 유지(app에서 트리거 안 함).
    - EAS 무료 quota 소진으로 빌드 실패 → Expo 앱 빌드를 **`eas build --local`** 기본으로 전환(quota 안 씀, JDK17+Android SDK 필요, `EAS_REMOTE=1`이면 cloud). 적용: estateflow-app(release-auto = sync+로컬빌드, 📦/🍎 task). stockflow-app(app-first, full 파이프라인)은 **기존 sync+bump+Notion+Backlog 유지하고 `eas build --local`을 맨 마지막 단계(④, version 기록 후)로 추가** — 사용자 요청대로 빌드를 끝에 둬서 **빌드가 실패해도 version 기록은 이미 완료/보존**되게 함. 원칙: 로컬 빌드는 항상 **추가**, 기존 release-auto 기능은 제거 금지. 추가로 gitignore된 `google-services.json`이 `eas build --local` 아카이브에서 빠져 prebuild가 실패 → **`.easignore`** 로 포함시킴(비밀 `.env`/`firebase-service-account*.json`은 계속 제외). bodyflow-app(monorepo)도 동일하게 release-auto(루트) 끝에 빌드 추가하되 Expo 앱이 `apps/mobile`이라 **`( cd apps/mobile && eas build --local )`**. → 3개 Expo 앱(estate/stock/body) 모두 build-last 일관 적용.

- **2026-06-22 · CoFlow App (GitHub repo + release secret onboarding)**
  - 발견: 새 `coflow-app` repo를 만들고 workflow wrapper를 올려도 app repo secrets가 비어 있으면 release/Notion/GAS 후처리가 실패한다. 기존 `coflow`/`taskflow` repo에는 secret 이름이 있어도 GitHub는 원문 값을 다시 읽을 수 없다.
  - 추가한 단계: §3 `GitHub Secrets And PAT`에 자동 등록 절차를 추가. Notion Releases DB id와 TaskFlow GAS webhook URL은 구조에서 유도해 등록하고, TaskFlow GAS Script Properties의 `NOTION_TOKEN`/`RELEASE_WEBHOOK_SECRET`은 임시 nonce-protected HEAD route로 읽어 바로 `gh secret set`에 넘긴 뒤 즉시 제거하는 방식. `RELEASE_PAT`는 `gh auth token`을 사용하되 `repo`/`workflow` scope 확인을 필수로 기록.
  - 부수 발견: `clasp` CLI가 Google API 호출에서 `Premature close`를 내도 curl 기반 Apps Script REST API는 정상 동작할 수 있음. `~/.clasprc.json`이 비었을 때는 로컬 OAuth callback + curl token exchange로 복구 가능.
  - 재발 방지: secret 값은 로그/문서에 남기지 않고, 검증은 `gh secret list`의 이름 목록과 Apps Script HEAD content의 임시 helper 부재만으로 한다. production deployment는 건드리지 않는다.

- **2026-06-10 · StockFlow / BodyFlow (Expo dev 셸 단축어)**
  - 발견: Expo dev 서버 실행에 매번 긴 `~/Projects/.../app` 경로를 `cd` 해야 해 번거로움. 사용자가 `~/Projects`의 **모든 Expo 앱**에 동일 단축어를 요청.
  - 추가한 단계: §7 Expo "dev 실행 셸 단축어" — `~/.npm-global/bin`에 `<flow>-start`/`-web`/`-stop` 3종(stockflow·estateflow·bodyflow 적용 완료). `.zshrc` alias가 자동화 환경에서 차단돼, PATH 폴더에 실행 스크립트를 두는 방식으로 회피.
  - 부수 발견: 어느 앱이 Expo인지 CLAUDE.md 구조표로 판단하면 누락된다 — **bodyflow는 문서상 GAS PWA지만 실제 `bodyflow-app/apps/mobile`에 Expo 앱이 별도 존재**. 식별은 `find ~/Projects -name node_modules -prune -o -name package.json -exec grep -l '"expo"' {} \;`.
  - 재발 방지: Known Pitfalls에 "Expo 앱 식별은 expo 의존성 스캔", "Expo 앱 dev 단축어 누락" 반영.
  - 후속(같은 날): 여러 앱 동시 실행·번갈아 확인 위해 **앱별 고정 포트** 지정 — estateflow 8081 / stockflow 8082 / bodyflow 8083, 새 앱 +1. start/web에 `--port`, stop은 해당 포트만 kill(→ `-stop`이 자기 앱만 정확히 종료, 8081 충돌 footgun 해소). §7 Expo "앱별 고정 포트" 반영.
  - 2026-06-18 후속: CoFlow Expo 앱 단축어 추가 — coflow 8084 배정. 다음 신규 Expo 앱은 8085부터 사용.

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
- 기존 GitHub secret 원문을 복사하려고 함 — secret 값은 조회 불가. 새 repo에는 `gh auth token` 또는 새로 발급한 PAT를 등록하고, byte-for-byte 동일성이 필요하면 모든 repo secret을 함께 회전한다.
- GitHub secret은 등록했지만 workflow permissions가 부족함
- GAS Script Properties의 `NOTION_TOKEN`/`RELEASE_WEBHOOK_SECRET`를 옮긴 뒤 임시 Apps Script helper를 HEAD에 남김 — 반드시 원래 file set으로 HEAD를 복원하고 content API에서 임시 파일 부재를 확인한다.
- `clasp` CLI의 Node fetch가 `Premature close`를 내는데 인증 자체가 깨졌다고 단정함 — curl 기반 token exchange/API 호출로 복구·우회 가능성을 먼저 확인한다.
- `clasp push`는 됐지만 production deployment가 옛 버전을 계속 서빙함
- 새 branch에서 upstream이 없어 release script가 `git push`에서 실패함
- Notion Releases row는 생겼지만 앱 페이지 relation이 비어 있음
- PWA service worker가 오래된 rev를 계속 잡고 있음
- Expo OTA로 해결할 수 없는 native 변경을 OTA로 배포하려고 함
- Expo app repo에 sync용 `.vscode/tasks.json` + git 전용 `sync-claude.sh`가 빠짐 (GAS 패턴을 그대로 복사하려다 clasp 단계 때문에 막힘)
- app의 Release Auto가 `bump-version`까지 트리거 → `source_repo` 때문에 소스 기준 패치노트 생성(app-only 변경 노트 불일치). app Release Auto는 `eas build`(배포)만, version 기록은 source에서.
- Expo app repo에 `CLAUDE.md` 누락
- `.claude-commit-msg`를 app repo `.gitignore`에 미등록
- release 워크플로가 app repo 원격에 `version.json`을 bump → 로컬이 매번 뒤처져 sync push가 `non-fast-forward`로 거부됨 (sync 스크립트에 `git pull --rebase` 필요)
- 첫 sync의 `git add -A`가 그동안 미커밋이던 secret/정크(`.env.save`·`__pycache__`·`.pyc`·`.claude/`)까지 통째로 커밋 → push 전에 `.gitignore` 보강 + 커밋에서 제거. (`.env`만 ignore돼 있고 `.env.save`는 빠지기 쉬움)
- EAS 무료 플랜 **월 빌드 quota 소진**(변경마다 cloud `eas build`를 돌린 결과) → 빌드 실패. Expo 앱 Release Auto는 **`eas build --local`** 기본(quota 안 씀, JDK17+Android SDK 필요)으로 두고, 잦은 테스트는 dev빌드+reload/`npm run web`로. cloud는 `EAS_REMOTE=1` 또는 quota 리셋·업그레이드 시.
- `eas build --local`은 프로젝트를 tar로 묶을 때 `.gitignore`(있으면 `.easignore`)를 따름 → **gitignore된 `google-services.json` 등 빌드 필수 파일이 아카이브에서 빠져 prebuild가 `ENOENT`로 실패**. `.easignore`를 만들어 그 파일만 포함(비밀 `.env`/`firebase-service-account*.json`은 계속 제외). (stockflow에서 발견)
- 로컬 빌드는 release-auto의 **맨 마지막 단계**에 둔다 — 빌드(느리고 실패 잦음)가 version 기록(bump/Notion/Backlog)을 막지 않도록. 빌드 실패해도 기록은 보존.
- Expo 앱 식별을 CLAUDE.md 구조표에 의존 → 숨은 Expo 앱 누락(예: bodyflow가 문서상 GAS지만 `bodyflow-app/apps/mobile`이 Expo). `package.json`의 expo 의존성 스캔으로 식별.
- Expo 앱에 dev 실행 셸 단축어(`<flow>-start`/`-web`/`-stop`)가 없어 매번 긴 app 경로를 `cd` (→ `~/.npm-global/bin`에 스크립트 3종, §7 Expo)
- 여러 Expo 앱을 고정 포트 없이 띄우면 둘째부터 자동으로 다른 포트(8082…)로 떠 `-stop`이 못 잡고 어느 게 어느 앱인지 헷갈림 → 앱별 고정 포트(§7 Expo: estate 8081·stock 8082·body 8083·신규 +1)
