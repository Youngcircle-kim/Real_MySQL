# 정리

# 정리

MySQL에서 사용자 계정을 생성하는 방법이나 각 계정의 권한을 설정하는 방법은 다른 DBMS와는 조금 차이가 있다.
가장 대표적으로 MySQL의 사용자 계정은 `User`뿐만 아니라 접속한 `IP`까지 확인한다.
또한 `MySQL 8.0`버전 부터는 권한을 묶어서 관리하는 역할(Role, 롤)의 개념이 도입됐기 때문에 각 사용자의 권한으로 미리 준비된 Role을 부여 가능.

### (번외) MySQL과 이외의 DB의 권한 설정 및 계정 생성

**MySQL**

```sql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
GRANT SELECT, INSERT ON db_name.* TO 'username'@'host';
```

- 계정은 `사용자@호스트(IP)` 단위로 구분
- 권한 부여는 `GRANT`로 수행
- 권한 변경은 `REVOKE, FLUSH PRIVILEGES` 사용
  **PostgrSQL**

```sql
CREATE ROLE username LOGIN PASSWORD 'password';
GRANT SELECT, INSERT ON TABLE table_name TO username;
```

- 사용자 = ROLE
- `LOGIN` 옵션이 있어야 로그인 가능
- `GRANT`는 개별 테이블 또는 전체 스키마 단위
- Super user는 `SUPERUSER` 옵션 포함
  **Oracle**

```sql
CREATE USER username IDENTIFIED BY password;
GRANT CONNECT, RESOURCE TO username;
GRANT SELECT, INSERT ON table_name TO username;
```

- `CONNECT`, `RESOURCE`는 Role
- 계정 활성화 시 `DEFAULT TABLESPACE`, `QUOTA` 설정 필요
- 개별 객체 권한은 `GRANT`로 부여
  **MSSQL**

```sql
CREATE LOGIN username WITH PASSWORD = 'password';
CREATE USER username FOR LOGIN username;
GRANT SELECT, INSERT ON dbo.table_name TO username;
```

- `LOGIN`은 서버 인증 단위
- `USER`는 데이터베이스 단위
- 권한 부여는 `GRANT`, `DENY`, `REVOKE`로 관리

## 3.1 사용자 식별

`MySQL`은 다른 DBMS와는 달리 사용자의 계정뿐 아니라 사용자의 접속지점(IP나 Domain, Host name)도 계정의 일부가 된다. 따라서 로그인 할 때 `'user_id'@'IP_address'`이 포맷으로 접속한다.

> `%`를 사용하면 모든 접속지점을 의미한다.
> 따라서 접속 지점과 상관 없는 계정을 생성할라면 `'user_id'@'%'`로 계정을 생성하면 된다.
> 참고로 노트북에서 돌리는 Local Server는 당연하지만 `localhost`로 등록되어있다.

그러면 동일한 `user_id`를 지니고 있을 때 `Host`(접속지점)을 어떻게 검사할까 ?
`mysql.user`테이블이 아래와 같이 존재한다고 생각하자.

| No  | User  | Host        |
| --- | ----- | ----------- |
| 1   | user1 | 192.168.1.1 |
| 2   | user1 | 192.168.1.% |
| 3   | user1 | 192.168.1.3 |
| 4   | user1 | 192.168.%   |
| 5   | user1 | %           |

그리고 다음 3개의 계정을 로그인한다고 생각해보자.

1. `'user1'@'192.168.1.3'`
   - 테이블의 순서로 판단하면 2번 호스트로 인식될거로 판단된다.
   - 하지만 `MySQL`은 가장 구체적인 일치 항목을 우선 선택한다.
   - 따라서 정확하게 일치하는 3번 user1으로 접속이 된다.
2. `'user1'@'192.168.2.3'`
   - 가장 구체적인 일치 항목인 4번으로 접속된다.
