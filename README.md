# plog-backend

> SWYP 웹 13기 4팀 · 2026.04 ~ 2026.05
> 공부한 장소를 지도에 기록·북마크하는 서비스의 백엔드

## 기술 스택

| 분류 | 기술 |
|---|---|
| Language / Framework | Java 21, Spring Boot 4.0.5 |
| Persistence | Spring Data JPA, QueryDSL 5.0 |
| DB | MySQL 8.0 (dev·local), H2 (CI 테스트) |
| Auth | Spring Security, OAuth2 (Kakao), JWT |
| Infra | GCP Compute Engine, Docker / Docker Compose, Nginx, GitHub Actions, Docker Hub, GCP Secret Manager, Let's Encrypt |
| Monitoring | Spring Actuator, Prometheus, Promtail |
| Etc | GCS(이미지), Spotless, Swagger |

## 도메인

Member(카카오 로그인) · Post(공부 기록: 공부시간·집중도) · Place · PlaceCategory · Bookmark · Tag · Image · Badge · **Map**

---

## 인프라 / 배포

```
feature → develop PR
  └─ CI (ci-check.yml): H2 테스트 + Spotless 포맷 검사
develop merge
  └─ CD (deploy-dev.yml)
       1. GCP Workload Identity Federation 인증 (GitHub OIDC, 키 파일 없음)
       2. Gradle 빌드 → Docker 이미지 → Docker Hub push
       3. SSH로 docker-compose·promtail 설정 전송
       4. Secret Manager 값으로 .env.dev 생성
       5. docker compose up
dev VM (GCE e2-medium, Rocky Linux 9, asia-northeast3)
  Nginx(HTTPS) ──app-tier──▶ Spring Boot ──data-tier──▶ MySQL (외부 미노출)
                                └─ Promtail (로그 수집)
```

| 항목 | 내용 |
|---|---|
| 시크릿 | 저장소에 두지 않음. GCP Secret Manager → CD 단계에서 `.env.dev` 생성 |
| CI 인증 | Workload Identity Federation, `attribute.repository` 조건으로 이 레포에서만 토큰 발급 |
| 네트워크 | compose 네트워크를 `app-tier`(nginx↔spring) / `data-tier`(spring↔mysql)로 분리, DB 포트 외부 미노출 |
| HTTPS | Let's Encrypt 인증서 (VM에서 certbot으로 관리, 레포 외부 설정) |
| Nginx | 악성 스캐닝 경로(`.php`, `.env`, `/wp-admin` 등) 444 차단, SSE 알림 경로 버퍼링 해제 (`infra/nginx/conf.d/default.conf`) |
| 헬스체크 | Spring 컨테이너 `/actuator/health` 30초 간격, Nginx는 `service_healthy` 이후 기동 |

### 로컬 실행

```bash
docker compose -f docker-compose.local.yml up -d   # MySQL
# IntelliJ Active Profile: local
# 환경변수: DB_URL, DB_USERNAME, DB_PASSWORD, JWT_SECRET_KEY
./gradlew bootRun
./gradlew spotlessApply   # 커밋 전 필수 (CI에서 검사)
```

---

## 지도 API (`/api/map`)

| Method | Path | 설명 |
|---|---|---|
| GET | `/pins/records`, `/pins/bookmarks` | 화면 영역(뷰포트) 안의 기록·북마크 장소 핀 + 장소별 개수·썸네일 |
| GET | `/pins/records/{placeId}`, `/pins/bookmarks/{placeId}` | 핀 상세 |
| GET | `/sheet/records`, `/sheet/bookmarks` | 하단 시트 장소 목록 (커서 페이징, 정렬) |
| GET | `/{placeId}/records`, `/{placeId}/bookmarks` | 장소별 게시글 목록 (커서 페이징, 정렬) |
| GET | `/places` | 내가 기록한 장소 키워드 검색 |
| GET | `/count` | 기록·북마크 개수 |

### 뷰포트 조회
남서·북동 위경도로 `latitude BETWEEN` / `longitude BETWEEN` 조건을 걸고 `GROUP BY place.id`로 장소별 개수를 집계한다. 위경도 범위는 요청 단계에서 검증한다.

### 커서 페이징 공통 인프라 (`global/support/paging`)
| 클래스 | 역할 |
|---|---|
| `Cursorable` / `CursorDefault` | 커서·limit 파라미터와 기본값 |
| `CursorableArgumentResolver` | 쿼리 파라미터 → `Cursorable` 자동 변환, Base64 URL-safe 디코딩 |
| `Slice` | `limit + 1`건 조회로 `hasNext` 판정, 다음 커서를 Base64로 인코딩해 응답 (opaque cursor) |

정렬마다 **정렬 키 + PK 복합 커서**를 사용해 동점에서도 중복·누락이 없다.

| 정렬 | 커서 |
|---|---|
| 최신순(장소) | `{최근 공부일}:{placeId}` |
| 기록 많은 순 | `{count}:{placeId}` |
| 장소 내 최신순 | `{postId}` |
| 공부시간 / 집중도 순 | `{studyTime}:{postId}` / `{focus}:{postId}` |

### 공통 응답 · 예외
```json
{ "resultType": "SUCCESS | ERROR", "data": {}, "error": { "errorCode": "E404", "message": "..." } }
```
`ApiResponse` + `ResultType` + `ErrorType`(HTTP 상태·코드·메시지) + 전역 예외 핸들러

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| dev 배포 실패 | Dockerfile이 이미지 안에서 Gradle 빌드를 한 번 더 수행 | CI에서 만든 jar만 `COPY`하도록 단순화 |
| 죽은 서버로 트래픽 전달 가능 | actuator 의존성 누락 + compose healthcheck 없음 | actuator 추가, healthcheck + `service_healthy` 의존 |
| 서버 기동 실패 | 카카오·JWT 환경변수 누락 | Secret Manager 등록, 설정 정리 |
| SSH(22)는 되는데 80/443만 timeout | GCE 기본 설정이 `net.ipv4.ip_forward=0` → Docker NAT가 조용히 드롭 | `/etc/sysctl.d/99-docker-ipforward.conf`로 `=1` 오버라이드 |
| 최신순 정렬 오류 | 장소 저장일 기준 정렬 → 최근에 기록한 오래된 장소가 아래로 | `MAX(post.studyDate)` 기준 정렬 |
| 테스트에서 unique 위반 | `PlaceCategory`를 앱 시작 시 `init.txt`로 초기화 | 테스트에서 새로 persist하지 않고 조회해서 재사용 |

## 팀

| 역할 | 담당 |
|---|---|
| 김민석 ([@kminseok-dev](https://github.com/kminseok-dev)) | dev CI/CD(`ci-check.yml`, `deploy-dev.yml`)·`docker-compose.dev.yml`·`Dockerfile`, 공통 응답·예외 체계(`ApiResponse`, `AppException`, `ErrorType`/`ErrorCode`, `GlobalExceptionHandler` 구현), QueryDSL 도입(빌드 설정·`QuerydslRepositorySupport`, 이후 #84에서 정리), 커서 페이징 인프라(`global/support/paging`), **지도 도메인**(`domain/map`) |
| 팀원 | 회원·인증(OAuth2/JWT), 게시글, 장소, 북마크, 뱃지, 이미지, 도메인별 에러 코드·SSE 예외 처리, Nginx 설정 등 |

> 원본 레포: [swyp-web-13th-team4/plog-backend](https://github.com/swyp-web-13th-team4/plog-backend) · 이 README는 개인 fork에서 내 담당 기준으로 정리한 문서다.
