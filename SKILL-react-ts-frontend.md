# SKILL: React + TypeScript 프론트엔드 개발

## 이 스킬을 사용하는 경우
- React 컴포넌트 신규 생성
- API 연동 (React Query)
- 커스텀 훅 작성
- 전역 상태 설계 (Zustand)
- 폼 처리 (React Hook Form)
- 라우팅 구조 설계

---

## 1단계: 구조 파악 먼저

```
src/
├── api/
│   ├── axios.ts          # axios 인스턴스 및 interceptor
│   └── user.api.ts       # 도메인별 API 함수
├── components/
│   ├── common/           # Button, Input, Modal 등 공용 컴포넌트
│   └── {domain}/         # 도메인별 컴포넌트
├── hooks/
│   └── useUser.ts        # 커스텀 훅
├── pages/
│   └── UserPage.tsx      # 라우트 단위 페이지
├── stores/
│   └── authStore.ts      # Zustand 전역 상태
├── types/
│   └── user.types.ts     # TypeScript 타입 정의
└── utils/
    └── format.ts         # 유틸리티 함수
```

기존 컴포넌트 하나를 먼저 읽고 패턴을 파악한 뒤 동일하게 작성한다.

---

## 2단계: 타입 정의

```typescript
// types/user.types.ts

// API 요청 타입
export interface CreateUserRequest {
  email: string;
  password: string;
}

// API 응답 타입 (백엔드 ApiResponse<T>에 대응)
export interface ApiResponse<T> {
  success: boolean;
  data: T;
  message: string | null;
}

// 도메인 모델 타입
export interface User {
  id: number;
  email: string;
  createdAt: string; // ISO 8601
}

// Enum 타입
export type UserRole = 'ADMIN' | 'USER' | 'GUEST';

// Pagination 타입
export interface PageResponse<T> {
  content: T[];
  totalElements: number;
  totalPages: number;
  number: number; // 현재 페이지 (0-indexed)
  size: number;
}
```

---

## 3단계: API 레이어 설계

### axios 인스턴스 설정
```typescript
// api/axios.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
});

// 요청 interceptor: 토큰 자동 첨부
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('accessToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 응답 interceptor: 공통 에러 처리
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // 토큰 만료 → 로그인 페이지로 이동
      localStorage.removeItem('accessToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### 도메인별 API 함수
```typescript
// api/user.api.ts
import apiClient from './axios';
import type { ApiResponse, User, CreateUserRequest } from '@/types/user.types';

export const userApi = {
  getUser: async (userId: number): Promise<User> => {
    const { data } = await apiClient.get<ApiResponse<User>>(`/api/v1/users/${userId}`);
    return data.data;
  },

  createUser: async (request: CreateUserRequest): Promise<User> => {
    const { data } = await apiClient.post<ApiResponse<User>>('/api/v1/users', request);
    return data.data;
  },

  deleteUser: async (userId: number): Promise<void> => {
    await apiClient.delete(`/api/v1/users/${userId}`);
  },
};
```

---

## 4단계: 커스텀 훅 (React Query 연동)

```typescript
// hooks/useUser.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { userApi } from '@/api/user.api';
import type { CreateUserRequest } from '@/types/user.types';

// Query Key 상수로 관리
export const userKeys = {
  all: ['users'] as const,
  detail: (id: number) => [...userKeys.all, id] as const,
};

// 단일 유저 조회
export function useUser(userId: number) {
  return useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => userApi.getUser(userId),
    enabled: !!userId, // userId가 있을 때만 실행
    staleTime: 1000 * 60 * 5, // 5분 캐싱
  });
}

// 유저 생성 mutation
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: CreateUserRequest) => userApi.createUser(request),
    onSuccess: () => {
      // 성공 시 유저 목록 캐시 무효화
      queryClient.invalidateQueries({ queryKey: userKeys.all });
    },
    onError: (error) => {
      console.error('유저 생성 실패:', error);
    },
  });
}
```

---

## 5단계: 컴포넌트 작성

### 기본 컴포넌트 구조
```tsx
// components/user/UserCard.tsx

// 1. 타입 정의 (파일 상단)
interface UserCardProps {
  userId: number;
  onDelete?: (userId: number) => void;
}

