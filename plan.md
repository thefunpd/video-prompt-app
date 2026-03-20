# 비디오 프롬프트 생성 앱

카메라 용어, 영상 포맷, 타겟/커리큘럼 옵션을 입력하면 AI가 대본 → 스토리보드 → Veo 프롬프트를 자동 생성하는 웹앱. 단일 HTML 파일과 Next.js 두 버전으로 제공.

---

## Phase 1 — 기반 세팅

목표: 프로젝트 뼈대 완성, 데이터 정의, 폼 UI 구현

### 1-1. 프로젝트 초기화
- [ ] `npx create-next-app@latest` (TypeScript, Tailwind, App Router)
- [ ] 디렉토리 구조 생성 (`src/lib/`, `src/components/`, `src/store/`, `src/app/api/`, `__tests__/`)
- [ ] `@anthropic-ai/sdk`, `zustand`, `jest`, `@testing-library/react` 설치
- [ ] Jest 설정 (`jest.config.js`, `jest.setup.ts`, module alias)
- [ ] 단일 HTML 버전 (`video-prompt-app.html`) 병행 관리

### 1-2. 데이터 레이어

#### `src/lib/types.ts` — 핵심 타입 정의
- [ ] `VideoOptions` 타입
  - `targetAudience`: string — 대상 (초등학생, 직장인, 대학생 등)
  - `language`: `'ko' | 'en'` — 출력 언어
  - `videoFormat`: `'tutorial' | 'explainer' | 'edutainment-short' | 'lecture'` — 영상 포맷
  - `curriculum`: string — 주제/커리큘럼 상세 내용
- [ ] `Script` 타입
  - `title`: string — 영상 제목
  - `sections`: `ScriptSection[]` — 섹션 배열
  - `totalDuration`: string — 총 영상 길이
- [ ] `ScriptSection` 타입
  - `heading`: string — 섹션 제목
  - `content`: string — 나레이션/대사 포함 내용
  - `duration`: string — 예상 소요 시간
- [ ] `StoryboardScene` 타입
  - `sceneNumber`: number — 씬 번호
  - `description`: string — 장면 설명
  - `dialogue`: string — 대사/나레이션
  - `visualDescription`: string — 화면에 보이는 것
  - `cameraAngle`: string — 카메라 앵글 (EWS, WS, MS, MCU, CU 등)
  - `duration`: string — 장면 길이
- [ ] `VeoPrompt` 타입
  - `sceneNumber`: number — 씬 번호
  - `prompt`: string — Veo 영상 생성 프롬프트 (영어)
  - `cameraAngle`: string — 카메라 앵글
  - `mood`: string — 분위기 (Energetic, Calm, Dramatic 등)
  - `action`: string — 주요 동작
  - `visualStyle`: string — 비주얼 스타일
  - `duration`: string — 장면 길이
- [ ] `VIDEO_FORMAT_LABELS` — 포맷 한글/영어 라벨 매핑

### 1-3. 옵션 입력 폼 컴포넌트

#### `src/components/OptionsForm.tsx`
- [ ] 타겟(대상) — text input, placeholder "예: 초등학생, 직장인"
- [ ] 언어 — select dropdown (한국어 / English)
- [ ] 영상 포맷 — select dropdown
  - 튜토리얼 (tutorial)
  - 설명 영상 (explainer)
  - 에듀테인먼트 숏폼 (edutainment-short)
  - 강의 (lecture)
- [ ] 커리큘럼 — textarea, 4줄, placeholder "영상에서 다룰 주제와 내용을 상세히 입력하세요"
- [ ] 제출 버튼 — "대본 생성 시작"
- [ ] 필수값 미입력 시 submit 방지 (HTML required + JS 검증)
- [ ] 테스트: 폼 렌더링, 필수값 검증, onSubmit 콜백 호출 확인

---

## Phase 2 — Claude API 연동 & 프롬프트 설계

목표: AI 프롬프트 작성, API 래퍼, 3개 API 라우트 구현 (TDD)

### 2-1. 프롬프트 빌더

#### `src/lib/prompts.ts` — 3개 함수, 각각 `{ system, user }` 반환

- [ ] `buildScriptPrompt(options: VideoOptions)`
  - system: 전문 영상 대본 작가 역할, 언어별 지시, JSON 형식 강제
  - user: 타겟, 포맷(언어별 라벨), 커리큘럼 포함
  - JSON 스키마: `{ title, sections: [{ heading, content, duration }], totalDuration }`
