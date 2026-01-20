# Nginx 단계만 (React 빌드는 로컬에서 수행)
FROM nginx:alpine

# 기본 nginx.conf 삭제
RUN rm /etc/nginx/conf.d/default.conf

# 커스텀 nginx 설정 복사 (프로젝트 루트가 context이므로 nginx/ 경로 사용)
COPY nginx/nginx.conf /etc/nginx/nginx.conf

# 로컬에서 빌드된 React 정적 파일 복사
# 배포 전에 로컬에서 npm run build를 실행하여 q-track-frontend/dist/ 폴더를 생성해야 함
COPY frontend/dist /usr/share/nginx/html

# 포트 노출
EXPOSE 80

# Nginx 실행
CMD ["nginx", "-g", "daemon off;"]