3. `'user1'@'localhost'` - 이것 또한 유일한 일치 항목인 5번으로 접속된다.
   위의 예시를 통해 알 수 있는 사실은 `MySQL`은 테이블에 존재하는 순서대로 매핑하는 것이 아닌(순서 무관), 가장 구체적으로 일치하는 항목을 기준으로 접속을 시킨다.

## 3.2 사용자 계정 관리

### 3.2.1 시스템 계정과 일반 계정

`MySQL 8.0`부터 계정은 `SYSTEM_USER` 권한의 유무에 따라 시스템 계정(`System Account`)과 일반 계정(`Regular Account`)으로 구분된다.

> 시스템 계정은 백그라운드 스레드(내부 시스템)와는 관련이 없다.

시스템 계정은 시스템 계정과 일반 계정을 관리(Creation, Deletion, and Modification) 할 수 있지만, 일반 계정은 불가능.

시스템 계정은 아래와 같은 작업을 수행할 수 있다.

- 계정 관리(계정 생성 및 삭제, 그리고 계정의 권한 부여 및 제거)
- 다른 [[CS지식/DB/커넥션과 세션.md|세션(Connection)]] 또는 그 세션에서 실행 중인 쿼리를 강제 종료
- [[CS지식/DB/StoredProgram.md|스토어드 프로그램]] 생성시 `DEFINER`를 타 사용자로 설정

시스템 계정은 DBA들이 가지고, 일반 계정은 개발자와 같은 일반 사용자들이 가진다.
두개의 계정은 `SYSTEM_USER` 권한을 차별적으로 부여하기 위해 도입된다.

> `'root'@'localhost'`외에 내장된 계정들 (자동으로 잠겨져있는 계정)
>
> 1. `'mysql.sys'@'localhost'` : `MySQL 8.0`부터 기본으로 내장된 `sys` 스키마의 객체들의 `DEFINER`로 사용되는 계정
> 2. `'mysql.session'@'localhost'`: `MySQL` 플러그인이 서버로 접근할 때 사용되는 계정
> 3. `'mysql.infoschema'@'localhost'`: `information_schema`에 정의된 뷰의 `DEFINER`로 사용되는 계정

### 3.2.2 계정 생성

`MySQL 5.7`까지는 `GRANT` 명령으로 권한을 부여와 동시에 계정 생성이 가능했음.
`MySQL 8.0`부터는 계정의 생성은 `CREATE USER`, 권한 부여는 `GRANT` 명령으로 구분해서 실행.
`CREATE USER`로 계정 생성할 때는 아래와 같은 옵션을 설정할 수 있다.

- 계정의 인증 방식과 비밀번호
- 비밀번호 관련 옵션(비밀번호 유효기간, 비밀번호 이력 개수, 비밀번호 재사용 불가 기간)
- 기본 역할(Role)
- SSL 옵션
- 계정 잠금 여부

`CREATE USER` 예시

```sql
CREATE USER 'user'@'%'
IDENTIFIED WITH 'mysql_native_password' BY 'password'
REQUIRE NONE
PASSWORD EXPIRE INTERVAL 30 DAY
ACCOUNT UNLOCK
PASSWORD HISTORY DEFAULT
PASSWORD REUSE INTERVAL DEFAULT
PASSWORD REQUIRE CURRENT DEFAULT;
```

#### `IDENTIFIED WITH`

- 사용자의 인증 방식과 비밀번호를 설정한다.
- 키워드(`IDENTIFIED WITH`) 뒤에는 반드시 인증 방식(인증 플러그인의 이름)을 명시
- 기본 인증은 `IDENTIFIED WITH 'password'`로 명시
  대표적인 4가지 인증 방식

1. `Native Pluggable Authentication`
   - 5.7버전까지 기본으로 사용되는 방식
   - 단순한 비밀번호를 Hashing(SHA-1)해서 저장
   - Client가 보낸 값이랑 비교
