# Runtime stage
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# 로컬에서 빌드된 JAR 파일 복사
# 배포 전에 로컬에서 ./gradlew clean bootJar를 실행하여 backend/build/libs/*.jar 파일을 생성해야 함
COPY build/libs/*.jar app.jar

# 포트 노출
EXPOSE 8080

# 애플리케이션 실행
ENTRYPOINT ["java", "-jar", "app.jar"]

