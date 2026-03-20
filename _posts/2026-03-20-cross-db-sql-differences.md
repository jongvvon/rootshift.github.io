---
title: "이기종 DB 혼용 환경에서 살아남기 — MySQL부터 DB2까지 SQL 차이 정리"
date: 2026-03-20 19:40:00 +0900
categories: [인프라 실무]
tags: [db, sql, mysql, mssql, postgresql, db2, oracle, mariadb, informix, 이기종]
---

## 배경

운영 환경에 DB가 하나면 얼마나 편할까.

현실은 그렇지 않다. 시스템마다 도입 시기가 다르고, 벤더가 다르고, 요구사항이 달랐기 때문에 한 환경 안에 MySQL, MSSQL, PostgreSQL, DB2, MariaDB, Informix, Oracle, SQLite가 공존하는 일이 생긴다.

각 DB마다 SQL 문법이 조금씩 다르다. 기본 SELECT야 같지만, 조금만 복잡해지면 바로 에러가 난다. 이 글은 **이기종 DB 환경에서 자주 마주치는 문법 차이**를 정리한 실무 참고용이다.

---

## 1. 현재 날짜/시간 조회

가장 자주 쓰는데 DB마다 다 다르다.

| DB | 현재 날짜 | 현재 날짜+시간 |
|---|---|---|
| MySQL / MariaDB | `CURDATE()` | `NOW()` |
| MSSQL | `CAST(GETDATE() AS DATE)` | `GETDATE()` |
| PostgreSQL | `CURRENT_DATE` | `NOW()` or `CURRENT_TIMESTAMP` |
| Oracle | `TRUNC(SYSDATE)` | `SYSDATE` |
| DB2 | `CURRENT DATE` | `CURRENT TIMESTAMP` |
| Informix | `TODAY` | `CURRENT` |
| SQLite | `DATE('now')` | `DATETIME('now')` |

```sql
-- MySQL / MariaDB
SELECT NOW(), CURDATE(), CURTIME();

-- MSSQL
SELECT GETDATE(), CAST(GETDATE() AS DATE), CAST(GETDATE() AS TIME);

-- PostgreSQL
SELECT CURRENT_TIMESTAMP, CURRENT_DATE, CURRENT_TIME;

-- Oracle
SELECT SYSDATE, TRUNC(SYSDATE) FROM DUAL;

-- DB2
SELECT CURRENT TIMESTAMP, CURRENT DATE FROM SYSIBM.SYSDUMMY1;

-- Informix
SELECT CURRENT, TODAY FROM SYSTABLES WHERE TABID = 1;
```

---

## 2. 문자열 연결 (Concatenation)

| DB | 연결 방법 |
|---|---|
| MySQL / MariaDB | `CONCAT(a, b)` |
| MSSQL | `a + b` 또는 `CONCAT(a, b)` (2012+) |
| PostgreSQL | `a \|\| b` 또는 `CONCAT(a, b)` |
| Oracle | `a \|\| b` 또는 `CONCAT(a, b)` |
| DB2 | `a \|\| b` 또는 `CONCAT(a, b)` |
| Informix | `a \|\| b` |
| SQLite | `a \|\| b` |

```sql
-- MySQL: CONCAT 함수 필수 (|| 는 OR 연산자로 해석됨 주의!)
SELECT CONCAT(first_name, ' ', last_name) FROM users;

-- MSSQL: + 연산자 사용 (NULL 포함 시 결과 NULL 주의)
SELECT first_name + ' ' + last_name FROM users;
-- NULL 안전하게: ISNULL(first_name, '') + ' ' + ISNULL(last_name, '')

-- PostgreSQL / Oracle / DB2
SELECT first_name || ' ' || last_name FROM users;
```

---

## 3. NULL 처리 함수

