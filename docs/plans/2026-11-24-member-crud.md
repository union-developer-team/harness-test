# 간단한 회원 CRUD — 실행 계획서

- 상태: **기본 설정 완료 — CRUD 기능은 별도 요청 대기**
- 대상: `/Users/mac/Desktop/harness-test/harness-test`, 현재 브랜치 `feat/test1`
- 작성일: 2026-11-24. 기존 준비 변경인 `AGENTS.md`·`README.md`를 보존한다. 빌드·실행 설정과 패키지 뼈대가 작성됐으며 회원 구현 코드는 없다.
- 사용자 최신 지시: “기본 셋팅 부터 구현 / 언어: Java21 / 아키텍쳐: Haxagonal”. Java 21·Hexagonal 기본 설정에 대한 직접 실행 지시다.
- 승인 범위: 프로젝트 뼈대·빌드·실행 설정·패키지 의존 방향. 전체 CRUD 계획·이슈 승인은 아니다. 이슈·브랜치·커밋·PR 생성은 이번 요청에 없다.

## 목적과 범위

Java 21·Spring Boot·Hexagonal과 H2 메모리 DB로 가상 데이터용 로컬 회원 CRUD를 준비한다.
**이번 실행 범위는 기본 설정뿐이다.** 빌드·Wrapper·실행 클래스·설정과 `package-info.java` 등 최소 패키지 뼈대를 준비한다. 미사용 포트·CRUD 메서드·회원 엔티티는 만들지 않는다.
이후 별도 기능 명령의 범위는 생성, 전체 목록 조회, 단건 조회, 이름·이메일 수정, 실제 DB 행 삭제다.
제외: 화면, 로그인·인증, 암호화, 소프트 삭제, 파일 DB·재시작 영속성, 버전 필드·잠금·별도 경합 처리, 페이지네이션, 외부 연동·배포.

## 조사한 문서와 확인한 사실

이전 조사 결과를 활용하며 새 외부 조사·버전 변경은 하지 않는다. 공통 문서는 이 계획서 기준 `../../../louie-kury-document/docs/`에 있다.

| 근거 | 확인한 사실과 이번 적용 |
| --- | --- |
| `prd.md` §6.1 ACC-01·03, §8·10.1·11, D11·D18, Q06·17·22 | 제품은 인증·본인 접근·개인정보 암호화·MongoDB 프로필·소프트 삭제를 요구한다. 이번 실험 예외로 제외하며 제품 기능 완료로 보고하지 않는다. |
| `erd.md`, `erd/svc_identity.md`의 account와 참조 관계 | account는 BIGINT PK, UUID·이메일 UNIQUE, name100자·email254자와 기타 필수정보·감사 필드가 있다. 자격증명·소셜·프로필·이력은 account_id를 참조한다. 실험은 별도 members 테이블에 id/name/email만 두며 제품 스키마·관계와 호환을 보장하지 않는다. |
| `api/README.md`, `api/auth.md` 공통 계약·이메일 비교·내 회원정보/탈퇴 | 성공 data 래퍼·오류 code/message, 201 Location·204 본문 없음, 이메일 도메인만 소문자화 규칙은 유지한다. 제품의 문자열 UUID·본인 /me·페이지 계약은 이번 실험에 적용하지 않는다. |
| `convension/BACKEND_CODE_CONVENTIONS.md` 계층·영속성·트랜잭션·예외·테스트 | Hexagonal로 유스케이스 입력 포트/서비스·출력 포트·입출력 어댑터와 domain/JPA를 분리한다. 서비스 트랜잭션·검증·DTO·오류 매핑도 유지한다. |
| `ai/WORKFLOW.md`·`HUMAN_GATES.md`·`templates/plan.md`, Git 컨벤션 | 기능 개발의 계획 승인 → 이슈 작성·승인 절차는 유지한다. 이번 기본 설정은 사용자의 직접 실행 지시 범위로 진행하고 전체 CRUD 승인으로 확대하지 않는다. |
| 대상 AGENTS·README·build.gradle·Wrapper·src/main | Java21 toolchain·release21, Boot4.2.0·Wrapper9.9.2, 메모리 H2·127.0.0.1 설정과 Hexagonal 패키지 문서를 확인했다. 빌드·런타임 결과는 검증 담당의 확인을 기다린다. |

