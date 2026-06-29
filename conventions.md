# 개발 컨벤션 (Next.js + FSD)

> 이 문서는 코드 작성·파일 배치의 **단일 진실 공급원(SSOT)**이다.
> 구조/네이밍 판단이 애매할 때는 항상 이 문서를 우선한다.
> 스택: **Next.js (App Router) · TypeScript · Zustand · TanStack Query · Tailwind**

---

## 0. 핵심 원칙 (요약)

1. **레이어는 한 방향으로만 의존한다.** (`application → screens → widgets → features → entities → shared`)
2. **다른 슬라이스는 그 Public API(`index.ts`)로만 import.** 내부 파일 직접 import 금지 (같은 슬라이스 내부는 상대경로 자유).
3. **파일명은 항상 `kebab-case`.** export 식별자만 PascalCase/camelCase.
4. **기본은 서버 컴포넌트(RSC).** `'use client'`는 상태·이벤트·브라우저 API가 필요한 잎(leaf)에만.
5. **서버 데이터 읽기는 RSC에서 controller 직접 호출이 기본.** 클라이언트 상호작용이 필요할 때만 TanStack Query + BFF.
6. **클라이언트 UI 상태는 Zustand, 서버 데이터·캐시는 TanStack Query.** 둘을 섞지 않는다.
7. **`app/`은 라우팅 껍데기, 전역 설정은 `application`.** 화면 로직은 `screens/`에 둔다.
8. **서버 전용 코드는 `server/`에 3계층(dao → controller)으로.** 컴포넌트는 controller만 부른다.

---

## 1. 디렉토리 구조

Next.js App Router와 FSD를 결합한다. FSD 최상위는 `application`(전역 Provider·앱 전역 설정)이고, Next의 `app/`은 그 위에서 **라우팅만** 담당하는 호스트다. 실제 화면은 FSD의 `screens/`가 담당한다.

```
src/
├── application/          # FSD 최상위 레이어 — 전역 Provider·앱 전역 설정·root 조립 (§2.2)
│   └── providers.tsx     # QueryClientProvider · store · theme ('use client')
│
├── app/                  # Next.js App Router — 라우팅 호스트 (application·screens 조립)
│   ├── layout.tsx        # application providers로 children 감쌈
│   ├── page.tsx          # screens 렌더
│   ├── (group)/...       # 라우트 그룹
│   └── api/              # BFF route handler (§8)
│
├── screens/              # 페이지 단위 조립 (FSD 'pages' 레이어 — Next 'app'과 충돌 피해 rename)
│   └── {screen}/
│       ├── ui/
│       ├── model/
│       └── index.ts
│
├── widgets/              # 페이지를 구성하는 큰 독립 블록 (header, sidebar, feed ...)
├── features/             # 사용자 액션 단위 (form 제출, toggle, 검색 ...)
├── entities/             # 비즈니스 엔터티의 데이터/표현 (§7 TanStack Query 패턴)
├── shared/               # 도메인 무관 재사용 (ui, hooks, lib, api, config)
│
└── server/               # FSD와 별개. 서버 전용 코드 (§9)
    ├── db/
    ├── dao/
    ├── controllers/
    ├── lib/
    └── types/
```

> **`app/` ↔ `screens/` 관계**: `app/**/page.tsx`는 해당 `screens/{screen}`을 import해서 렌더만 한다. 데이터 패칭·상태·마크업은 `screens/`로 내린다. `app/`에는 `metadata`, `generateStaticParams`, route segment config 같은 **Next.js 라우팅 관심사만** 남긴다.

---

## 2. 서버/클라이언트 컴포넌트 경계 & 데이터 패칭

App Router의 기본 단위는 **서버 컴포넌트(RSC)**다. 이 경계가 모든 데이터 흐름의 출발점이다.

### 2.1 `'use client'` 경계

- 기본은 **서버 컴포넌트**. `'use client'`는 아래가 필요한 **잎(leaf)** 모듈에만 선언한다.
  - `useState`/`useEffect` 등 React hook, 이벤트 핸들러
  - Zustand, TanStack Query
  - 브라우저 API (`window`, `localStorage`, `IntersectionObserver` 등)
