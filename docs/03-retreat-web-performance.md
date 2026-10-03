# 01. Retreat Web 성능, 안정성, 가독성 리팩토링

> 대상: 참가자용 신청 사이트(Next.js 14, React Hook Form, React Query, Zustand)
> 브랜치: `refactor/portfolio`

## 한 줄 요약

모든 개선을 **측정 → 원인 확인 → 수정 → 같은 방법으로 재측정** 순서로 진행했다.
측정 도구(렌더 카운터, 워터폴 측정, 이미지 폭 검증, 날짜 벤치마크)도 저장소에 함께 남겼다.

| 항목 | Before | After | 확인 방법 |
|---|---|---|---|
| 전화번호 13자 입력 시 렌더된 컴포넌트 수 | 882 | **14** | `e2e/perf/render-count.spec.ts` |
| "전참" 체크 시 렌더된 컴포넌트 수 | 477 | **124** | 위와 같음 |
| `getKSTDateString` 1회 호출 | 297µs | **7µs** (40배) | 입력 2,000개로 결과 동일 확인 후 측정 |
| 3배 밀도 휴대폰의 포스터 요청 폭 | 3840px | **1200px** | `e2e/poster-image-size.spec.ts` |
| 리더 대시보드 첫 화면(요청당 300ms 지연) | 2508ms | **1467ms** | `e2e/perf/leader-waterfall.spec.ts` |
| 로그아웃 후 다른 리더에게 이전 리더의 조원 명단 노출 | 발생 | **차단** | `e2e/leader-session.spec.ts` (기존 코드에서 실패하는 것 확인) |
| iOS 15 이하 Safari에서 `toSorted` 크래시 | 발생 가능 | **제거, 컴파일 단계에서 차단** | `tsconfig` lib를 es2022로 |

검증 결과:

- `tsc` 통과
- Vitest 92개 통과 (79개에서 13개 추가)
- `next build` 성공
- Playwright E2E 170여 개 (PERF 전용 측정 spec 2개는 기본 실행에서 제외)
  - 3 workers 병렬 실행에서는 매번 1~2개가 dev 서버 컴파일 지연으로 타임아웃 났다. 실패하는 테스트는 실행할 때마다 달랐다.
  - 실패한 테스트는 직렬로 다시 돌렸을 때 모두 통과했다.

## 1. 폼 리렌더 범위 축소: `useWatch` / `useFormState`

**측정 방법.** 소스를 건드리지 않고 테스트에서 React DevTools 전역 훅(`__REACT_DEVTOOLS_GLOBAL_HOOK__`)을 흉내 냈다.
커밋마다 실제로 렌더 함수가 실행된 컴포넌트를 셌다.

- 처음 측정값은 부풀려져 있었다.
  - 렌더를 건너뛴(bailout) 서브트리의 fiber는 이전 커밋의 객체가 그대로 남는다.
  - 이때 `PerformedWork` 플래그도 지워지지 않아서, 렌더되지 않은 컴포넌트까지 렌더된 것으로 셌다.
- 직전 커밋에서 본 fiber 객체는 제외하도록 카운터를 고친 뒤 다시 측정했다.

**원인.** 조사 단계에서는 "키 입력마다 폼 전체가 리렌더된다"고 봤는데, 측정해 보니 정확하지 않았다.

- `watch("x")`는 지켜보는 필드가 바뀔 때만 루트를 다시 그린다. 그래서 이름 입력은 렌더 0회였다.
- 실제 원인은 두 가지였다.
  1. 필드 컴포넌트들이 `useFormContext()`의 `formState.errors`를 읽는다. 이 Proxy 구독은 `useForm`을 호출한 루트에 걸린다. 그래서 에러가 생기거나 사라질 때마다 폼 전체(약 450개 컴포넌트)가 다시 그려졌다.
  2. 일정 체크나 셀렉트 변경은 `watch`로 지켜보는 값을 바꾸므로 역시 루트부터 다시 그려졌다.

**수정.**

```tsx
// Before: 루트(useForm 호스트)를 구독
const { register, formState: { errors } } = useRetreatForm();
const univGroup = watch("univGroup");

// After: 이 필드만 구독
const { control, register } = useRetreatForm();
const { errors } = useFormState({ control, name: "name" });
const univGroup = useWatch({ control, name: "univGroup" });
```

## 2. 리더 세션 캐시 누수 수정 (개인정보)

**문제.**

- 리더 쿼리 키가 `["leader-me", slug]`, `["leader-members", slug]`처럼 평평하게 흩어져 있었다.
- 로그아웃할 때는 `me` 쿼리만 지웠다.
- `staleTime`이 60초이므로, 같은 휴대폰에서 다음 리더가 로그인하면 **이전 리더의 조원 이름이 다시 요청되지도 않고 캐시에서 바로 보였다.**

**재현.** 두 번째 리더의 명단 응답을 1.5초 늦추는 E2E를 만들었다.
그 사이 이전 리더의 조원이 화면에 한 번이라도 나타나는지 확인한다. 기존 코드에서는 이 테스트가 **실패했다.**

