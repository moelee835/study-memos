# Oracle query processing
https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/sql-processing.html#GUID-8CF633B1-EAC4-47C7-9189-C479ADEF1FFA

## About SQL Processing
SQL processing은 SQL 파싱, 최적화, 행소스 생성, SQL 문장 생성으로 구성된다.

![SQL 프로세스](C:/temp/오라클_SQL_실행과정.PNG)


SQL 문장에 따라 몇 단계는 생략될 수 있음.

### SQL Parsing
파싱 단계에선 SQL을 조각으로 나누고, 다른 루틴에서 처리할 수 있는 데이터 구조로 연관시킴.

데이터베이스는 애플리케이션에 의해 지시받았을 때만 파싱을 수행하고 스스로는 수행할 수 없다. 이를 통해 애플리케이션에 의해서만 파싱 횟수를 감소시킬 수 있다.

애플리케이션이 SQL 문장을 DB에 주면, parse call이 발생한다. 그럼 DB는 SQL 문장 실행을 준비하게 된다.

Parse Call은 **커서**를 만들거나 열게 됨. Session specific private SQL area를 다루는 용도로 사용됨.

Session specific private SQL area는 파싱된 SQL 문장과 다른 실행용 정보들을 포함하고 있음.

- Session specific private SQL area가 뭔가?

커서랑 Session --- SQL area는 Program Global Area에 있음. (PGA)

Parse call 동안에 데이터베이스는 문장 실행 전 에러가 있는지는 체크함.
어떤 에러는 파싱에 의해 검출되지 않음.
예를 들면, 데드락이나 데이터 변환(Conversion)에 의한 오류가 있음.


#### Syntax check
DB는 문장 구문의 유효성을 검증함.
SQL 구문을 망가뜨리는 규칙 위반을 검사를 실패하게 함.

```sql
SQL> SELECT * FORM employees;
SELECT * FORM employees
         *
ERROR at line 1:
ORA-00923: FROM keyword not found where expected
```

위와 같이 FROM을 FORM으로 쓰는 경우임.

#### Semantic Check
의미 체크 단계. 여기선 문장이 의미 있는지를 체크함. 예를들면 문장 안의 컬럼이나 개체가 존재하는지를 체크.

```
SQL> SELECT * FROM nonexistent_table;
SELECT * FROM nonexistent_table
              *
ERROR at line 1:
ORA-00942: table or view does not exist
```

#### Shared pool check
파싱 과정 중에는 Shared pool check 단계가 있고 여기선 문장 처리 과정 중 자원을 많이 소모하는 단계를 건너 뛸 수 있는지 체크함. 즉, Full parse하는 비용을 피하기 위한 작업임.
단계의 마지막에 DB는 해싱 알고리즘을 통해 각 SQL 문장들의 해시값을 계산함.

해시 결과를 13자리 문자열로 표현한 값임.

이 값은 `SQL_ID`라는 이름으로 저장됨. `V$SQL.SQL_ID`
이 해시값은 오라클 DB 버전이 달라도 동일하게 계산됨.

유저가 SQL문을 제출하면 DB는 해시값으로 Shared SQL area를 검색해서 이미 파싱된 문장이 있는지 검색함.

다음 요소들로 해시값들이 구분됨.
1. 메모리 주소.
오라클 DB는 SQL_ID를 사용해서 룩업 테이블에서 Read를 수행함. 이 해시가 룩업 테이블에 존재한다면 해당 SQL의 메모리 주소를 획득함.

2. 실행 계획의 해시값
하나의 SQL문은 Shared pool에서 여러개의 실행 계획을 가질 수 있음. 보통 각 계획은 다른 해시값을 가짐. 만약 동일한 SQL ID가 여러 계획의 해시값을 가지고 있다면, 데이터베이스는 이 존재를 알 수 있음.

이를 통해 문장의 메모리 주소와 실행 계획을 조회하거나 새로운 계획을 생성함.

파싱 작업은 다음 두 가지 케이스로 진행됨.
1. **Hard parse**
만약 다시 사용할 SQL이 없다면, 새로운 실행 버전을 빌드해야함.
이 과정에서 데이터베이스는 데이터 딕셔너리를 체크하기 위해서 라이브러리 캐시와 데이터 딕셔너리 캐시를 매우 많이 접근하게 됨. 
이 단계에서 DB는 '래치'라고 불리는 직렬 통신 장치를 통해서 요구된 개체에 접근함. Latch에 대한 '경쟁'은 SQL 실행시간을 증가시키고, 동시성을 감소시킴.
래치는 여러 프로세스가 동일한 개체에 접근할 때, 데이터를 보호하기 위한 lock 장치의 일종이다. 이를 통해 해당 객체가 처리되는 동안에 다른 사용자가 이 객체를 변경하지 못하게 막는 역할을 함.
정리 : Hard parse 단계에서 제출된 SQL 문을 완전히 다시 빌드하는데, 이때 이 SQL문에서 사용하는 처리 대상을 latch 라는 장치로 접근하고, 동시에 다른 사용자들이 접근하지 못하게 막음.
2. **Soft parse**
hard parse가 아닌 모든 파싱을 의미함. shared pool에서 재사용할 수 있는 SQL을 발견한 경우, 오라클은 이 코드를 재사용 함. 이런 재사용을 **library cache hit** 라고 함.
soft parse는 그 작업량에 따라 효율성이 달라지는데, 예를들어 세션 공유 SQL 영역을 잘 설정하면 필요한 latch의 수를 줄일 수 있음. 따라서 동시성 제어가 감소하여 더 효율적인 수행이 가능함.

Soft parse는 일반적으로 더 선호됨. 왜냐하면 hard parse에 비해 최적화와 row source generation 단계를 건너 뛰고 바로 실행이 가능하기 때문임.

![Shared pool check](C:/temp/shared_pool_check.PNG)

1. SGA와 PGA는 다른 영역이다.
PGA는 프로세스별 메모리 공간이다. private SQL Area가 이 안에 포함된다.

2. SGA는 오라클 인스턴스 전체가 공유하는 공간이다.


Program Global Area는 오라클 데이터베이스 서버 내부에서 각 서버 프로세스마다 할당되는 전용 메모리 영역임. Spring boot 서버가 실행된다고 PGA를 갖지 않음. 오직 Oracle 데이터베이스 서버 프로세스 내에 생성됨.
Oracle DB 서버의 응용 프로그램(서버 프로세스, 백그라운드 프로세스)이 수행되는 머신의 메모리.

구조 : 스프링 부트 서버 => Oracle DB 서버(PGA) => SGA
