# 배포 가이드

이 문서는 EC2 우분투 서버에 Docker Compose를 사용하여 애플리케이션을 배포하는 전체 과정을 설명합니다.

## 배포 전략

1. **Frontend**: 로컬에서 빌드하여 HTML 정적 파일 생성 (`frontend/dist/` 폴더)
2. **Backend**: 로컬에서 빌드하여 JAR 파일 생성 (`backend/build/libs/*.jar`)
3. **Nginx**: EC2에서 Docker 이미지 빌드 (정적 파일과 JAR 파일만 복사)
4. **배포**: GitHub에 푸시 후 EC2에서 클론하여 Docker Compose로 실행

이 방식의 장점:
- ✅ EC2 메모리 절약 (React와 Java 빌드가 메모리를 많이 사용함)
- ✅ 빌드 시간 단축 (로컬에서 빠르게 빌드)
- ✅ 로컬 개발 환경 활용 (빌드 캐시 등)
- ✅ EC2에서 빌드 도구 불필요 (JDK, Node.js 등)

---

## 1. Frontend 빌드

### 1.1 Frontend 빌드 실행

```bash
# Frontend 디렉토리로 이동
cd frontend

# 의존성 설치 (처음 한 번만)
npm install

# 프로덕션 빌드 실행
npm run build
```

빌드가 완료되면 `frontend/dist/` 디렉토리에 정적 파일이 생성됩니다.

### 1.2 Frontend 빌드 결과 확인

```bash
# 빌드된 파일 확인
ls -la frontend/dist/

# 예상 출력:
# - index.html
# - assets/
#   - index-xxx.js
#   - index-xxx.css
#   ...
```

**중요**: `frontend/dist/` 폴더가 생성되어야 합니다. 이 폴더는 Nginx가 서빙할 정적 파일을 포함합니다.

---

## 2. Backend 빌드

### 2.1 Backend 빌드 실행

```bash
# Backend 디렉토리로 이동
cd backend

# Gradle wrapper 실행 권한 부여 (Linux/Mac)
chmod +x gradlew

# 프로덕션 빌드 실행
./gradlew clean bootJar

# Windows의 경우
# gradlew.bat clean bootJar
```

빌드가 완료되면 `backend/build/libs/` 디렉토리에 JAR 파일이 생성됩니다.

### 2.2 Backend 빌드 결과 확인

```bash
# 빌드된 JAR 파일 확인
ls -la backend/build/libs/

# 예상 출력:
# - q-track-backend-1.0.jar
# - q-track-backend-1.0-plain.jar (선택사항)
```

**중요**: `q-track-backend-1.0.jar` 파일이 생성되어야 합니다. (`-plain.jar`는 라이브러리 포함 안 된 버전이므로 사용하지 않음)

---

## 3. 배포 패키지 준비

### 3.1 필수 파일 확인

배포 전에 다음 파일들이 포함되어야 합니다:

```
프로젝트 루트/
├── docker-compose.yml          # 필수
├── nginx/
│   ├── Dockerfile              # 필수
│   └── nginx.conf              # 필수
├── backend/
│   ├── Dockerfile              # 필수
│   └── build/
│       └── libs/
│           └── q-track-backend-1.0.jar  # 필수 (로컬에서 빌드된 JAR)
└── frontend/
    └── dist/                    # 필수 (로컬에서 빌드된 정적 파일)
        ├── index.html
        └── assets/
```

### 3.2 .gitignore 확인

`.gitignore` 파일에 빌드 산출물이 제외되지 않았는지 확인:

```
# 빌드 산출물은 포함해야 함
!frontend/dist/
!backend/build/libs/*.jar
```

**중요**: 
- `frontend/dist/` 폴더는 Git에 포함되어야 합니다!
- `backend/build/libs/*.jar` 파일은 Git에 포함되어야 합니다!

---

## 4. Nginx 폴더 구성 확인

### 4.1 Nginx 폴더 구조

```
nginx/
├── Dockerfile      # Nginx 이미지 빌드 파일
└── nginx.conf      # Nginx 설정 파일
```

### 4.2 Nginx Dockerfile 확인

`nginx/Dockerfile`이 다음과 같이 구성되어 있는지 확인:

```dockerfile
# Nginx 단계만 (React 빌드는 로컬에서 수행)
FROM nginx:alpine

# 기본 nginx.conf 삭제
RUN rm /etc/nginx/conf.d/default.conf

# 커스텀 nginx 설정 복사
COPY nginx/nginx.conf /etc/nginx/nginx.conf

# 로컬에서 빌드된 React 정적 파일 복사
COPY frontend/dist /usr/share/nginx/html

# 포트 노출
EXPOSE 80

# Nginx 실행
CMD ["nginx", "-g", "daemon off;"]
```

