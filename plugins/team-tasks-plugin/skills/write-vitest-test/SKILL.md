---
name: write-vitest-test
description: 함수 시그니처(또는 함수 코드)를 받아 vitest 단위 테스트 파일을 생성. 트리거 발화 — "X 함수 테스트 짜줘", "이 시그니처로 vitest 테스트 만들어줘", "X.ts 에 spec 추가", "유닛 테스트 작성해줘". happy path 와 edge case 를 한 파일에 담아 코로케이션 위치(`<target>.test.ts`)에 만든다. 통합/E2E·기존 테스트 디버깅·테스트 러너 설정 변경은 범위 밖.
allowed-tools: Read, Write, Edit, Bash
---

# write-vitest-test

함수 시그니처(또는 구현 코드)를 받아 vitest 테스트 파일 한 개를 만든다. happy path 와 edge case 를 *모두* 포함한다. 한 번에 끝내고 사용자에게 중간 질문하지 않는다.

## 왜 이 스킬이 필요한가

- **edge case 누락** — happy path 만 짜면 회귀 시 테스트가 못 잡는다. 빈값·경계·에러 분기를 *체크리스트로 강제* 한다.
- **위치 불일치** — `foo.ts` 인데 `foo.spec.js` 로 만들면 vitest `include` 패턴에 안 잡혀 "성공처럼 보이는 침묵 실패" 가 된다. 코로케이션 + 동일 확장자가 기본.

## 사용 시점

자동 호출 신호

- "이 함수 vitest 테스트 짜줘", "시그니처 줄 테니 테스트 만들어줘"
- "X.ts spec 추가해줘", "유닛 테스트 추가"
- 새 함수 작성 직후 *테스트가 같이 와야 한다* 는 흐름

**쓰지 않는 경우**

- 브라우저·E2E → `qa` / Playwright
- 기존 테스트 실패 분석 → `investigate`
- vitest config 변경

## 진행 순서

1→4 일괄. 시그니처가 모호할 때만 1 에서 한 번 묻는다.

1. **시그니처 파악** — 사용자가 코드를 줬으면 그대로, 파일 경로를 줬으면 Read 로 본다. 인자 타입·반환·throw/reject 분기 추출.
2. **테스트 파일 위치 결정** — 아래 표.
3. **케이스 도출 + 작성** — 체크리스트 4분류를 채워 템플릿에 끼운다.
4. **검증** — `npx vitest run <test-path>` 한 번. 실패 시 1회 자동 수정, 안 풀리면 stop + 보고.

## 테스트 파일 위치 표

| 대상 | 테스트 파일 |
|---|---|
| `src/lib/foo.ts` | `src/lib/foo.test.ts` |
| `src/lib/foo.tsx` | `src/lib/foo.test.tsx` |
| 기존에 `__tests__/` · `tests/` 관행 있음 | 기존 관행 따름 |
| `.spec.` 접미사 | 프로젝트가 *명시적으로* `.spec.` 일 때만. 기본은 `.test.` |

## 케이스 체크리스트 (4분류 — 빠뜨리지 않는다)

함수 하나당 전부 검토. 해당 없으면 "해당 없음" 으로 명시 패스.

- [ ] **happy path** — 일반 입력 1~2개. 가장 흔한 사용 형태.
- [ ] **경계값** — 시그니처에 해당하는 것만:
  - 숫자: `0`, 음수, 최대값
  - 문자열: `""`, 공백만, 매우 김
  - 배열: `[]`, 단일 요소, 중복 요소
  - 객체: `{}`, 누락 필드, optional 필드 부재
  - nullable: `null`, `undefined`
- [ ] **에러 분기** — `throw` 하거나 reject 하는 입력. `expect(() => fn(x)).toThrow(/msg/)` / `.rejects.toThrow(/msg/)`
- [ ] **부수효과·외부 의존** — fetch·DB·시간·랜덤이면 `vi.mock` / `vi.useFakeTimers()`. 없으면 패스.

## 템플릿

### 순수 함수 (sync)

```ts
import { describe, it, expect } from "vitest"
import { <fnName> } from "./<target>"

describe("<fnName>", () => {
  it("<happy path 설명>", () => {
    expect(<fnName>(<input>)).toBe(<expected>)
  })

  it("<edge — 빈 입력 등>", () => {
    expect(<fnName>(<edgeInput>)).toEqual(<edgeExpected>)
  })

  it("throws on <조건>", () => {
    expect(() => <fnName>(<badInput>)).toThrow(/<message>/)
  })
})
```

