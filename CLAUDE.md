# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 배포

```bash
cd ~/Desktop/claude-code-folder/morning-club
firebase deploy          # hosting + firestore rules 동시 배포
firebase serve           # 로컬 미리보기 (선택)
```

`firebase.json`에 `firestore.rules`가 포함되어 있으므로 `firebase deploy` 한 번으로 rules와 hosting이 함께 배포된다.

## 프로젝트 개요

매일 아침 새벽 6시 모임을 위한 출석 체크 웹 앱.
**단일 파일 구조**: 모든 HTML·CSS·JS가 `index.html` 하나에 있음. 파일 분리 금지.

- **호스팅**: https://monring-club.web.app (Firebase Hosting)
- **인증**: Firebase Auth (Google 로그인)
- **DB**: Firebase Firestore
- **차트**: Chart.js 4.4.0 (CDN)

## Firestore 데이터 구조

```
members/{uid}
  - name, emoji, email, photo, uid, createdAt

attendance/{uid}/records/{date}        ← 내 출석 기록 (YYYY-MM-DD 형식, 로컬 시간 기준)
  - date, day, log, isPublic, uid, userName, userEmoji, timestamp

feed/{date}/entries/{uid}             ← 오늘 피드 + 반응 저장
  - date, day, log, isPublic, uid, userName, userEmoji, timestamp  (attendance와 동일)
  - r0: [uid, ...]   ← 👍 반응한 멤버 uid 배열
  - r1: [uid, ...]   ← ❤️
  - r2: [uid, ...]   ← 💪
  - r3: [uid, ...]   ← 🎉
```

> `feed` 컬렉션은 `collectionGroup` 쿼리의 인덱스 의존성을 피하기 위해 별도로 운용.
> `attendance`를 수정(수정/삭제/공개토글)할 때 오늘 날짜면 `feed`도 반드시 함께 수정해야 함.
> 반응(r0~r3)은 `feed` 엔트리에만 저장. `attendance`에는 저장하지 않음.

## Firestore Security Rules (`firestore.rules`)

```js
match /members/{uid}               { allow read: if auth != null; allow write: if auth.uid == uid; }
match /attendance/{uid}/records/{date} { allow read: if auth != null; allow write: if auth.uid == uid; }
match /feed/{date}/entries/{uid}   {
  allow read: if auth != null;
  allow create, delete: if auth.uid == uid;   // 본인 엔트리만 생성·삭제
  allow update: if auth != null;              // 모든 멤버가 반응(r0~r3) 업데이트 가능
}
```

## 핵심 상태 변수 (JS 전역)

| 변수 | 설명 |
|---|---|
| `currentUser` | Firebase Auth 유저 객체 (uid, name, email, photo) |
| `userProfile` | 앱 내 프로필 (name, emoji, uid) — Firestore members에서 로드 |
| `myRecords` | 내 전체 출석 기록 배열 — attendance onSnapshot 실시간 동기화 |
| `allMembers` | 전체 멤버 배열 — members onSnapshot 실시간 동기화 |
| `todayAttendanceMap` | `{ uid: feedEntry }` — feed onSnapshot 실시간 동기화. r0~r3 반응 데이터 포함 |
| `myFeedCache` | `{ date: feedEntry }` — 과거 날짜 feed 엔트리 캐시 (프로필 탭 반응 표시용) |
| `allFeedEntryCache` | `{ "uid-date": feedEntry }` — 전체 피드 탭 로드 시 채워지는 캐시. `toggleReaction` 에서 참조 |
| `currentFeedTab` | `'today'` 또는 `'all'` — 오늘 탭 내 피드 서브탭 상태 |
| `checkedIn` | 오늘 출석 여부 (myRecords에서 todayStr 존재 여부로 결정) |
| `todayStr` | **로컬 시간** 기준 YYYY-MM-DD. `dateToLocal(today)` 헬퍼로 생성 |
| `ADMIN_UID` | 모임장 UID — **"demo"에서 실제 Firebase UID로 교체 필요** |

> `todayStr`은 반드시 로컬 시간 기준이어야 함. `toISOString()`은 UTC 기준이므로 사용 금지.
> 날짜 문자열이 필요할 때는 항상 `dateToLocal(dateObj)` 헬퍼 사용.

## 실시간 구독 구조 (onAuthStateChanged 내부)

```
unsubscribeAttendance  →  attendance/{uid}/records  (내 기록, 날짜 내림차순)
                           → myRecords 갱신 → renderToday/Profile/Stats
                           → feed 엔트리 누락 감지 시 feed에 자동 재동기화 (fallback)

unsubscribeMembers     →  members  (전체 멤버 목록)
                           → allMembers 갱신 → renderMemberFeed + renderRanking

unsubscribeFeed        →  feed/{todayStr}/entries  (오늘 피드, 반응 포함)
                           → todayAttendanceMap 갱신 → renderTodayFeedData + renderMemberFeed
                           → myFeedCache[todayStr] 동기화
                           → 프로필 탭 활성 상태면 renderProfile 호출
```

