# 알:음 (AL:EUM) — 시스템 아키텍처 설계 문서 (Full Implementation)

> **전체 기능 구현 목표 버전**
> **집중 개발 기간**: 3일 MVP + 3주 보강
> **최종 공모 제출**: 2026-05-31
> **목표**: 교육부 공모전 대상 + 범정부 본선 진출

---

## 0. 문서 목적

이 문서는 알:음의 **모든 기능을 구현하는 것을 목표**로 하는 완전한 아키텍처 설계입니다.
기획서에 명시된 11개 기능 모듈을 전부 포괄하며, 각 모듈이 참조하는 GitHub 레퍼런스를 명시합니다.

---

## 1. 개요 (Overview)

### 1.1 서비스 정의
한국어를 모르는 다문화 학부모·학생·교사에게 학교의 모든 정보를 **모국어 + 맥락 해설 + Action Item + 안전 경고 + 위험 예측**으로 변환하는 통합 교육 접근성 AI 플랫폼.

### 1.2 핵심 가치
- **실시간성**: NEIS API 자동 수집 (교사 수동 개입 X)
- **정확성**: 교육 용어 RAG + Groundedness Check + HITL
- **행동성**: 단순 번역이 아닌 "해야 할 일" 추출
- **포용성**: 6개 언어 + TTS/STT + WCAG 2.1 AA
- **예방성**: ML 기반 학업중단 위험 사전 감지
- **정책성**: MEGI 지수로 정책 공정성 측정

### 1.3 3-Sided Platform 구조

| 대상 | 채널 | 주요 기능 |
|---|---|---|
| **학부모** | 웹 + PWA + 카카오톡 봇 | 공지 번역, 급식 안전, Action Items |
| **학생** | 웹 + 핸드트래킹 학습 | 교육 용어 플래시카드, 위험 조기 감지 |
| **교육청** | 대시보드 | MEGI 지수, 위험 학생 리스트, 정책 효과 측정 |

### 1.4 Goals (Full Implementation)

**Must Have (Phase 1 - MVP 3일):**
- NEIS 공지 실시간 변환
- 6개 언어 번역 + 맥락 해설
- Action Item 자동 추출
- 급식 알레르기/할랄 감지
- TTS 모국어 재생
- 신뢰도 공개 (Responsible AI)

**Should Have (Phase 2 - 보강 2주):**
- 학업중단 예측 ML
- MEGI 지수 계산
- 교육청 대시보드
- STT 음성 질문
- 쉬운 한국어 버전
- Upstage Document Parser (PDF 가정통신문 OCR)

**Nice to Have (Phase 3 - 최종 1주):**
- 교육 용어 핸드트래킹 학습 (웹 MediaPipe)
- 카카오톡 봇 연동
- HITL 교사 검토 UI
- 수어 영상 (중요 공지 AI 생성)

---

## 2. High-Level Architecture

### 2.1 시스템 구조도

```
╔═══════════════════════════════════════════════════════════════════╗
║                      USERS (3 Personas)                           ║
║  학부모 (Parent)    학생 (Student)    교육청 (District)           ║
╚═══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
╔═══════════════════════════════════════════════════════════════════╗
║                   CHANNELS (Omnichannel)                          ║
║  PWA Web App  KakaoTalk Bot  Admin Dashboard                     ║
╚═══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
╔═══════════════════════════════════════════════════════════════════╗
║              FRONTEND (Vercel / Lovable)                           ║
║  Parent UI (6 langs) | Student UI (HandTrack) |                   ║
║  District Dashboard | Teacher HITL Queue                          ║
╚═══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
╔═══════════════════════════════════════════════════════════════════╗
║              API GATEWAY (Supabase Edge Functions)                 ║
║  Auth | Rate Limit | CORS | Request Routing | Audit Log           ║
╚═══════════════════════════════════════════════════════════════════╝
                               │
      ┌────────────┬───────────┼───────────┬────────────┐
      ▼            ▼           ▼           ▼            ▼
  INGESTION    AI PIPELINE  ML PIPELINE  ANALYTICS   DELIVERY
  NEIS Poll    Translate    Dropout      MEGI        Push
  PDF OCR      Context      Prediction   Compute     Kakao
  Schedule     Action       Anomaly      Aggregate   Email
  Meals        Safety       Detection    Trends      SMS
               TTS/STT
      │            │           │           │            │
      └────────────┴───────────┼───────────┴────────────┘
                               ▼
╔═══════════════════════════════════════════════════════════════════╗
║                     DATA LAYER (Supabase)                          ║
║  PostgreSQL (relational) | pgvector (RAG) | Storage (audio, pdf)  ║
╚═══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
╔═══════════════════════════════════════════════════════════════════╗
║                 EXTERNAL SERVICES & DATA SOURCES                   ║
║  NEIS API | Gemini | Upstage | GCP TTS | Whisper | KEDI | 학교알리미 ║
╚═══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
╔═══════════════════════════════════════════════════════════════════╗
║                    RESPONSIBLE AI LAYER                            ║
║  Groundedness | HITL Queue | PII Redaction | Audit Log | Bias Test║
╚═══════════════════════════════════════════════════════════════════╝
```

### 2.2 데이터 흐름 요약

```
NEIS API ─┐
학교알리미 ├─→ Ingestion ─→ Normalize ─→ DB
KEDI     ─┘                              │
                                         ▼
                                  LLM Pipeline
                                  (Translate +
                                   Context +
                                   Actions +
                                   Safety)
                                         │
                                         ▼
                                  Quality Gate
                                  (Groundedness
                                   >= 0.7 pass /
                                   < 0.7 HITL)
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
            Parent Feed         Student Learning      District Analytics
            (PWA + Kakao)       (Web HandTrack)       (MEGI + Dropout)
```

---

## 3. 기술 스택 (Full Stack)

### 3.1 Frontend Layer