**수정.**

- 세션에 속한 쿼리를 모두 `["leader", slug, ...]` 아래로 모았다.
- 로그아웃 시 `removeQueries(session)`으로 한 번에 지운다.
- 로그인 성공 시 `resetQueries(session)`을 호출한다. 세션이 만료되어 로그아웃 없이 넘어온 경우도 이것으로 정리된다.
- 리더 포털 오픈 여부(`open`)는 수양회 단위 데이터라서 세션 키와 분리했다.

## 3. 리더 대시보드 상태 구독 정리

**드래프트 우선 규칙 공통화.** "드래프트 값이 있으면 그것을, 없으면 서버 값을 쓴다"는 규칙이 카드, 진행률, 제출 payload, 검증의 4곳에 따로 구현되어 있었다.

- 이 규칙을 `effectiveAttendance` / `effectiveMemo` / `effectiveScheduleIds`로 모았다.
- 비고의 `""`는 서버 값을 비우겠다는 명시적인 입력이다. 이 의미가 4곳 모두에서 같게 유지되도록 했다.

**구독 범위 축소.**

| 대상 | Before | After |
|---|---|---|
| MemberCard | 드래프트 일부를 구독 | 자기 멤버의 원시 값만 반환하는 셀렉터 |
| 대시보드 | `draft.attendance` 객체 전체 구독 (한 명만 체크해도 모든 카드가 리렌더) | 체크한 인원 수(숫자)만 구독 |
| 제출 바 | 렌더 시 드래프트 구독 | 제출 시점에 `getState()`로 읽음 (은혜나눔을 입력할 때마다 리렌더되던 문제 해결) |

**테스트 먼저.** 리팩토링 전에 리더 영역 테스트가 하나도 없었다. 검증 로직과 드래프트 store에 특성화 테스트 13개를 먼저 작성해 기존 동작을 고정한 뒤 코드를 바꿨다.

## 4. 날짜 유틸: Intl formatter 재사용

- **문제.** `toLocaleDateString`은 호출할 때마다 내부에서 `Intl.DateTimeFormat`을 새로 만든다. `getKSTDateString` 한 번이 이 호출을 3번 했고, 일정 매트릭스의 셀마다 실행됐다.
- **수정.** 모듈 단위 formatter 하나를 만들어 두고 `formatToParts`로 연·월·일·요일·시·분을 한 번에 뽑는다.
- **검증.** 기존 KST 경계 테스트(자정 롤오버, 연말)가 그대로 통과했다. 시간 단위로 2,000개 입력을 넣어 기존 결과와 같은지 비교한 뒤 성능을 측정했다.

## 5. `toSorted` 제거와 재발 방지

- **문제.** `Array.prototype.toSorted`는 ES2023 API다. Safari에서는 iOS 16부터만 지원하고, Next 14는 이를 polyfill하지 않는다. iPhone 7 이하처럼 iOS 15까지만 올라가는 기기에서는 신청 폼이 크래시한다.
- **수정.** 복사 후 `sort` 하도록 바꿨다.
- **재발 방지.** `tsconfig`의 `lib`를 `esnext`에서 `es2022`로 좁혔다. 이제 같은 종류의 API를 쓰면 컴파일 단계에서 에러가 난다.

## 6. 포스터 이미지 `sizes`

- **원인.** `sizes`가 없으면 `next/image`는 1x/2x 밀도 기준으로 srcset을 고른다. 그래서 3배 밀도 휴대폰이 3840px 원본을 받았다.
- **수정.** `sizes="100vw"`를 지정했다.
- **검증.** E2E에서 `/_next/image?w=` 요청을 가로채, 브라우저가 고른 폭이 1200 이하인지 확인한다.
- **보류.** `remotePatterns: hostname "**"`(누구나 이미지 최적화 서버를 프록시로 쓸 수 있음)는 그대로 두었다. 포스터 호스트가 서버의 `CLOUDFRONT_DOMAIN` 환경변수와 관리자 직접 입력으로 정해지므로 운영 도메인을 확인해야 한다. → 운영 확인 후 제한 권장.

## 7. 리더 대시보드 요청 워터폴

- **Before.** `/open` → (레이아웃 렌더) → `/me` → (대시보드 렌더) → today/members/info. 왕복이 3단계였다.
- **After.**
  - 레이아웃이 `/open`과 `/me`를 동시에 보낸다.
  - 대시보드 페이지는 `/me`를 기다리는 동안 today/members/info를 `prefetchQuery`로 미리 받는다.
  - 로그인되지 않은 상태라면 prefetch는 401로 끝나고 재시도하지 않는다. 이 경우 로그인 페이지로 이동하는지 E2E로 확인했다.

## 8. 정리