2. `Caching SHA-2 Pluggable Authentication`
   - `Native Pluggable Authentication`과 비슷
   - 차이점은 Hashing algorithm(SHA-2)
   - 1번은 항상 동일 해시값이 출력되지만, 이 방식은 수천번 시행해서 동일한 키값이라도 달라짐
   - 시간 소모가 커, 해시 결괏값을 메모리에 Caching
   - 사용을 위해서 SSL/TLS or RSA key pair 필수라 SSL 옵션 활성화 해아함
3. `PAM Pluggable Authentication`
   - `UNIX`, `LINUX` 비밀번호 or `LDAP(Lightweight Directory Access Protocol)`같은 외부 인증 사용할 수 있게 해주는 인증 방식
   - 엔터프라이즈 에디션만 사용 가능
4. `LDAP Pluggable Authentication`
   - `LDAP`를 이용한 외부 인증 사용 가능
   - 엔터프라이즈 에디션에서만 사용 가능

`MySQL 8.0`부터는 2번이 기본 인증 방식이라, 이전 버전과 호환성을 생각한다면 1번을 따로 세팅해야 한다. 세팅 명령어는 아래와 같다.

```sql
SET GLOBAL default_authentication_plugin="mysql_native_password"
```

#### `REQUIRE`

- `MySQL` 서버에 접속할 때 암호화된 SSL/TLS 채널을 사용할지 여부를 설정
- 설정 안하면 비암호화 채널로 연결
- `Caching SHA-2 Authenticaion` 인증 방식을 사용하면, 암호화된 채널만으로 접속 가능(`REQUIRE` 설정안해도)

#### `PASSWORD EXPIRE`

- 비밀번호의 유효기간을 설정하는 옵션
- Default는 `default_password_lifetime` 시스템 변수에 저장된 기간으로 설정
  설정 가능한 옵션은 다음과 같음.

1. `PASSWORD EXPIRE`: 계정 생성과 동시에 비밀번호의 만료 처리
2. `PASSWORD EXPIRE NEVER`: 계정 비밀번호의 만료 기간이 없음
3. `PASSWORD EXPIRE DEFAULT`: `default_password_lifetime` 시스템 변수에 저장된 기간으로 비밀번호의 유효기간을 설정
4. `PASSWORD EXPIRE INTERVAL n DAY`: 비밀번호의 유효 기간을 오늘부터 n일자로 설정

#### `PASSWORD HISTORY`

- 한 번 사용했던 비밀번호를 재사용하지 못하게 설정하는 옵션
  설정 가능한 옵션은 다음과 같음

1. `PASSWORD HISTORY DEFAULT`: `password_history` 시스템 변수에 저장된 개수만큼 비밀번호 이력을 저장하며, 저장된 이력이 있으면 재사용 X
2. `PASSWORD HISTORY n`: 비밀번호의 이력을 최근 n개까지만 저장, 저장된 이력에 남아있는 비밀번호는 재사용할 수 없다.
   비밀번호 이력들은 `password_history` 테이블에 저장한다.

#### `PASSWORD REUSE INTEVAL`

- 한 번 사용했던 비밀번호의 재사용 금지 기간을 설정
  설정 가능한 옵션은 다음과 같다.

1. `PASSWORD REUSE INTERVAL DEFAULT`: `password_reuse_interval` 변수에 저장된 기간으로 설정
2. `PASSWORD REUSE INTEVAL n DAY`: n일자 이후에 비밀번호를 재사용할수 있게 설정

#### `PASSWORD REQUIRE`

- 비밀번호가 만료되어 새로운 비밀번호로 변경할 때 현재 비밀번호를 필요로 할지 말지를 결정하는 옵션
  설정 가능한 옵션은 아래와 같다.

1. `PASSWORD REQUIRE CURRENT`: 비밀번호를 변경할 때 현재 비밀번호를 먼저 입력하도록 설정
2. `PASSWORD REQUIRE OPTIONAL`: 비밀번호를 변경할 때 현재 비밀번호를 입력하지 않아도 되도록 설정
3. `PASSWORD REQUIRE DEFAULT`: `password_require_current` 시스템 변수의 값으로 설정

#### `ACCOUNT LOCK / UNLOCK`

