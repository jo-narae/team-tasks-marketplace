---
name: scaffold-crud
description: 새 도메인을 추가하려는 요청에 자동 호출. 트리거 발화 — "X 리소스 만들어줘/추가해줘", "X CRUD 짜줘", "X 테이블 추가", "X 보드 만들어줘", "X 컬렉션 추가", 단수형 영어 명사(users, notes, comments)나 한글 도메인 명사가 등장하는 모든 신규 모델 도입 요청. 사용자가 'CRUD' 를 명시하지 않아도, 새 테이블·새 API·새 페이지가 함께 필요하다는 신호면 이 스킬이 우선. DB 테이블+RLS migration, REST API(/api/<plural>), 목록·작성·삭제 페이지를 한 번에 생성. 기존 리소스에 컬럼·필터만 추가하는 경우는 feature-extension, 버그 수정은 investigate.
---

# scaffold-crud

리소스 한 단어를 받아 풀스택 CRUD 를 일괄 생성한다. 단계 사이에 사용자에게 묻지 않고 1→6 을 한 번에 끝낸다.

## 왜 이 스킬이 필요한가

즉흥으로 만들면 두 가지가 자주 어긋난다.

- **RLS 누락** — `enable row level security` 또는 4종 policy 중 일부가 빠진 채 production 에 적용되면 anon key 로 모든 row 가 노출된다. 정적 보안 사고.
- **plural 오류** — `users` 가 입력으로 들어오면 *이미 복수* 인데 다시 plural 화 하려다 `userss` 같은 사고가 나거나, `category → categorys` 처럼 영어 규칙을 어긴 테이블명이 박힌다.

본문의 *템플릿* 과 *결정 표* 를 그대로 따르면 둘 다 자동으로 안전해진다. 매번 즉흥으로 짜지 않는 게 핵심.

## 사용 시점

자동 호출 신호 (description 과 동기화)

- "X 리소스 만들어줘/추가해줘", "Y CRUD 짜줘", "Z 보드 추가해줘"
- 단수형 영어 명사 (`note`, `comment`, `user`) 또는 한글 도메인 명사
- 새 테이블·새 API·새 페이지가 *모두* 필요하다는 신호

**쓰지 않는 경우**

- 기존 리소스에 컬럼·필터·뷰만 추가 → `feature-extension`
- 버그 수정·로직 변경 → `investigate`
- DB·API 가 불필요한 정적/마케팅 페이지

## plural 결정 규칙 (반드시 이 표 참조)

입력 단어를 먼저 분석. 표 안에서 결정되면 사용자에게 다시 묻지 않는다.

| 입력 패턴 | plural 처리 | singular 처리 | 예 |
|---|---|---|---|
| 단수 영어 일반 | `+s` | 입력 그대로 | `note → notes` |
| 단수 + `-y` (자음 앞) | `-y → -ies` | 입력 그대로 | `category → categories`, `entry → entries` |
| 단수 + `-s/-x/-z/-ch/-sh` | `+es` | 입력 그대로 | `box → boxes`, `wish → wishes` |
| 단수 + `-f/-fe` | `-f → -ves` | 입력 그대로 | `leaf → leaves` |
| **이미 복수형** (`users`, `notes`, `tasks`) | 입력 그대로 사용 | 마지막 `s` 제거 | `users → users` (plural), `user` (singular) |
| 불규칙 영어 | 표준 불규칙형 | 입력 그대로 | `person → people`, `child → children`, `man → men` |
| 불가산 명사 (`data`, `news`, `series`) | 거부 | — | 사용자에게 명사 교체 요청 (`articles`, `headlines` 등) |
| 한글·합성어 | 거부 | — | 영어 단수형 한 단어 다시 요청. 제멋대로 transliterate 금지. |
| SQL 키워드·시스템 충돌 (`order`, `user`, `group`) | prefix 권장 | — | `app_users`, `customer_orders`. 사용자에게 한 번 알리고 진행. |

가장 흔한 함정 — `users 리소스 만들어줘` 입력. *이미 복수* 분기로 가서 `users` 그대로 쓰고, 타입명은 `User` 로. `usersS` 같은 사고 방지.

## 진행 순서

단계 사이에 묻지 않고 1→6 일괄.

1. **plural·singular 결정** — 위 표대로.
2. **migration 작성** — 아래 템플릿 그대로, `<plural>` 치환.
3. **migration 적용** — Supabase MCP `apply_migration` 호출. production 즉시 반영.
4. **REST API 생성** — 아래 skel + `api-conventions` 스킬 참조.
5. **페이지 생성** — `src/app/<plural>/page.tsx`. 기존 `src/app/page.tsx` (tasks) 의 구조·상태관리 그대로.
6. **검증** — `npm run lint && npm run typecheck`. 실패 시 자동 수정 1회 시도, 안 풀리면 stop + 사용자에 보고.

## 마이그레이션 템플릿 (이 블록을 *그대로* 사용)

`<plural>` 만 치환. enable·policy 4종이 모두 들어있다 — 이 중 하나라도 빠지면 보안 사고.

```sql
-- 1. 테이블
create table public.<plural> (
  id uuid primary key default gen_random_uuid(),
  created_by uuid not null references auth.users(id) default auth.uid(),
  created_at timestamptz not null default now()
  -- 도메인 컬럼 (예시):
  -- , title text not null
  -- , status text not null default 'todo'
);

-- 2. RLS 활성화 — 이 줄이 빠지면 아래 policy 들이 모두 무시된다
alter table public.<plural> enable row level security;

-- 3. policy 4종 — select/insert/update/delete 가 *모두* 있어야 한다
create policy "<plural>_select_own" on public.<plural>
  for select using (auth.uid() = created_by);

create policy "<plural>_insert_own" on public.<plural>
  for insert with check (auth.uid() = created_by);

create policy "<plural>_update_own" on public.<plural>
  for update using (auth.uid() = created_by) with check (auth.uid() = created_by);

create policy "<plural>_delete_own" on public.<plural>
  for delete using (auth.uid() = created_by);

-- 4. 인덱스
create index <plural>_created_by_idx on public.<plural>(created_by);
create index <plural>_created_at_idx on public.<plural>(created_at desc);
```

