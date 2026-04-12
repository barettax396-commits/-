# CLAUDE.md — 프로젝트 전체 룰

## 프로젝트 스택
- **백엔드**: Java 17+, Spring Boot 3.x, JPA/Hibernate, MySQL or PostgreSQL
- **프론트엔드**: React 18+, TypeScript 5+, Vite, Tailwind CSS (또는 CSS Modules)
- **API 통신**: RESTful API (JSON), 필요 시 WebSocket

---

## 공통 원칙

- 코드를 작성하기 전에 항상 기존 코드 구조와 패턴을 먼저 파악한다
- 새 파일을 만들기 전에 유사한 파일이 이미 있는지 확인한다
- 라이브러리나 패턴을 추가할 때 기존 프로젝트에서 사용하는 것을 우선한다
- 추측으로 코드를 작성하지 말고, 불확실한 부분은 명확히 질문한다

---

## 백엔드 룰 (Java / Spring Boot)

### 아키텍처
- **레이어 구조** 엄수: Controller → Service → Repository
- Controller는 요청/응답 변환만 담당 (비즈니스 로직 금지)
- Service는 트랜잭션 단위로 설계
- 도메인 로직은 Entity 또는 Domain Service에 위치

### 코딩 컨벤션
- 클래스명: `PascalCase`
- 메서드/변수명: `camelCase`
- 상수: `UPPER_SNAKE_CASE`
- 패키지명: 모두 소문자, 단수형 (`controller`, `service`, `repository`)

### 필수 규칙
- `@Transactional`은 Service 레이어에서만 사용
- Entity를 Controller에서 직접 반환 금지 → 반드시 DTO로 변환
- `Optional` 사용 시 `.get()` 직접 호출 금지 → `.orElseThrow()` 사용
- `NullPointerException` 방지를 위해 null 체크 또는 `Optional` 활용
- 예외는 커스텀 Exception 클래스로 명확히 정의
- 모든 API 응답은 공통 응답 포맷(`ApiResponse<T>`) 사용

### 금지 사항
- `System.out.println()` 사용 금지 → SLF4J Logger 사용
- 비밀번호, API Key 등 민감 정보 하드코딩 금지 → `application.yml` 또는 환경변수
- 단일 메서드에 50줄 초과 금지 (초과 시 분리)
- 와일드카드 import (`import com.example.*`) 금지

### 테스트
- Service 레이어는 단위 테스트 필수
- Repository 레이어는 `@DataJpaTest`로 검증
- 테스트 메서드명: `메서드명_상황_기대결과` 형식

---

## 프론트엔드 룰 (React / TypeScript)

### TypeScript
- `any` 타입 사용 절대 금지
- 모든 함수의 파라미터와 반환 타입을 명시
- API 응답 타입은 반드시 interface 또는 type으로 정의
- `as` 타입 단언(assertion)은 최소화, 불가피한 경우 주석 필수

### React 컴포넌트
- 함수형 컴포넌트만 사용 (Class 컴포넌트 금지)
- 컴포넌트 파일명: `PascalCase.tsx`
- 컴포넌트당 하나의 파일 원칙
- Props 타입은 `interface`로 파일 상단에 정의
- 컴포넌트는 200줄 초과 시 분리를 검토

### 상태 관리
- 서버 상태: React Query (TanStack Query) 사용
- 전역 클라이언트 상태: Zustand 또는 Context API
- 단순 로컬 상태: `useState`로 충분히 처리

### 폴더 구조
```
src/
├── api/          # API 호출 함수 (axios 인스턴스, endpoint 별 함수)
├── components/   # 공용 컴포넌트
├── hooks/        # 커스텀 훅
├── pages/        # 페이지 컴포넌트 (라우트 단위)
├── stores/       # Zustand 전역 상태
├── types/        # TypeScript 타입 정의
└── utils/        # 유틸리티 함수
```

### 금지 사항
- `useEffect` 안에서 직접 API 호출 금지 → React Query 사용
- Props Drilling 3단계 초과 금지 → Context 또는 전역 상태 사용
- `index.ts` 배럴 파일 남용 금지 (순환 참조 위험)
- 인라인 스타일(`style={{}}`) 남용 금지 → CSS 클래스 또는 Tailwind 사용
- 비즈니스 로직을 컴포넌트 안에 직접 작성 금지 → 커스텀 훅으로 분리

---

## API 연동 룰

- 모든 API URL은 `src/api/` 에서 중앙 관리
- API 에러 처리는 axios interceptor에서 공통 처리
- HTTP 상태 코드 의미를 정확히 사용
  - `200 OK`: 조회 성공
  - `201 Created`: 생성 성공
  - `204 No Content`: 삭제 성공
  - `400 Bad Request`: 클라이언트 입력 오류
  - `401 Unauthorized`: 인증 필요
  - `403 Forbidden`: 권한 없음
  - `404 Not Found`: 리소스 없음
  - `500 Internal Server Error`: 서버 오류

---

## Git 커밋 컨벤션

```
feat: 새 기능 추가
fix: 버그 수정
refactor: 코드 리팩토링 (기능 변화 없음)
style: 코드 포맷팅 (로직 변화 없음)
test: 테스트 코드 추가/수정
docs: 문서 수정
chore: 빌드 설정, 패키지 업데이트
```

예시: `feat: 회원가입 API 구현`

---

## 스킬 파일 참고
- Java 백엔드 개발: `SKILL-java-backend.md`
- React + TypeScript 프론트엔드 개발: `SKILL-react-ts-frontend.md`