| DB | NULL 대체 함수 |
|---|---|
| MySQL / MariaDB | `IFNULL(col, 대체값)` |
| MSSQL | `ISNULL(col, 대체값)` |
| PostgreSQL | `COALESCE(col, 대체값)` |
| Oracle | `NVL(col, 대체값)` |
| DB2 | `COALESCE(col, 대체값)` or `VALUE(col, 대체값)` |
| Informix | `NVL(col, 대체값)` |
| SQLite | `IFNULL(col, 대체값)` or `COALESCE(col, 대체값)` |

> `COALESCE`는 ANSI SQL 표준이라 대부분 DB에서 동작한다. 이기종 환경이라면 `COALESCE` 쓰는 게 이식성 좋다.

```sql
-- 어느 DB에서나 동작
SELECT COALESCE(phone, '미등록') FROM users;

-- Oracle / Informix 전통 방식
SELECT NVL(phone, '미등록') FROM users;

-- MSSQL 전통 방식
SELECT ISNULL(phone, '미등록') FROM users;
```

---

## 4. 페이징 (LIMIT / OFFSET)

대용량 데이터 조회할 때 꼭 필요한데, DB마다 문법이 제각각이다.

```sql
-- MySQL / MariaDB / PostgreSQL / SQLite
SELECT * FROM logs ORDER BY created_at DESC LIMIT 10 OFFSET 20;

-- MSSQL (2012 이상)
SELECT * FROM logs ORDER BY created_at DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- MSSQL (구버전 - TOP + 서브쿼리)
SELECT TOP 10 * FROM (
    SELECT TOP 30 * FROM logs ORDER BY created_at DESC
) sub ORDER BY created_at ASC;

-- Oracle (12c 이상)
SELECT * FROM logs ORDER BY created_at DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle (구버전 - ROWNUM)
SELECT * FROM (
    SELECT a.*, ROWNUM rn FROM (
        SELECT * FROM logs ORDER BY created_at DESC
    ) a WHERE ROWNUM <= 30
) WHERE rn > 20;

-- DB2
SELECT * FROM logs ORDER BY created_at DESC
LIMIT 10 OFFSET 20;
-- 또는
FETCH FIRST 10 ROWS ONLY;

-- Informix
SELECT FIRST 10 SKIP 20 * FROM logs ORDER BY created_at DESC;
```

---

## 5. 조건부 표현식 (CASE / DECODE)

```sql
-- ANSI SQL (대부분 DB 공통)
SELECT
    CASE status
        WHEN 1 THEN '활성'
        WHEN 0 THEN '비활성'
        ELSE '알 수 없음'
    END AS status_name
FROM users;

-- Oracle 전용 DECODE (간결하지만 Oracle만 됨)
SELECT DECODE(status, 1, '활성', 0, '비활성', '알 수 없음') FROM users;

-- MySQL IF 함수
SELECT IF(status = 1, '활성', '비활성') FROM users;

-- MSSQL IIF (2012+)
SELECT IIF(status = 1, '활성', '비활성') FROM users;
```

---

## 6. 자동 증가 컬럼 (Auto Increment)

테이블 생성 시 PK 자동 증가 방식:

```sql
-- MySQL / MariaDB
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100)
);

-- MSSQL
CREATE TABLE users (
    id INT IDENTITY(1,1) PRIMARY KEY,
    name VARCHAR(100)
);

-- PostgreSQL
CREATE TABLE users (
    id SERIAL PRIMARY KEY,  -- 또는 BIGSERIAL
    name VARCHAR(100)
);
-- PostgreSQL 10+ IDENTITY
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100)
);

-- Oracle
-- 시퀀스 별도 생성 필요 (12c 이전)
CREATE SEQUENCE users_seq START WITH 1 INCREMENT BY 1;
CREATE TABLE users (id NUMBER PRIMARY KEY, name VARCHAR2(100));
-- INSERT 시: INSERT INTO users VALUES (users_seq.NEXTVAL, '홍길동');

-- Oracle 12c+
CREATE TABLE users (
    id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR2(100)
);

-- DB2
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100)
);

-- Informix
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- SQLite
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT
);
```

---

## 7. 문자열 자르기 (SUBSTRING)