- **경계는 최대한 아래로 내린다.** 상호작용 잎만 클라이언트로 만들고, 트리 상단(`layout`/`page`/screen 컨테이너)은 서버로 유지한다.
- 서버 컴포넌트는 클라이언트 컴포넌트를 import해 렌더할 수 있다. 반대로 클라이언트가 서버 컴포넌트를 품어야 하면 `children`(슬롯)으로 주입한다.

**FSD 세그먼트 ↔ 경계 대응**

| 세그먼트 | 기본 |
| --- | --- |
| `model/` (hook·store·TanStack Query) | 클라이언트 (`'use client'`) |
| `ui/` — 이벤트·에페메랄 뷰 상태(open/hover 등) 있음 | 클라이언트 |
| `ui/` — 순수 표시만 | 서버 가능 |
| `lib/` (React 비의존) | 서버/클라 무관 |

> screen·widget·feature를 통째로 서버/클라로 정하지 않는다. **서버 컨테이너 + 클라이언트 잎**으로 쪼개서, 데이터는 서버 컨테이너가 fetch해 props로 내리고 상호작용은 클라이언트 잎이 맡는다.

> `ui/`는 server든 client든 **순수 표현**이다 — 핸들러는 props로 받아 DOM에 꽂고, 보유 가능한 상태는 open/hover/focus 같은 **에페메랄 뷰 상태**까지. 도메인 로직·데이터 패칭·store 접근은 `model/`에 둔다. (위 server/client 구분은 '상호작용 유무'일 뿐, 순수함과는 별개 축)

### 2.2 전역 Provider — `application` 레이어

`QueryClientProvider`, Zustand store provider, theme provider 등 전역 설정은 **`application` 레이어**(예: `application/providers.tsx`, `'use client'`)에 모은다. Next의 `app/layout.tsx`는 이 Provider로 `children`을 감싸기만 한다.

> FSD 최상위 레이어 이름은 `application`이다 — Next의 라우팅 디렉토리 `app/`과 구분한다. **전역 Provider·앱 전역 설정·root 조립은 `application`에, 라우팅 관심사만 `app/`에** 둔다.

### 2.3 데이터 읽기 (read) — RSC 우선

- **기본**: 서버 컴포넌트(`page.tsx`/screen 컨테이너)에서 `server/controllers`의 `fetch*`를 **직접 `await`** 한다. (SSR·서버 캐시 이점, 클라 번들 절감)
- **예외 — 아래처럼 클라이언트 상호작용이 필요할 때만** TanStack Query + `/api/` BFF (§7, §8):
  - 무한 스크롤·페이지네이션·클라이언트 필터로 재패칭
  - 실시간/폴링, optimistic update, 포커스 리패치
- **하이브리드**: 서버에서 prefetch 후 `HydrationBoundary`로 클라이언트 TanStack Query에 넘기면 둘의 이점을 같이 가져간다.

### 2.4 데이터 변경 (write) — Server Action 또는 BFF

- mutation은 `server/controllers`의 `create*/modify*/remove*`를 사용한다.
- 클라이언트에서 직접 호출이 필요하면 controller에 `'use server'`(Server Action)를 붙인다.
  - **`'use server'`는 변경(mutation)에만 쓴다. 읽기에는 쓰지 않는다** (Server Action은 POST·순차 실행이라 데이터 읽기용이 아니다).
- 폼 위주면 Server Action, 클라이언트 상태와 얽히면 TanStack Query `useMutation` + BFF route를 쓴다.

---

## 3. 파일 · 네이밍 컨벤션

### 3.1 파일/폴더명 — 항상 `kebab-case`

```
user-profile-card.tsx      ✅
use-debounced-value.ts      ✅
UserProfileCard.tsx         ❌
useDebouncedValue.ts        ❌
```

- 컴포넌트 파일도 kebab-case로 두고 **export 식별자만 PascalCase**:
  `user-card.tsx` → `export function UserCard()`
- hook 파일은 `use-` 접두사: `use-auth.ts` → `export function useAuth()`

### 3.2 Next.js App Router 예약 파일 (이건 Next 규칙 그대로)

| 파일 | 역할 |
| --- | --- |
| `page.tsx` | 라우트의 UI 진입점 — `screens/`를 렌더 |
| `layout.tsx` | 공유 레이아웃 (상태 유지) |
| `template.tsx` | 매 네비게이션마다 리마운트되는 레이아웃 |
| `loading.tsx` | Suspense fallback |
| `error.tsx` | 에러 바운더리 (`'use client'` 필수) |
| `not-found.tsx` | 404 UI |
| `route.ts` | API route handler (§8) |

