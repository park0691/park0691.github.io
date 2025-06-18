---
title: (MS-SQL) 여러 행 문자열 합치기
date: 2025-06-14 15:00 +0900
categories: DB
tags: ms-sql,stuff,value,row
---
이번 포스트에서는 MS-SQL에서 여러 행의 문자열을 합치는 방법에 대해 알아보고자 한다.

## 예시 데이터
```sql
CREATE TABLE #TMP_TBL(
	JOB VARCHAR(30)
	, ENAME VARCHAR(30)
);

INSERT INTO #TMP_TBL
VALUES ('PRESIDENT', 'KING')
	, ('MANAGER', 'CLARK')
	, ('MANAGER', 'BLAKE')
	, ('MANAGER', 'CLARK')
	, ('MANAGER', 'JONES')
	, ('ANALYST', 'SCOTT')
	, ('ANALYST', 'FORD')
	, ('CLERK', 'SMITH')
	, ('CLERK', 'SMITH')
	, ('SALESMAN', 'ALLEN')
	, ('SALESMAN', 'MARTIN')
	, ('SALESMAN', 'WARD')
	, ('SALESMAN', 'MARTIN')
	, ('SALESMAN', 'TURNER')
	, ('CLERK', 'ADAMS')
	, ('CLERK', 'JAMES')
	, ('CLERK', 'MILLER');

```
![images](https://1drv.ms/i/c/9251ef56e0951664/IQRtXgAZjoHrTr3mLAb6qfEjAaG1z1uBZidJMZveggLFxoc)

왼쪽의 데이터를 같은 JOB 별로 ENAME을 중복 제거하여 가나다 역순으로 정렬하여 문자열을 합쳐보려 한다.

여러 행의 컬럼 값을 하나의 컬럼으로 합치기 위해 SQL Server 2017 버전 이상에서는 `STRING_AGG()` 함수를 사용하며, SQL Server 2017 이전 버전에서는 `FOR XML PATH` 서브 쿼리를 사용한다.

2017 이전 버전의 방법을 적용하기 위한 `STUFF, FOR XML PATH` 함수에 대해 알아보고 적용한 후 SQL Server 2017 이상 버전에서 `STRING_AGG()` 이용 방법에 대해 각각 알아보자.

## STUFF()
- 입력한 문자열의 특정 위치, 길이를 지정하여 다른 문자로 치환한다.
- `STUFF(문자열, 위치, 길이, 치환할 문자)`의 형태로 사용한다.

다음은 ENAME 칼럼의 위치 1에서 길이 1만큼의 문자를 'X'로 치환한다.

```sql
SELECT JOB
	, STUFF(ENAME, 1, 1, 'X') AS ENAME
FROM #TMP_TBL;
```

![images](https://1drv.ms/i/c/9251ef56e0951664/IQQlsdobUuuKS449t6V9vgU4AU2JDq_qEsaXBFsKfJvN0Hc)


## FOR XML PATH()
- 실행한 쿼리의 결과를 XML 형태로 표현하여 반환한다.
- `FOR XML PATH([Row Element Name])`의 형태로 사용한다.

```sql
SELECT *
FROM #TMP_TBL
FOR XML PATH;
```

```xml
<row>
  <JOB>PRESIDENT</JOB>
  <ENAME>KING</ENAME>
</row>
<row>
  <JOB>MANAGER</JOB>
  <ENAME>CLARK</ENAME>
</row>
<row>
  <JOB>MANAGER</JOB>
  <ENAME>BLAKE</ENAME>
</row>
<row>
  <JOB>MANAGER</JOB>
  <ENAME>CLARK</ENAME>
</row>
<row>
  <JOB>MANAGER</JOB>
  <ENAME>JONES</ENAME>
</row>
...
```
Row Element Name이 안 들어간 경우 `<row>`로 감싸지는 것을 확인할 수 있다.

```sql
SELECT *
FROM #TMP_TBL
FOR XML PATH('replaced')
```

```xml
<replaced>
  <JOB>PRESIDENT</JOB>
  <ENAME>KING</ENAME>
</replaced>
<replaced>
  <JOB>MANAGER</JOB>
  <ENAME>CLARK</ENAME>
</replaced>
<replaced>
  <JOB>MANAGER</JOB>
  <ENAME>BLAKE</ENAME>
</replaced>
<replaced>
  <JOB>MANAGER</JOB>
  <ENAME>CLARK</ENAME>
</replaced>
<replaced>
  <JOB>MANAGER</JOB>
  <ENAME>JONES</ENAME>
</replaced>
...
```
Row Element Name 파라미터 `'replaced'`를 주어 row 태그가 해당 명칭으로 바뀐 것을 확인할 수 있다.


## 여러 행의 문자열 합치기 (SQL Server 2017 이전 버전)

1. FOR XML PATH 적용<br/>
XML 형태로 데이터를 포현하되 Root Element Name 옵션을 `''` 빈 문자로 주어 `<row></row>`을 빈 값으로 치환하여 없앤다.

    ```sql
    SELECT ENAME
    FROM #TMP_TBL
    FOR XML PATH('')
    ```
    ```xml
    <ENAME>KING</ENAME>
    <ENAME>CLARK</ENAME>
    <ENAME>BLAKE</ENAME>
    <ENAME>CLARK</ENAME>
    <ENAME>JONES</ENAME>
    ...
    ```

    > 태그 없이 값을 결합하려면?<br/>
    > `FOR XML PATH`와 `TYPE` 및 `value()`를 조합하면 순수 문자열을 얻을 수 있다.
    > 
    > ```sql
    > SELECT
    >   (SELECT ENAME
    >   FROM #TMP_TBL
    >   FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)') AS NAME
    > ```
    > ![images](https://1drv.ms/i/c/9251ef56e0951664/IQRT0tUKR_qYTbZ2hDPVTW6SATEVc2OcFh4svgkqpHqhVpg)
    > - `value()` : XML을 일반 문자열로 변환한다.
    {: .prompt-tip }

2. 구분자 추가<br/>
각 행 ENAME 칼럼의 문자가 합쳐질 때 각 행들을 구분하기 위해 넣을 구분자 `/`를 추가한다.
추가 칼럼의 명칭이 `null`이 되어 `<name></name>` 칼럼명이 사라진다.

    ```sql
    SELECT '/' + ENAME
    FROM #TMP_TBL
    FOR XML PATH('')
    ```

    > `&gt; &lt;`를 문자 그대로 나타내려면?<br/>
    > ```sql
    > SELECT '>' + ENAME
    > FROM #TMP_TBL
    > FOR XML PATH('')
    > ```
    > ![images](https://1drv.ms/i/c/9251ef56e0951664/IQSK6L4lwweGTpfkA_I8vg4iAYYk3N2o_QkfMcL3DluBons)
    >
    > `>`를 추가하면 `&gt;`로 표시된다. 다음과 같이 `TYPE, value()`를 조합하면 순수 문자열을 얻을 수 있다.
    > ```sql
    > SELECT
    >     (SELECT '>' + ENAME
    >     FROM #TMP_TBL
    >     FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)') AS NAME
    > ```
    > ![images](https://1drv.ms/i/c/9251ef56e0951664/IQRFTAG-ppPTSpCW-CRy_imVAflpHgRITL_cMx6il6wBaMY)
    {: .prompt-tip }

3. STUFF()를 사용하여 맨 앞 구분자 제거

    ```sql
    SELECT STUFF(
        (SELECT '/' + ENAME
        FROM #TMP_TBL
        FOR XML PATH('')), 1, 1, ''
    )
    ```
    
    ![images](https://1drv.ms/i/c/9251ef56e0951664/IQSqraYNI09cR6MC-IlFfCniASJxa7xQx2fKoAaqBHAdKro)
    
    STUFF() 함수를 사용하여 문자열의 첫 번째 구분자 `/`를 빈 문자로 치환한다.

4. 같은 JOB 끼리 묶기
    ```sql
    SELECT 
        JOB
        , STUFF(
            (SELECT '/' + ENAME
            FROM #TMP_TBL t
            WHERE t.JOB = a.JOB
            FOR XML PATH('')), 1, 1, ''
        )
    FROM #TMP_TBL a;
    ```
    ![images](https://1drv.ms/i/c/9251ef56e0951664/IQSBPc-HFBKERZohp0qa6I1rASQhOX_qNotEbIqPaBhUQrA)
    

5. 동일 행 제거, DISTINCT
    ```sql
    SELECT DISTINCT
        JOB
        , STUFF(
            (SELECT '/' + ENAME
            FROM #TMP_TBL t
            WHERE t.JOB = a.JOB
            FOR XML PATH('')), 1, 1, ''
        )
    FROM #TMP_TBL a;
    ```
    ![images](https://1drv.ms/i/c/9251ef56e0951664/IQS-SkmYtZhoTpv2jov5K10bAc37gbMv78Is-5HtX-9N88g)
    

6. 동일한 ENAME 제거 및 역순 정렬
    ```sql
    SELECT DISTINCT
        JOB
        , STUFF(
            (SELECT '/' + ENAME
            FROM #TMP_TBL t
            WHERE t.JOB = a.JOB
            GROUP BY ENAME
            ORDER BY ENAME DESC
            FOR XML PATH('')), 1, 1, ''
        ) AS NAMES
    FROM #TMP_TBL a;
    ```
    ![images](https://1drv.ms/i/c/9251ef56e0951664/IQQ5BaY-J2u1SYq7yoBR6KBoAecrqe2tuynYZvj2Zx0rvgw)
    
    GROUP BY를 통해 중복 ENAME 제거하며, ORDER BY를 통해 ENAME 역순 정렬하였다.

## STRING_AGG() (SQL Server 2017 이상 버전)
SQL Server 2017 이상 버전에서는 여러 행의 컬럼 값을 한 칼럼으로 합치기 위한 `STRING_AGG()` 함수를 지원한다.

GROUP BY 절과 함께 사용하며, ORDER BY 절을 사용하여 정렬 가능하며 ORDER BY 절은 생략할 수 있다.

```sql
STRING_AGG("합칠컬럼명", "구분자") WITHIN GROUP(ORDER BY "컬럼명")
```

### 기본 사용법
```sql
SELECT JOB
	, STRING_AGG(ENAME, ',') NAMES
FROM #TMP_TBL
GROUP BY JOB
```
![images](https://1drv.ms/i/c/9251ef56e0951664/IQRW-RK0ib47R6wI1nf4k003Abts7-GXmgIzyHvwiOZEvow)

ENAME을 정렬하려면

```sql
SELECT JOB
	, STRING_AGG(ENAME, '/') WITHIN GROUP(ORDER BY ENAME DESC) AS NAMES
FROM #TMP_TBL
GROUP BY JOB
```
![images](https://1drv.ms/i/c/9251ef56e0951664/IQRKTSov5huGQ6XEM_PGH9cjAVSM8000eUo6O3JC-mYnCis)

그러나 중복 행을 제거할 수 없으므로 STRING_AGG 함수 사용 전에 중복 항목을 제거해야 한다.

```sql
SELECT JOB
	, STRING_AGG(ENAME, '/') WITHIN GROUP(ORDER BY ENAME DESC) AS NAMES
FROM (
	SELECT DISTINCT JOB
		, ENAME
	FROM #TMP_TBL
) t
GROUP BY JOB
```
![images](https://1drv.ms/i/c/9251ef56e0951664/IQSkDcpvHG9JRbsVw3_zDUBmATb21f2kq61KCkpuQx8aNJM)

## References
- https://sungeune97.tistory.com/96
- https://da-new.tistory.com/13
- https://gent.tistory.com/344
- https://gent.tistory.com/345