- 계정 생성 시 또는 `ALTER USER` 명령을 사용해 계정 정보를 변경할 때 계정을 사용하지 못하게 잠글지 여부

1. `ACCOUNT LOCK`: 계정을 사용하지 못하게 잠금
2. `ACCOUNT UNLOCK`: 잠긴 계정을 다시 사용 가능 상태로 잠금 해제

## 3.3 비밀번호 관리

### 3.3.1 고수준 비밀번호

`MySQL` 서버는 위에서 살펴봤듯이 비밀번호 재사용 금지, 유효기간 설정 등 많은 기능뿐만 아니라 비밀번호를 쉽게 유추할 수 있는 단어들이 사용되지 않게, 글자의 조합을 강제하거나 금칙어를 만들 수 있다.

이 기능을 사용할라면 `validate_password` 컴포넌트를 이용하면 된다. 설치 명령어는 아래와 같다.

```sql
-- 컴포넌트 설치
INSTALL COMPONENT 'file://component_validate_password;';

-- 설치된 컴포넌트 확인
SELECT *  FROM mysql.component;
```

`vlaidate_password`의 비밀번호 정책은 크게 다음 3가지 중에서 선택 가능하다. default는 medium

- `LOW`: 비밀번호의 길이만 검증
- `MEDIUM`: 비밀번호의 길이를 검증하며, 숫자와 대소문자, 그리고 특수문자의 배합을 검증
- `STRONG`: `MEDIUM`레벨의 검증을 모두 수행하며, 금칙어가 포함됐는지 여부까지 검증

금칙어 파일은 한 줄씩 기록해서 저장한 텍스트 파일로 작성하면 된다.

### 3.3.2 이중 비밀번호

`MySQL` 서버의 이중 비밀번호 기능은 하나의 계정에 대해 2개의 비밀번호를 동시에 설정할 수 있다.
2개의 비밀번호는 다음으로 구분된다.

- Primary (최근에 설정된 비밀번호)
- Secondary (이전 비밀번호)

```sql
-- // 비밀번호를 "ytrewq"로 설정
ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password';

-- // 비밀번호를 "qwerty"로 변경하면서 기존 비밀번호를 세컨더리 비밀번호로 설정
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;
```

설정한 비밀번호 2개를 아무거나 기입해도 로그인이 된다.
Dual Password를 생성하고 모든 Application의 DB connection 비밀번호를 변경하면, 보안상 Secondary는 삭제한다.

### 3.4 권한(Privilege)

`MySQL 5.7` 버전까지 권한은 글로벌(Global) 권한과 객체 단위의 권한으로 구분되었음.

- 글로벌 권한 : 데이터베이스나 테이블 이외의 객체에 적용되는 권한
- 객체 단위의 권한 : 데이터베이스나 테이블을 제어하는 데 필요한 권한
  `MySQL 8.0`부터는 동적 권한도 추가됐다.
- 동적 권한 서버가 시작되면서 동적으로 생성하는 권한.
  - ex : 서버의 컴포넌트나 플러그인의 권한
- 정적 권한 : `MySQL` 서버의 소스코드에 고정적으로 명시돼 있는 권한

### 3.5 역할(Role)

`MySQL 8.0`부터는 권한을 묶어서 역할을 사용할 수 있게 됐다.
리눅스에서 파일 권한을 부여하는거랑 비슷하다.
사용자 계정이 재로그인하면 초기화된다.(이건 시스템 변수로 설정 가능)
실제로 계정과 권한은 큰 차이가 없이 취급받음.

# 읽으면서 들은 질문

- 엔터프라이즈급 MySQL 서버에만 지원되는 기능이 많은거 같다. 우리가 개발환경에서(백엔드 개발자일 때) 이중 고려해야 될점은 뭐가 있을까 ? 한번 고민해보는것도 나쁘지 않을거 같다.

# 처음본 개념 정리

[커넥션과 섹션](../Appendix/connection_session.md)
[StoredProgram](../Appendix/stored_program.md)