- 라우트 그룹 `(group)`, 동적 세그먼트 `[id]`, catch-all `[...slug]`는 Next 표기 그대로.
- 예약 파일 외 컴포넌트를 라우트 폴더에 두지 않는다 — 화면 코드는 `screens/`로.

### 3.3 슬라이스 내부 표준 파일명

| 파일 | 내용 |
| --- | --- |
| `index.ts` | Public API (외부 노출 정의) |
| `api.ts` | fetch 함수 + 요청/응답 타입 |
| `factory.ts` | `queryOptions` / `mutationOptions` |
| `use-*.ts` | hook |
| `*.store.ts` | Zustand store |
| `types.ts` | 슬라이스 공용 타입 |
| `*.const.ts` / `config.ts` | 상수·설정값 |

### 3.4 식별자 네이밍

| 대상 | 규칙 | 예 |
| --- | --- | --- |
| 컴포넌트 | PascalCase | `UserCard` |
| hook | `use` + camelCase | `useUserData` |
| 변수/함수 | camelCase | `fetchUser` |
| 타입/인터페이스 | PascalCase | `UserDto` |
| 상수 | UPPER_SNAKE | `MAX_RETRY` |
| 불리언 | `is/has/should` 접두사 | `isLoading` |

### 3.5 import 경로

- **슬라이스/레이어를 넘는 import는 절대경로 alias `@/*`** (`tsconfig.json`의 `paths`)로, 항상 상대 슬라이스의 Public API를 통해서.
- **같은 슬라이스 내부 import는 상대경로 허용** — 세그먼트를 오르내리는 `../../`도 슬라이스 경계 안이면 OK.
- 단, 슬라이스 **밖**을 `../../`로 끌어오는 건 금지. 경계를 넘을 땐 반드시 `@/*` + Public API.

### 3.6 스타일 (Tailwind)

- 클래스 병합은 `cn()`(clsx + tailwind-merge) 유틸로. 조건부 클래스를 문자열 직접 연결하지 않는다.
- 정렬은 `prettier-plugin-tailwindcss`에 맡긴다(수동 정렬 금지).

---

## 4. FSD 레이어 & 의존성 규칙

```
application → screens → widgets → features → entities → shared
```

- **상위 레이어만 하위 레이어를 import.** 역방향 금지.
- Next의 `app/`(라우팅 호스트)는 `application`과 `screens`를 조립한다 — 사실상 최상위에서 동작하며, 라우팅 관심사만 담는다.
- **같은 레이어 내 슬라이스 간 import 금지.**
  예) `features/auth`가 `features/profile`을 직접 import ❌ → 공통 부분을 `entities`/`shared`로 내린다.
- **import는 항상 슬라이스 루트 `index.ts`(Public API)로만.** `features/auth/model/store.ts` 직접 import ❌.
- `entities`/`shared`는 상위 레이어를 모른다. 필요한 값은 **파라미터로 주입**받는다.
- **entities 간 교차 참조는 허용한다.** `comment` → `user` 처럼 엔터티가 다른 엔터티를 참조할 수 있다. 단 **Public API(`index.ts`)로만, 단방향**으로. 순환 참조는 금지하고, 양쪽이 서로 알아야 하면 더 낮은 레이어로 끌어내린다. (entities를 제외한 다른 레이어는 동일 레이어 import 금지를 유지)

### 새 기능 추가 절차

1. 어느 레이어인지 먼저 판단한다.
   - 비즈니스 엔터티의 데이터/표현 → `entities`
   - 사용자 액션(폼 제출, 토글) → `features`
   - 페이지의 큰 독립 블록 → `widgets`
   - 페이지 조립 → `screens`
   - 도메인 무관 재사용 → `shared`
2. 슬라이스 폴더를 만들고 필요한 세그먼트만 채운다.
3. `index.ts`에 외부 노출 API를 정의한다.
4. 사용처는 슬라이스 루트로만 import한다.

---

## 5. 슬라이스 내부 세그먼트

