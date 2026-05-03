---
name: api-conventions
description: src/app/api/ 아래 새 라우트를 추가하거나 기존 라우트를 수정·리뷰할 때 참조. 응답 포맷·HTTP 상태 코드·인증 가드·입력 검증·소유자 강제·RLS·페이지네이션·에러 처리 등 Next.js App Router REST 핸들러의 프로젝트 규칙.
allowed-tools: Read, Edit, Write
---

# API 컨벤션

Next.js App Router (`src/app/api/.../route.ts`) REST 핸들러 작성 규칙. 새 라우트를 만들거나 기존 라우트를 수정할 때 이 문서를 따른다. 절차가 아닌 *규칙·예시* 모음.

## 라우트 구조

| 경로 | 핸들러 | 비고 |
|---|---|---|
| `src/app/api/<plural>/route.ts` | GET (목록), POST (생성) | |
| `src/app/api/<plural>/[id]/route.ts` | GET (단건), PATCH (부분 수정), DELETE | |
| `src/app/api/auth/{callback,logout}/route.ts` | OAuth 콜백·로그아웃 | 인증 가드 예외 |

`[id]` 라우트의 `params` 는 Next 15+ 에서 **Promise**. 시그니처:

```ts
type RouteContext = { params: Promise<{ id: string }> }

export async function GET(_request: Request, { params }: RouteContext) {
  const { id } = await params
  // ...
}
```

## 인증 가드 (필수)

콜백 외 **모든** 핸들러는 첫 줄에서 `auth.getUser()` 검증. 미인증은 401 즉시 리턴. 인라인으로 둔다 — 두 줄짜리라 헬퍼 추상화 비용 > 이득.

```ts
import { NextResponse } from "next/server"
import { createClient } from "@/lib/supabase/server"

export async function GET() {
  const supabase = await createClient()
  const {
    data: { user },
  } = await supabase.auth.getUser()

  if (!user) {
    return NextResponse.json({ error: "unauthorized" }, { status: 401 })
  }
  // 이후 비즈니스 로직
}
```

5+ 라우트가 같은 인증 + 부가 검사를 공유하게 되면 그때 헬퍼·middleware 도입 재검토.

## 응답 포맷

| 종류 | 형태 |
|---|---|
| 성공 (조회·수정) | `NextResponse.json(data)` — 데이터를 그대로 (래핑 없음) |
| 생성 성공 | `NextResponse.json(data, { status: 201 })` |
| 빈 응답 (DELETE 등) | `new Response(null, { status: 204 })` |
| 에러 | `NextResponse.json({ error: "<keyword>" }, { status })` |

에러 메시지는 짧은 영문 키워드(`unauthorized`, `not_found`, `title_required`) 또는 Supabase `error.message`. 스택·내부 path 노출 금지.

## HTTP 상태 코드

| 상황 | 코드 |
|---|---|
| 인증 실패 | 401 |
| 권한 없음 (RLS 통과 못함 등) | 403 |
| 단건 조회 실패 | 404 |
| 입력 검증 실패·수정·삭제 실패 | 400 |
| 생성 성공 | 201 |
| 삭제 성공 (본문 없음) | 204 |
| Supabase·서버 측 에러 | 500 |

## 입력 검증

**현재 라우트는 `...body` spread — 알려진 약점.** 새 라우트·리팩터링 시 화이트리스트로 좁힌다. zod 같은 라이브러리 미도입 (의존성 추가 의사결정 미합의), 도입 전엔 수동 패턴.

```ts
// 안 됨 — 클라이언트가 임의 컬럼 주입 가능
.insert({ ...body, created_by: user.id })

// 권장 — 화이트리스트
const { title, status } = (await request.json()) as {
  title?: unknown
  status?: unknown
}
if (typeof title !== "string" || !title.trim()) {
  return NextResponse.json({ error: "title_required" }, { status: 400 })
}

const insert = {
  title: title.trim(),
  status: typeof status === "string" ? status : "todo",
  created_by: user.id,
  assignee_id: user.id,
}
```

