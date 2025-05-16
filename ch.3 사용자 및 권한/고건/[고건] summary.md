# 정리

## 3.1 사용자 식별

- Mysql 계정 이름 형식: '아이디'@'IP 또는 호스트명'
- 예시: 'gogun'@'192.168.0.10'

모든 외부 컴퓨터에서 접속하려면, 호스트 부분을 '@'로 지정하면 됨


## 3.2 사용자 계정 관리

### 3.2.1 시스템 계정과 일반 계정

- MySQL 8.0 이후로 'SYSTEM_USER' 권한의 유무에 따라 DBA 계정, 일반 사용자 계정이 나눠짐.

### 3.2.2 계정 생성

GRANT와 CREATE USER

- MySQL 5.7 까지는 GRANT 명령으로 권한 부여 & 계정 생성 가능
- MySQL 8.0 이후로 GRANT 명령으로는 권한 부여, CREATE USER 로 계정 생성

아래는 CREATE USER 명령의 옵션이다.

#### IDENTIFIED WITH

사용자 인증 방식 & 비밀번호 설정
  - 이 구문 뒤에는 반드시 인증 방식을 명시해야 함
  - EX: IDENTIFIED WITH 'mysql_native_password' BY 'password'
 
#### REQUIRE

접속 시 암호화된 SSL / TLS 채널 사용 여부 설정

#### PASSWORD EXPIRE

비밀번호 유효 기간 설정

#### PASSWORD HISTORY

한 번 사용했던 비밀번호를 재사용하지 못하게 설정

- PASSWORD HISTORY DEFAULT: 시스템 변수에 저장된 개수만큼 비밀번호 이력 저장
- PASSWORD HISTORY n: 최근 n개까지 비밀번호 이력 저장

#### PASSWORD REUSE INTERVAL

한 번 사용한 비밀번호 재사용 금지 기간 설정

#### PASSWORD REQUIRE

비밀번호가 만료되어 새로운 비밀번호로 변경 시 현재 비밀번호의 필요 여부를 결정

#### ACCOUNT LOCK / UNLOCK

계정 생성 / 정보 변경 시 계정을 잠글지 결정


## 비밀번호 관리

### 3.3.1 고수준 비밀번호

- validate_password 컴포넌트: 비밀번호의 유효성 체크
  - LOW: 길이만 검증
  - MEDIUM: 길이 + 숫자, 대소문자, 특수문자까지 검증
  - STRONG: MEDIUM + 금칙어 포함 여부 검증
 
- DUAL PASSWORD

  - 이전의 문제점: 서비스 실행 중 비밀번호 변경 불가 (서비스를 모두 멈추어야 변경 가능)
      
  Mysql 8.0부터는 계정 비밀번호로 두 개의 값을 사용하는 기능이 추가됨
  Primary(최근 비밀번호), Secondary(이전 비밀번호)로 구분됨

  - ALTER USER 'root@localhost' IDENTIFIED BY 'new_password' RETAIN CURRNET PASSWORD;

  기존 비밀번호를 Secondary로 바꾸고, 새로운 비밀번호를 Primary로 설정

  - ALTER USER 'root@localhost' DISCARD OLD PASSWORD;
  
  작업이 끝나고 Secondary 비밀번호를 삭제


## 3.4 권한

- Mysql 5.7까지의 권한
  - 글로벌 권한: DB나 테이블 이외 객체에 적용되는 권한 (뒤에 특정 객체 명시 X)
  - 정적 권한: DB나 테이블을 제어하는 데 필요한 권한 (뒤에 특정 객체 명시)
  
  Mysql 8.0의 권한
  - Mysql 5.7의 권한 + 동적 권한
  - 동적 권한: Mysql 서버가 시작되면서 동적으로 생성되는 권한 (컴포넌트나 플러그인 설치 후 등록되는 권한)
 
- GRANT
  권한을 부여할 때 사용
  GARNT priv_list ON db.table TO 'user@host';

- 글로벌 권한
  특정 DB, 테이블에 부여할 수 없음.
  GRANT SUPER ON *.* TO 'user@localhost';
  *.*은 DB의 모든 오브젝트를 포함한 Mysql 서버 전체를 의미함

- DB 권한
  GRANT EVENT ON *.* TO 'user@localhost';
  혹은 ON employees.* 가능

- 테이블 권한
  GRANT SELECT,INSERT,UPDATE,DELETE ON *.* TO 'user.localhost';
  혹은 ON employees.*, ON employees.department 가능
  잘 사용하지 않음
  
- 특정 칼럼에 권한 부여
  테이블 권한과 다르게 INSERT, UPDATE, SELECT 3가지 권한만 부여할 수 있음
  잘 사용하지 않음

## 3.5 역할

- 역할(Role): 권한의 묶음

CREATE ROLE
  role_emp_read,
  role_emp_write;

로 정의 가능

역할 자체로 사용 불가, 계정에 부여해야 사용 가능
단, 매번 로그인 시 SET ROLE을 통해 역할을 활성화해야 사용 가능 (기본 설정)

- 권한은 Mysql 서버 내부적으로 계정과 동일 취급 (구분 없이 같은 테이블에 저장)
- 권한 부여 시 하나의 계정에 다른 계정을 병합하는 방식
- 역할이 mysql.user에 저장될 때 account_locked 칼럼이 'Y'로 설정


# 읽으면서 생긴 궁금증

---

### Q1. 이전에 사용했던 비밀번호 정보가 저장된 `password_history` 테이블에는 현재 사용하는 비밀번호가 저장되어 있을까?

- **A1.** No. 현재 사용하는 비밀번호는 `mysql.user` 테이블에 해싱된 형태로 저장되어 있으며,  
  `password_history` 테이블에는 과거에 사용된 비밀번호만 저장된다.

---

### Q2. 칼럼 단위의 권한을 하나라도 설정하게 되면, 나머지 모든 테이블의 칼럼에 대해서도 권한 체크를 하는 이유는 무엇일까?

- **A2.** 다른 모든 칼럼에 대해 권한 검사를 수행하지 않으면,  
  은밀하게 보호되어야 할 칼럼의 존재나 내용을 간접적으로 추론하거나 노출할 위험이 있기 때문이다.  
  이는 보안을 강화하기 위한 설계다.

---

### Q3. MySQL 서버에서 역할이 자동으로 활성화되지 않게 설정한 이유는 무엇일까?

- **A3.** 보안 강화와 **최소 권한 원칙**에 따라, 로그인 시 매번 불필요한 권한이 자동 부여되지 않도록 하기 위해서이다.  
  역할은 명시적으로 활성화(`SET ROLE`)해야 적용된다.

---

### Q4. 역할이 저장되어 있는 `mysql.user` 테이블의 `account_locked` 값을 'Y'에서 'N'으로 바꾸면 로그인할 수 있을까?

- **A4.** 불가능하다.  
  역할(role)은 로그인 기능이 없는 **비로그인 전용 계정**이므로,  
  `account_locked` 값을 'N'으로 바꾸더라도 인증 정보가 없어서 로그인 중 오류가 발생한다.

---
# 처음 본 개념 정리
[3강 개념정리](https://mju-my.sharepoint.com/:w:/g/personal/rhrjs0131_mju_ac_kr/ERzTVVqnxiJFoyYm5HpOoscBt62VWjhl5KXe8Qb6leofAg?e=5NAfIO)  