로그아웃 시 세 구독 모두 해제 및 `myFeedCache`, `allFeedEntryCache` 초기화, `switchFeedTab('today')` 호출.

## 렌더링 함수 역할 분리

| 함수 | 담당 DOM | 데이터 소스 |
|---|---|---|
| `renderToday()` | 오늘 탭 출석 폼/완료 뱃지, 요일 뱃지. **`#todayFeed`는 건드리지 않음** | `checkedIn`, `myRecords` |
| `renderTodayFeedData(entries)` | `#todayFeed` 단독 관리 | feed onSnapshot entries |
| `switchFeedTab(tab)` | `#todayFeed` / `#allFeed` 표시·숨김 전환 | `currentFeedTab` |
| `renderAllFeed()` | `#allFeed` 단독 관리 — 최근 14일 feed를 날짜별로 표시 | Firestore (fetch, 실시간 구독 없음) |
| `renderMemberFeed()` | `#memberFeed` 단독 관리 | `allMembers` + `todayAttendanceMap` |
| `renderStats()` | 통계 탭 (스트릭, 출석률, 차트) + `renderRanking()` 호출 | `myRecords` |
| `renderRanking()` | `#rankingSection` — 비동기, 멤버별 attendance 레코드 수 쿼리 | Firestore |
| `renderProfile()` | 내 정보 탭 (출석 기록 + 받은 반응) | `myRecords`, `myFeedCache`, `todayAttendanceMap` |
| `loadProfileReactions()` | 과거 날짜 feed 엔트리를 `myFeedCache`에 로드 후 `renderProfile` 호출 | Firestore (캐시 미스 시만) |

## 반응(이모지) 시스템

반응은 `feed/{date}/entries/{uid}` 문서의 `r0`~`r3` 필드에 **반응한 멤버 uid 배열**로 저장된다.

```js
const REACTION_KEYS      = ["r0","r1","r2","r3"];
const REACTION_EMOJIS_ARR = ["👍","❤️","💪","🎉"];
```

- **`buildReactionBar(entryData, cardId, isMe)`** — 피드/멤버 탭용. isMe=true면 읽기 전용 span, false면 클릭 가능 button
- **`buildReceivedBar(feedEntry)`** — 프로필 탭 전용. 반응 수 + 반응한 멤버 이모지 표시 (`👍 2 · 🌟 🦁`)
- **`toggleReaction(cardId, emoji)`** — `arrayUnion`/`arrayRemove`로 Firestore 업데이트. 현재 반응 상태는 `todayAttendanceMap[authorUid]` → `allFeedEntryCache[cardId]` → `myFeedCache[date]` 순으로 조회. 오늘 피드는 onSnapshot이 자동 갱신, **전체 피드 탭에서는 400ms 후 `renderAllFeed()` 재호출**

## 디자인 시스템

CSS 변수 (`index.html` `<style>` 상단):
```
--peach: #FFB347      포인트
--sunrise: #FF8C69    주요 액션 (버튼, 강조)
--mint: #98D8C8       보조
--lavender: #C8A8E9   관리자/특수
--cream: #FFF8F0      배경
--text: #5C4A3A       본문
--sub: #A08070        보조 텍스트
```

- 버튼: `border-radius: 50px`, 그라디언트
- 카드: `border-radius: 20px`
- 다크 모드: `body.dark` 클래스로 전환. 모든 신규 스타일에 다크 오버라이드 추가 필요
- UI 언어: 한국어 유지

## 주의사항

- **날짜 계산**: `today.toISOString()`은 UTC 기준 → KST 새벽에 전날 날짜가 됨. 반드시 `dateToLocal(d)` 헬퍼 사용
- **`#todayFeed` / `#allFeed`**: `renderTodayFeedData()`와 `renderAllFeed()`가 각각 단독 관리. `renderToday()`에서 건드리면 타이밍 충돌로 피드가 빈 상태로 덮어써짐
- **전체 피드 탭**: `renderAllFeed()`는 실시간 구독이 없으므로 반응 토글 후 자동 재fetch됨. 날짜 파싱 시 `new Date(date+'T00:00:00')` 형태로 로컬 시간 기준 처리
- **`attendance` 수정 시**: 날짜가 `todayStr`이면 `feed`도 반드시 동기화 (수정·삭제·공개토글 모두)
- **차트 재렌더링**: `weekChart`, `detailChart`는 재렌더링 전 반드시 `.destroy()` 호출
- **`ADMIN_UID`**: 실제 Firebase UID로 교체 전까지 관리자 기능 비활성화
- **`renderRanking()`**: 멤버 수만큼 Firestore read 발생. 소규모 모임 전제로 설계됨