```sql
-- MySQL / MariaDB / MSSQL / PostgreSQL / SQLite
SELECT SUBSTRING(name, 1, 3) FROM users;  -- 1번째부터 3자

-- Oracle / DB2 / Informix
SELECT SUBSTR(name, 1, 3) FROM users;

-- MSSQL 전용 (LEFT, RIGHT)
SELECT LEFT(name, 3), RIGHT(name, 3) FROM users;

-- MySQL도 LEFT, RIGHT 지원
SELECT LEFT(name, 3) FROM users;
```

---

## 8. 형변환 (CAST / CONVERT)

```sql
-- ANSI SQL 표준 (대부분 DB 지원)
SELECT CAST(price AS VARCHAR(20)) FROM products;
SELECT CAST('2026-03-20' AS DATE) FROM DUAL;

-- MSSQL 전용 CONVERT (포맷 지정 가능)
SELECT CONVERT(VARCHAR(10), GETDATE(), 120);  -- 'YYYY-MM-DD' 형식

-- MySQL
SELECT CONVERT(price, CHAR);
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d');

-- Oracle
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD') FROM DUAL;
SELECT TO_DATE('2026-03-20', 'YYYY-MM-DD') FROM DUAL;
SELECT TO_NUMBER('12345') FROM DUAL;

-- PostgreSQL
SELECT price::VARCHAR FROM products;  -- :: 캐스팅 연산자
SELECT CAST(price AS TEXT) FROM products;
```

---

## 9. 더미 테이블 (FROM 절 없는 SELECT)

DB마다 FROM 절이 필수인지 아닌지 다르다.

```sql
-- MySQL / MariaDB / MSSQL / PostgreSQL / SQLite
-- FROM 절 없이 가능
SELECT 1+1, NOW(), 'hello';

-- Oracle: FROM DUAL 필수
SELECT 1+1, SYSDATE, 'hello' FROM DUAL;

-- DB2: FROM SYSIBM.SYSDUMMY1 필요
SELECT 1+1 FROM SYSIBM.SYSDUMMY1;

-- Informix: FROM SYSTABLES WHERE TABID=1 사용
SELECT TODAY FROM SYSTABLES WHERE TABID = 1;
```

---

## 10. OUTER JOIN 문법

ANSI JOIN 문법은 대부분 지원하지만, 구형 DB나 Oracle 구버전은 다르다.

```sql
-- ANSI 표준 (권장 - 대부분 DB 호환)
SELECT a.name, b.dept_name
FROM users a
LEFT OUTER JOIN departments b ON a.dept_id = b.id;

-- Oracle 구버전 (+) 표기법
SELECT a.name, b.dept_name
FROM users a, departments b
WHERE a.dept_id = b.id(+);  -- (+)가 붙은 쪽이 outer

-- MSSQL 구버전 *= 표기법 (현재는 지원 안 함)
-- 사용 금지, ANSI JOIN으로 대체 필요
```

---

## 실무 팁

**1. 이기종 환경에선 ANSI SQL 표준을 우선 쓴다**
`COALESCE`, `CASE WHEN`, `LEFT/RIGHT OUTER JOIN` 등 표준 문법은 대부분 DB에서 동작한다.

**2. DB별 함수명 차이 메모해두기**
특히 날짜 함수와 문자열 함수가 DB마다 크게 다르다. 자주 쓰는 것만 따로 정리해두면 시간 절약된다.

**3. NULL 처리 주의**
MSSQL에서 `NULL + '문자열' = NULL`이 된다. 문자열 연결 시 NULL 값이 있으면 결과 전체가 NULL이 되는 함정이 있다. `ISNULL` 또는 `COALESCE`로 미리 처리해야 한다.

**4. 페이징은 DB 버전 확인 필수**
Oracle이나 MSSQL은 버전마다 페이징 문법이 다르다. 구버전 서버가 남아있는 환경이라면 `ROWNUM`이나 `TOP` 방식도 알아둬야 한다.

---

## 한 줄 요약

> DB는 달라도 데이터는 같다. 문법 차이만 알면 어디서든 쿼리 짤 수 있다.