## 소유자 강제

INSERT 시 `created_by` 는 **항상 서버 측 `user.id`** 로 강제. 클라이언트 입력 신뢰 금지.

```ts
.insert({
  ...validated,
  created_by: user.id,                              // 강제
  assignee_id: validated.assignee_id ?? user.id,   // 기본값 본인
})
```

UPDATE 시 클라이언트가 `created_by` 를 보내도 무시 — 위 화이트리스트 검증 단계에서 차단.

## RLS 의존

서버 라우트는 **anon key + RLS** 로 동작. Service Role Key 로 RLS 우회 금지.

- `select("*")` 만으로 본인 데이터만 노출됨 — RLS 가 처리. 별도 `eq("created_by", user.id)` 중복 추가하지 않는다.
- 정책: `enable row level security` + select/insert/update/delete 4종 policy 를 같은 migration 에 묶어 작성.
- 패턴 예시는 `scaffold-crud` 스킬 참조.

## 페이지네이션 (미도입, 도입 시 권장 패턴)

현재 라우트는 전부 풀 셋 반환. 큰 데이터가 들어오는 리소스에 한해 cursor 기반 페이지네이션 도입.

```ts
const { searchParams } = new URL(request.url)
const limit = Math.min(Number(searchParams.get("limit") ?? 50), 200)
const cursor = searchParams.get("cursor")  // ISO timestamp 또는 id

let query = supabase
  .from("tasks")
  .select("*")
  .order("created_at", { ascending: false })
  .limit(limit)
if (cursor) query = query.lt("created_at", cursor)

const { data, error } = await query
if (error) return NextResponse.json({ error: error.message }, { status: 500 })

const nextCursor = data.length === limit ? data[data.length - 1].created_at : null
return NextResponse.json({ data, nextCursor })
```

cursor 가 도입되면 응답 형태가 단순 배열에서 `{ data, nextCursor }` 로 바뀐다 — 라우트 단위로 일관되게 적용하고 호출 측을 같이 갱신.

offset 페이지네이션은 큰 데이터셋에서 성능 함정 → 지양.

## 보안

- **Service Role Key 노출 금지** — `NEXT_PUBLIC_*` 또는 클라이언트 컴포넌트에 절대 박지 않는다. 서버 전용.
- **에러 본문 정보 누출 금지** — SQL 쿼리·스택트레이스·내부 path·env 값 노출 금지.
- **사용자 입력 로그 그대로 찍지 않기** — PII·비밀번호 후보가 흘러들 수 있다.
- **CSRF** — 현재는 same-origin + cookie 기반 세션 (Supabase) 이라 별도 방어 미도입. 외부 origin 에서 호출 받게 되면 그 시점에 토큰 기반으로 전환.

## 성능

- API P95 ≤ 500ms 가 전역 비기능 요구.
- Supabase 호출은 핸들러당 **1~2회**. N+1 패턴이면 join·view 또는 단일 RPC 로 합친다.
- 핸들러 내부에서 외부 fetch 가 필요하면 그 자체로 P95 위협 — Edge runtime 또는 캐시 레이어 검토.

## 자주 보이는 안티패턴

| 패턴 | 왜 문제인가 | 대안 |
|---|---|---|
| `...body` 그대로 insert/update | 클라이언트가 임의 컬럼 주입 가능 | 화이트리스트 |
| `created_by` 를 클라이언트가 지정 | 위변조 가능 | 서버에서 `user.id` 강제 |
| 인증 가드를 헬퍼·middleware 로 일찍 추상화 | 라우트 1~2개일 때 과한 추상 | 인라인, 5+ 시점에 재검토 |
| `eq("created_by", user.id)` 중복 추가 | RLS 가 이미 처리, 중복은 노이즈 | RLS 신뢰 |
| `error: err.stack` 노출 | 내부 구조 leak | `error.message` 또는 키워드 |
| offset 페이지네이션 (`range()`) 으로 큰 데이터 순회 | 깊은 offset 성능 저하 | cursor 기반 |
