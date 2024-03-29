## 인덱스 구조

### 범위 스캔

~~~ 
- 필요한 데이터만 빠르고 효율적으로 액세스할 목적으로 사용하는 오브젝트
~~~

###  인덱스 기본 구조

~~~ 
- 루트 블록 -> 브랜치 블록 -> 리프 블록으로 구성
- 인덱스 구성 컬럼은 모두 null인 레코드는 저장하지 않음
- 상위 레코드 불록 개수는 하위 블록 개수와 일치

주요 특징
- 리프 노드상의 인덱스 레코드와 테이블 레코드 간에는 1:1 관계
- 리프 노드상의 키 값과 테이블 레코드 키 값은 일치
- 리프 노드상의 레코드 개수는 하위 레벨의 블록 개수와 일치
- 리프 노드상의 키 값은 하위 노드가 갖는 값의 범위를 의미
~~~

### 인덱스 탐색

~~~ 
수직적 탐색 : 하위 블록으로 탐색
수평적 탐색 : 리프 블록에서 레코드 간의 논리적 순서로 탐색

수직적 탐색은 수평적 탐색을 위한 시작 지점을 찾는 과정
~~~

수직적 탐색 시작 -> 루트 블록 확인 -> 브랜치 블록 확인 -> 리프 블록 확인 -> 시작점 찾음

수평적 탐색 시작 -> 값 찾을 시 rowId를 이용하여 다른 컬럼 값을 읽음

**브랜치 블록 스캔**

- 브랜치 블록 스캔할 때는 뒤에서 부터 스캔

- 찾고자 하는 키 값보다 작은 엔트리를 따라 내려감 

**결합 인덱스 구조와 탐색**

- 인덱스 값이 두개 이상일 시 
- 하나의 인덱스 값 일치 뿐만 아니라 다음 인덱스 값도 확인해야하므로 탐색을 멈추지 않음



### ROWID 포맷

테이블 레코드의 물리적 위치 정보를 포함

즉, 테이블 레코드(row)를 찾아가는 데 필요한 주소 정보로 인덱스에 저장.

~~~
오라클 7버전(6btye) - 제한 rowid 포맷 : 블록, 로우, 데이터파일 번호 순으로 구성
오라클 8버전 이상 (10byte) - 확장 rowid 포맷 : 데이터 오브젝트, 데이터파일, 블록, 로우 번호 순으로 구성
~~~



## 인덱스 기본 원리

B*Tree(루트, 브랜치, 리프)를 정상적으로 사용하려면 수직적 탐색 과정을 거쳐야 함

즉, 인덱스 선두 컬럼이 조건절에 사용되지 않으면 범위 스캔의 시작점을 찾을 수 없으므로 옵티마이저는 **인덱스 전체를 스캔**하거나 **테이블 전체를 스캔**

### 인덱스 사용이 불가능하거나 범위 스캔이 불가능한 경우

~~~ 
- 인덱스 컬럼을 조건절에서 가공할 시
	- where substr(업체명, 1, 2) = '대한'
- 부정형 비교
	- where 직업 <> 학생 OR is not null
이 경우 정상적인 인덱스 범위 스캔은 불가능 but, index full scan은 가능
~~~

### 컬럼 조건절에 is null 혹은 is not null

~~~
where 절에 is null이나 is not null이 있을 시 정상적인 인덱스 범위를 탐색 못함
but, is null일 경우에도 index range scan 사용 가능 -> 해당 컬럼이 not null일 경우 옵티마이저는 이를 이미 알고 있어 가능
~~~

### 인덱스 컬럼의 가공

~~~
튜닝 시 가공으로 인덱스 탐색 가능
sbstr -> like
월급여 * 12 -> 월급여 = x/12
to_char(일시, 'yyymmdd') -> 일시 >= to_data
연령 || 직업 -> 연령 = x and 직업 = y
~~~

**인덱스 사용을 위한 가공**

