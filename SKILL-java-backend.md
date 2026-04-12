# SKILL: Java 백엔드 개발 (Spring Boot)

## 이 스킬을 사용하는 경우
- Spring Boot API 엔드포인트 생성
- JPA Entity / Repository 설계
- Service 레이어 비즈니스 로직 구현
- 예외 처리 구조 설계
- Spring Security 인증/인가 설정

---

## 1단계: 구조 파악 먼저

새 기능을 구현하기 전에 다음을 확인한다:
```bash
# 기존 패키지 구조 파악
src/main/java/com/{project}/
├── controller/
├── service/
├── repository/
├── domain/        # 또는 entity/
├── dto/
├── exception/
└── config/
```

기존 컨트롤러/서비스 하나를 먼저 읽고, 패턴을 동일하게 따른다.

---

## 2단계: Entity 설계

```java
@Entity
@Table(name = "users")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED) // JPA용 기본 생성자는 protected
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING) // 반드시 STRING 사용 (ORDINAL 금지)
    private Role role;

    @CreatedDate
    private LocalDateTime createdAt;

    // 정적 팩토리 메서드로 생성 (생성자 직접 노출 금지)
    public static User create(String email, String password, Role role) {
        User user = new User();
        user.email = email;
        user.password = password;
        user.role = role;
        return user;
    }

    // 도메인 로직은 Entity 안에
    public void changePassword(String newPassword) {
        this.password = newPassword;
    }
}
```

### Entity 체크리스트
- [ ] `@NoArgsConstructor(access = AccessLevel.PROTECTED)` 설정
- [ ] `@Enumerated(EnumType.STRING)` 사용
- [ ] Setter 금지 → 의미 있는 메서드로 상태 변경
- [ ] `@CreatedDate`, `@LastModifiedDate` → BaseEntity로 분리

---

## 3단계: DTO 설계

```java
// 요청 DTO
public record CreateUserRequest(
    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "올바른 이메일 형식이 아닙니다")
    String email,

    @NotBlank(message = "비밀번호는 필수입니다")
    @Size(min = 8, message = "비밀번호는 8자 이상이어야 합니다")
    String password
) {}

// 응답 DTO
public record UserResponse(
    Long id,
    String email,
    LocalDateTime createdAt
) {
    // Entity → DTO 변환은 DTO 안에서 처리
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getEmail(), user.getCreatedAt());
    }
}
```

---

## 4단계: Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    // 복잡한 쿼리는 @Query 또는 QueryDSL 사용
    @Query("SELECT u FROM User u WHERE u.role = :role AND u.createdAt >= :from")
    List<User> findActiveAdmins(@Param("role") Role role, @Param("from") LocalDateTime from);
}
```

---

## 5단계: Service

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // 기본은 readOnly
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    // 조회: readOnly 상속
    public UserResponse getUser(Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new EntityNotFoundException("사용자를 찾을 수 없습니다. id=" + userId));
        return UserResponse.from(user);
    }

    // 변경: readOnly = false 명시
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException("이미 사용 중인 이메일입니다.");
        }

        String encodedPassword = passwordEncoder.encode(request.password());
        User user = User.create(request.email(), encodedPassword, Role.USER);
        User saved = userRepository.save(user);

        return UserResponse.from(saved);
    }
}
```

---

## 6단계: Controller

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{userId}")
    public ResponseEntity<ApiResponse<UserResponse>> getUser(@PathVariable Long userId) {
        UserResponse response = userService.getUser(userId);
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @PostMapping
    public ResponseEntity<ApiResponse<UserResponse>> createUser(
        @RequestBody @Valid CreateUserRequest request
    ) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.success(response));
    }
}
```

---

## 7단계: 공통 응답 / 예외 처리

### 공통 응답 포맷
```java
@Getter
public class ApiResponse<T> {
    private final boolean success;
    private final T data;
    private final String message;

    private ApiResponse(boolean success, T data, String message) {
        this.success = success;
        this.data = data;
        this.message = message;
    }

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, null);
    }

    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message);
    }
}
```

### 전역 예외 핸들러
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(EntityNotFoundException e) {
        log.warn("Not found: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiResponse.error(e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining(", "));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(ApiResponse.error(message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleAll(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("서버 오류가 발생했습니다."));
    }
}
```

---

## 자주 쓰는 패턴 모음

### Soft Delete (논리 삭제)
```java
// Entity에 추가
@Column(nullable = false)
private boolean deleted = false;

public void delete() {
    this.deleted = true;
}

// Repository에 추가
@Where(clause = "deleted = false") // Hibernate 기본 필터
```

### Pagination
```java
// Controller
@GetMapping
public ResponseEntity<ApiResponse<Page<UserResponse>>> getUsers(
    @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
    Pageable pageable
) {
    return ResponseEntity.ok(ApiResponse.success(userService.getUsers(pageable)));
}

// Service
public Page<UserResponse> getUsers(Pageable pageable) {
    return userRepository.findAll(pageable).map(UserResponse::from);
}
```

---

## 체크리스트 (PR 전 확인)

- [ ] Entity → Controller 직접 반환 없음
- [ ] `@Transactional` 위치 적절한가 (Service만)
- [ ] 예외 메시지가 명확한가
- [ ] 로그 레벨이 적절한가 (info/warn/error)
- [ ] `application.yml`에 민감 정보 없음
- [ ] N+1 문제 없는가 (`fetch join` 또는 `@EntityGraph` 확인)
- [ ] 핵심 Service 로직 단위 테스트 존재하는가
