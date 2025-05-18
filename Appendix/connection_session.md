# Connection
- `connection`은 DB와 Client 간의 TCP socket을 통한 물리적인 연결(Channel)이다.
	- Client가 DB Server에 연결을 요청하면 커넥션을 맺게 된다.
- 한 `connection`에서 여러 개의 트랜잭션을 동시에 실행시킬 수 없다.
# Session
- 사용자가 DB에 연결된 상태를 의미한다.
	- `session`은 DB와 Client 간의 논리적인 연결.
	- `connection` 하나 당 여러 개(0이상)의 `session`을 가질 수 있다.
- `connection`을 통한 모든 요청은 세션을 통해서 실행.
	- 쉽게 말해서 실제 SQL 쿼리 실행은 `session`을 통해 이루어진다.
- 사용자가 `connection`을 닫으면 세션이 종료된다.
	- `connection`이 닫히면 `session`은 종료되고, commit되지 못한 트랜잭션은 롤백된다.

# 정리
Client가 DB와 `connection`을 맺고 사용자 인증이 성공하면 바로 `session`이 생긴다.
Client는 `session`을 통해 Query를 날리고, `session` 안에서 {`시작`, `유지`, `종료`}됨.
이외에도 `state`를 갖는 작업(트랜잭션, 변수 등)은 세션에 종속.
