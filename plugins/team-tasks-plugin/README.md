# team-tasks-plugin

사내 일감 할당 앱(team-tasks) 작업 시 자동화 자산 묶음. slash command·skill·subagent 를 한 번에 설치한다.

## 전제 조건

이 플러그인은 다음 스택의 프로젝트를 가정한다.

- **Next.js 15** App Router (`src/app/api/.../route.ts`)
- **React 19**, **TypeScript strict**
- **Supabase** Postgres + Auth (RLS 사용), **Google OAuth**
- 배포: **Vercel**
- UI: **shadcn/ui + Radix**, E2E: **Playwright**

스택이 이와 다르면 `api-conventions`·`scaffold-crud` 스킬과 모든 reviewer 에이전트는 그대로 적용되지 않는다 (경로·RLS 가정·Supabase 클라이언트 패턴이 박혀 있음). `commit-summary`·`git-commit` 정도만 범용.

## 자산 목록

### Slash commands — `commands/`

| 명령 | 한 줄 요약 |
|---|---|
| `/team-tasks-plugin:commit-summary` | 워킹트리 변경(`git status` + `git diff HEAD`)을 한 문장 커밋 메시지로 요약. |
| `/team-tasks-plugin:config-review` | `package.json`·`tsconfig.json`·`.env.example` 을 보안·구버전·잘못된 설정 관점에서 일괄 검토. |
| `/team-tasks-plugin:git-commit` | 워킹트리 변경을 적절한 메시지로 커밋. `git` 계열 Bash 만 허용. |
| `/team-tasks-plugin:security-scan` | 코드베이스에서 SQL injection·XSS·노출된 비밀키를 탐색 (read-only). |

### Skills — `skills/`

| 스킬 | 한 줄 요약 |
|---|---|
| `team-tasks-plugin:api-conventions` | `src/app/api/` REST 핸들러 작성·수정 시 자동 참조 — 응답 포맷, HTTP 상태 코드, 인증 가드, 입력 검증, RLS, 페이지네이션 규칙 모음. |
| `team-tasks-plugin:scaffold-crud` | 새 도메인 추가 요청 시 자동 호출 — 테이블+RLS 마이그레이션, REST API, 목록/작성/삭제 페이지를 한 번에 생성. |

### Subagents — `agents/`

| 에이전트 | 한 줄 요약 |
|---|---|
| `team-tasks-plugin:team-tasks-reviewer` | 종합 리뷰어 — 보안·정확성·성능·가독성·컨벤션 5개 관점으로 최근 변경 코드 점검. |
| `team-tasks-plugin:reviewer-security` | 보안 전용 — RLS 누락/우회, OAuth/세션/보호 라우트 누수, 비밀키 노출, SQLi·XSS 4개 관점만. |
| `team-tasks-plugin:reviewer-perf` | 성능 전용 — Supabase N+1·인덱스, Server/Client 경계 & 번들, 리스트 렌더 비용, 자산 로딩 4개 축만. |
| `team-tasks-plugin:team-tasks-test-coverage-reviewer` | 테스트 커버리지 전용 — unit·integration·E2E·회귀 위험 누락 항목을 P0/P1/P2 로 분류. |

## 네임스페이스 호출 예시

플러그인이 설치되면 모든 자산은 `team-tasks-plugin:` 네임스페이스로 호출한다.

```text
# slash command
/team-tasks-plugin:commit-summary
/team-tasks-plugin:security-scan

# skill (보통 자동 트리거지만 명시 호출도 가능)
team-tasks-plugin:api-conventions
team-tasks-plugin:scaffold-crud

# subagent (Task tool 의 subagent_type 또는 자연어 위임)
"team-tasks-plugin:team-tasks-reviewer 로 최근 변경 리뷰해줘"
"team-tasks-plugin:reviewer-security 만 돌려줘"
```

리뷰어 에이전트끼리는 역할이 분리돼 있다 — 종합 리뷰는 `team-tasks-reviewer`, 한 관점만 빠르게 보려면 `reviewer-security` / `reviewer-perf` / `team-tasks-test-coverage-reviewer` 중 하나를 직접 호출.

## 설치

이 플러그인은 같은 저장소 루트의 `team-tasks-marketplace` 를 통해 배포된다.

```text
/plugin marketplace add jo-narae/team-tasks-marketplace
/plugin install team-tasks-plugin@team-tasks-marketplace
```