### 비동기 함수

```ts
import { describe, it, expect } from "vitest"
import { <fnName> } from "./<target>"

describe("<fnName>", () => {
  it("resolves with <expected>", async () => {
    await expect(<fnName>(<input>)).resolves.toEqual(<expected>)
  })

  it("rejects when <조건>", async () => {
    await expect(<fnName>(<badInput>)).rejects.toThrow(/<message>/)
  })
})
```

### 외부 모듈 mock 이 필요한 경우

`vi.mock` 은 파일 최상단에 hoist 된다. 함수 안에서 호출 금지.

```ts
import { describe, it, expect, vi, beforeEach } from "vitest"

vi.mock("./api", () => ({
  fetchUser: vi.fn(),
}))

import { fetchUser } from "./api"
import { greet } from "./greet"

beforeEach(() => {
  vi.mocked(fetchUser).mockReset()
})

it("greets a fetched user", async () => {
  vi.mocked(fetchUser).mockResolvedValue({ name: "Ellen" })
  await expect(greet(1)).resolves.toBe("Hello, Ellen")
})
```

## assertion 빠른 참조

| 의도 | matcher |
|---|---|
| 원시값 동등 | `toBe` |
| 객체·배열 깊은 동등 | `toEqual` |
| 부분 일치 (객체) | `toMatchObject` |
| 배열 요소 포함 | `toContain` |
| 함수 호출 검증 | `toHaveBeenCalledWith(...)`, `toHaveBeenCalledTimes(n)` |
| 정규식 매치 | `toMatch(/.../)` |
| 비동기 성공/실패 | `.resolves` / `.rejects` 체이닝 |

스냅샷은 기본 사용 안 함. 출력이 *길고 안정적* 일 때만 `toMatchInlineSnapshot()`.

## 자주 하는 실수

- `expect(fn(x)).toThrow(...)` — `fn(x)` 가 즉시 실행되어 throw 가 expect 밖에서 터진다. **반드시** `expect(() => fn(x)).toThrow(...)`.
- `vi.mock` 을 함수 안에서 호출 — hoist 안 됨. 파일 최상단 only.
- 비동기 테스트에 `await` 누락 — 통과처럼 보이지만 unhandled rejection 으로 샘. `async/await` 또는 `.resolves`/`.rejects` 둘 중 하나.
- `it.only` / `describe.only` 커밋 — 1개만 돌고 CI 통과해 보임. 작성 후 제거.

## 예시 — 입력에서 출력까지

**입력 시그니처**

```ts
// src/lib/clamp.ts
export function clamp(value: number, min: number, max: number): number
```

**도출한 케이스**

- happy: `clamp(5, 0, 10) === 5`
- edge: `clamp(-1, 0, 10) === 0`, `clamp(11, 0, 10) === 10`, `clamp(0, 0, 10) === 0` (lower 경계 동일)
- 에러: `min > max` 일 때 throw — 시그니처에 명시 없으면 *구현을 한 번 확인* 후 추가, 구현이 throw 하지 않으면 이 케이스 생략

**출력**

```ts
// src/lib/clamp.test.ts
import { describe, it, expect } from "vitest"
import { clamp } from "./clamp"

describe("clamp", () => {
  it("returns value when within range", () => {
    expect(clamp(5, 0, 10)).toBe(5)
  })

  it("clamps to min when below range", () => {
    expect(clamp(-1, 0, 10)).toBe(0)
  })

  it("clamps to max when above range", () => {
    expect(clamp(11, 0, 10)).toBe(10)
  })

  it("returns boundary value as-is", () => {
    expect(clamp(0, 0, 10)).toBe(0)
    expect(clamp(10, 0, 10)).toBe(10)
  })
})
```

## 검증

작성 직후 *해당 파일만* 한 번 실행한다.

```bash
npx vitest run <test-file-path>
```

전체 스위트는 사용자가 따로 요청했을 때만. 한 파일 추가에 전체 실행은 시간 낭비.

실패 시:

1. 출력을 그대로 읽고 1회 자동 수정.
2. 그래도 실패하면 멈추고 사용자에게 진단 보고. 추측 기반 반복 수정 금지 — `investigate` 영역.