프론트·상담·미디어·소셜 등 다른 도메인 구현은 이번 범위에 해당하지 않는다.

## 제안하는 동작과 선택

Java **21**·Hexagonal은 사용자가 지정했다. 작성된 빌드는 Spring Boot4.2.0·Gradle Wrapper9.9.2, Web MVC·Validation·JPA·H2·테스트를 사용한다. Wrapper 배포 SHA-256도 설정돼 있다.
기존 근거: [Spring Boot 요구사항](https://docs.spring.io/spring-boot/system-requirements.html), [Gradle 호환성 표](https://docs.gradle.org/current/userguide/compatibility.html). Java21 toolchain과 release21을 지정했으며 실제 빌드·기동 결과는 검증 뒤 기록한다.
H2 메모리 DB와 JPA 스키마 생성으로 실행하고 종료하면 데이터가 사라진다. `127.0.0.1:8080`에 바인딩하며 H2 웹 콘솔은 비활성으로 둔다.

루트는 `com.louiekury.harnesstest`이며 `member/domain`, `member/application`, `member/application/port/out`, `member/adapter/in/web`, `member/adapter/out/persistence`에 `package-info.java`를 작성했다. 유스케이스별 입력 포트·서비스는 기능 구현 시 추가한다.
domain·포트·입력/결과 모델은 Spring/JPA/웹 타입을 참조하지 않는다. 서비스는 포트에 의존하고 입력/출력 어댑터가 해당 포트로 연결된다. JPA 엔티티·매퍼는 영속성 어댑터에 두며 도메인과 분리한다. 기본 설정 단계에는 이 의존 방향만 패키지 문서로 남긴다.

| 필드 | 제안 규칙 |
| --- | --- |
| id | 서버 발급 자동 증가 long PK, 응답 JSON 숫자. 클라이언트가 지정·변경하지 않음 |
| name | 앞뒤 공백 제거 후 1~100자·필수 |
| email | 앞뒤 공백 제거, 유효 이메일·최대254자·필수·UNIQUE. 도메인만 소문자화하고 로컬 부분은 보존 |

회원 응답은 위 3개 필드다. 생성·수정 요청에는 name/email만 받는다. 수정은 두 값 모두 필요한 PUT이며 자기 이메일 유지는 허용한다.
이메일 중복은 DB UNIQUE와 409 `EMAIL_ALREADY_REGISTERED`로 처리한다. 실패한 수정은 롤백한다.
잘못된 본문·양수가 아닌/파싱 불가 ID는 400 `VALIDATION_ERROR`, 없는 ID는 404 `NOT_FOUND`, 예상 밖 장애는 500 `INTERNAL_SERVER_ERROR`다. 내부 SQL·스택은 응답에 담지 않는다.
삭제는 실제 행 삭제이며 재삭제·삭제 후 조회/수정은 404다. 삭제한 이메일은 다시 사용할 수 있다.

| API | 성공 상태·본문 |
| --- | --- |
| POST /api/members | 201, Location 헤더, `{"data":{"id":N,"name":"...","email":"..."}}` |
| GET /api/members | 200, `{"data":[...]}`, id 오름차순·빈 목록은 [] |
| GET /api/members/{id} | 200, `{"data":{"id":N,"name":"...","email":"..."}}` |
| PUT /api/members/{id} | 200, 변경된 회원을 data에 반환 |
| DELETE /api/members/{id} | 204, 본문 없음 |

향후 CRUD의 승인 대상 예외는 최소 필드·숫자 ID·단순 목록, 인증/암호화 없는 가상 데이터, 단일 메모리 DB·실제 삭제다. 감사 필드·복잡한 공통 예외 체계와 별도 독립 QA 단계는 생략하는 제안이다.
Hexagonal 계층·서비스 트랜잭션·DTO·최소 오류 형식과 코드 리뷰는 유지한다. 위 제품 예외와 CRUD 전체는 아직 미승인이고, Java21·Hexagonal 기본 설정은 직접 지시된 범위다.

## 진행 순서와 변경 파일

1. 이번에는 사용자 직접 지시에 따라 **Java21·Hexagonal 기본 설정만 구현·검증·보고**한다. 기존 브랜치·변경을 보존하고 CRUD를 자동 진행하지 않는다.
2. 향후 CRUD 계획을 승인받으면 메인이 이슈를 작성하고 실제 링크로 이슈 승인을 받는다. 그 뒤 `feat/<이슈번호>-member-crud` 등 작업 브랜치를 만든다. 이번 기본 설정 중에는 이슈·브랜치를 생성하지 않는다.
3. 이후에는 **사용자가 명령한 기능만 구현·최소 검증·보고**하며 다음 기능을 자동 진행하지 않는다.
4. 해당 기능 내부 구현→검증→지적 수정→수정 확인은 승인 범위에서 이어간다. 이미 받은 같은 권한을 다시 묻지 않는다.
5. CRUD 전체 완료 시 새 reviewer가 코드 리뷰를 한 번 수행하고 planner가 실행/API 안내를 동기화한다. 별도 대규모 QA·경합 검사는 하지 않는다.
6. 커밋·push·PR·배포는 미요청으로 제외한다. 머지는 사람이 한다.

| 사용자 작업 명령 | 작업 범위·최소 확인 | 선행 조건 |
| --- | --- | --- |
| 기본 설정 구현 | Java21 toolchain·빌드·실행 설정·Hexagonal 패키지 뼈대, 빌드·서버 기동 확인. CRUD 없음 | 현재 직접 지시 수신 |
| 회원 생성 구현 | POST, 정상201·잘못된 입력400·중복409 | 기본 설정·기능 개발 승인 절차 |
| 회원 목록 조회 구현 | GET 목록, 데이터 포함·빈 배열200 | 생성 기능 |
| 회원 단건 조회 구현 | GET 단건, 저장값200·없는 ID404·잘못된 ID400 | 생성 기능 |
| 회원 수정 구현 | PUT, 응답값200·없는 ID404·잘못된 입력400·중복409·실패 시 기존값 유지 | 생성 기능 |
| 회원 삭제 구현 | DELETE, 204·DB 행 제거·재삭제404 | 생성 기능 |

목록·단건·수정·삭제는 서로 선행할 필요가 없다. tester는 생성 API나 테스트 fixture로 데이터를 준비하고 JPA 조회로 수정·삭제 결과를 확인하므로 아직 없는 다른 API에 의존하지 않는다.

| 담당 | 예정 파일 |
| --- | --- |
| implementer | 이번: Gradle/Wrapper·Java21 toolchain·실행 클래스·application.yml·Hexagonal `package-info.java`. 이후 기능 요청별: 입력 포트/서비스·출력 포트·웹/영속성 어댑터·domain/JPA·DTO·최소 예외 |
| tester | 이번: 빌드·기동 확인. 이후 `src/test/java/**/MemberApiTest.java` 등 기능별 검사·curl 확인. 구현 수정 금지 |
| reviewer / planner | 최종 읽기 전용 코드 리뷰 / 본 계획 상태와 README·API 안내 동기화 |

## 완료 기준과 검증

**이번 기본 설정 완료 기준:** Java21 toolchain·컴파일, `./gradlew --version`·`./gradlew build`, `./gradlew bootRun` 기동·로컬 바인딩·메모리 H2 확인 후 종료. macOS 표준 명령은 `JAVA_HOME="$(/usr/libexec/java_home -v 21)" ./gradlew build`와 같은 방식이며 검증 캐시는 명령 범위 `GRADLE_USER_HOME="$PWD/.gradle-user-home"`을 사용한다. Java21이 없으면 환경 차단으로 보고한다.
패키지 문서의 의존 방향과 domain/port의 기술 독립성을 확인한다. 이 단계에는 회원 API·테이블·포트 메서드가 없어야 하며 CRUD 검증으로 보고하지 않는다. README 구조 설명은 실제 구현 확인 후 동기화한다.
이후 기능별 `./gradlew test --tests '*MemberApiTest.createMember*'`처럼 해당 기능 검사만 실행한다.
테스트 메서드 접두사는 순서대로 `createMember`, `listMembers`, `getMember`, `updateMember`, `deleteMember`로 정한다. 최소 정상/실패 조건은 위 표대로 검증한다.
생성 검사는 Web/JPA/H2를 포함한 최소 API 통합 검사로 구성하고 각 테스트는 독립 데이터를 사용한다. 순서나 ID=1에 의존하지 않는다. 전체 완료 시 새 변경을 포함한 전체 테스트를 한 차례 실행한다.

아래는 구현 후 실행할 독립 확인 명령이다. 생성 구현이 완료된 서버를 켠 상태에서 사용하며 예제 이메일 중복 시 서버를 재시작하거나 이메일을 바꾼다.
각 생성 응답의 id는 서버가 발급한다. 조회·수정·삭제 예제는 자기 사전 데이터를 POST로 만들며 다른 조회 API가 필요하지 않다.

생성:
```sh
curl -i -X POST http://127.0.0.1:8080/api/members -H 'Content-Type: application/json' -d '{"name":"생성테스트","email":"create@example.com"}'
```
기대: 201·Location, `{"data":{"id":N,"name":"생성테스트","email":"create@example.com"}}`.

목록 조회:
```sh
curl -fsS -X POST http://127.0.0.1:8080/api/members -H 'Content-Type: application/json' -d '{"name":"목록테스트","email":"list@example.com"}'
curl -i http://127.0.0.1:8080/api/members
```
기대: 200·data 배열에 생성한 회원 포함. 빈 DB에서는 `{"data":[]}`.

단건 조회:
```sh
member_id=$(curl -fsS -X POST http://127.0.0.1:8080/api/members -H 'Content-Type: application/json' -d '{"name":"조회테스트","email":"get@example.com"}' | python3 -c 'import json,sys; print(json.load(sys.stdin)["data"]["id"])')
curl -i "http://127.0.0.1:8080/api/members/$member_id"
```
기대: 200·data의 id가 member_id이고 name/email이 사전 생성값과 동일.

수정:
```sh
member_id=$(curl -fsS -X POST http://127.0.0.1:8080/api/members -H 'Content-Type: application/json' -d '{"name":"수정전","email":"update@example.com"}' | python3 -c 'import json,sys; print(json.load(sys.stdin)["data"]["id"])')
curl -i -X PUT "http://127.0.0.1:8080/api/members/$member_id" -H 'Content-Type: application/json' -d '{"name":"수정후","email":"updated@example.com"}'
```
기대: 200·동일 id, name=수정후·email=updated@example.com. 저장값은 해당 기능 통합 검사에서 확인.

삭제:
```sh
member_id=$(curl -fsS -X POST http://127.0.0.1:8080/api/members -H 'Content-Type: application/json' -d '{"name":"삭제테스트","email":"delete@example.com"}' | python3 -c 'import json,sys; print(json.load(sys.stdin)["data"]["id"])')
curl -i -X DELETE "http://127.0.0.1:8080/api/members/$member_id"
curl -i -X DELETE "http://127.0.0.1:8080/api/members/$member_id"
```
기대: 첫 요청204·본문 없음, 재요청404·`{"code":"NOT_FOUND","message":"..."}`. 실제 행 제거는 해당 기능 통합 검사에서 확인.

위 CRUD 명령은 아직 구현·실행되지 않았다. 기본 설정의 빌드·서버/H2 결과는 별도로 확인하며 하네스 검사 결과를 제품 검증으로 대체하지 않는다.
기능 보고에는 변경 파일·실행 명령·결과·미검증 범위만 적는다. 지적 수정 후에는 해당 실패 사례만 확인한다.

## 승인과 진행 상태

- 현재: **Java21·Hexagonal 기본 설정 완료.** Wrapper `bootJar`·`build` 성공, Gradle9.9.2/JVM21.0.2·클래스 major65 확인. `ApplicationBootstrapTest` 1개 통과(HTTP404·H2 연결·SELECT1). `bootRun`의 Java21·8080 기동·H2 연결을 확인하고 종료했으며 8080 리스너가 남지 않았음을 확인했다. 독립 정적 리뷰는 실질 결함 없음. README 구조/명령 동기화 완료. CRUD 기능은 미구현이며 전체 CRUD 계획·이슈는 미승인이다.
- 이번 기본 설정 결과를 보고한 뒤 다음 기능 지시를 기다린다. 향후 기능 개발에는 계획·이슈 승인 절차를 적용한다.
- 대기 조건: 향후 기능 승인, 중요한 범위 변경, 권한·환경 차단, 수정 확인 후 미해결 사항, 사용자 중단 요청.
- 기존 사용자 변경·공통 하네스 문서는 보존한다. 현재 계획에 구현 결과나 승인 사실을 만들지 않는다.
