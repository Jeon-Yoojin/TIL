## Issue

### [Database] 페이지네이션 방법론

1. 문제

   1. typeorm 사용 중 case-insensitive한 정렬을 위해 queryBuilder로 생성한 쿼리에 LOWER()를 추가함
   2. offset pagination 방식을 적용하기 위해 skip, take를 사용했을 때, typeorm에서 다음과 같은 오류 발생

   ```
   "* alias was not found. Maybe you forgot to join it?"
   ```

2. 고민한 방법
   1. skip, take 대신 limit, offset 사용하기
      - **1-N 관계에서 JOIN을 이용할 경우** 문제 발생
      - 예를 들어 Writer(1) - Post(N) 관계에서 JOIN 결과 하나의 writer에 대해 여러 중복된 행이 생성된다. 즉, 원하는 수만큼 writer를 제대로 가져오지 못함
      - 그럼 skip과 take는 어떻게 제대로 가져오는거지?라는 의문이 들 수 있는데,
      - skip & take는 내부적으로 **1+1 쿼리 방식**을 사용한다. 위의 예시를 가져오면 writer만 대상으로 offset/limit을 적용하고, 원하는 writer + post의 묶음을 정상적으로 불러온다.
   2. 백엔드에서 직접 페이지네이션하기
      - 모든 결과를 불러와서 전달할 때 직접 가져올 page를 start-end로 정한 후, slice로 잘라서 보여준다.
      - 이 방법의 경우 당연히 모든 데이터를 불러와서 처리하는 것이기 때문에 메모리를 많이 사용하게 되고, 성능 상의 이슈가 발생할 수 있다.
   3. DB 설정 변경하기
      1. colloate 설정 변경 - 어떻게 문자열을 비교하고 정렬할지 정의한 규칙인 collate 설정을 변경한다.

         ```sql
         -- 적용된 collate를 확인한다 --
         select datname, datcollate from pg_database;
         ```

         이때 case-insensitive order를 적용하고 싶다면, 해당 규칙을 포함하는 설정으로 변경하면 된다. 쿼리에 특정한 collate 값을 지정하는 것도 가능하다.

         단, 이미 DB에 데이터가 존재한다면 전체 설정은 변경할 수 없다. (백업 후 진행해야 함)

      2. icu 설정 - PostgreSQL 12 이상부터 ICU를 지원한다. ICU는 기존의 OS locale 기반 collation보다 **더 정밀하고 국제화된 정렬 규칙**을 제공
   4. \_lower column 생성
      - case-insensitive하게 정렬 / 검색하고자 하는 column에 대하여, column_low라는 이름으로 새로운 컬럼을 생성한다.
      - 이 컬럼은 오직 정렬 또는 검색을 위해 사용되는 컬럼으로, 데이터가 CREATE / UPDATE될 때마다 변경되어야 한다.
        - DB Trigger를 통해 위의 작업을 자동화
      - 하지만, 인덱스를 탈 수 있기에 속도는 더 빠르다.
   5. 쿼리에 alias 만들고 select 하는 구문 추가

      - 소문자로 변환된 컬럼에 대해 별칭을 지정하고, `SELECT` 구문에 추가한다.
      - `ORDER BY`에서 이 별칭을 사용해 정렬한다.

      ```sql
      query
        .addSelect('LOWER(writer.name)', 'writer_name_lower')
        .orderBy('writer_name_lower', 'ASC');
      ```
3. 해결
   1. 지금 상황은..
      - 특정 컬럼에 대해서만 대소문자 구분 없는 정렬이 필요
      - 전체 DB의 collate 변경이나 백업/초기화를 되도록 피하고 싶음
      - 1-N 관계에서의 정확한 페이지네이션 결과도 보장돼야 함 (JOIN에 의한 오류가 발생하지 않아야 함)
   2. 해결 방법
      - alias + select 를 이용, query에 구문을 추가하는 함수를 만들어 전역에서 사용하도록 했다.
   3. 왜 이 방식이 효과가 있었을까?
      - TypeORM에서 `skip` / `take`가 동작하지 않던 문제는 별칭(alias)이 select되지 않아 인식되지 못해 발생했다.
      - 이를 `addSelect()`로 명시적으로 포함시키면서 문제가 해결되었다.