**왜 4종 모두 필요한가** — RLS 가 켜진 후엔 매치되는 policy 가 없으면 그 동작이 *기본 거부* (default deny). select 만 만들고 insert 를 빼면 POST API 가 RLS 에 막혀 조용히 실패한다. 한 묶음으로 두는 이유.

**왜 `enable row level security` 가 필수인가** — 이 줄이 없으면 policy 가 만들어져도 적용되지 않고 anon key 로 전체 테이블이 노출된다. policy 위에, 같은 migration 에 *항상* 포함.

## API 라우트 skel

상세 컨벤션은 `api-conventions` 스킬. 여기는 새 라우트의 최소 골격 — 인증 가드 + 화이트리스트 검증 + 소유자 강제가 박혀있다.

`src/app/api/<plural>/route.ts`:

```ts
import { NextResponse } from "next/server"
import { createClient } from "@/lib/supabase/server"

export async function GET() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return NextResponse.json({ error: "unauthorized" }, { status: 401 })

  const { data, error } = await supabase
    .from("<plural>")
    .select("*")
    .order("created_at", { ascending: false })
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function POST(request: Request) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return NextResponse.json({ error: "unauthorized" }, { status: 401 })

  // 입력 화이트리스트 — ...body 그대로 spread 금지
  const body = (await request.json()) as Record<string, unknown>
  const { title } = body  // 도메인에 맞게 추출
  if (typeof title !== "string" || !title.trim()) {
    return NextResponse.json({ error: "title_required" }, { status: 400 })
  }

  const { data, error } = await supabase
    .from("<plural>")
    .insert({
      title: title.trim(),
      created_by: user.id,  // 서버 강제, 클라이언트 입력 무시
    })
    .select()
    .single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data, { status: 201 })
}
```

`[id]/route.ts` 의 GET/PATCH/DELETE 는 `src/app/api/tasks/[id]/route.ts` 와 동일 구조 — Promise `params`, 인증 가드, `eq("id", id)`, 같은 응답 포맷.

## 컨벤션

| 항목 | 규칙 |
|---|---|
| 파일 경로 | API: `src/app/api/<plural>/route.ts` + `[id]/route.ts`. 페이지: `src/app/<plural>/page.tsx`. 한 컴포넌트 = 한 파일. |
| 도메인 타입 | `src/lib/types.ts` 에 인터페이스 (`User`, `Note` 등 PascalCase singular) + 필요시 `as const` 튜플 (status·priority 등). |
| Supabase 클라이언트 | 서버 라우트는 `@supabase/ssr` server client (`src/lib/supabase/server.ts`), 클라이언트는 browser client. anon key 만. |
| 응답 포맷·상태 코드·에러 | `api-conventions` 스킬 참조 (단일 출처). |
| UI | shadcn/ui — Card·Button·Input·Dialog·Label·Select 재사용. CVA + `cn()`. 아이콘 Lucide. |
| 주석·식별자 | 주석 한국어, 코드 영어 (CLAUDE.md). |
| 경로 별칭 | `@/*` → `src/*`. 상대경로 import 지양. |

## 완료 자가 검증 (생략 금지)

마지막에 이 체크리스트를 *선언적으로* 확인. 하나라도 ✗ 면 미완 — 사용자에게 항목 명시 후 수정.

- [ ] migration 에 `alter table ... enable row level security` 가 포함됐는가?
- [ ] migration 에 select·insert·update·delete policy 4종이 *모두* 있는가?
- [ ] migration 이 `apply_migration` 으로 실제 적용됐고, 응답에 성공이 확인됐는가?
- [ ] plural 이 위 표 규칙대로 결정됐는가? 입력이 *이미 복수* (`users`) 면 그대로 사용했는가?
- [ ] singular 타입명이 PascalCase 로 `src/lib/types.ts` 에 추가됐는가?
- [ ] API 라우트의 모든 핸들러 첫 줄이 `auth.getUser()` 가드인가?
- [ ] POST 의 INSERT 가 `created_by: user.id` 를 서버 측에서 강제하는가?
- [ ] `...body` spread 가 *없고* 화이트리스트 추출만 사용했는가?
- [ ] `npm run lint && npm run typecheck` 가 모두 통과했는가?

migration 이 적용된 상태에서 코드만 미완이면 "DB 에 테이블이 생긴 상태이고 코드는 미완" 임을 사용자에게 명확히 보고. 조용히 끝내지 않는다.

## 주의사항 (요약)

- **production DB 직접 DDL** — `apply_migration` 은 즉시 반영. 학습용 더미 리소스면 시작 시 "정리 단계에서 drop 예정" 한 줄 명시.
- **Service Role Key 노출 금지** — 새 라우트에서 anon + RLS 기본. service role 은 RLS 우회가 정말 필요한 경우만, 서버 전용 env. `NEXT_PUBLIC_*` 에 service role 절대 금지.
- **기존 `tasks` 패턴과의 일관성** — 다른 응답 형태·디렉터리 구조를 만들지 않는다. 학습자가 한 패턴만 알면 모든 리소스를 다룰 수 있어야.