| 세그먼트 | 무엇을 두는가 | 기준 |
| --- | --- | --- |
| `ui/` | 표현 + 에페메랄 뷰 상태(open/hover/focus) | 도메인 로직·데이터 패칭·store 접근 없음 |
| `model/` | hook, store, 도메인 로직 | React에 의존하는 모든 것 |
| `lib/` | 순수 유틸, 브라우저 API 헬퍼, 상수 | React 없이 동작하는 것 |
| `config/` | 슬라이스 전용 상수·설정값 | 코드가 아닌 값 |

### shared 전용 추가 세그먼트

`shared`는 도메인이 없으므로 아래를 추가로 쓴다.

| 세그먼트 | 무엇을 두는가 | 기준 |
| --- | --- | --- |
| `ui/` | 디자인시스템 / 범용 프리미티브 | 도메인 지식 없는 UI |
| `hooks/` | 도메인 무관 범용 hook | React 의존 + 도메인 지식 없음 |
| `api/` | BFF(`/api/*`) 호출 공통 헬퍼 | 클라이언트→서버 통신 유틸 |
| `lib/` | 순수 유틸·상수 | React 없이 동작 |

- `shared/hooks/` 예: `useStorageState`, `useDebounce`, `useMediaQuery`
- React hook은 절대 `shared/lib/`에 두지 않는다 → 반드시 `shared/hooks/`.

### feature 루트 컴포넌트 역할

feature 루트는 state·action을 보유하고 `model/` hook을 호출하는 **조립 지점**이다.

- 로직은 `model/`이 갖고 루트는 `훅 호출 → ui에 props로 내리기`라 얇게 유지된다. **그래서 `ui/`를 순수하게 둬도 루트가 커지지 않는다.** (핸들러는 props로 받아 꽂을 뿐, 소유하지 않으면 ui는 여전히 순수)
- props가 너무 많아지면(prop drilling) → ui를 더 잘게 쪼개거나, 슬라이스 단위 view-model 객체로 묶어 내린다.
- **`ui/`로 분리하는 기준**: 상태별 마크업이 충분히 달라 루트에 다 두면 흐름 파악이 어려울 때.
- **루트에 마크업을 둬도 되는 경우**: 포지셔닝 컨테이너, 로딩 오버레이, 단순 UI 변형.
  분리 기준이 안 되는 JSX를 억지로 빼서 의미 없는 Wrapper 파일을 만들지 않는다.

---

## 6. 전역 상태 관리 (Zustand)

전역 상태는 React Context 대신 **Zustand**를 쓴다.

- **서버 초기 데이터가 필요한 경우**: Context에 store **레퍼런스**를 담는 패턴.
- **클라이언트 전용 singleton**: `create()`로 모듈 레벨 생성.
- 소비자는 **셀렉터**로 필요한 슬라이스만 구독: `useStore(s => s.value)`.
- Context에 값을 직접 담지 않는다 (레퍼런스만).

### store의 역할 범위

store는 **UI가 즉시 필요한 클라이언트 상태**만. 서버 데이터·캐시는 TanStack Query에.

- 넣는 것: 세션 한정 UI 상태 (모달 open, 선택된 탭 등)
- 넣지 않는 것: 서버에서 온 데이터, 다른 state에서 파생 가능한 값

### 액션은 메서드로, state 직접 수정 금지

- 컴포넌트·hook에서 state 필드를 직접 조작하지 않는다.
- store는 **의도가 드러나는 액션(메서드)**만 외부에 노출한다. (한 액션 = 한 사용자 의도)

### 읽기도 메서드로 캡슐화

내부 자료구조를 셀렉터에서 직접 꺼내지 않는다. store에 읽기 메서드를 두고 셀렉터에서 호출한다.

```ts
// ✅ useStore((s) => s.getValue(id))   — 셀렉터 안에서 호출, 반응성 유지
// ❌ useStore((s) => s.getValue)(id)   — 함수 레퍼런스만 구독, 반응성 깨짐
```

---

## 7. entities — TanStack Query 패턴

서버 데이터를 **클라이언트에서** 다루는 entity는 아래 구조를 따른다. (서버에서 읽을 땐 §2.3대로 controller 직접 호출)