| 영역 | 기술 | 참조 프로젝트 |
|---|---|---|
| **프레임워크** | React 18 + TypeScript + Vite | (기존 안실) |
| **스타일링** | Tailwind CSS + shadcn/ui | (기존) |
| **빌더** | Lovable (AI 코드 생성) | (기존) |
| **국제화 (i18n)** | react-i18next | [Shafihu/InSnip](https://github.com/Shafihu/InSnip) — 다국어 K-12 i18n 패턴 |
| **라우팅** | React Router v6 | (표준) |
| **상태관리** | TanStack Query + Zustand | (기존) |
| **핸드트래킹** | MediaPipe Hands (웹) | [VirtualQuizGame-OpenCV](https://github.com/) — 제스처 매핑 패턴 |
| **3D/수어** | Three.js (아바타) | (BridgeCast 재활용) |
| **차트** | Recharts + d3.js | (기존 안실) |
| **지도** | Leaflet / Mapbox | (기존 안실) |
| **PWA** | Workbox + vite-plugin-pwa | [Shafihu/InSnip](https://github.com/Shafihu/InSnip) — 저대역폭 패턴 |
| **a11y** | WCAG 2.1 AA, ARIA | (BridgeCast Accessibility Report) |

### 3.2 Backend Layer

| 영역 | 기술 | 참조 프로젝트 |
|---|---|---|
| **BaaS** | Supabase (Auth + DB + Edge + pgvector + Realtime) | (안실) |
| **Edge Functions** | Deno (TypeScript) | (표준) |
| **DB** | PostgreSQL 15 + pgvector | (표준) |
| **Queue** | Supabase Realtime Channels | (표준) |
| **Storage** | Supabase Storage (audio, PDF) | (표준) |
| **Cron** | Supabase pg_cron | (표준) |
| **Auth** | Supabase Auth (이메일/소셜) | (기존) |
| **Security** | Row Level Security (RLS) | (표준) |
| **Python Worker** (무거운 ML) | FastAPI + Modal.com or Cloud Run | [dssg/student-early-warning](https://github.com/dssg/student-early-warning) 패턴 |

### 3.3 AI / ML Layer

| 모듈 | 기술 | 참조 프로젝트 |
|---|---|---|
| **번역 LLM** | Gemini 1.5 Flash (비용 효율) | [lalah-chatbot](https://github.com/) — Multi-LLM 패턴 |
| **대안 LLM (한국어 특화)** | KoAlpaca (fine-tune 옵션) | [Beomi/KoAlpaca](https://github.com/Beomi/KoAlpaca) |
| **번역 모델 (로컬)** | NLLB (6개 언어 지원) | [vEduardovich/dodari_nllb](https://github.com/vEduardovich/dodari_nllb) — 정확히 6개 언어 커버 |
| **한국어 번역** | Gugugo (EN↔KO) | [jwj7140/Gugugo](https://github.com/jwj7140/Gugugo) |
| **환각 방지** | Upstage Groundedness Check | (Shellter 기존) |
| **RAG 임베딩** | OpenAI text-embedding-3-small / Upstage Solar Embedding | (Shellter 기존) |
| **Vector DB** | Supabase pgvector | (표준) |
| **프롬프트 관리** | 자체 버전 관리 + YAML | [Shubhamsaboo/awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps) — 에이전트 패턴 |
| **Action Item 추출** | LLM + structured output | [robocorp/example-llm-emails](https://github.com/robocorp/example-llm-emails) — 이메일→태스크 패턴 |
| **다국어 공공 RAG** | 11개 언어 공공 서비스 | [amrutha0001/Sahayak](https://github.com/amrutha0001/Sahayak_Indic_Muliti-lingual_Chatbot) — 인도 복지 RAG 참조 |
| **다국어 99개 RAG** | Whisper + 99언어 | [Saptarshiii/Multilingual-RAG-ChatBot](https://github.com/Saptarshiii/Multilingual-RAG-ChatBot) |
| **교육 용어집 RAG** | 용어집 기반 번역 | [Multi-language-Course-Content-Translator-Agent](https://github.com/) — IBM Watsonx 패턴 |
| **Document OCR** | Upstage Document Parser | (Shellter 기존) |
| **알레르기 감지** | 키워드 매칭 + E-code | [UsmanAsad87/Halal_Food_Detector](https://github.com/UsmanAsad87/Halal_Food_Detector) — Halal E-code DB |
| **알레르기 Taxonomy** | 알레르기 분류 체계 | [kevinkchen1/SafeGrub](https://github.com/kevinkchen1/SafeGrub) |
| **할랄 검증** | 원료 기반 분류 | [AI_Halal_Product_Detector](https://github.com/) — 결정 엔진 패턴 |
| **알레르기 ML** | 성분 기반 분류 | [AllergySavvy/ML-AllergySavvy-Dev](https://github.com/AllergySavvy/ML-AllergySavvy-Dev) |
| **학업중단 예측 ML** | LightGBM + Random Forest 앙상블 | [dropout-prediction](https://github.com/) (기존) + [dssg/student-early-warning](https://github.com/dssg/student-early-warning) — 형평성 프레임 |
| **소수자 중도탈락 ML** | 멀티태스크 러닝 | [ismailelbouknify/Student-At-Risk-Identification](https://github.com/ismailelbouknify/Student-At-Risk-Identification) — 모로코 사례 |
| **학생 성과 분석 UI** | Streamlit 대시보드 | [Vivekchary2607/EduPredict](https://github.com/Vivekchary2607/AI-Powered-EduPredict-Student-Performance-Analytics-System) |
| **액션 아이템 후속** | 이메일 에이전트 | [deshwalmahesh/outlook-email-agent](https://github.com/deshwalmahesh/outlook-email-agent) |
| **TTS** | Google Cloud TTS (6개 언어) | (Shellter 기존) |
| **STT** | OpenAI Whisper (다국어) | (Shellter 기존) |

### 3.4 Data Sources (교육 공공데이터)

| No | 데이터 | 제공기관 | 활용 | 참조 |
|----|---|---|---|---|
| 1 | 학교기본정보 | NEIS | 학교 정보, 밀집도 | [my-school-info/neis-api](https://github.com/my-school-info/neis-api) (TS) |
| 2 | 급식식단정보 | NEIS | 급식 다국어 + 할랄/알레르기 | [agemor/neis-api](https://github.com/agemor/neis-api) (Kotlin, 72★) |
| 3 | 학사일정 | NEIS | Action Item 추출 | [star0202/neis.ts](https://github.com/star0202/neis.ts) |
| 4 | 시간표 | NEIS | 일일 일정 다국어 | [SaidBySolo/neispy](https://github.com/SaidBySolo/neispy) (Python) |
| 5 | 교육기본통계 | KEDI | MEGI 시계열 | [kevinwang09/learningtower](https://github.com/kevinwang09/learningtower) — OECD 방법론 |
| 6 | 학교알리미 공시 | 교육부 | 학교별 다문화 비율 | (직접 파싱) |
| 7 | 학구도안내서비스 | 교육부 | 다문화 밀집 매핑 | (직접 파싱) |
| 8 | 다문화가족 실태조사 | 여가부 | 학부모 고충 정량화 | (정적 데이터) |
| 9 | 한국교육고용패널 | 직능연 | 진로·학업 지속률 | [dssg/student-early-warning](https://github.com/dssg/student-early-warning) 패턴 |
| 10 | 다문화교육포털 | 교육부 | 정책학교 + 다국어 원시 | (정적 데이터) |

### 3.5 Integration / Delivery (Omnichannel)

**채널 전략**: 다문화 가정 국가별 주요 메신저가 다르므로 멀티 채널 지원.
사업자 등록 전 단계에서는 **대화형 봇 + 웹 푸시 + 이메일** 조합으로 시작.

| 채널 | 기술 | 사업자 필요 | 비용 | 참조 |
|---|---|---|---|---|
| **PWA 웹 푸시** | Web Push API + Service Worker | ❌ | 무료 | (표준) |
| **카카오톡 채널 봇** | 카카오 i 오픈빌더 (대화형만) | ❌ | 무료 | [getsolaris/laravel-kakaobot](https://github.com/getsolaris/laravel-kakaobot) |
| **WhatsApp Business** | Meta Cloud API | ❌ | 월 1,000통 무료 | (다문화 가정 최적) |
| **LINE 봇** | LINE Messaging API | ❌ | 월 500통 무료 | (베트남/일본 특화) |
| **Telegram 봇** | Telegram Bot API | ❌ | 무료 | (러시아/중앙아 특화) |
| **이메일** | Resend API | ❌ | 월 3,000통 무료 | (표준) |
| **SMS** | NHN Cloud SMS | ❌ | 건당 ~9원 | (긴급 알림용) |
| **알림톡 (카카오)** | 카카오 비즈 | ✅ **사업자 필수** | 건당 ~8원 | (파일럿 이후) |
| **친구톡 (카카오)** | 카카오 비즈 | ✅ **사업자 필수** | 건당 ~11원 | (파일럿 이후) |
| **학부모-교사 통신** | 알림장 데이터 모델 | ❌ | 무료 | [Bennacci/eCommunicationBook](https://github.com/Bennacci/eCommunicationBook) |
| **학부모-교사 소통** | 커뮤니케이션 플랫폼 | ❌ | 무료 | [cardosakv/Faculti-v2](https://github.com/cardosakv/Faculti-v2) |
| **상담 스케줄링** | Parent-Teacher Scheduler | ❌ | 무료 | [axyjo/ptscheduler](https://github.com/axyjo/ptscheduler) |

**단계별 채널 전략:**

```
Phase 1 (MVP - 공모전):
  1순위: PWA 웹 푸시 (브라우저 권한만)
  2순위: 카카오 i 오픈빌더 (대화형 챗봇)
  3순위: 이메일 (백업)

Phase 2 (파일럿 - 10개 학교):
  + WhatsApp Business API (다문화 가정 최적)
  + LINE 봇 (베트남/일본 학부모)
  + Telegram 봇 (러시아/우즈벡/몽골)

Phase 3 (시범 - 100개 학교):
  + 사업자 등록 완료
  + 카카오 알림톡 (자동 푸시)
  + 카카오 친구톡 (안내 메시지)
  + SMS 긴급 발송
```

**중요**: 카카오 알림톡/친구톡 = **사업자등록번호 필수**. 공모전 단계에서는 **카카오 i 오픈빌더 (챗봇)** 로만 시작 — 학부모가 채널 친추 후 **"오늘 공지" / "급식 확인"** 같은 대화형 응답만 가능. 이것도 "카카오 연동"이라 볼 수 있어 기획서 어필 가능.

### 3.6 Infrastructure & DevOps

| 영역 | 기술 |
|---|---|
| **Frontend 호스팅** | Vercel (무료 티어) |
| **Backend** | Supabase (무료 → Pro $25) |
| **ML 워커** | Modal.com (serverless GPU) or Google Cloud Run |
| **Monitoring** | Sentry (에러) + Vercel Analytics + PostHog |
| **CI/CD** | GitHub Actions + Vercel Auto-deploy |
| **Secret Management** | Supabase Vault + Vercel Env |
| **Domain/CDN** | Cloudflare |

---

## 4. 기능 모듈 상세 설계 (11개 모듈)

### 4.1 Module A — NEIS Integration (수집)

**목적**: 전국 11,800개 학교의 공지를 실시간 자동 수집

**구성 요소:**
```
NEIS Poller (Supabase Edge Function + Cron)
  - 매일 05:00 KST 자동 실행
  - 학교별 변경분만 fetch (etag 기반)
  - 실패 시 exponential backoff
     │
     ▼
Normalizer
  - NEIS 원문 → 정규화된 JSON 스키마
  - HTML 태그 제거, 공백 정리
  - 중복 제거
     │
     ▼
   [DB 저장]
```

**참조:**
- [my-school-info/neis-api](https://github.com/my-school-info/neis-api) — Node.js/TypeScript 구조
- [agemor/neis-api](https://github.com/agemor/neis-api) — 빠른 파싱 패턴
- [star0202/neis.ts](https://github.com/star0202/neis.ts) — 2025 활성 TS
- [SaidBySolo/neispy](https://github.com/SaidBySolo/neispy) — 타입 안정성

**엔드포인트 사용:**
- `schoolInfo` — 학교 기본
- `mealServiceDietInfo` — 급식
- `SchoolSchedule` — 학사
- `elsTimetable` / `misTimetable` / `hisTimetable` — 시간표
- `classInfo` — 반 정보

---

### 4.2 Module B — Document OCR (가정통신문)

**목적**: PDF/이미지 가정통신문을 텍스트로 변환

**기술:**
- Upstage Document Parser API (Shellter 경험)
- Fallback: Tesseract OCR

**플로우:**
```
교사 업로드 (PDF/PNG)
  → Upstage Parser
  → 구조화된 텍스트 + 표 추출
  → LLM 번역 파이프라인으로 전달
```

---

### 4.3 Module C — AI Translation Pipeline (핵심)

**목적**: 한국어 공지 → 6개 언어 맥락 번역

**계층 구조:**
```
Input (한국어 공지)
   │
   ▼
[Step 1] PII Redaction
   - 학생/학부모 실명 → [학생A]
   - 주소/연락처 → [주소]
   │
   ▼
[Step 2] 교육 용어 RAG 검색
   - pgvector similarity search
   - Top-10 관련 용어 + 번역 추출
   │
   ▼
[Step 3] LLM 호출 (Gemini 1.5 Flash)
   - System: 교육 맥락 번역기
   - Context: 용어집 + 학부모 프로필 (국적, 아이 학년)
   - Output: JSON {translation, explanation, actions, warnings}
   │
   ▼
[Step 4] Groundedness Check (Upstage)
   - 원문 vs 번역 일치도 측정
   - 날짜/숫자/이름 정합성 검증
   │
   ▼
[Step 5] 품질 게이트
   - score >= 0.9: 바로 배포
   - 0.7 <= score < 0.9: 경고 배지 표시하고 배포
   - score < 0.7: HITL 큐에 추가 (미배포)
   │
   ▼
[Step 6] PII 복원 (로컬)
   │
   ▼
Output (학부모 모국어)
```

**프롬프트 템플릿:**
```
[SYSTEM]
You are "알:음", an educational context translator for multicultural
parents in Korea. Your role is to translate Korean school notices into
{target_language} while preserving educational context and extracting
actionable items for the parent.

RULES:
1. Never invent dates, names, or facts not in the source.
2. Use the provided glossary for educational terms.
3. If uncertain, mark in `confidence_notes`.
4. Write at elementary-school reading level in target language.
5. Preserve Korean terms in parentheses where culturally significant.
6. Always extract action items with explicit deadlines.

[GLOSSARY] (retrieved from RAG)
{terms_json}

[PARENT CONTEXT]
- Origin country: {country}
- Korean proficiency: {level}
- Child grade: {grade}
- Dietary: {restrictions}

[INPUT - Korean School Notice]
"{original_text}"

[OUTPUT FORMAT - JSON ONLY]
{
  "translation": "...",
  "context_explanation": "...",
  "action_items": [
    {"task": "...", "deadline": "YYYY-MM-DD", "priority": "high|med|low"}
  ],
  "warnings": ["..."],
  "cultural_notes": "...",
  "confidence_notes": "..."
}
```

**참조:**
- [lalah-chatbot](https://github.com/) — Multi-LLM + 문화 민감성
- [Multi-language-Course-Content-Translator-Agent](https://github.com/) — 용어집 RAG
- [amrutha0001/Sahayak](https://github.com/amrutha0001/Sahayak_Indic_Muliti-lingual_Chatbot) — 공공 RAG
- [robocorp/example-llm-emails](https://github.com/robocorp/example-llm-emails) — Action Item 구조화

---

### 4.4 Module D — Meal Safety Detector (급식 안전)

**목적**: 급식 메뉴에서 할랄/알레르기 자동 감지

**알고리즘:**
```
메뉴 아이템 "돼지고기찌개"
   │
   ├─→ [1] NEIS 알레르기 코드 확인
   │       → allergen_codes: [10=돼지고기, 2=우유]
   │
   ├─→ [2] 메뉴명 키워드 매칭
   │       → 한국어 키워드 DB: {돼지, 돈, 삼겹, 목살, ...}
   │       → 할랄 여부 결정
   │
   ├─→ [3] 학부모 프로필 매칭
   │       → dietary_restrictions.halal = true
   │       → dietary_restrictions.allergens = [1, 3] (난류, 메밀)
   │
   └─→ [4] 경고 생성
         → 🔴 "돼지고기 포함 — 도시락 준비 필요"
         → 번역 파이프라인 → 모국어 경고
```

**알레르기 코드 매핑 (NEIS 19개 표준):**
```
1. 난류    2. 우유    3. 메밀    4. 땅콩    5. 대두
6. 밀     7. 고등어  8. 게      9. 새우    10. 돼지고기
11. 복숭아 12. 토마토 13. 아황산류 14. 호두 15. 닭고기
16. 쇠고기 17. 오징어 18. 조개류  19. 잣
```

**참조:**
- [UsmanAsad87/Halal_Food_Detector](https://github.com/UsmanAsad87/Halal_Food_Detector) — E-code 매핑 DB
- [kevinkchen1/SafeGrub](https://github.com/kevinkchen1/SafeGrub) — 알레르기 taxonomy
- [AI_Halal_Product_Detector](https://github.com/) — 결정 엔진 (다중 신호)
- [AllergySavvy/ML-AllergySavvy-Dev](https://github.com/AllergySavvy/ML-AllergySavvy-Dev) — ML 분류 (추가 정확도)

---

### 4.5 Module E — Action Item Extractor

**목적**: 공지문에서 학부모 "해야 할 일" 자동 추출

**추출 항목:**
- **할 일 (task)**: "도시락 준비"
- **기한 (deadline)**: "2026-05-12"
- **우선순위 (priority)**: high/med/low
- **연관 인물 (actor)**: 학부모 / 학생 / 교사
- **연관 항목 (related)**: 준비물, 서류, 참석

**LLM 프롬프트 (Action 특화):**
```
[TASK] Extract action items for the parent.

[INPUT]
"5월 15일(목) 현장체험학습 실시, 도시락 지참, 참가동의서 5/12까지 제출"

[OUTPUT]
[
  {
    "actor": "parent",
    "task": "Prepare lunchbox",
    "related_date": "2026-05-15",
    "priority": "high",
    "reason": "School meal not provided on field trip day"
  },
  {
    "actor": "parent",
    "task": "Sign and submit consent form",
    "deadline": "2026-05-12",
    "priority": "high",
    "reason": "Required by school"
  },
  {
    "actor": "child",
    "task": "Wear comfortable clothes",
    "related_date": "2026-05-15",
    "priority": "medium"
  }
]
```

**참조:**
- [robocorp/example-llm-emails](https://github.com/robocorp/example-llm-emails) — GPT-4 이메일→태스크
- [deshwalmahesh/outlook-email-agent](https://github.com/deshwalmahesh/outlook-email-agent) — 분류 + 회신

---

### 4.6 Module F — Dropout Prediction ML (학업중단 예측)

**목적**: 다문화 학생 학업중단 위험 조기 감지

**Feature 설계 (다문화 특화):**
```python
features = {
    # 기본 학생 정보
    "age": int,
    "grade": int,
    "gender": str,
    "student_type": str,  # 국내출생 / 중도입국 / 외국인

    # 다문화 특화
    "country_of_origin": str,
    "korean_proficiency_level": str,  # A1~C2
    "years_in_korea": int,

    # 학업 지표
    "attendance_rate": float,
    "grade_average": float,
    "homework_completion_rate": float,
    "teacher_interaction_count": int,

    # 알:음 사용 지표 (독창)
    "notice_read_rate": float,  # 학부모 공지 확인률
    "parent_korean_level": str,
    "action_item_completion_rate": float,

    # 가정 환경
    "parent_education_level": str,
    "household_income_band": str,
    "single_parent": bool,

    # 지역/학교
    "region": str,
    "school_multicultural_ratio": float,
    "MEGI_score": float,
}
```

**모델:**
- **LightGBM** (주) + **CatBoost** (보조) 앙상블
- 5-fold 층화 교차검증
- SHAP 값으로 설명가능성

**HITL 통합:**
- 예측 위험도 70+ → 교사 알림
- 교사 검토 → "실제 위험 / 오진" 레이블
- 레이블로 모델 재학습

**공정성 검증:**
- 국가별 모델 성능 평가 (DEI 프레임)
- 편향 테스트: 국적/성별/지역별 false positive 균형

**참조:**
- [dropout-prediction](https://github.com/) (기존, Decision Tree 기본)
- [dssg/student-early-warning](https://github.com/dssg/student-early-warning) — DSSG 대표작, 형평성
- [ismailelbouknify/Student-At-Risk-Identification](https://github.com/ismailelbouknify/Student-At-Risk-Identification) — 소수자 멀티태스크
- [Vivekchary2607/EduPredict](https://github.com/Vivekchary2607/AI-Powered-EduPredict-Student-Performance-Analytics-System) — 대시보드 패턴

---

### 4.7 Module G — MEGI Index Calculator (정책 지수)

**MEGI (Multicultural Education Gap Index) 산출:**

**수식:**
```
4차원 변수 (OECD 복합지표 가이드라인):

1. 수요 (Demand):   다문화 학생 비율
2. 위험 (Risk):     학업중단율
3. 공급 (Supply):   전담교원 비율
4. 인프라 (Infra):  한국어학급 수용률

정규화: Min-Max Scaling (0~1)
집계: 가중합 (OECD 권장)

MEGI = 0.25 × demand + 0.30 × risk +
       0.20 × (1 - supply) + 0.25 × (1 - infra)

(risk, demand는 높을수록 격차 ↑
 supply, infra는 낮을수록 격차 ↑)
```

**데이터 흐름:**
```
교육통계(KEDI) + 학교알리미 + 학구도
   ↓
파이썬 워커 (월 1회 재계산)
   ↓
시도별 MEGI 점수 + 시계열
   ↓
교육청 대시보드 시각화
```

**참조:**
- [kevinwang09/learningtower](https://github.com/kevinwang09/learningtower) — OECD PISA ESCS 복합지표 R 구현
- OECD Handbook on Constructing Composite Indicators (2008)

---

### 4.8 Module H — TTS/STT Voice Module

**TTS (모국어 음성 재생):**
```
텍스트 (베트남어 번역)
  → Google Cloud TTS
  → 여성 음성 (친근함)
  → MP3 파일 Supabase Storage 저장
  → CDN 캐싱 (같은 텍스트 재요청 시)
  → 프론트 재생
```

**STT (학부모 음성 질문):**
```
학부모 음성 녹음
  → OpenAI Whisper (다국어)
  → 언어 자동 감지
  → 텍스트 변환
  → "교사에게 전달" or "FAQ 매칭"
```

**참조:**
- **Shellter** 기존 (Google Cloud TTS/STT)
- **BridgeCast** 기존 (Azure Speech)

---

### 4.9 Module I — Educational Terms Learning (학생 학습)

**목적**: 다문화 학생이 교육 용어를 재미있게 학습

**기능:**
- 교육 용어 250개 × 6개 언어 플래시카드
- 웹 MediaPipe 핸드트래킹 (설치 불필요)
- 4지선다 퀴즈 (제스처로 선택)
- AI 난이도 자동 조절
- 학습 진도 기록

**제스처 인터랙션:**
```
4개 답 영역 (화면 4분할)
  ↑
  │
  │  검지+중지 거리 <= 30px → 선택
  ↓
MediaPipe Hands (브라우저)
```

**참조:**
- [VirtualQuizGame-OpenCV](https://github.com/) — 원본 Python, 웹 포팅 필요
- MediaPipe Hands (Google)

---

### 4.10 Module J — KakaoTalk Bot (배포 채널)

**목적**: 한국 다문화 가정의 카카오톡 사용률 높으므로 접근성 확대

**기능 (사업자 등록 없이 MVP 가능):**
- 학교 코드 등록
- 당일 공지 요약 (대화형 질의 응답)
- 급식 경고 (할랄/알레르기) 질의
- 자연어 질문 응답
- **주의**: 자동 푸시 발송은 사업자 필수 (알림톡) — 파일럿 이후 확장

**참조:**
- [getsolaris/laravel-kakaobot](https://github.com/getsolaris/laravel-kakaobot) — 급식·학사 봇 구조

---

### 4.11 Module K — Teacher HITL Review (검토 UI)

**목적**: 저신뢰 번역을 교사가 검토해 품질 보장

**UI 구성:**
```
┌───────────────────────────────────────┐
│  검토 대기 (신뢰도 < 0.7)  [23건]       │
├───────────────────────────────────────┤
│ [원문]                                 │
│ "5월 15일 현장체험학습..."              │
│                                        │
│ [AI 번역 (베트남어)]                    │
│ "Ngày 15/5 đi học tập..."              │
│                                        │
│ [Groundedness: 0.62] ⚠️                │
│ 문제: "참가동의서" → 번역 누락           │
│                                        │
│ [ 승인 ]  [ 수정 ]  [ 거절 ]            │
└───────────────────────────────────────┘
```

**학습 루프:**
교사 수정 → 수정 패턴 수집 → 분기별 프롬프트 개선

---

## 5. Data Model (Full Schema)

### 5.1 Core Entities

```sql
-- ═══════════════════════════════════════════════════════════
-- USERS & AUTH
-- ═══════════════════════════════════════════════════════════

CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  role TEXT CHECK (role IN ('parent','student','teacher','district_admin')),
  name TEXT,
  phone TEXT,
  preferred_language TEXT NOT NULL CHECK (
    preferred_language IN ('ko','vi','zh','ru','mn','uz')
  ),
  origin_country TEXT,
  korean_proficiency TEXT CHECK (
    korean_proficiency IN ('A1','A2','B1','B2','C1','C2','native')
  ),
  dietary_restrictions JSONB DEFAULT '{}',
    -- {"halal": true, "allergens": [1,5,10], "vegetarian": false}
  accessibility_prefs JSONB DEFAULT '{}',
    -- {"font_size": "large", "tts_speed": 0.9, "high_contrast": true}
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE children (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES profiles(id),
  name TEXT,
  school_id UUID REFERENCES schools(id),
  grade INT,
  class_num INT,
  student_type TEXT CHECK (
    student_type IN ('domestic','mid_entry','foreign')
  ),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════════════════════════
-- SCHOOLS
-- ═══════════════════════════════════════════════════════════

CREATE TABLE schools (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  neis_edu_code TEXT NOT NULL,      -- ATPT_OFCDC_SC_CODE
  neis_school_code TEXT NOT NULL,   -- SD_SCHUL_CODE
  name TEXT NOT NULL,
  type TEXT CHECK (type IN ('elem','mid','high','special')),
  region TEXT,
  address TEXT,
  lat FLOAT, lng FLOAT,
  multicultural_ratio FLOAT,
  teacher_count INT,
  student_count INT,
  multicultural_teacher_count INT,
  korean_class_capacity INT,
  dropout_rate FLOAT,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (neis_edu_code, neis_school_code)
);

-- ═══════════════════════════════════════════════════════════
-- NOTICES & TRANSLATIONS
-- ═══════════════════════════════════════════════════════════

CREATE TABLE notices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID REFERENCES schools(id),
  source TEXT CHECK (
    source IN ('neis_schedule','neis_timetable','pdf_upload','teacher_manual')
  ),
  category TEXT,
  title TEXT,
  original_text TEXT NOT NULL,
  event_date DATE,
  event_end_date DATE,
  target_grades INT[] DEFAULT ARRAY[]::INT[],
  raw_data JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE translations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  notice_id UUID REFERENCES notices(id),
  language TEXT NOT NULL,
  translated_text TEXT NOT NULL,
  context_explanation TEXT,
  action_items JSONB,
  warnings JSONB,
  cultural_notes TEXT,
  confidence_score FLOAT,
  groundedness_score FLOAT,
  llm_model TEXT,
  prompt_version TEXT,
  review_status TEXT DEFAULT 'auto'
    CHECK (review_status IN ('auto','pending_review','approved','rejected')),
  reviewer_id UUID REFERENCES profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (notice_id, language)
);

CREATE TABLE tts_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  text_hash TEXT UNIQUE NOT NULL,
  language TEXT NOT NULL,
  audio_url TEXT NOT NULL,
  duration_ms INT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════════════════════════
-- MEALS & SAFETY
-- ═══════════════════════════════════════════════════════════

CREATE TABLE meals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID REFERENCES schools(id),
  meal_date DATE NOT NULL,
  meal_type TEXT CHECK (meal_type IN ('breakfast','lunch','dinner')),
  menu_items JSONB NOT NULL,
    -- [{"name":"돼지고기찌개","allergen_codes":[10,2],"is_halal":false}]
  calories INT,
  raw_data JSONB,
  UNIQUE (school_id, meal_date, meal_type)
);

CREATE TABLE meal_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  meal_id UUID REFERENCES meals(id),
  child_id UUID REFERENCES children(id),
  severity TEXT CHECK (severity IN ('info','warning','danger')),
  alert_type TEXT,  -- 'halal_violation', 'allergen_match'
  trigger_items TEXT[],
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════════════════════════
-- EDUCATIONAL TERMS (RAG)
-- ═══════════════════════════════════════════════════════════

CREATE TABLE education_terms (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  term_ko TEXT NOT NULL UNIQUE,
  category TEXT,  -- 학사/행사/행정/안전/수업
  translations JSONB NOT NULL,
    -- {"vi":{"word":"","pronunciation":"","usage":""},"zh":{...}}
  explanation JSONB,
    -- {"vi":"베트남어 문화 맥락 설명","zh":"..."}
  difficulty_level INT CHECK (difficulty_level BETWEEN 1 AND 5),
  frequency_rank INT,
  embedding VECTOR(1536),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE term_learning_progress (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id UUID REFERENCES profiles(id),
  term_id UUID REFERENCES education_terms(id),
  mastery_level INT CHECK (mastery_level BETWEEN 0 AND 100),
  attempts INT DEFAULT 0,
  correct_count INT DEFAULT 0,
  last_attempt_at TIMESTAMPTZ,
  UNIQUE (student_id, term_id)
);

-- ═══════════════════════════════════════════════════════════
-- DROPOUT PREDICTION
-- ═══════════════════════════════════════════════════════════

CREATE TABLE student_risk_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID REFERENCES children(id),
  risk_score FLOAT CHECK (risk_score BETWEEN 0 AND 100),
  features JSONB,             -- Feature 값들
  shap_values JSONB,           -- 설명 가능성
  model_version TEXT,
  predicted_at TIMESTAMPTZ DEFAULT NOW(),
  teacher_reviewed BOOLEAN DEFAULT FALSE,
  teacher_label TEXT           -- 'actual_risk','false_positive','monitor'
);

-- ═══════════════════════════════════════════════════════════
-- MEGI INDEX
-- ═══════════════════════════════════════════════════════════

CREATE TABLE megi_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  region_code TEXT NOT NULL,
  region_name TEXT NOT NULL,
  year INT NOT NULL,
  month INT,
  demand_score FLOAT,
  risk_score FLOAT,
  supply_score FLOAT,
  infra_score FLOAT,
  megi_total FLOAT,
  rank INT,
  UNIQUE (region_code, year, month)
);

-- ═══════════════════════════════════════════════════════════
-- RESPONSIBLE AI
-- ═══════════════════════════════════════════════════════════

CREATE TABLE ai_audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  translation_id UUID REFERENCES translations(id),
  check_type TEXT,  -- 'groundedness','pii','bias','hallucination'
  result JSONB,
  flagged BOOLEAN DEFAULT FALSE,
  action_taken TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE error_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES profiles(id),
  notice_id UUID REFERENCES notices(id),
  translation_id UUID REFERENCES translations(id),
  issue_type TEXT,
  description TEXT,
  language TEXT,
  status TEXT DEFAULT 'open',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE bias_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  report_period TEXT,  -- 'Q1-2026'
  language TEXT,
  metric TEXT,
  value FLOAT,
  baseline FLOAT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════════════════════════
-- ACTIVITY & EVENTS
-- ═══════════════════════════════════════════════════════════

CREATE TABLE user_events (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID REFERENCES profiles(id),
  event_type TEXT,  -- view, tts, question, action_done, etc.
  notice_id UUID REFERENCES notices(id),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_events_user_time ON user_events(user_id, created_at DESC);
```

### 5.2 주요 인덱스

```sql
CREATE INDEX idx_notices_school_date ON notices(school_id, event_date);
CREATE INDEX idx_notices_source ON notices(source);
CREATE INDEX idx_translations_notice_lang ON translations(notice_id, language);
CREATE INDEX idx_translations_review ON translations(review_status)
  WHERE review_status = 'pending_review';
CREATE INDEX idx_meals_school_date ON meals(school_id, meal_date);
CREATE INDEX idx_terms_embedding ON education_terms
  USING ivfflat (embedding vector_cosine_ops) WITH (lists=100);
CREATE INDEX idx_risk_scores_child ON student_risk_scores(child_id, predicted_at DESC);
CREATE INDEX idx_megi_region_year ON megi_scores(region_code, year, month);
```

### 5.3 Row Level Security 정책

```sql
-- 학부모: 자기 자녀의 학교 공지만
CREATE POLICY parent_notices ON notices FOR SELECT USING (
  school_id IN (
    SELECT school_id FROM children WHERE parent_id = auth.uid()
  )
);

-- 학생: 자기 학교 공지만
CREATE POLICY student_notices ON notices FOR SELECT USING (
  school_id = (SELECT school_id FROM profiles WHERE id = auth.uid())
);

-- 교사: 자기 학교 전체
CREATE POLICY teacher_all_notices ON notices FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid() AND role = 'teacher'
  )
);

-- 교육청: 전체 (정책 분석용)
CREATE POLICY district_analytics ON megi_scores FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid() AND role = 'district_admin'
  )
);
```

---

## 6. API 설계 (전체)

### 6.1 Parent API
```
GET    /api/v1/feed                   — 개인화 피드
GET    /api/v1/notices/:id            — 공지 상세
POST   /api/v1/notices/:id/tts        — 음성 생성
GET    /api/v1/meals/today            — 오늘 급식 + 경고
POST   /api/v1/action-items/:id/done  — 체크
POST   /api/v1/questions              — 학부모 질문 (STT 포함)
POST   /api/v1/errors                 — 오역 신고
GET    /api/v1/profile                — 프로필
PUT    /api/v1/profile                — 설정 변경
```

### 6.2 Student API
```
GET    /api/v1/student/terms/next     — 다음 학습 단어
POST   /api/v1/student/terms/answer   — 퀴즈 답변
GET    /api/v1/student/progress       — 학습 진도
```

### 6.3 Teacher API
```
GET    /api/v1/teacher/review-queue   — HITL 큐
POST   /api/v1/teacher/review/:id     — 검토 승인/거절
POST   /api/v1/teacher/notices        — 수동 공지 등록
POST   /api/v1/teacher/upload-pdf     — 가정통신문 업로드
GET    /api/v1/teacher/students/risk  — 위험 학생 리스트
```

### 6.4 District API
```
GET    /api/v1/district/megi          — MEGI 점수 전국
GET    /api/v1/district/megi/:region  — 지역별 상세
GET    /api/v1/district/trends        — 시계열 분석
GET    /api/v1/district/bias-report   — 공정성 리포트
```

### 6.5 Internal (Cron/Worker)
```
POST   /api/v1/internal/poll-neis     — NEIS 폴링 (05:00 KST)
POST   /api/v1/internal/compute-megi  — MEGI 재계산 (월 1회)
POST   /api/v1/internal/predict-risk  — 위험 예측 배치 (주 1회)
POST   /api/v1/internal/bias-audit    — 편향 감사 (분기 1회)
```

---

## 7. Responsible AI 완전 설계

### 7.1 5계층 환각 방지
```
Layer 1: 프롬프트 제약
Layer 2: 교육 용어 RAG (출처 기반)
Layer 3: Groundedness Check (Upstage)
Layer 4: Regex 정합성 (날짜/숫자)
Layer 5: HITL (score < 0.7)
```

### 7.2 PII 처리
```
입력 전 마스킹:
  실명 → [학생A], [학부모A]
  전화 → [전화1]
  주소 → [주소1]

출력 후 복원:
  로컬 매핑 테이블로 [학생A] → 실명
  LLM에는 절대 PII 전달 안 함
```

### 7.3 공정성 모니터링
- 언어별 BLEU/chrF 자동 측정
- 국가별 사용자 신고율
- 분기별 `bias_reports` 자동 생성
- 임계치 초과 시 알림

### 7.4 설명 가능성
- 번역: 출처 용어집 명시
- 예측: SHAP 값 시각화
- MEGI: 4차원 기여도 차트

### 7.5 감사 로그
- 모든 번역 → `ai_audit_log` 기록
- 30일 보관
- 분쟁 시 재현 가능

---

## 8. 접근성 (WCAG 2.1 AA 완전 준수)

### 8.1 Perceivable (지각 가능)
- 대체 텍스트 모든 이미지
- 명도 대비 4.5:1 이상
- 색상만으로 정보 전달 X
- 최소 글자 크기 14pt
- 큰 글씨 옵션 (3단계)

### 8.2 Operable (운용 가능)
- 키보드 전용 네비
- Skip navigation
- 포커스 표시
- 시간 제한 해제 가능
- 자동 재생 X

### 8.3 Understandable (이해 가능)
- 페이지별 언어 선언
- 일관된 네비게이션
- 오류 메시지 명확
- 레이블 + 지침

### 8.4 Robust (견고)
- 유효한 HTML
- ARIA 라벨
- 스크린 리더 테스트 (NVDA/VoiceOver)

### 8.5 추가 포용
- TTS 모국어 (문해력 낮은 학부모)
- 쉬운 한국어 버전
- 수어 영상 (중요 공지, Phase 3)
- 오프라인 모드 (저대역폭)

---

## 9. 보안

### 9.1 인증/인가
- Supabase Auth + MFA 옵션
- Role 기반 (parent/student/teacher/district_admin)
- Row Level Security 전 테이블 적용

### 9.2 API 보안
- Rate Limiting: 60 req/분
- API Key는 Edge Function 환경변수
- CORS: 자사 도메인만
- HTTPS 전용

### 9.3 데이터 보안
- PostgreSQL 저장 암호화
- 백업 7일 보존
- PII 마스킹 (LLM 전달 전)
- PIPA 준수 개인정보 처리방침
- GDPR 호환 설계 (해외 확장 대비)

### 9.4 비밀번호/세션
- bcrypt 12 rounds
- JWT 1시간 만료 + refresh token
- 세션 하이재킹 방지 (IP 바인딩 옵션)

---

## 10. 확장성

### 10.1 확장 로드맵

| Phase | 학교 | 동시사용자 | 인프라 | 비용/월 |
|---|---|---|---|---|
| MVP | 1 | 10 | 무료 티어 | $0 |
| 파일럿 | 10 | 500 | Supabase Pro | $25 |
| 시범 | 100 | 5,000 | Pro + CDN | $100 |
| 전국 | 11,800 | 200,000 | Enterprise | $2,000 |

### 10.2 비용 최적화

**LLM:**
- 캐싱: 동일 (notice, language) = DB만 조회
- Gemini Flash ~$0.075/1M tokens
- 일일 1,000 공지 × 6언어 × 500 tokens = $0.23/일

**TTS:**
- 사전 렌더링 + CDN 캐시
- 같은 텍스트 재요청 = 0 비용

**DB:**
- 30일 이후 공지 아카이브 (S3)
- 임베딩 압축 (int8 quantization)

### 10.3 성능 목표

| 지표 | MVP | 전국 운영 |
|---|---|---|
| API p95 응답 | < 300ms | < 200ms |
| 번역 지연 | < 3s | < 1s (캐시) |
| 가용성 | 99% | 99.9% |
| 동시 요청 | 10 | 1,000 |

---

## 11. 배포 & 운영

### 11.1 환경 분리
```
dev:     localhost + Supabase Local
staging: Vercel Preview + Supabase Staging
prod:    Vercel Prod + Supabase Prod
```

### 11.2 CI/CD (GitHub Actions)
```yaml
name: Deploy
on: [push]
jobs:
  test: ...
  lint: ...
  build: ...
  deploy-db: # Supabase migration
  deploy-functions: # supabase functions deploy
  deploy-frontend: # Vercel auto
```

### 11.3 모니터링

| 도구 | 목적 |
|---|---|
| Sentry | 런타임 에러 |
| Vercel Analytics | 프론트 성능 |
| PostHog | 사용자 행동 |
| Supabase Logs | API 로그 |
| 자체 대시보드 | 번역 품질, 편향 |

### 11.4 장애 대응

**SLO:** 99.5% uptime (월 3.6시간 허용)

**Runbook:**
- LLM API 다운 → 구글 번역 폴백
- NEIS 다운 → 캐시 데이터 서빙
- DB 다운 → 읽기 전용 모드

---

## 12. 타임라인 (3일 MVP + 3주 보강)

### Phase 1 - MVP (D-Day ~ D+3)

**Day 1: Foundation**
- Supabase 프로젝트 생성
- DB 스키마 마이그레이션
- NEIS API 키 발급
- Lovable 프론트 시작
- NEIS Edge Function (학교 정보 fetch)

**Day 2: AI Pipeline**
- Gemini API 연결
- 번역 프롬프트 v1
- Groundedness Check 통합
- Action Item 추출
- 급식 알레르기/할랄 감지
- Google TTS 연결 (베트남어)

**Day 3: UI + Deploy**
- 메인 피드 UI
- 공지 상세 UI
- TTS 플레이어
- 신뢰도 배지
- i18n 설정 (한/베)
- Vercel 배포
- 시연 시나리오 녹화

### Phase 2 - 보강 (D+4 ~ D+17)

**Week 1 (D+4 ~ D+10):**
- 나머지 5개 언어 i18n (중/러/몽/우즈)
- STT 통합
- Upstage Document Parser (가정통신문 PDF)
- HITL 교사 검토 UI
- 쉬운 한국어 버전
- 에러 리포팅 + 오디트

**Week 2 (D+11 ~ D+17):**
- 학업중단 예측 ML 파이프라인
- Python 워커 배포 (Modal)
- MEGI 지수 계산기
- 교육청 대시보드 (차트 + 지도)
- 편향 감사 자동화

### Phase 3 - 최종 (D+18 ~ D+30)

**Week 3 (D+18 ~ D+24):**
- 교육 용어 핸드트래킹 학습
- 카카오톡 봇 (기본)
- 성능 최적화
- 접근성 감사

**Week 4 (D+25 ~ D+30):**
- 기획서 15페이지 최종
- 시연 영상 편집
- 프롬프트 · 코드 문서화
- 제출 준비

### 제출 (D+30 = 5/31)
- 기획서 PDF 15페이지
- URL (실서비스 링크)
- 시연 영상 10분
- AI 활용 상세 (도구·프롬프트·산출물)
- 공공데이터 출처 명시

---

## 13. 시연 시나리오 (2차 심사용)

**페르소나: 팜티짠 (베트남, 35세, 초3 딸 어머니)**

### Scene 1: 온보딩 (30초)
- 베트남어 선택 → 학교 코드 입력 → 완료

### Scene 2: 피드 (20초)
- 오늘의 공지 3건
- 색상 경고 (빨강/노랑/초록)
- 신뢰도 배지 표시

### Scene 3: 공지 상세 (45초)
- 베트남어 번역 + 교육 용어 해설
- Action Items 체크리스트
- 모국어 TTS 재생
- 신뢰도 94%
- [한국어 원문] 토글

### Scene 4: 급식 경고 (30초)
- "오늘 급식에 돼지고기"
- 도시락 준비 권고
- 대체 메뉴 추천

### Scene 5: 교사 HITL (30초)
- 낮은 신뢰도 번역 검토
- 교사 수정 → 승인

### Scene 6: 교육청 MEGI (30초)
- 전국 지도 시각화
- 서울 강남 vs 경기 시흥 격차
- 시계열 추이

### Scene 7: Responsible AI (20초)
- 편향 리포트 대시보드
- 국가별 번역 품질

### Scene 8: 학업중단 예측 (25초)
- 위험 학생 리스트
- SHAP 값 설명

**총 시연: 약 4분 30초**

---

## 14. GitHub 레퍼런스 완전 목록 (31개)

### 14.1 NEIS API 래퍼 (6개)

| Repo | 언어 | 상태 | 활용 |
|---|---|---|---|
| [neisapi-ng](https://github.com/) | Python | 활성 | 학교정보/급식/학사 — 기본 레퍼런스 |
| [my-school-info/neis-api](https://github.com/my-school-info/neis-api) | Node.js/TS | 활성 | TS/Node 백엔드 |
| [agemor/neis-api](https://github.com/agemor/neis-api) | Kotlin (72★) | 2025.10 활성 | Android 클라이언트 |
| [star0202/neis.ts](https://github.com/star0202/neis.ts) | TypeScript | 2025 활성 | 최신 TS 구현 |
| [kimcore/neis.kt](https://github.com/kimcore/neis.kt) | Kotlin + Ktor | 활성 | 타입 강제 DTO |
| [SaidBySolo/neispy](https://github.com/SaidBySolo/neispy) | Python | 활성 | Sync+Async 모두 |

### 14.2 한국 다문화 교육 유사 프로젝트 (2개, 경쟁 분석 필수)

| Repo | 설명 | 알:음 차별화 |
|---|---|---|
| [kusitms-com/29th_Meetup_TeamA_SchoolPoint_Front](https://github.com/kusitms-com/29th_Meetup_TeamA_SchoolPoint_Front) | "스쿨포인트" — AI 통합 알림장 + 다문화 번역 계획 | **MEGI + 할랄/알레르기 + 학업중단 예측** 걔들 없음 |
| [kookmin-sw/capstone-2024-30](https://github.com/kookmin-sw/capstone-2024-30) | "KuKu" — 외국인 학생 RAG+LLM Q&A | **NEIS 실시간 + 학부모 타겟** 걔들 없음 (학생 Q&A만) |

### 14.3 한국어 LLM / 번역 (4개)

| Repo | 설명 | 활용 |
|---|---|---|
| [Beomi/KoAlpaca](https://github.com/Beomi/KoAlpaca) | 한국어 instruction LLM | Action Item 추출 fine-tune 옵션 |
| [jwj7140/Gugugo](https://github.com/jwj7140/Gugugo) | 한국어 OSS 번역 (EN↔KO) | 로컬 폴백 |
| [vEduardovich/dodari_nllb](https://github.com/vEduardovich/dodari_nllb) | Facebook NLLB 로컬 실행 UI | **6개 언어 전부 커버** — 로컬 번역 폴백 |
| [NomaDamas/awesome-korean-llm](https://github.com/NomaDamas/awesome-korean-llm) | 한국어 LLM 큐레이션 | 모델 선택 shortcut |

### 14.4 교육 AI 챗봇 / RAG (4개)

| Repo | 설명 | 활용 |
|---|---|---|
| [lalah-chatbot](https://github.com/) | 아프간 학생용 다국어 교육 챗봇 | Multi-LLM + 문화 민감성 프레임 |
| [Multi-language-Course-Content-Translator-Agent](https://github.com/) | IBM Watsonx RAG 번역 | 용어집 기반 환각 방지 |
| [amrutha0001/Sahayak](https://github.com/amrutha0001/Sahayak_Indic_Muliti-lingual_Chatbot) | 인도 11개 언어 복지 RAG | 공공 서비스 RAG 참조 |
| [Saptarshiii/Multilingual-RAG-ChatBot](https://github.com/Saptarshiii/Multilingual-RAG-ChatBot) | Whisper + 99언어 RAG | STT + 다국어 RAG 구조 |

### 14.5 할랄 / 알레르기 감지 (4개)

| Repo | 설명 | 활용 |
|---|---|---|
| [AI_Halal_Product_Detector](https://github.com/) | YOLOv8 + OCR + DB (기존) | 다중 신호 투표 결정 엔진 |
| [UsmanAsad87/Halal_Food_Detector](https://github.com/UsmanAsad87/Halal_Food_Detector) | Android OCR + E-code | **E-code 매핑 DB 재사용** |
| [kevinkchen1/SafeGrub](https://github.com/kevinkchen1/SafeGrub) | 성분 파싱 알레르기 | Taxonomy 참고 |
| [AllergySavvy/ML-AllergySavvy-Dev](https://github.com/AllergySavvy/ML-AllergySavvy-Dev) | ML 알레르기 분류 | 정확도 보강 |

### 14.6 학업중단 예측 (4개)

| Repo | 설명 | 활용 |
|---|---|---|
| [dropout-prediction](https://github.com/) (기존) | Decision Tree 기본 예측 | sklearn 파이프라인 기반 |
| [dssg/student-early-warning](https://github.com/dssg/student-early-warning) | DSSG 형평성 중심 | **대표 레퍼런스 — 문서 풍부** |
| [ismailelbouknify/Student-At-Risk-Identification](https://github.com/ismailelbouknify/Student-At-Risk-Identification) | 모로코 소수자 멀티태스크 | 다문화 프레임 가장 유사 |
| [Vivekchary2607/EduPredict](https://github.com/Vivekchary2607/AI-Powered-EduPredict-Student-Performance-Analytics-System) | Streamlit 대시보드 | 위험 surfacing UI 패턴 |

### 14.7 Action Item / Email Agent (3개)

| Repo | 설명 | 활용 |
|---|---|---|
| [robocorp/example-llm-emails](https://github.com/robocorp/example-llm-emails) | GPT-4 이메일→태스크 | **가정통신문 Action Item 추출 직접 적용** |
| [deshwalmahesh/outlook-email-agent](https://github.com/deshwalmahesh/outlook-email-agent) | 분류 + 회신 생성 | 교사 자동 답변 초안 |
| [Shubhamsaboo/awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps) | LLM 앱 큐레이션 | 에이전트 패턴 참고 |

### 14.8 학부모-교사 소통 (3개)

| Repo | 설명 | 활용 |
|---|---|---|
| [axyjo/ptscheduler](https://github.com/axyjo/ptscheduler) | 부모-교사 상담 스케줄링 | 플러그인 확장 구조 |
| [cardosakv/Faculti-v2](https://github.com/cardosakv/Faculti-v2) | 학부모-교사 플랫폼 | UI 패턴 |
| [Bennacci/eCommunicationBook](https://github.com/Bennacci/eCommunicationBook) | 디지털 알림장 | **데이터 모델 재사용** |

### 14.9 교육 플랫폼 (2개)

| Repo | 설명 | 활용 |
|---|---|---|
| [Shafihu/InSnip](https://github.com/Shafihu/InSnip) | 다국어 학습 플랫폼 | i18n + 저대역폭 패턴 |
| [classroomio/classroomio](https://github.com/classroomio/classroomio) | OSS Moodle 대안 | LMS 스캐폴딩 참고 |

### 14.10 핸드트래킹 & UI (1개)

| Repo | 설명 | 활용 |
|---|---|---|
| [VirtualQuizGame-OpenCV](https://github.com/) | MediaPipe 핸드 퀴즈 | **웹 포팅** (TensorFlow.js) |

### 14.11 KakaoTalk 채널 (1개)

| Repo | 설명 | 활용 |
|---|---|---|
| [getsolaris/laravel-kakaobot](https://github.com/getsolaris/laravel-kakaobot) | 급식·학사 봇 | **한국 학부모 배포 채널** |

### 14.12 교육 지수 방법론 (1개)

| Repo | 설명 | 활용 |
|---|---|---|
| [kevinwang09/learningtower](https://github.com/kevinwang09/learningtower) | OECD PISA ESCS | **MEGI 복합 지수 설계 근거** |

---

## 15. 학술 근거 (7개 논문)

| 논문 | 출처 | 알:음 근거 |
|---|---|---|
| AI in Multicultural Dialogue Between Teachers and Parents | Springer 2025 | 학부모-교사 AI 소통 효과 |
| Inclusive Education with AI | arXiv 2504.14120 | 언어 장벽 + 특수교육 동시 해결 |
| Multilingual RAG for Knowledge-Intensive Tasks | arXiv 2504.03616 | 다국어 RAG 성능 +3.6~4.4% |
| RAGtrans: Retrieval-Augmented MT | EMNLP 2025 | BLEU +1.6~3.1 향상 |
| Early Prediction of Student Dropout using ML | EDM 2024 | immigrant 변수 활용 선행 연구 |
| Student Dropout Prediction | Nature Sci. Reports 2025 | 로그 기반 예측 |
| Using AI to Support Emergent Bilingual Students | Language Magazine 2024 | 교실 포용성 향상 |

---

## 16. 독창성 증명 (GitHub 검색 결과)

| 검색어 | 결과 | 의미 |
|---|---|---|
| "NEIS + 다국어 번역 + 실시간" | **0건** | 기술 조합 전례 없음 |
| "핸드트래킹 + 교육 용어 학습" | **0건** | 웹 기반 교육 용어 학습 없음 |
| "다문화 + AI + 한국 교육 + MEGI" | **0건** | 정책 지수 최초 |
| "RAG + 번역 + 교육 맥락" | 6건 | 존재하지만 한국 교육 특화 없음 |
| "학생 중퇴 예측 ML" | 271건 | 활발하지만 다문화 특화 거의 없음 |
| "교육 Action Item 추출" | 0건 | Action Extraction은 비즈니스 이메일에만 |

→ **알:음 = 위 6개 축을 "한국 다문화 교육"이라는 맥락으로 통합한 유일한 시스템.**

---

## 부록 A — NEIS API 엔드포인트 상세

| 엔드포인트 | 용도 | 응답 필드 |
|---|---|---|
| `/schoolInfo` | 학교 정보 | 학교명, 주소, 유형, 교원수 |
| `/mealServiceDietInfo` | 급식 | 일자, 메뉴, 알레르기 코드, 칼로리 |
| `/SchoolSchedule` | 학사 | 일정명, 일자, 내용 |
| `/elsTimetable` / `/misTimetable` / `/hisTimetable` | 시간표 | 교시, 과목 |
| `/classInfo` | 반 정보 | 학년, 반, 담임 |

---

## 부록 B — 환경 변수

```bash
# NEIS
NEIS_API_KEY=

# LLM
GEMINI_API_KEY=
OPENAI_API_KEY=       # Whisper STT
UPSTAGE_API_KEY=       # Groundedness + Document Parser

# Google Cloud
GOOGLE_CLOUD_TTS_KEY=

# Supabase
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Modal (ML Worker)
MODAL_TOKEN_ID=
MODAL_TOKEN_SECRET=

# KakaoTalk
KAKAO_REST_API_KEY=

# Monitoring
SENTRY_DSN=
POSTHOG_KEY=
```

---

## 부록 C — 초기 교육 용어 시드 (50개)

**학사 (Academic)**
현장체험학습, 수행평가, 중간고사, 기말고사, 학부모상담, 학부모총회, 운영위원회, 학습지

**행사 (Events)**
운동회, 학예회, 졸업식, 입학식, 수학여행, 수련회, 체육대회, 소풍

**행정 (Administrative)**
가정통신문, 동의서, 확인서, 서약서, 급식비, 체험학습비, 방과후학교, 돌봄교실

**안전 (Safety)**
안전교육, 비상대피훈련, 학교폭력, 생활지도, 학생상담, 보건실, 양호교사

**수업 (Class)**
시간표, 교과서, 준비물, 숙제, 알림장, 독서록, 일기장, 독후감

**진로 (Career)**
진로상담, 진로체험, 직업탐색, 적성검사, 입학설명회

**기타**
학부모 참여, 자원봉사, 학교 홈페이지, 학생증, 생활기록부

---

## 부록 D — 데이터 라이선스

| 데이터 | 라이선스 | 상업 이용 | 고지 |
|---|---|---|---|
| NEIS 학교기본 | 공공누리 1유형 | O | 출처 명시 |
| NEIS 급식 | 공공누리 1유형 | O | 출처 명시 |
| NEIS 학사 | 공공누리 1유형 | O | 출처 명시 |
| KEDI 통계 | 공공누리 1유형 | O | 출처 명시 |
| 학교알리미 | 공공누리 1유형 | O | 출처 명시 |
| 학구도 | 공공누리 1유형 | O | 출처 명시 |

---

## 부록 E — 프롬프트 버전 관리

**구조:**
```
prompts/
├── translate/
│   ├── v1.0.yaml   (초기 MVP)
│   ├── v1.1.yaml   (용어집 강화)
│   └── v1.2.yaml   (현재)
├── action_extract/
│   ├── v1.0.yaml
│   └── v1.1.yaml
├── safety_check/
│   └── v1.0.yaml
└── dropout_explain/
    └── v1.0.yaml
```

각 YAML:
```yaml
name: translate
version: 1.2.0
model: gemini-1.5-flash
temperature: 0.3
max_tokens: 2000
system: |
  ...
user_template: |
  ...
examples:
  - input: ...
    output: ...
evaluation:
  bleu: 0.42
  user_report_rate: 0.03
```

---

**문서 버전**: v2.0 (Full Implementation)
**작성일**: 2026-04-16