~~~
- nvl(주문수량, 0) >= 100 -> 주문수량 >= 100
	- 100보다 크므로 이미 null인것 없지
- nvl(주문수량, 0) < 100 -> 
	- 주문수량이 not null 일시 nvl은 제거
	- 아니면 full scan 때려야하되 양 많으면 괜춘 but 작으면 FBI를 사용해야함
~~~

**튜닝 사례**

~~~
where A || B in ('1100') -> 튜닝 : ('1', '100')
~~~

**튜닝 사례2**

![1주차_튜닝사례2](https://user-images.githubusercontent.com/40616436/68543826-743e3880-03ff-11ea-9942-44c673f4a3fe.jpg)

### 묵시적 형변환

~~~ 
ex) 대상년월(+) = substr(파트너지원요청일자, 1, 6) -1
	- NL join은 outer 테이블이 먼저 드라이빙 됨(substr쪽)
	- substr쪽은 문자형에서 숫자형으로 변환 후 대상년월과 비교하는데 대상년월은 숫자형으로 형변환 이루어짐
	- why? 숫자형과 문자형 비교될 때는 숫자형이 우선
	- 고로 인해 형변환 후 인덱스 컬럼 탐색
	- 해결 : 대상년월(+) = to_char(add_months(to_data(파트너지원요청일자, 'yyyymmdd') -1) 'yyyymmdd')로 개발자가 먼저 형변환을 해줌
~~~

### 묵시적 형변환 시 주의사항

~~~
- 문자 -> 숫자로 형변환 시 오류 발생 가능성 높음 why? 문자를 숫자로 변환 못할 수도 있으므로.
	- where 숫자 = 문자 -> to_char(숫자) like 문자 || '%'

- decode 형변환 시
	- deocde(a, b, c, d)
	- decode할 시 데이터 타입은 세번째 인자 c에 의해 결정

- 또 다른 형변환
	- 문자와 숫자는 숫자형
	- 문자와 날짜는 날짜
~~~

**함수적 기반 인덱스(FBI)**

성능 이슈 발생으로 원인이 묵시적 형변환일 경우 일일이 바꿀 수가 없으면, FBI로 급한 불을 끄되, 권장할 만한것은 아님.



## 다양한 인덱스 스캔 방식

### Index Range Scan

~~~
- 수직적으로 탐색한 후 수평적으로 탐색
- 리프 블록의 필요한 범위만 스캔하는 방식
- 가장 일반적이고 정상적인 형태의 액세스 방식

- 인덱스를 구성하는 선두 컬럼이 조건절에 사용해야 함.
- 인덱스 스캔 범위 줄이고, 테이블로 액세스하는 횟수를 줄일 수록 성능 더 좋아짐.
- sort order by 연산 생략 및 min/max 값 추출 가능
~~~

### Index Full Scan

~~~
- 가장 왼쪽에 있는 리프 블럭으로 간 후 수평정 탐색
- 인덱스가 조건절에 없으면 옵티마이저는 우선적으로 Table Full Scan을 하지만, 대용량 테이블일 시 Idnex Full Scan을 고려함.

- Sort Order By 연산을 생략 가능 but, 힌트를 잘못 하용하게 되면 Table Full Scan보다 더 안 좋아 질 수 있음.
~~~

### Index Unique Scan

~~~
- 수직적 탐색만으로 데이터를 찾는 스캔 방식
- Unique 인덱스를 통해 = 조건으로 탐색하는 경우에 작동
- Unique 인덱스더라도 범위검색 조건(between, 부등호, like)로 검색할 시 Index Range Scan으로 처리
- 인덱스 a,b,c 중 a,c만 이용해도 Index Ragne Scan으로 검색
~~~

### Index Skip Scan

- 인덱스 선두 컬럼이 조건절로 사용되지 않으면, 기본적으로 Table Full Scan을 선택
- Table Full Scan을 보다 줄이기 위해 정렬된 결과를 쉽게 얻을 수 있다면 Index Full Scan을 사용