- **타입 정리.** `types/index.ts`에는 admin/server 테이블 타입 30여 개가 복사되어 있었는데, 실제로 쓰는 것은 4개뿐이었다. 쓰는 타입만 남겼다.
  - JSON 날짜 필드를 `Date`에서 `string`으로 바꿨다. `Date`로 두면 `a.time - b.time` 같은 잘못된 코드가 타입 검사를 통과한다.
- **삭제.**
  - 사용하지 않는 의존성: `date-fns`, `clipboardy`, `@radix-ui/react-tabs`
  - 사용하지 않는 코드: `ui/tabs`, `IS_E2E`, `useLeaderReport`, `fetchLeaderReport`, `fetchLeaderScheduleChangeRequests`, `hasUnsavedChanges`, 쓰이지 않던 `<style jsx>`
- **전광판.**
  - 숫자 애니메이션 컴포넌트를 `components/registration-status/`로 분리했다.
  - 자체 polling은 그대로 두었다. 응답이 끝난 뒤 1초 후 다음 요청을 보내므로 요청이 겹치지 않고, 프로젝터 화면이라 탭 숨김 시 일시정지할 필요도 없다. React Query로 옮겨도 얻는 것이 적다고 판단했다.

## 9. 폼 필드 중복 제거와 모달 통일

**중복 필드 통합.** 수양회 폼과 셔틀버스 폼에 부서·학년·이름·성별·전화번호 필드 컴포넌트가 5개씩, 총 10개가 복사되어 있었다. 서로 다른 점은 폼 훅 이름뿐이었다.

- 공용 `components/registration/basic-info-fields.tsx` 하나로 합쳤다(파일 12개 삭제).
- 실제로 다른 점 3가지는 props로 받는다:
  - 이름 예시(placeholder)
  - 전화번호 안내 문구(수양회만 있음)
  - 수양회 폼에만 있는 리더 필드(`children`)
- 필드 id는 `BASIC_INFO_FIELDS` 한 곳에 정의했다. `satisfies`로 두 스키마 모두의 키인지 컴파일 타임에 검사한다.

**모달 통일.** 직접 그린 `fixed inset-0` 모달 5개를 `components/ui/modal`(Radix Dialog)로 바꿨다.

- 새로 생긴 동작:
  - 포커스 트랩(Tab을 눌러도 배경 폼으로 나가지 않음)
  - Esc로 닫기 (제출 중에는 막음)
  - `role="dialog"`와 제목 연결
  - 배경 스크롤 잠금
- 그대로 유지한 동작: 바깥 클릭으로는 닫히지 않는다. 휴대폰에서 실수로 닫히는 것을 막기 위해서다.

**스피너.** 똑같은 SVG 7개를 `components/ui/spinner` 하나로 대체했다.

**검증.**

- 추가한 E2E:
  - `e2e/modal-a11y.spec.ts`: dialog 역할, 포커스 트랩, Esc 닫기, 바깥 클릭 무시
  - `e2e/leader-session.spec.ts`: 리더 조원 모달에서 출석을 저장하면 진행률에 반영되는지 (리더 영역에 처음 생긴 UI 테스트)
- 전체 E2E 172개를 2 workers로 실행해 **모두 통과**했다.
- 모바일(390px)과 데스크톱(1280px) 스크린샷으로 레이아웃과 한국어 줄바꿈을 확인했다.

**비용(트레이드오프).** `@radix-ui/react-dialog`를 추가하면서 해당 페이지들의 First Load JS가 늘었다.

| 페이지 | Before | After |
|---|---|---|
| 신청 페이지 | 182kB | 195kB (+13kB) |
| 리더 대시보드 | 125kB | 138kB (+13kB) |

접근성 문제(키보드·스크린리더 사용자가 모달을 쓸 수 없음)를 해결하는 대가로 받아들였다. 용량이 문제가 되면 모달을 `next/dynamic`으로 지연 로딩하면 된다.

## 남은 후보 (이번에 하지 않은 것)

- 페이지를 Server Component + `HydrationBoundary`로 바꿔 포스터(LCP)를 첫 HTML에 포함하는 작업.
- 앞에서 보류한 `remotePatterns` 제한 (운영 이미지 도메인 확인 필요).
- 모달 지연 로딩 (위 13kB).

## 면접에서 설명할 때

- **"추측하지 않고 측정했다."** 조사 단계에서 세운 가설("키 입력마다 폼 전체가 리렌더된다")은 측정해 보니 틀렸다. 실제 원인은 `formState` Proxy 구독과 `watch`의 범위였다. 측정 도구 자체의 버그(bailout fiber 중복 집계)도 발견해서 고쳤다.
- **"버그는 재현 테스트부터 만들었다."** 리더 캐시 누수는 기존 코드에서 실패하는 E2E를 먼저 만들고, 수정 후 그 테스트가 통과하는 것을 확인했다.
- **"재발을 구조로 막았다."** `toSorted`는 고치는 데서 끝내지 않고 `tsconfig`로 같은 종류의 API 사용 자체를 막았다. 리더 키는 계층 구조로 바꿔 "세션 데이터 지우기"가 한 줄로 끝나게 했다.
