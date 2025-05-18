# Definition
`Procedure`, `Function`, `Trigger`, `Event`를 총칭하며, `MySQL` 서버 내부에 저장되고, 특정 시점 또는 조건에서 실행되는 코드 블록.
# StoredProcedure
### 목적
- 반복적이고 복잡한 작업을 서버 측에서 명령어 단위로 실행
	- 일반 프로그래밍 언어에서의 함수 ? 메소드 ? 개념인듯
	- 비즈니스 로직을 DB 계층에 캡슐화하는 도구.
- 파라미터를 통해 외부에서 동적 값 입력 가능
### 특징
- `Call procedure_name(args..)`으로 실행
- DML에서 사용 가능
- `IN`, `OUT`, `INOUT` 매개변수 사용
	- `IN` 
		- 호출자 => 프로시저(Procedure)로 값을 전달
		- 프로시저 내에서 값 읽기 전용(값 수정은 가능, but 호출자에겐 반영 안됨)
		- 기본 파라미터 타입 (Default 타입이라, 명시 안 하면 `IN`으로 처리)
		- 내부에서 수정해도 호출자 변수에 영향 없음
	- `OUT`
		- 프로시저 => 호출자로 값 전달
		- 프로시저가 내부에서 값을 설정해야 호출자가 값을 받을 수 있음
		- 호출 시 변수 초기화 상태는 무시
			- 프로시저 시작시 자동으로 `NULL` 할당
		- 프로시저 내에서 반드시 `SET` 필요
	- `INOUT`
		- 호출자 => 프로시저로 값을 전달하고
		- 프로시저 => 호출자로 수정된 값을 반환
		- 변수는 입력값으로 시작하여, 내부에서 변경되면 호출자 변수에도 반영
```sql	
-- IN 매개변수
CREATE PROCEDURE square_val(IN num INT)
BEGIN
  SELECT num * num AS squared;
END;

SET @a = 5;
CALL square_val(@a);  -- 출력: 25

-- OUT 매개변수 
CREATE PROCEDURE get_constant(OUT val INT)
BEGIN
  SET val = 42;
END;

CALL get_constant(@v);
SELECT @v;  -- 결과: 42

--INOUT 매개변수
CREATE PROCEDURE add_ten(INOUT num INT)
BEGIN
  SET num = num + 10;
END;

SET @val = 15;
CALL add_ten(@val);
SELECT @val;  -- 결과: 25
```
- 여러 `SQL` 문 블록 구성 가능
- 로직 내 조건문, 반복문 가능(`IF`, `WHILE`, `LOOP`, `CASE` 등)

# StoredFunction
### 목적
- 하나의 값을 계산해서 반환하는 로직
- 일반 SQL query 내에서 함수처럼 사용 가능
### 특징
- 반드시 하나의 `RETURN` 값을 가짐
- `OUT`, `INOUT` 파타미터 없음. 오직 `IN`
- `CREATE FUNCTION`, `RETURNS` 문으로 정의
- 트리거, 이벤트에서는 사용 불가(제약 있음), DML에서 사용 불가.
# Trigger
### 목적
- 특정 테이블의 DML event 발생 시 자동 실행
- 데이터 무결성, 자동 값 설정 등 처리에 유용
### 특징
- `BEFORE` 또는 `AFTER` 발생 시점 지정
- `FOR EACH ROW` 행 단위로 실행
- `NEW`,`OLD` 키워드로 값 접근 가능
- 한 테이블당 동일 이벤트에 트리거 1개만 가능
# Event
### 목적
- 특정 시간이나 주기로 자동 실행되는 작업
- 스케쥴링 기능을 SQL 내에서 지원
### 특징
- `CREATE EVENT`, `ON SCHEDULE` 구문 사용
- `EVERY`, `AT`, `INTERVAL` 사용
- `EVENT SCHEDULER` 활성화 필요
```sql
--Event Scheduler 활성화 구문
SET GLOBAL event_scheduler = ON;
```