~~~
- 조건절이 빠져 있더라도 Index를 사용하는 스캔 방식
- 루트 또는 브랜치 블록에서 읽은 컬럼값 정보를 이용해 가능성 있는 리프 블록만 골라 액세스 하는 방식
- 항상 처음과 마지막은 방문함
- 리프 블록을 보면서 가능성 있기만 하면 방문을 함
	- 남 연봉 4000이하를 찾는데, 순차 탐색 중 남에서 여성으로 성별이 변경될 시 남성의 최대값도 한번 방문해야함 -> 이유는 남녀 말고 다른 성별이 있을 수 있기 때문
	- 또한, 처음과 마지막은 항상 방문! 성별이 무엇들이 있는지 모르니깐!

- a,b,c의 인덱스 컬럼이 적용되어 있는데, a 컬럼이 누락되고 b,c등 뒤 컬럼들이 많을 때 유용
- 두개의 선두 컬럼(a,b)가 없어도 사용 가능
- 선두 컬럼이 부등호, between, like 같은 범위검색일 경우에도 사용 가능
- index_ss(유도), no_index_ss(방지) 힌트 사용
~~~

**버퍼 Pinning을 이용한 skip 원리**

~~~
- 항상 그 위쪽에 있는 블록을 재방문해서 다음 방문할 블록에 주소 정보를 얻어야 함
- 재방문을 하더라도 Pinning을 한 상태이기 때문에 추가적인 블록 I/O는 발생하지 않음
~~~

**Index Skip Scan과 In-List Iterator와의 비교**

~~~
- 조건절에 in (남, 여)등으로 인덱스 탐색을 반복 수행한다는 뜻
- 해당 테이블의 성별이 남과 여로만 구성되어 있어야 함
- 즉, 옵티마이저가 성별을 이미 알고 있으므로 방문해야할 곳과 하지 말아야 할 곳을 더욱 알 수 있음.
~~~

### Index Fast Full Scan

~~~
- Index Full Scan보다 빠름.
- 인덱스 트리 구조를 무시하고 인덱스 세그먼트 전체를 멀티블록 리드 방식으로 탐색
- 논리적인 순서(오름, 내림차순 정렬)을 물리적인 순서로 재배치

주요 특징
- 대량의 인덱스 블록을 읽어야하는 상황에서 큰 효과를 발휘
- 쿼리에 사용되는 모든 컬럼이 인덱스 컬럼으로 되어있어야만 사용 가능
- index_ffs, no_index_ffs의 힌트를 사용
- 사용빈도가 낮고, 디스크 I/O가 많이 발생할 때 iffs 사용시 좋음
- 테이블에 컬럼개수가 많아 테이블보다 인덱스 크기가 현저히 작은 상황에서 효과를 발휘
~~~

### Index Range Scan Desc

~~~
- Index Range Scan의 내림 차순
- 옵티마이저가 알아서 인덱스를 거꾸로 읽을 실행계획을 수립
- 만약 max값을 구할 시 인덱스를 뒤에서부터 읽으므로 한건만 보면 됨! 매우 유용
	- 이 전에는 index_desc 힌트를 사용하거나 rownum <= 1 조건을 사용했음
~~~

### Index Combine 및 Index Join

~~~
[Index Combine]
- rowId 목록을 얻음 -> rowId 목록을 가지고 비트맵 인덱스 구조를 하나씩 만듦 -> bit-wise 오퍼레이션을 수행한 결과가 참인 비ㅡ값들을 rowid 값으로 환산하여 최종적으로 방문할 rowid 목록을 얻음
- 즉, 테이블 Random 액세스량을 줄이는 데에 목적이 있음.

[Index Join]
- 테이블에 액세스 없이 한 테이블에 속한 인덱스를 이용해서 결과집합을 만듬
- 크기가 작은 쪽 인덱스를 기준으로 큰 쪽 인덱스를 탐색하면서 같은 rowId일 시 결과집합에 포함
- where 절에 인덱스가 모두 포함되어야 함.
~~~

