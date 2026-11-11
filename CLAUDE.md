# CareerFit — 프로젝트 컨텍스트

> 이 파일은 Claude Code가 매 세션 시작 시 읽는 프로젝트 규칙입니다. 레포 루트에 둡니다.

## 서비스 한 줄 정의
사용자의 경험(이력서·프로젝트·업무)을 구조화하고, 채용공고 요구사항과 근거 중심으로 매칭해 지원 전략을 제안하는 AI 취업 지원 웹 서비스.

## 기술 스택 (변경 시 반드시 먼저 물어볼 것)
- Next.js (App Router) + TypeScript (strict)
- Tailwind CSS
- Supabase: Auth, PostgreSQL, Storage (이력서 PDF)
- LLM API: 서버(Route Handler / Server Action)에서만 호출
- 검증: zod (폼 입력, API 응답, LLM 출력 모두)
- PDF 텍스트 추출: 서버에서 처리 (unpdf 또는 pdf-parse)
- 채용공고 목록: 공식 API만 사용 (사람인 Open API 또는 로켓펀치 Open API). 크롤링 금지.
- 배포: Vercel

## 폴더 구조
```
src/
  app/
    (auth)/login, signup
    onboarding/            # 직군 선택
    experiences/           # 경험 등록·목록·수정
    jobs/                  # 공고 탐색(목록) · [id] 상세
    analyses/[id]/         # 분석 결과
    api/                   # Route Handlers
  components/
    ui/                    # 버튼, 인풋, 카드, 스켈레톤, 빈 상태, 에러 박스
  lib/
    supabase/              # client.ts, server.ts
    jobs/                  # 공고 소스 어댑터 (JobSource 인터페이스)
    llm/                   # LLM 호출, 프롬프트, 출력 스키마
    pdf/                   # PDF 텍스트 추출
  types/
supabase/migrations/       # SQL 마이그레이션
```

## 핵심 데이터 모델
- `profiles` (id = auth.uid, job_category, created_at)
- `experiences` (id, user_id, title, job_category, role, tools text[], problem, solution, result, evidence_url, created_at)
- `resumes` (id, user_id, file_path, extracted_text, created_at)
- `job_postings` (id, user_id, source, external_id, url, title, company, summary jsonb, raw_text, created_at)
- `analyses` (id, user_id, job_posting_id, result jsonb, created_at)
- 모든 테이블에 RLS: `user_id = auth.uid()` 인 행만 읽기/쓰기

## 공고 소스 어댑터
```ts
interface JobSource {
  search(params: { keyword: string; jobCategory: string; page: number }): Promise<JobSummary[]>;
  getDetail(externalId: string): Promise<JobSummary>;
}
```
- `mockSource` (로컬 JSON) → `saraminSource` 또는 `rocketpunchSource` 순서로 구현
- API 키 승인 전에도 mock으로 개발이 막히지 않게 할 것

## 공고 본문 확보 규칙
1. 목록·요약은 공식 API 데이터 사용
2. "이 공고로 분석하기" 클릭 → 서버에서 공고 URL 1건 fetch 시도
3. 실패/이미지 공고/본문 부족 → 사용자에게 붙여넣기 입력창 표시 (API 요약 정보는 미리 채워둠)

## AI 분석 원칙 (가장 중요)
- 매칭 결과는 반드시 3단계로 구분: `confirmed`(확인된 근거) / `needs_check`(추가 확인 필요) / `missing`(정보 부족)
- `confirmed`에는 반드시 근거(어떤 경험의 어떤 내용인지)를 포함
- 합격 가능성 점수·단정 표현 금지
- LLM 출력은 JSON으로 받고 zod로 검증, 실패 시 1회 재시도 후 에러 상태 표시

## 코딩 규칙
- `any` 금지. 타입은 `src/types` 또는 zod 스키마에서 추론
- API 키·서비스 롤 키는 서버 전용. `NEXT_PUBLIC_` 접두사에 비밀값 금지
- 모든 비동기 화면에 로딩 / 에러 / 빈 상태 3가지 UI를 구현
- 모바일(375px)부터 반응형
- 컴포넌트는 작게, 서버 컴포넌트 기본, 상호작용 필요한 곳만 `"use client"`

## 작업 방식
- 한 번에 PLAN.md의 태스크 하나만 진행
- 작업 전 변경할 파일 목록과 계획을 먼저 보여주고 확인받기
- 작업 후: `npm run lint`, `npm run build` 통과 확인 → PLAN.md 체크박스 체크 → 커밋 메시지 제안
- 커밋 형식: `feat: ...`, `fix: ...`, `chore: ...`, `docs: ...`