# MemoDoc - 개발 일지 (DEVLOG)

---

## 2026-03-13 | 주제 선정

- 아이디어 확정: 업무 메모 → 지식 문서 자동 생성
- 핵심 가치: 메모 → 즉시 공유 가능한 문서로 변환
- 서버 없이 브라우저 단독 실행 방식 채택

---

## 2026-03-13 | 문서 작성

- PRD, ROADMAP, CLAUDE, DEVLOG, TEST_PLAN 작성
- 6탭 구조 확정: 요약 / 개념 정리 / 구조화 문서 / 도식화 / 단계별 설명 / 설계 활용
- `hello-world/` 폴더로 기존 index.html 이동

---

## 2026-03-13 | 설계

- 2패널 레이아웃 (입력 | 출력 탭)
- 스트리밍 방식으로 6탭 순차 생성 (rate limit 대응)
- Mermaid.js로 다이어그램 탭 렌더링

---

## 2026-03-13 | 구현

- `hackathon.html` / `hackathon.css` / `hackathon.js` 구현 완료
- OpenRouter API 연동 (OpenAI 호환 형식, SSE 스트리밍)
- 다크모드 토글 및 OS 설정 자동 감지
- 마크다운 내보내기 (탭별 / 전체)
- 프리텐다드 폰트 적용

---

## 2026-03-13 | 트러블슈팅

### 🔴 문제 1: Gemini API 할당량 초과
**현상:** Gemini API 무료 티어 호출 한도 초과로 전체 기능 동작 불가
**원인:** 무료 티어 일일 할당량 소진
**해결:** OpenRouter로 마이그레이션. 브라우저 직접 호출 가능(CORS 허용) + 무료 모델 제공 + SSE 스트리밍 지원으로 적합한 대안 확인
**변경:** 엔드포인트 / 인증 방식(Bearer) / 요청 형식(OpenAI 호환) / 스트리밍 파싱(`choices[0].delta.content`) 전면 수정

---

### 🔴 문제 2: 기존 Gemini API 키 자동 로드 오류
**현상:** 마이그레이션 후 `Missing Authentication header` 오류 발생
**원인:** localStorage에 저장된 이전 Gemini 키(`AIza...`)가 그대로 로드되어 OpenRouter로 전송됨
**해결:** `loadApiKey()` 에서 `sk-or-` 접두사 검증 추가. 불일치 시 자동 삭제 후 재입력 유도

---

### 🔴 문제 3: 무료 모델 엔드포인트 오류 연속 발생
**현상:** `No endpoints found` / `Provider returned error` 오류 반복
**원인:** OpenRouter 무료 모델은 서버 상태에 따라 가용성이 유동적. 특정 모델이 일시적으로 비활성화되거나 upstream provider rate limit에 걸림
**시도한 모델:** `gemini-2.0-flash-exp:free` → `gemini-2.5-pro-exp:free` → `gemini-2.0-flash-thinking-exp:free` → `deepseek-chat-v3:free` → `llama-3.3-70b:free` → `qwen/qwq-32b:free` → `glm-4.5-air:free` → `stepfun/step-3.5-flash:free` (채택)
**해결:** 실시간 가용 모델을 OpenRouter 대시보드에서 직접 확인 후 교체. 모델 ID를 상수(`OPENROUTER_MODEL`)로 분리해 교체 용이하게 설계한 것이 주효

---

### 🔴 문제 4: 6탭 병렬 요청 시 Rate Limit 오류
**현상:** 6개 탭 동시 생성 시 2번째 탭부터 `rate-limited upstream` 오류
**원인:** `Promise.all()`로 6개 요청을 동시 발송 → 무료 모델의 분당 요청 제한 초과
**해결:** `Promise.all` → `for...of` 순차 루프로 변경, 탭 간 3초 딜레이 삽입
**트레이드오프:** 전체 생성 시간 증가(약 30~40초). 무료 모델 사용의 근본적 한계로 판단, 향후 유료 전환 시 병렬 복원 예정

---

### 🟡 문제 5: Mermaid 다이어그램 `renderId is not defined` 오류
**현상:** 도식화 탭에서 다이어그램 렌더링 후 콘솔 에러 발생
**원인:** `renderId`를 첫 번째 `if (mermaidCode)` 블록 안에서 `const`로 선언했으나, 두 번째 `if` 블록에서 참조 → 블록 스코프 밖 접근 불가
**해결:** `renderId` 선언을 두 `if` 블록 밖으로 이동

---

### 🟡 문제 6: 프롬프트가 메모 유형 무관하게 회의 형식 고정 출력
**현상:** 교육/학습 메모를 입력해도 "결정 사항", "다음 할 일" 섹션이 반복 출력
**원인:** 무료 소형 모델이 조건부 섹션 생략 지시를 잘 따르지 않음
**해결:** AI 판단에 의존하지 않고 사용자가 직접 메모 유형(회의/교육/아이디어)을 선택하는 UI 추가. 유형별로 별도 프롬프트를 분기하여 적용

---

## 2026-03-13 | UX 개선

- 탭별 로딩 상태 표시: 대기 중(⏳) → 생성 중(spinner) → 완료
- 생성 버튼 진행률 표시: `생성 중... (1/6)`
- 오류 탭에 🔄 재시도 버튼 추가 (탭 단위 재생성)
- 입력 검증 강화: 빈 메모 / 10자 미만 / 잘못된 API 키 형식

---

## 테스트

- [x] 6탭 스트리밍 정상 동작 확인
- [x] Mermaid 다이어그램 렌더링 확인
- [x] 메모 유형별 요약 형식 분기 확인
- [x] 오류 시 재시도 버튼 동작 확인
- [ ] 내보내기 파일 Notion/GitHub 적용 확인
- [ ] 엣지 케이스: 긴 메모(5,000자 이상) / 한영 혼용