### 4.3 Nginx 설정 확인

`nginx/nginx.conf`가 올바르게 설정되어 있는지 확인:

- `/api` 경로는 `backend:8080`으로 프록시
- `/` 경로는 정적 파일 서빙 (`/usr/share/nginx/html`)
- React Router를 위한 `try_files` 설정 포함

---

## 5. 로컬에서 Docker Compose 실행 테스트

### 5.1 Docker Compose 실행

```bash
# 프로젝트 루트에서 실행
docker compose up
```

또는 백그라운드로 실행:

```bash
docker compose up -d
```

### 5.2 컨테이너 상태 확인

```bash
# 실행 중인 컨테이너 확인
docker compose ps

# 로그 확인
docker compose logs -f

# 특정 서비스 로그만 확인
docker compose logs -f nginx
docker compose logs -f backend
```

### 5.3 로컬 테스트

브라우저에서 다음을 확인:

1. **Frontend 접속**: `http://localhost`
   - React 앱이 정상적으로 로드되는지 확인

2. **API 테스트**: `http://localhost/health`
   - Backend API가 정상적으로 응답하는지 확인

3. **Nginx 프록시 테스트**: `http://localhost/api/...`
   - API 요청이 Backend로 올바르게 프록시되는지 확인

### 5.4 테스트 완료 후 정리

```bash
# 컨테이너 중지 및 제거
docker compose down

# 이미지까지 제거하려면
docker compose down --rmi all
```

**중요**: 로컬에서 정상적으로 동작하는 것을 확인한 후에만 GitHub에 푸시합니다!

---

## 6. GitHub에 푸시

GitHub 웹사이트에서 다음을 확인:

- `frontend/dist/` 폴더가 존재하는지
- `backend/build/libs/` 폴더에 JAR 파일이 있는지
- `docker-compose.yml` 파일이 있는지
- `nginx/` 폴더가 있는지

---

## 7. AWS EC2 인스턴스 생성

### 7.1 EC2 인스턴스 생성

t3.micro 메모리 부족하면 더 늘려야함
엘라스틱빈스톡 환경에 배포하면 불편함. 그리고 대부분 메모리 부족함 (프리티어)

## 8. Git, Docker, Docker Compose 설치 (우분투 환경)

### 8.1 시스템 업데이트

```bash
# EC2에 SSH 접속 후 실행
sudo apt-get update
sudo apt-get upgrade -y
```

### 8.2 Git 설치

```bash
# Git 설치
sudo apt-get install -y git

# 설치 확인
git --version
```

### 8.3 Docker 및 Docker Compose 설치

#### 방법 1: 스냅 사용 (가장 간단)

```bash
# 스냅이 설치되어 있지 않은 경우 먼저 설치
sudo apt-get install -y snapd

# Docker 설치 (스냅 사용)
sudo snap install docker

# Docker Compose 별도 설치 (GitHub에서 다운로드)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 실행 권한 부여
sudo chmod +x /usr/local/bin/docker-compose

# 설치 확인
docker --version
docker-compose --version
```

**중요**: 
- **sudo 사용 시**: 위의 권한 설정 단계는 **필요 없습니다**. 바로 `sudo docker-compose` 명령어를 사용하면 됩니다.
---

## 9. GitHub에서 프로젝트 클론

**중요**: 다음 파일들이 존재해야 합니다:
- `docker-compose.yml`
- `nginx/Dockerfile`
- `nginx/nginx.conf`
- `backend/Dockerfile`
- `frontend/dist/` 폴더 (빌드된 정적 파일)
- `backend/build/libs/*.jar` 파일 (빌드된 JAR 파일)

---

## 10. Docker Compose로 실행

### 10.1 Docker Compose 실행

```bash
sudo docker-compose up -d
```

### 10.2 컨테이너 상태 확인

```bash
# 실행 중인 컨테이너 확인
docker-compose ps

# 로그 확인
docker-compose logs -f

# 특정 서비스 로그만 확인
docker-compose logs -f nginx
docker-compose logs -f backend
```

### 10.3 서비스 상태 확인

```bash
# Backend API 테스트
curl http://localhost/health

# 컨테이너 내부 확인
docker-compose exec nginx sh
docker-compose exec backend sh
```

### 10.4 외부 접속 확인

브라우저에서 EC2 퍼블릭 IP로 접속:

```
http://43.200.170.228/
```

**중요**: 보안 그룹에서 포트 80이 열려있는지 확인하세요!

---