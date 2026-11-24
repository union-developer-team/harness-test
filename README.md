# harness-test

`louie-kury-document` 공통 하네스를 활용하는 회원 CRUD 실험용 서비스 레포입니다. Java 21·Spring Boot·Hexagonal 기본 설정을 준비했으며 회원 기능은 아직 구현하지 않았습니다.

## 기본 설정과 실행

- Java 21 toolchain·컴파일 대상 21, Spring Boot 4.2.0, Gradle Wrapper 9.9.2
- H2 메모리 DB(`jdbc:h2:mem:harnesstest`), JPA `create-drop`, Open Session in View 비활성
- 서버 `127.0.0.1:8080`, H2 웹 콘솔 비활성. 프로세스를 종료하면 DB 데이터가 사라집니다.

JDK 21이 필요합니다. Gradle은 Wrapper가 내려받으므로 별도 설치하지 않습니다. 최초 실행에는 Gradle·Maven 저장소 네트워크 접근이 필요합니다.
서비스 레포에서 다음 명령을 실행합니다. macOS에서는 명령별 `JAVA_HOME`으로 설치된 JDK 21을 선택하며 전역 설정을 바꾸지 않습니다.

```sh
JAVA_HOME="$(/usr/libexec/java_home -v 21)" ./gradlew build
JAVA_HOME="$(/usr/libexec/java_home -v 21)" ./gradlew bootRun
```

서버 종료는 실행 터미널에서 `Ctrl+C`입니다. 기본 JVM이 다른 버전이어도 빌드 설정의 toolchain과 `release`는 Java 21입니다. 다른 OS에서는 JDK 21의 경로를 `JAVA_HOME`으로 지정합니다.
검증용 캐시를 레포 안에 두려면 명령 앞에 `GRADLE_USER_HOME="$PWD/.gradle-user-home"`을 함께 지정합니다. `.gradle-user-home/`, `.gradle/`, `build/`는 Git 제외되어 있습니다.

## Hexagonal 구조

현재 각 패키지에는 의존 방향을 설명하는 `package-info.java`만 있습니다. 사용하지 않는 포트·회원 엔티티·CRUD 메서드는 만들지 않았습니다.

```text
com.louiekury.harnesstest
├── HarnessTestApplication
└── member
    ├── domain                    # Spring/JPA와 독립적인 도메인
    ├── application               # 기능 구현 때 <usecase>/입력 포트·서비스 추가
    │   └── port/out              # 유스케이스가 요구하는 출력 포트
    └── adapter
        ├── in/web                # HTTP 변환·입력 포트 호출
        └── out/persistence       # 출력 포트 구현·JPA 엔티티·매퍼
```

서비스는 도메인·포트에 의존하고 어댑터 구현을 직접 참조하지 않습니다. 입력 포트·모델과 출력 포트에는 웹/JPA 타입을 노출하지 않으며, JPA 엔티티는 도메인 모델과 분리합니다.

## 작업 경로

- 공통 문서와 역할 정의: `../louie-kury-document/`
- 실제 구현·빌드·테스트: 이 레포(`harness-test/`)
- 서비스 작업 규칙: [AGENTS.md](AGENTS.md)
- 공통 문서 탐색: [ROUTING.md](../louie-kury-document/docs/ai/ROUTING.md)
- 역할과 개발 순서: [WORKFLOW.md](../louie-kury-document/docs/ai/WORKFLOW.md)

공통 하네스의 역할·스킬을 사용하는 세션은 `louie-kury-document/`를 기준으로 시작하고, 실제 구현과 서비스 검사는 이 레포를 작업 경로로 지정합니다. 공통 레포의 `.codex/` 설정이나 `.agents/skills/`가 이 레포에 자동 설치되는 것으로 간주하지 않습니다.

## 하네스 검사

Python 3.11 이상으로 공통 문서 레포에서 실행합니다.

```sh
cd ../louie-kury-document
python3 -B scripts/ai/harness.py check-config
python3 -B scripts/ai/harness.py verify
```

현재 로컬 환경의 기본 `python3`는 3.9.6입니다. 하네스 훅도 `python3`를 호출하므로 부모 명령과 자식 훅 모두 설치된 Python 3.12를 사용하도록 명령 범위의 `PATH`를 지정합니다. 전역 Python 설정은 변경하지 않습니다.

```sh
# 위와 같이 louie-kury-document/에서 실행
PATH="/opt/homebrew/opt/python@3.12/libexec/bin:$PATH" python3 -B scripts/ai/harness.py check-config
PATH="/opt/homebrew/opt/python@3.12/libexec/bin:$PATH" python3 -B scripts/ai/harness.py verify
```

이 경로는 현재 컴퓨터 기준입니다. 다른 환경에서는 Python 3.11 이상이 있는 경로를 사용합니다.

하네스 검사 성공은 회원 API·서버·DB 검증을 의미하지 않습니다. 서비스 빌드·실행은 위 기본 설정 명령으로 따로 확인합니다.

## 회원 CRUD 진행 순서

관련 문서 조사 → planner의 `docs/plans/` 계획서 작성 → 사람의 계획 승인 → 이슈 작성 → 사람의 이슈 승인 → 브랜치 생성 → 구현·검증·독립 리뷰·문서 정리 순서로 진행합니다.

기본 설정 이후에는 `회원 생성 구현`, `회원 목록 조회 구현`, `회원 단건 조회 구현`, `회원 수정 구현`, `회원 삭제 구현` 중 지정한 기능만 구현·검증·보고합니다. 다음 기능은 자동 진행하지 않습니다.
기존 제품 명세와의 차이 및 기능별 API·확인 명령은 [실행 계획서](docs/plans/2026-11-24-member-crud.md)에 있습니다. 생성·목록/단건 조회·수정·실제 삭제, `id/name/email` 최소 모델은 이후 기능 제안이며 제품 회원 기능 완료를 의미하지 않습니다.
