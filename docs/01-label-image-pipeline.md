# 01. GBS 꼬리표 이미지 파이프라인 리팩토링

> 대상: `src/utils/puppeteerInstance.ts`, `convertHtmlToImage.ts`, `render.ts`,
> `src/api/retreat/line-up/line-up.service.ts`(꼬리표 ZIP 2종), `line-up.model.ts`, 신규 `line-up.label.ts`
> 커밋: `refactor: rebuild GBS label image pipeline with page pool and single query`

## 한 줄 요약

리더마다 DB를 조회하고 이미지마다 브라우저 탭을 새로 열던 구조를 바꿨다.
**DB 조회 1번 → 메모리에서 분배 → 탭을 재사용하는 워커 큐로 렌더링**하는 구조로 바꿔서
렌더링 시간을 **29.9초에서 9.9초로 67% 줄였다**(150장 기준).

## 측정 결과

`tools/bench-label-image.ts` 결과. 합성 데이터 150장(리더 1명 + 조원 9명 = 1장, 약 1,500명 규모)으로 측정했다.
브라우저 2개 × 페이지 4개, 3회 실행한 값의 중앙값이다.

| 방식 | 총 시간 | 장당 평균 |
|---|---|---|
| Before: 작업마다 `newPage` → 캡처 → `close`, 브라우저는 랜덤으로 선택 | 29.89s | 199ms |
| After: 워커마다 페이지 하나를 재사용하는 작업 큐 | **9.85s** | **66ms** |

DB N+1 제거 효과는 이 수치에 들어 있지 않다. 운영 환경에서는 리더 수(약 150)만큼 하던 쿼리가 3개의 병렬 쿼리로 줄어든 만큼 추가로 단축된다.

```bash
PUPPETEER_EXECUTABLE_PATH="C:/Program Files/Google/Chrome/Application/chrome.exe" npx ts-node tools/bench-label-image.ts 150 3
```

## 구조 변화

```
Before (부서 꼬리표)
  리더 목록 조회
  └─ 리더마다 (p-limit, 브라우저 랜덤 선택)
       ├─ DB: 그 리더의 조원 조회            ← N+1 (리더 수만큼)
       ├─ browser.newPage()                  ← close() 없음 (탭 누수)
       └─ 캡처 → zip
  누락 인원 보정
  ├─ DB: 리더 목록 다시 조회
  ├─ DB: 전체 인원 조회
  ├─ 메모리: 누락 계산 (같은 루프가 두 번 돌아 중복 push)
  └─ DB: 누락 인원 상세 조회 → 개인별 캡처

After
  Promise.all([인원 전체, 리더 목록, 일정])   ← DB 3회, 병렬
  planLeaderLabels(인원, 리더)                ← 순수 함수: "누구를 / 어느 파일에"
  renderHtmlBatch(jobs)                       ← 페이지 풀: "어떻게 그릴지"
  zip.generateAsync()
```

**책임 분리:**

- **model**: 데이터 조회
- **line-up.label.ts**: 분배 규칙 (테스트 가능)
- **puppeteerInstance.ts**: 렌더링 인프라
- **service**: 위 셋을 이어 붙이기

## 변경 내역과 이유

### 1. 페이지 풀: p-limit + 랜덤 분배를 워커 큐로 교체 (성능)

```ts
// Before: 작업 하나마다 탭을 열고 닫는다. 브라우저는 랜덤으로 고른다.
const poolIndex = Math.floor(Math.random() * browserPool.length);
return limit(async () => {
  const page = await browser.newPage();   // 매번 수십 ms
  ...
  await page.close();
});

// After: 워커 = 페이지 1개. 공용 cursor에서 다음 작업을 가져가며 같은 페이지를 계속 쓴다.
while (cursor < jobs.length) {
  const { item, html } = jobs[cursor++];
  onImage(item, await convertHtmlToImage(html, page));
}
```

- **탭 생성 비용:** 작업 수(150)만큼 들던 비용이 워커 수(8)만큼으로 줄어든다. `setViewport`도 워커당 한 번만 한다.
- **분배 편중:** 랜덤 분배는 한 브라우저에 작업이 몰릴 수 있었다. 작업 큐에서는 먼저 끝난 워커가 다음 작업을 가져가므로 편중이 없다.
- **`cursor++`의 안전성:** JS 이벤트 루프는 단일 스레드라서 `cursor++` 사이에 경쟁 조건이 생기지 않는다. 별도 lock이 필요 없다.
- **워커 수 상한:** 작업이 워커보다 적으면 브라우저를 필요한 만큼만 띄운다.

### 2. 리소스 누수 수정 (안정성)

- 부서 꼬리표 경로에서 `page = await browser.newPage()` 뒤에 `close()`가 없었다. 리더 수만큼 탭이 쌓인 채로 끝까지 갔다.
- 지금은 워커의 `finally`에서 반드시 닫는다. 브라우저 종료도 `Promise.allSettled`로 처리해서 하나가 실패해도 나머지는 닫힌다.