- [ ] `buildStoryboardPrompt(script: Script, options: VideoOptions)`
  - system: 전문 스토리보드 작가 역할, 장면(scene) 분해 지시, JSON 배열 강제
  - user: 대본 제목, 총 길이, 섹션별 내용 전달
  - JSON 스키마: `[{ sceneNumber, description, dialogue, visualDescription, cameraAngle, duration }]`
- [ ] `buildVeoPrompt(storyboard: StoryboardScene[], options: VideoOptions)`
  - system: Veo 프롬프트 전문가 역할, 영어 프롬프트 생성 지시
  - user: 각 씬의 설명/대사/화면/앵글/길이 전달
  - JSON 스키마: `[{ sceneNumber, prompt, cameraAngle, mood, action, visualStyle, duration }]`
- [ ] 테스트 (11개): 타겟/포맷/커리큘럼 포함 확인, JSON 지시 확인, 언어별 분기 확인

### 2-2. Claude API 래퍼

#### `src/lib/claude.ts`
- [ ] `generateJSON<T>(system, user): Promise<T>`
  - Anthropic SDK 클라이언트 싱글턴 생성 (`ANTHROPIC_API_KEY`)
  - 모델: `claude-sonnet-4-20250514`, max_tokens: 4096
  - 응답에서 text 블록 추출
  - JSON 파싱: 직접 파싱 시도 → 실패 시 `{...}` 또는 `[...]` 정규식 매칭
  - 파싱 실패 시 에러 throw
- [ ] 테스트 (3개): 정상 JSON 파싱, 혼합 텍스트에서 JSON 추출, 비정상 응답 에러 처리

### 2-3. API 라우트 (3개)

| 라우트 | 메서드 | 입력 | 출력 | 검증 |
|--------|--------|------|------|------|
| `/api/generate-script` | POST | `VideoOptions` | `Script` | targetAudience, language, videoFormat, curriculum 필수 |
| `/api/generate-storyboard` | POST | `{ script, options }` | `StoryboardScene[]` | script.title, script.sections, options 필수 |
| `/api/generate-veo-prompts` | POST | `{ storyboard, options }` | `VeoPrompt[]` | storyboard[].length > 0, options 필수 |

각 라우트 공통:
- [ ] 입력 검증 → 실패 시 400 + 에러 메시지
- [ ] 프롬프트 빌더 호출 → Claude API 호출 → JSON 반환
- [ ] API 오류 시 500 + 에러 메시지
- [ ] 테스트: 정상 응답, 400 검증, 500 에러 처리 (claude.ts mock)

---

## Phase 3 — 상태관리 & 결과 표시 UI

목표: 4단계 플로우 상태 관리, 결과 카드 UI 구현

### 3-1. Zustand 스토어

#### `src/store/useAppStore.ts`
- [ ] 상태: `options`, `script`, `storyboard`, `veoPrompts`, `currentStep` (1~4)
- [ ] 액션: `setOptions` (→ step 2), `setScript` (→ step 3), `setStoryboard` (→ step 4), `setVeoPrompts`, `setStep`, `reset`
- [ ] 페이지 새로고침 시 상태 초기화 (별도 persist 없음, MVP)

### 3-2. 공통 UI 컴포넌트

#### `src/components/StepNavigation.tsx`
- [ ] 4단계 표시: 옵션 입력 → 대본 생성 → 스토리보드 → Veo 프롬프트
- [ ] 현재 단계: 파란색 원 + 볼드 라벨
- [ ] 완료 단계: 초록색 ✓ + 연결선 초록
- [ ] 미완료 단계: 회색 원

#### `src/components/CopyButton.tsx`
- [ ] `navigator.clipboard.writeText` 호출
- [ ] 복사 후 2초간 "복사됨!" 피드백
- [ ] 테스트 (3개): 렌더링, clipboard 호출, 피드백 표시

#### `src/components/LoadingSpinner.tsx`
- [ ] 스피너 애니메이션 + 커스텀 메시지 ("대본을 생성하고 있습니다..." 등)

### 3-3. 결과 표시 컴포넌트

#### `src/components/ScriptView.tsx`
- [ ] 제목 + 전체 복사 버튼
- [ ] 총 길이 표시
- [ ] 섹션별 카드: 제목, 내용(pre-wrap), 시간 배지, 개별 복사 버튼
- [ ] "← 이전" + "스토리보드 생성" 버튼

#### `src/components/StoryboardView.tsx`
- [ ] "스토리보드" 제목 + 전체 복사 버튼
- [ ] 씬별 카드: 씬 번호(파란색), 시간 배지
- [ ] 2열 그리드: 설명, 대사, 화면, 카메라
- [ ] "← 이전" + "Veo 프롬프트 생성" 버튼

