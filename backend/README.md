# 11th Assignment - Backend Starter Kit

FE/APP 개발자가 JVM 지식 없이도 로컬에서 백엔드를 실행하고 API 명세를 확인할 수 있도록 만든 실행 패키지입니다.

## 목적

- FE, APP 개발자의 로컬 연동 환경 제공
- assignment 모듈 실행 JAR + API 명세 제공

## 결과물

- `jar/` 폴더
  - `prography-be-assignment-*.jar` (로컬 서버 실행용)
- 원본 API 명세: `docs/api-spec.md`

## 빠른 시작 (공통)

1. Java 설치 (아래 환경별 안내 참고)
2. 프로젝트 받기
3. JAR 실행

## Java 설치 (환경별)

### macOS

- 권장: Adoptium Temurin 24+ 설치
- Homebrew 사용 시:

```bash
brew install --cask temurin
```

- 설치 확인:

```bash
java -version
```

### Windows

- 권장: Adoptium Temurin 24+ MSI 설치
- 설치 후:
  - `JAVA_HOME` 환경변수 설정
  - `PATH`에 `%JAVA_HOME%\bin` 추가
- 설치 확인 (PowerShell):

```powershell
java -version
```

### Linux

- 권장: Adoptium Temurin 24+ 설치
- 설치 확인:

```bash
java -version
```

## 실행

```bash
java -jar jar/prography-be-assignment-*.jar
```

- API Base URL: `http://localhost:8080/api/v1`
- H2 Console: `http://localhost:8080/h2-console`
  - JDBC URL: `jdbc:h2:mem:prography`
- 데이터는 메모리 DB라서 **서버 재시작 시 초기화**됩니다.

## API 명세

- `docs/api-spec.md` 를 참조해주세요.

## 환경 변수 (선택)

아래 값이 없으면 기본값으로 실행됩니다.

- `JWT_SECRET`
- `CORS_ALLOWED_ORIGINS`

예시 (macOS / Linux):

```bash
export JWT_SECRET="your-local-secret"
export CORS_ALLOWED_ORIGINS="http://localhost:3000"
java -jar dist/prography-be-assignment-*.jar
```

## 실행이 잘 되지 않는다면?

- `java` 명령이 안 보이면: `JAVA_HOME`/`PATH` 설정 확인
- `8080` 포트 충돌 시: 실행 전 `SERVER_PORT=8081`로 변경
- H2 콘솔 접속 실패 시: `application.yml`의 `spring.h2.console.enabled` 확인
