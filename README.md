# gradle_Jenkins_mount
gradle 프로젝트를 Jenkins를 통해 build하고, docker의 volume mount, bind mount


# 📦 Jenkins + Spring Boot 실행 환경 (Bind Mount & Volume Mount)

## 1. 개요
- **Jenkins 컨테이너**: GitHub Webhook으로 빌드 → JAR 생성  
- **호스트(Ubuntu)**: Jenkins 빌드 산출물(JAR)을 bind mount로 공유  
- **ubt-bind 컨테이너**: 호스트 디렉토리를 마운트하여 JAR 실행  
- **Volume**: 애플리케이션 데이터 영구 보관  

---

## 2. Bind Mount

📌 개념  
- 호스트 디렉토리와 컨테이너 디렉토리를 직접 연결하는 방식  
- 호스트에서 변경 → 컨테이너에서 즉시 반영됨  

📌 설정  
-v /home/ubuntu/09.test:/app  

- 호스트 경로: /home/ubuntu/09.test  
- 컨테이너 경로: /app  
- Jenkins 빌드 결과물(JAR)을 자동 반영  

📌 확인  
호스트에 파일 생성:  
echo "bind test" > /home/ubuntu/09.test/test.txt  

컨테이너에서 확인:  
docker exec -it ubt-bind ls /app  

✅ test.txt 존재 확인 시 bind mount 정상 작동  

---

## 3. Volume Mount

📌 개념  
- Docker가 관리하는 영구 저장소  
- 컨테이너 삭제/재생성에도 데이터 유지  

📌 생성 & 사용  
docker volume create gradle.vol  

컨테이너 실행 시:  
-v gradle.vol:/data  

- 컨테이너 경로 /data 는 영구 데이터 보관용  
- 예: 로그, 업로드 파일, 캐시 등 저장  

---

## 4. Jenkins 컨테이너 실행 예시
docker run -d \
  --name myjenkins \
  -p 9090:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /home/ubuntu/09.test:/var/jenkins_home/workspace/step03_teamArt/build/libs \
  -v gradle.vol:/data \
  jenkins/jenkins:lts  

- jenkins_home → Jenkins 설정/Job 영구 보관  
- /home/ubuntu/09.test → 빌드된 JAR이 호스트에 공유됨  
- gradle.vol → 추가 데이터 저장소  

---

## 5. Spring Boot 실행 컨테이너 (ubt-bind)
docker run -d \
  --name ubt-bind \
  -p 8087:8087 \
  -v /home/ubuntu/09.test:/app \
  -v gradle.vol:/data \
  openjdk:17-jdk-slim \
  java -jar /app/app.jar  

- JAR 파일은 항상 /home/ubuntu/09.test/app.jar 로 최신화  
- ubt-bind 컨테이너는 bind mount 덕분에 최신 JAR을 자동 실행  

---

## 6. 동작 흐름
```mermaid
flowchart LR
    A[GitHub Push] -->|Webhook| B[Jenkins Container]
    B -->|Build JAR| C["Host: /home/ubuntu/09.test"]
    C -->|Bind Mount| D[ubt-bind Container]
    D -->|Run| E[Spring Boot App]
    D -->|Volume Mount| F[(gradle.vol)]
