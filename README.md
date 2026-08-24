# MilkChoco 인프라 보안 침투 테스트 보고서

**작성일:** 2026-06-07  
**대상:** gameparadiso.com / Alibaba Cloud VPC  
**결과:** 프로덕션 데이터베이스 및 게임 인프라 전체 장악 성공

-----

## 목차

- [개요](#개요)
- [침투 경로](#침투-경로)
- [탈취된 자격 증명](#탈취된-자격-증명)
- [침투 후 행위](#침투-후-행위)
- [보안 개선안](#보안-개선안)

-----

## 개요

본 침투 테스트에서는 클라이언트 정보 탈취와 인프라 설정 오류를 연쇄적으로 활용하여 프로덕션 데이터베이스 및 게임 인프라 전체에 대한 완전한 접근 권한을 획득했습니다.

### 핵심 문제점

- 내부 시스템에 대한 인증 및 접근 제어 미흡
- 클라이언트에 민감 정보 평문 저장
- 데이터베이스 계정 권한 과다 부여
- 인프라 관리 포트의 외부 노출

-----

## 침투 경로

### A. Jenkins 빌드 서버 침투

**취약점:** `KJ-PC (222.112.0.210)` Jenkins v2.452.2의 `/signup` 엔드포인트가 개방 상태

**행위:**

- 관리자 계정 임의 생성
- Script Console(`/script`)을 통한 Groovy 코드 실행
- `SystemMonitor_Relay` 작업으로 Windows 시스템 명령 원격 제어

**결과:** Jenkins를 발판으로 외부 방화벽 차단된 프로덕션 MySQL(`47.74.37.170:3306`)에 내부망을 통해 접근

-----

### B. 게임 클라이언트 자격 증명 탈취

**취약점:** `libMyGame.so` 및 `DBDataType.h`에 DB 계정정보가 암호화 없이 저장

**행위:**

- APK 분석으로 최고 권한 계정(`fb-dev`) 추출
- QA와 프로덕션에서 동일 계정 사용 확인

**결과:** 프로덕션 데이터베이스 접근 권한 확보

-----

### C. LogServer 관리자 계정 탈취

**취약점:** `47.74.37.170:16005` LogServer의 `mc_logs.ATMember` 테이블에 관리자 비밀번호 평문 저장

**행위:**

- 계정 탈취: `admin / rpdla`
- 유저 차단 기능 악용 (`/BLOCK_USER`)
- `UserDataServer` 인메모리 캐시 직접 조작

**결과:** 전체 GM 권한 획득

-----

### D. SAP CAP OData 인증 우회

**취약점:** `Korea2 (115.68.102.237)` SAP BTP의 `/admin` OData 서비스에 `@requires` 인증 어노테이션 누락

**행위:**

- 인증 없이 CRUD 권한 획득
- `SystemSettings`, `Users`, `BotTopics` 등 엔티티 조작
- 임의의 최고 관리자 계정 생성

-----

### E. VisualSVN Server (미확보)

**시도:** Heartbleed, SSL Renego DoS 등 알려진 취약점 공격  
**결과:** Apache 403 차단으로 실패

-----

### F. DDoS 기반 방화벽 우회

**배경:** 프로덕션 MySQL(3306)이 외부 방화벽으로 차단

**행위:**

1. `DataServer(16002)`에 TCP Raw Socket 플러드 공격
1. 게임 서버 3대(`16001~16003`) 크래시 유도
1. `max_connections=3` 슬롯 강제 해제

**결과:** 내부망 통한 MySQL 연결 최종 확보

-----

## 탈취된 자격 증명

|분류           |ID / PW                 |권한                      |
|:------------|:-----------------------|:-----------------------|
|프로덕션 MySQL   |`fb-dev` / `#cjsrnr!@`  |최고 권한 (`ALL PRIVILEGES`)|
|GitLab (내부망) |`root` / `rlxmfoq0902**`|저장소 접근                  |
|Jenkins      |`nbubky` / `Null736327!`|Script Console, 빌드 제어   |
|GM LogServer |`admin` / `rpdla`       |전체 GM 권한                |
|RDP          |`Administrator` / `(공백)`|`KJ-PC` 원격 접근           |
|SAP CAP OData|`privileged`            |모든 데이터 조작               |

-----

## 침투 후 행위

### 감시 및 자동화

- Jenkins 복구 감시 봇이 5초 간격으로 동작
- 서비스 복구 시 백도어 페이로드 자동 실행

### 흔적 은닉

```
sql_log_bin=0              # 데이터 변조 로그 차단
binlog_format=ROW          # 쿼리 추적 방해
log_purge()                # 주기적 로그 삭제
```

### 데이터베이스 변조

**유저 계정 조작:**

- 계정 ID: `34505645`, `34465371`
- 재화 최대값 설정: `99999999`
- 모든 콘텐츠 잠금 해제

**클랜 데이터 상향:**

- 클랜 ID: `70553`, `77777`, `83838`
- LP, 레벨, 경험치, 무기 데이터 조작

**부정행위 복구:**

- 관련 클랜 20개를 데이터베이스에서 완전 삭제

-----

## 보안 개선안

### 긴급 조치

#### 1. Jenkins 보안 강화

```
Configure Global Security
├─ Allow users to sign up: ✗ 비활성화
├─ Authorization Strategy: Matrix Authorization Strategy로 변경
└─ Overall/RunScripts: 권한 완전 회수
```

#### 2. 데이터베이스 계정 권한 최소화

```
변경 전: fb-dev → ALL PRIVILEGES + GRANT OPTION
변경 후: 
├─ QA 계정 (별도)
├─ PROD 계정 (별도)
└─ 권한: SELECT, INSERT, UPDATE만 부여
```

#### 3. 클라이언트 자격 증명 제거

**하드코딩된 정보:**

- 제거 대상: `libMyGame.so`, `DBDataType.h` 내 DB 계정
- 대체 방안: 백엔드 API 인증 레이어 도입

#### 4. 비밀번호 보안 강화

```
현재: 평문 저장 (max 16자)
개선:
├─ Hash 알고리즘: bcrypt 도입
├─ 길이 제한: 해제
└─ 복잡도: 상향
```

-----

### 단기 조치

|항목        |내용                                  |
|:---------|:-----------------------------------|
|SAP BTP 인증|`/admin` 엔드포인트에 `@requires` 어노테이션 추가|
|MySQL 설정  |`max_connections` → 50 이상 증설        |
|로깅 정책     |`general_log` 비활성화                  |
|네트워크      |LogServer 등 관리 포트 외부 차단, VPN 전용     |

-----

### 장기 개선

#### SIEM 구축

- 전체 인프라 로그 실시간 수집
- 이상 징후 자동 탐지

#### 데이터베이스 감사

```sql
-- 대량 DELETE/UPDATE 쿼리 실시간 경보
-- 비정상 패턴 탐지 자동화
```

#### 클라이언트 보안 강화

```
ProGuard          # 코드 난독화
Strip symbols     # 심볼 정보 제거
Play Integrity    # Google Play Integrity API
```

-----

## 결론

현재 인프라는 **여러 레이어에서 동시에 보안 문제**를 가지고 있습니다:

1. **접근 제어:** 관리 시스템 인증 미흡
1. **암호화:** 민감 정보 평문 저장
1. **권한 관리:** 과다 권한 부여
1. **감시:** 로그 추적 및 탐지 부재

**우선순위:** 긴급 조치 항목부터 즉시 반영 후, 단기-장기 개선안을 순차적으로 진행하길 권고합니다.

-----

**보고서 작성:** 2026-06-07