```
entities/{slice}/
├── api.ts            # fetch 함수 — 요청/응답 타입 정의 포함 (shared/api 헬퍼 사용)
├── factory.ts        # queryOptions, mutationOptions (queryClient 파라미터 패턴)
├── model/
│   ├── use-*-data.ts # useQuery + useMutation (Raw Data 위주)
│   └── use-*.ts      # 소비자 hook — 비즈니스 로직
└── index.ts
```

- `factory.ts`의 `mutationOptions`는 `(userId, queryClient)` 형태로 **캐시 전략을 함께** 정의한다.
- `use-*-data.ts`는 데이터 레이어만, `use-*.ts`는 비즈니스 로직.
- entity hook은 상위 레이어(`application`, `screens`, `widgets`, `features`)를 import하지 않는다. 필요한 값은 파라미터로 받는다.

---

## 8. BFF API route (`app/api/`)

클라이언트 → 서버 데이터 통신은 `/api/` 경로를 경유한다. (서버 컴포넌트는 BFF를 거치지 않고 controller를 직접 부른다)

- **인증**: `proxy.ts`(또는 `middleware.ts`)에서 Bearer 토큰 검증 후 `x-user-id` 헤더로 주입.
  route handler는 `request.headers.get('x-user-id')`만 읽는다.
- **새 보호 라우트 추가**: `PROTECTED_API_PREFIXES` 배열에만 추가한다.
- route handler는 **controller만** 호출한다. dao를 직접 import하지 않는다.

---

## 9. 서버 레이어 (`server/`)

FSD와 별도로 서버 전용 코드는 `server/`에 3계층으로 구성한다.

```
server/
├── db/          # DB 클라이언트 팩토리
├── dao/         # 테이블 단위 raw DB 접근 — 'use server' 없음
├── controllers/ # dao 호출 + 캐시 무효화 — 외부 노출 유일 진입점
├── lib/         # 서버 공통 유틸
└── types/       # 서버 공통 타입
```

- **dao**: 순수 DB 접근만. 프레임워크 비의존. 파일은 테이블(도메인) 단위.
- **controllers**: dao 호출 + 캐시 무효화(`revalidatePath`/`revalidateTag`) 처리.
  - 서버 컴포넌트에서 직접 `await` 호출(읽기·쓰기 모두) → `'use server'` **불필요**
  - 클라이언트 컴포넌트에서 직접 호출하는 **mutation** → `'use server'`(Server Action) 선언
  - 읽기를 클라이언트에 노출할 땐 `'use server'`가 아니라 **BFF route(§8)**를 쓴다
- `app/`, `screens/` 등에서는 **controller만 import**. dao 직접 import 금지.
- **`server/`는 서버 실행 코드에서만 import**한다 (RSC · `app/api` route handler · `'use server'` Server Action). 클라이언트 컴포넌트는 일반 controller를 직접 import하지 않는다 — 읽기는 BFF route(§8), 변경은 Server Action(§2.4)으로.
- 타입은 dao에 정의하고, controller가 `export type { ... } from`으로 노출한다.

### 네이밍: DAO ↔ Controller

| 구분 | DAO | Controller |
| --- | --- | --- |
| 읽기 | `find*` | `fetch*` |
| 생성 | `insert*` | `create*` |
| 수정 | `update*` | `modify*` |
| 삭제 | `delete*` | `remove*` |
| 집계 | `count*` | — |

---

## 10. import / 데이터 흐름 빠른 참조

| 상황 | 규칙 |
| --- | --- |
| 레이어 간 | 상위 → 하위만. 역방향·동일레이어 슬라이스 간 ❌ (단 entities 간 교차는 Public API·단방향 허용) |
| 슬라이스 진입 | 항상 `index.ts`(Public API)로만 |
| 경로 | 슬라이스 밖은 `@/*` 절대경로(Public API). 슬라이스 내부는 상대경로(`../../` 허용) |
| 화면 코드 | `app/`이 아니라 `screens/`에 |
| 컴포넌트 종류 | 기본 서버 컴포넌트. 상호작용 잎만 `'use client'` |
| 데이터 읽기 | RSC에서 controller 직접 `await`. 클라 상호작용만 TanStack Query + BFF |
| 데이터 변경 | controller mutation. 클라 직접 호출은 Server Action(`'use server'`) |
| 서버 접근 | 컴포넌트 → controller만. dao 직접 ❌ |
| 상태 종류 | 서버 데이터·캐시 = TanStack Query · 클라 UI 상태 = Zustand |