// 2. 컴포넌트 (named export 권장)
export function UserCard({ userId, onDelete }: UserCardProps) {
  // 3. 훅 호출
  const { data: user, isLoading, isError } = useUser(userId);

  // 4. 로딩/에러 처리
  if (isLoading) return <Skeleton />;
  if (isError || !user) return <ErrorMessage message="유저를 불러올 수 없습니다." />;

  // 5. 이벤트 핸들러 (컴포넌트 내부에 정의)
  const handleDelete = () => {
    onDelete?.(userId);
  };

  // 6. 렌더링
  return (
    <div className="rounded-lg border p-4">
      <p className="font-semibold">{user.email}</p>
      <p className="text-sm text-gray-500">{formatDate(user.createdAt)}</p>
      {onDelete && (
        <button onClick={handleDelete} className="mt-2 text-red-500">
          삭제
        </button>
      )}
    </div>
  );
}
```

---

## 6단계: 폼 처리 (React Hook Form)

```tsx
// components/user/CreateUserForm.tsx
import { useForm } from 'react-hook-form';
import { useCreateUser } from '@/hooks/useUser';
import type { CreateUserRequest } from '@/types/user.types';

export function CreateUserForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
  } = useForm<CreateUserRequest>();

  const { mutateAsync: createUser } = useCreateUser();

  const onSubmit = async (data: CreateUserRequest) => {
    try {
      await createUser(data);
      reset();
      alert('생성 완료!');
    } catch (error) {
      alert('생성 실패. 다시 시도해주세요.');
    }
  };

  return (
    // form 태그 사용 (React Hook Form에서는 form 태그 OK)
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <input
          {...register('email', {
            required: '이메일을 입력해주세요',
            pattern: { value: /\S+@\S+\.\S+/, message: '올바른 이메일 형식이 아닙니다' },
          })}
          placeholder="이메일"
          className="w-full border rounded px-3 py-2"
        />
        {errors.email && <p className="text-red-500 text-sm">{errors.email.message}</p>}
      </div>

      <div>
        <input
          {...register('password', {
            required: '비밀번호를 입력해주세요',
            minLength: { value: 8, message: '8자 이상 입력해주세요' },
          })}
          type="password"
          placeholder="비밀번호"
          className="w-full border rounded px-3 py-2"
        />
        {errors.password && <p className="text-red-500 text-sm">{errors.password.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting} className="w-full bg-blue-500 text-white rounded py-2">
        {isSubmitting ? '처리 중...' : '가입하기'}
      </button>
    </form>
  );
}
```

---

## 7단계: 전역 상태 (Zustand)

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware'; // 새로고침 후에도 유지

interface AuthState {
  accessToken: string | null;
  user: { id: number; email: string } | null;
  isAuthenticated: boolean;
  // 액션
  login: (token: string, user: { id: number; email: string }) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      accessToken: null,
      user: null,
      isAuthenticated: false,

      login: (token, user) =>
        set({ accessToken: token, user, isAuthenticated: true }),

      logout: () =>
        set({ accessToken: null, user: null, isAuthenticated: false }),
    }),
    {
      name: 'auth-storage', // localStorage key
      partialize: (state) => ({ accessToken: state.accessToken }), // 토큰만 저장
    }
  )
);
```

---

## 8단계: 라우팅

```tsx
// App.tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { useAuthStore } from '@/stores/authStore';

// Protected Route 컴포넌트
function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated);
  return isAuthenticated ? <>{children}</> : <Navigate to="/login" replace />;
}

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<LoginPage />} />
        <Route path="/" element={<ProtectedRoute><Layout /></ProtectedRoute>}>
          <Route index element={<HomePage />} />
          <Route path="users" element={<UserListPage />} />
          <Route path="users/:id" element={<UserDetailPage />} />
        </Route>
        <Route path="*" element={<NotFoundPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

---

## 환경변수 설정

```bash
# .env.development
VITE_API_BASE_URL=http://localhost:8080

# .env.production
VITE_API_BASE_URL=https://api.example.com
```

```typescript
// 타입 안전한 env 접근
const apiUrl = import.meta.env.VITE_API_BASE_URL as string;
```

---

## 체크리스트 (PR 전 확인)

- [ ] `any` 타입 없음
- [ ] `useEffect` 안에서 직접 fetch 없음 (React Query 사용)
- [ ] API 호출 함수가 `api/` 폴더에 위치
- [ ] 로딩/에러 상태 처리 있음
- [ ] Props 타입이 interface로 정의됨
- [ ] 비즈니스 로직이 커스텀 훅으로 분리됨
- [ ] 환경변수가 `.env` 파일에 정의됨 (하드코딩 없음)
- [ ] 컴포넌트가 200줄 이하 (초과 시 분리 검토)