### 3. N+1 쿼리 제거 (성능)

- 쿼리 3개(`getGbsMembersByLeader`, `getAllGbsMembers`, `getMismatchGbsMembers`)를 하나로 합쳤다. 새 쿼리 `getLabelMembers`는 전체 인원을 한 번에 조회한다.
- 일정 id 집계는 서브쿼리로 바꿨다. 결과가 `ARRAY[]::integer[]`로 고정되므로 `[null]`이 섞이지 않는다.

### 4. 분배 로직을 순수 함수로 분리 (가독성·테스트)

- 기존 코드베이스의 `line-up.bulk-gbs.ts` 패턴을 그대로 따랐다(DB 없는 결정 로직 + jest 단위 테스트).
- `planLeaderLabels`의 반환값은 `{ path, leaderName, rows }[]` 하나뿐이다. 서비스는 이것을 렌더링만 한다.
- 테스트 `test/api/retreat/line-up/label.test.ts`가 다루는 것:
  - 리더를 맨 위에 두고 조원 순서를 유지하는지
  - 이름이 같은 리더가 다른 부서에 있을 때
  - 미매칭 인원의 경로
  - 파일명 치환
  - HTML 이스케이프

### 5. 버그 수정 (동작 변화가 있는 부분)

| 항목 | Before | After |
|---|---|---|
| 다른 부서에 같은 이름의 리더 | `Map`을 이름으로만 만들어서 한쪽 리더 그룹이 통째로 "미매칭"이 됨 | (부서, 이름) 쌍으로 구분 |
| 누락 인원 집계 | 같은 그룹핑 루프가 두 번 돌아 중복 push(결과는 미사용) | 제거 |
| 수양회 꼬리표 파일 확장자 | 내용은 JPEG인데 `.png`로 저장 | `.jpeg` |
| GBS 미배정 인원 | `gbsNumber ?? 0` 처리로 파일명은 `GBS_0_…`, 제목은 "GBS 0" | `GBS_미배정_…`, 제목 "GBS 미배정" |
| 활성 학년이 없는 인원 | 리더 그룹에도, 누락 집계(inner join)에도 빠져서 이미지에서 사라짐 | `0/미매칭/`에 개인 이미지로 생성 |
| 사용자 입력 HTML | 이름을 그대로 템플릿에 삽입 (`<`가 들어가면 표가 깨지고, 스크립트가 들어가면 headless 브라우저에서 실행) | `esc()`로 이스케이프 |
| env 미설정 | `Number(undefined)`가 `NaN`이 되어 브라우저 0개 → 크래시 | 기본값 2 × 4 |

### 6. 정리

- **주석 처리된 코드 삭제:** 쓰이지 않던 `generateSingleGbsLabelImage`(node-html-to-image 시절) 약 40줄, 디버그 `console.log` 블록 약 60줄을 지웠다.
- **템플릿 중복 제거:** 두 템플릿에 똑같이 있던 `getColor`를 `DAY_COLORS` 하나로 합쳤다.
- **필드 정리:** `partialAttend`는 `!fullAttend`라서 필드에서 뺐다. 템플릿 row 필드명은 snake_case에서 camelCase로 통일했다.
- **도달 불가능한 분기 삭제:** 쿼리가 이미 STAFF를 제외하므로 `isStaff` 분기는 실행될 수 없었다. 같은 블록 안에 서로 다른 `folderPath`가 두 번 계산되던 코드도 이때 함께 지웠다.
- **엑셀 꼬리표 2종:** 서로 의존하지 않는 조회 3개를 `Promise.all`로 병렬화했다. 어디에도 쓰이지 않던 `leaderUserIds` / `aIsLeader` 계산은 삭제했다.
- **보존한 것:** env 이름 `LABLE_CHROMIUM_*`의 오타는 배포 설정과 맞물려 있어서 그대로 두었다(AGENTS.md 규칙).

## 면접에서 설명할 때

1. **문제:** 1,500명분 꼬리표 ZIP을 만드는 데 수십 초가 걸렸고, 리더 수만큼 DB 쿼리와 브라우저 탭이 생겼다.
2. **원인 분리:** 렌더링(탭 생성 비용 + 분배 편중)과 데이터 조회(N+1) 두 가지로 나눠서 봤다.
3. **해결:**
   - 조회는 한 번으로 줄였다.
   - 분배 규칙은 순수 함수로 빼서 테스트했다.
   - 렌더링은 페이지를 재사용하는 작업 큐로 바꿨다.
4. **검증:** 벤치마크 스크립트로 같은 조건에서 비교했다(67% 단축). 단위 테스트 15개가 통과한다.
5. **부수 효과:** 리팩토링 과정에서 탭 누수, 동명 리더 누락, 확장자 오류, HTML 미이스케이프를 발견해 함께 고쳤다.