#### `src/components/VeoPromptView.tsx`
- [ ] "Veo 프롬프트" 제목 + 전체 복사 + JSON 내보내기 버튼
- [ ] 씬별 카드: 씬 번호(보라색), 개별 복사 버튼
- [ ] 프롬프트 박스: 모노스페이스, 회색 배경
- [ ] 4열 메타 그리드: 앵글(파랑), 분위기(주황), 동작(초록), 스타일(보라)
- [ ] "← 이전" 버튼

---

## Phase 4 — 페이지 조립 & 통합

목표: 메인 페이지에서 전체 플로우 연동

### 4-1. 메인 페이지 (`src/app/page.tsx`)
- [ ] `'use client'` 선언
- [ ] Zustand 스토어에서 상태/액션 가져오기
- [ ] `isLoading`, `error` 로컬 상태 관리
- [ ] Step 1: `OptionsForm` → fetch `/api/generate-script` → `setScript`
- [ ] Step 2: `ScriptView` → fetch `/api/generate-storyboard` → `setStoryboard`
- [ ] Step 3: `StoryboardView` → fetch `/api/generate-veo-prompts` → `setVeoPrompts`
- [ ] Step 4: `VeoPromptView` + JSON 내보내기 (Blob 다운로드)
- [ ] 에러 시 빨간 배너 표시
- [ ] 로딩 시 스피너 표시 (단계별 메시지)
- [ ] "처음부터 다시 시작" 버튼 (step > 1일 때 표시)

### 4-2. 레이아웃 (`src/app/layout.tsx`)
- [ ] `lang="ko"`, Geist 폰트
- [ ] 메타데이터: "비디오 프롬프트 생성기"
- [ ] 배경: `bg-gray-50`

---

## Phase 5 — 단일 HTML 버전

목표: 프레임워크 없이 `video-prompt-app.html` 하나로 동작

### 5-1. 구조
- [ ] 인라인 CSS (Tailwind 없이, 커스텀 스타일)
- [ ] 인라인 JS (모듈 없이, 전역 함수)
- [ ] 동일한 4단계 플로우

### 5-2. API 연동
- [ ] `anthropic-dangerous-direct-browser-access` 헤더로 브라우저 직접 호출
- [ ] API Key 입력란 (비워두면 테스트 모드)

### 5-3. 테스트 모드 (API Key 없이)
- [ ] `generateMockScript(opts)` — 입력값(타겟, 포맷, 커리큘럼) 기반 mock 대본 생성
- [ ] `generateMockStoryboard(script, opts)` — 대본 기반 mock 스토리보드
- [ ] `generateMockVeoPrompts(storyboard, opts)` — 스토리보드 기반 mock Veo 프롬프트
- [ ] 언어별 분기: 한국어/영어 mock 데이터 각각 생성
- [ ] 포맷 라벨: `FORMAT_LABELS_KO`, `FORMAT_LABELS_EN` 분리

### 5-4. UI 기능
- [ ] 단계 인디케이터 (1-2-3-4, 색상 변화)
- [ ] 각 단계별 복사/전체 복사 버튼
- [ ] JSON 내보내기 (Blob 다운로드)
- [ ] 이전 단계 이동, 처음부터 다시 시작
- [ ] 로딩 스피너 + 에러 메시지 표시

---

## Phase 6 — 마무리 & 검증

### 6-1. 테스트 실행
- [ ] `npx jest` — 전체 27개 테스트 통과 확인
  - lib/prompts: 11개 (프롬프트 빌더)
  - lib/claude: 3개 (API 래퍼)
  - api 라우트: 7개 (3개 라우트 × 정상/에러)
  - components: 6개 (OptionsForm 3 + CopyButton 3)
- [ ] `npm run build` — Next.js 빌드 성공 확인

### 6-2. 수동 테스트
- [ ] HTML 버전: 테스트 모드로 전체 플로우 확인 (한국어 + 영어)
- [ ] HTML 버전: API Key 입력 후 실제 Claude API 호출 확인
- [ ] 복사 기능 동작 확인
- [ ] JSON 내보내기 파일 확인
- [ ] 모바일 반응형 확인

### 6-3. 완료 조건
- [ ] 4단계 플로우 정상 동작 (옵션 → 대본 → 스토리보드 → Veo 프롬프트)
- [ ] 언어 선택 시 해당 언어로 전체 콘텐츠 생성
- [ ] 각 단계 결과물 복사/내보내기 가능
- [ ] 에러 발생 시 사용자 안내
- [ ] 테스트 전체 통과, 빌드 성공
