# Real Mysql Chapter4

## 4.1 Mysql 엔진 아키텍처
- MySql 서버는 크게 MySql 엔진과 스토리지 엔진으로 구분할 수 있다.

### 4.1.1.1 MySql 엔진
- 클라이언트로부터의 접속 및 쿼리 요청을 처리하는 `컨넥션 핸들러` `Sql 파서` 및 `전처리기`, 쿼리의 최적화된 실행을 위한   
`옵티마이저가` 중심을 이룬다.

### 4.1.1.2 스토리지 엔진 (InnoDb, MyISam)
- 요청된 Sql 문장을 분석하거나 최적화하는 등 DBMS 의 두뇌에 해당하는 처리를 수행
- 실제 데이터를 디스크 스토리지에 저장하거나 데이터를 읽어오는 부분을 전담
- MySql 엔진은 하나지만 스토리지 엔진은 여러 개를 동시에 사용할 수 있다.

### 4.1.1.3 핸들러 API
- MySql 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때는 각 스토리지 엔진에 쓰기 또는 읽기를 요청하는데, 이러한   
  요청을 핸들러 요청이라고 하고, 여기서 사용되는 API를 핸들로 API 라고 한다. InnoDB 스토리지 엔진 또는 이 핸들러 API를 이용해   
  MySql 엔진과 데이터를 주고받는다.
- `show glbal status linke 'Handler%;`

### 4.2.1 MySql 스레딩 구조
- Mysql 서버는 프로세스 기반이 아니라 스레드 기반으로 작동 (포그라운드, 백그라운드 스레드로 구분)

### 4.1.2.1 포그라운드 스레드(클라이언트 스레드)
- 사용자가 요청하는 쿼리 문장을 처리
- 사용자가 작업을 마치고 커넥션을 종료하면 해당 커넥션을 담당하던 스레드는 스레드 캐시로 돌아간다.
- 이미 스레드 캐시에 일정 개수 이상의 대기 중인 스레드가 있다면 스레드를 종료시켜 일정 개수를 유지

###4.1.2.2 백그라운드 스레드
- InnoDB 는 다음과 같이 여러 가지 작업이 박그라운드로 처리된다.
  - 인서트 버퍼를 병합
  - 로그를 디스크로 기록
  - InnoDB 버퍼 풀의 데이터를 디스크에 기록
  - 데이터를 비퍼로 읽음
  - 잠금이나 데드락을 모니터링
- 가장 중요한 것은 `로그 스레드`와 버퍼의 데이터를 디스크로 내려쓰는 작업을 처리하는 `쓰기 스레드`
- 사용자의 요청을 처리하는 중 데이터의 쓰기 작업은 지연되어 처리될 수 있지만 데이터의 읽기 작업은 절대 지연될 수 없다.
- InnoDB 는 대부분 쓰기 작업을 `버퍼링해서 일괄 처리하는 기`능을 사용한다.
- Insert, Update, Delete 쿼리로 데이터가 변경되는 경우 완전히 저장될 때까지 기다리지 않아도 된다.

### 4.1.3 메로리 할당 및 사용 구조
- MySql 에서 사용되는 메로리 공간은 크게 `글로벌 메모리 영역`과 `로컬 메모리 영역`으로 구분
- 글로벌 메모리 영역의 모든 메모리 공간은 서버가 시작되면서 운영체제로부터 할당된다.

### 4.1.3.1 글로벌 메모리 영역
- 클라이언트 스레드의 수와 무관하게 하나의 메모리 공간만 할당(필요에 따라 2개 이상 할당 가능, 클라이언트의 스레드 수와 무관)
- 모든 스레드에 의해 공유되며 대표적인 글로벌 메모리 영역은 아래와 같다.
  - 테이블 캐시
  - InnoDB 버퍼 풀
  - InnoDB 어댑티브 해시 인덱스
  - InnoDB 리두 로그 버퍼

### 4.1.3.2 로컬 메모리 영역
- 세션 메모리 영역이라고 표현하며, [클라이언트 스레드](#4122--)스레드가 쿼리를 처리하는 데 사용하는 메모리 영역
- 대표적인 로컬 메모리 영역
  - 정렬 버퍼
  - 조인 버퍼
  - 바이너리 로그 캐시
  - 네트워크 버퍼
- 클라이언트 스레드가 사용하는 메모리 공간이라고 해서 클라이언트 메모리 영역이라고도 한다.
- 클라이언트와 MySql 서버와의 커넥션을 세션이라고 하기 때문에 로컬 메모리 영역을 세션 메모리 영역이라고 표현한다.
- 각 클라이언트 스레드별로 `독립적으로 할당`되며 절대 공유되어 사용되지 않는다.

### 4.1.4플로그인 스토리지 엔진 모델
- MySql 에서 쿼리가 실행되는 과정은 거의 대부분의 작엽이 MySql 엔진에서 처리되고, 데이터 읽기/쓰게 작업만 스토리지 엔진에 의해   
  처리된다.
- MySql 엔진이 각 스토리지 엔진에게 데이터를 읽어오거나 저장하도록 명령하려면 반드시 핸들러를 통해야한다.
- 실질적인 group by 나 order by 복잡한 처리는 스토리지 엔진 영역이 아니라 MySql 엔진의 처리 영역인 쿼리 실행기 에서 처리된다.

### 4.1.5 컴포넌트
- 8.0 버전부터는 기존의 플로그인 아키텍처의 단점을 보안하기 위해 컴포넌트 아키텍처가 지원된다.
  - 플로그인은 오직 MySql 서버와 인터페이스할 수 있고, 플로그인끼리는 통신할 수 없음
  - 플로그인 MySql 서버의 변수나 함수를 직접 호출하기 때문에 안전하지 않음(캡슐화X)
  - 플로그인은 상호 의존 관계를 설정할 수 없어서 초기화가 어렵다.

### 4.1.6 쿼리 실행 구조

### 4.1.6.1 쿼리 파서
- 사용자 요청으로 들어온 쿼리 문장을 토킨(MySql 이 인식할 수 있는 최소단위의 어휘나 기호)으로 분리해 트리 형태의 구조로 만들어  
  내는 작업을 의미
- 쿼리 문장의 기본 문법 오류는 이 과정에서 발견되고 사용자에게 오류 메세지를 전달.

### 4.1.6.2 전처리기
- 파서 과정에서 만들어진 트리를 기반으로 쿼리 문장에 구조적 문제점이 있는지 확인
  - 테이블 이름, 컬럼 이름, 내장함수의 존재 여부와 접근 권항 등을 확인
  - 실제 존재하지 않거나 권한이 없다면 개체의 토큰은 이 과정에서 걸러진다.
### 4.1.6.3 옵티마이저
- 사용자의 요청으로 들어온 쿼리 문장을 `저렴한 비용`으로 `가장 빠르게 처리`할지를 결정하는 역할을 담당

### 4.1.6.5 실행 엔진
- 만들어진 계획대로 각 핸들러에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할을 수행
- 핸들러에게 명령을 내리는 역할을 담당
### 4.1.6.5 핸들러(스토리지 엔진)
- MySql 서버의 가장 밑단에서 MySql 실행 엔진의 요청에 따라 데이터를 디스크로 저장하거나 읽어오는 역할을 담당
- 핸들러는 스토리지 엔진을 의미

### 4.1.8 쿼리 캐시(Deleted)
- Sql 의 실행 결과를 메로리에 캐시, 동일한 Sql 쿼리가 실행되면 테이블을 읽지 않고 즉시 결과를 반환하여 빠른 성능을 보였다.
- 테이블의 데이터가 변경되면 캐시에 저장된 결과 중에서 변경된 테이블과 관련된 것을 모두 삭제해야 했다.
- 8.0 으로 올라오면서 완전히 제거되고 관련 시스템 변수도 모두 제거되었다.

### 4.1.9 스페드 풀
- 엔터프라이즈 에디션에서는 기능을 제공한다.
- 내부적으로 사용자의 요청을 처리하는 스레드 개수를 줄여서 동시 처리되는 요청이 많다하더라도 제한된 개수의 스레 처리에만 집중할 수    
  있게 해서 서버의 자원 소모를 줄이는 것이 목적
- 제한된 수의 스레드만으로 CPU 가 처리하도록 적절히 유도한다면 CPU 의 프로세스 친화도도 높이고 운영체제 입장에서는 불필요한   
  컨텍스트 스위치를 줄여서 오버헤드를 낮출 수 있다.

### 4.1.10 트랜잭션 지원 메타데이터
- 테이블의 구조 정보와 스토어 프로그램 등의 정보를 `데이터 딕셔너리` 또는 `메타 데이터`라고 한다.
- 8.0 이전에서는 파일을 이용하여 메타데이터를 관리했지만 메타데이터는 생성 및 변경 작업이 트랜잭션을 지원하지 않기 때문에 예상치   
  상황에 비정상적으로 종료되면 일관되지 않은 상태로 남는 문제가 발생
- 8.0 부터는 이런 메타 데이터를 모두 `InnoDB의 데이블에 저장`하도록 개선
- MySql 서버가 동작하는 데 기본적으로 필요한 테이블들을 묶어서 시스템 테이블이라고, 이 역시 InnoDB 스토리지 엔진을 사용하여 관리
- 시스템 테이블과 데이터 딕셔너리 정보를 모두 모아서 `mysql DB` 에 저장하고 있다.(mysql.ibd 테이블스페이스에 저장)

## 4.2 InnoDB 스토리지 엔진 아키텍처
- InnoDB는 MySql 에서 사용할 수 있는 스토리지 엔진 중 거의 유일하게 레코드 기반의 잠금을 제공
- 높은 동시성 처리가 가능하고 안정적이며 성능이 뛰어나다.

### 4.2.1 프라이머리 키에 의한 클러스터링
- InnoDB 의 모든 테이블은 기본적으로 PK 를 기준으로 클러스터링되어 저장된다.
- 즉, PK 값의 순서대로 디스크에 저장된다는 뜻이며, 모든 세컨더리 인덱스는 레코드의 주소 대신 PK의 값을 논리적인 주소로 사용한다.
- MyISAM 테이블에서는 프라이머리 키와 세컨더리 인덱스는 구조적으로 아무런 차이가 없다. PK는 유니크 제약을 가진 세컨더리 인덱스일 뿐
### 4.2.2. 외래키 지원
- InnoDB 에서 외래 키는 부모 자식 테이블 모두 해당 컬럼에 인덱스 생성이 필요하고, 변경 시에는 부모 테이블이나 자식 테이블에 데이터   
  가 있는지 체크하는 작업이 필요하므로 잠금이 여러 테이블로 정파되고, 그로 인해 데드락이 발생할 때가 많으므로 개발할 때도 외래 키의    
  의 존재에 주의하는 것이 좋다.
- ```SET foreign_key_checks=OFF|ON``` 로 잠시 외래키 체크 작업을 멈출 수 있다.
### 4.2.3 MVCC(Multi Version Concurrency Control)
- MVCC 의 가장 큰 목적은 잠금을 사용하지 않는 일관된 읽그를 제공하는 데 있다.
- InnoDB 는 언두 로그(Undo Log)를 이용하여 이 기능을 구현한다.
- 멀티 버전(Multi Version): 하나의 레코드에 대해 여러 개의 버전이 동시에 관리된다는 의미
- InnoDB 스토리지 엔진을 이용한 테이블 변경
  - A 사용자가 데이터 변경을 시도한다면 해당 요청은 InnoDB 버퍼 풀에 적재되어 쓰기 실행을 기다리거나 혹은 바로 실행된다.(커밋전)
  - 이때 B 사용자가 해당 데이터를 조회, 격리 상태에 따라 다른 값을 반환
    - READ_UNCOMMITTED: 아직 커밋되지 않은 버퍼 풀이나 데이터 파일로부터 변경되지 않은 데이터를 읽어서 반환
    - READ_COMMITTED 이상: InnoDB 버퍼 풀이나 데이터 파일에 있는 내용 대신 변경되기 전 내용을 보관하고 있는 언두 영역의    
      데이터를 반환
- 이러한 과정을 DBMS 에서는 MVCC 라고 한다.
- Commit 명령을 실행하면 지금의 상태를 영구적인 데이터로 만든다.
- 롤백이 실행되면 언두 영역에 있는 백업된 데이터를 InnoDB 버퍼 풀로 다시 복구하고, 언두 영역의 내용을 삭제한다.
- 커밋이 된다고 언두 영역의 백업 데이타가 항상 바로 삭제되는 것은 아니다. 필요로 하는 트랜잭션이 더이상 없을 때 삭제 된다.

### 4.2.4 잠금 없는 일관된 읽기(None-Locking Consistent Read)
- MVCC 기술을 이용해 `잠금을 걸지 않고 읽기 작업`을 수행
- InnoDB 에서 읽기 작업은 다른 트랜잭션이 가지고 있는 잠금을 기다리지 않고, 읽기 작업이 가능
- `SERIALIZABLE` 을 제외한 격리 수준에서 `INSERT` 와 연결되지 않은 순수한 읽기 작업은 다른 트랜잭션의 변경 작업과 관계없이    
  항상 잠금을 대기하지 않고 바로 실행된다.
- 변경되기 전의 데이터를 읽기 위해 언두 로그를 사용, 이러한 방식을 `잠금 없는 일관된 읽기` 라고 한다.

### 4.2.7 InnoDB 버퍼 풀
- InnoDB 스토리지 엔진에서 가장 핵심적인 부분, 디스크의 데이터 파일이나 인덱스 정보를 메로리에 캐시해 두는 공간
- 쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 버퍼 역할도 함께 한다.
- 내부적으로 128MB 청크 단위로 쪼개어 관리, 크기를 줄일 때는 128MB 단위로 처리된다.

### 4.2.7.2 버퍼 풀의 구조
- 거대한 메모리 공간을 페이지 크기의 조각으로 쪼개어 InnoDB 스토리지 엔진이 데이터를 필요로 할 때 해당 데이터 페이지를 읽어서   
  각 조각에 저장한다.
- LRU 리스트와 플러시 리스트, 프리 리스트라는 3개의 자료구조를 관리한다.
  - 프리 리스트: InnoDB 버퍼 풀에서 실제 사용자 데이터로 채워지지 않은 비어 있는 페이지들의 목럭 사용자의 쿼리가 새롭게 디스크의    
    데이터 페이지를 읽어와야 하는 경우 사용된다.
  - LRU 리스트: 디스크로부터 한 번 읽어온 페이지를 최대한 오랫동안 버퍼 풀의 메모리에 유지해서 디스크 읽기를 최소화하기 위해 관리
- 처음 한 번 읽힌 데이터 페이지가 이후 자주 사용된다면 그 데이터 페이지는 InnoDB 버퍼 풀의 MRU 영역에서 계속 살아남게 되고   
  반대로 거의 사용되지 않는다면 새롭게 디스크에서 읽히는 페이지들에 밀려서 LUR 의 끝으로 밀려나 결국 제거된다.
- 데이터가 변경되면 변경 내용을 리두 로그에 기록하고 버퍼 풀의 데이터 페이지에도 변경 사항을 버퍼 풀에도 반영

### 4.2.7.3 버퍼 풀과 리두 로그
- InnoDB 버퍼 풀은 데이터베이스 서버의 성능 향상을 위해 데이터 캐시와 쓰기 버퍼링이라는 두가지 용도가 있다.
- 버퍼 풀의 메모리 공간만 단순히  늘리는 것은 데이터 캐시 기능만 향상시키는 것이다.
- 버퍼 풀은 디스크에서 읽은 상태로 전혀 변경되지 않은 클린 페이지와 변경된 데이터를 가진 더티 페이지도 가지고 있다.
- 더티 페이지는 디스크와 메모리의 데이터 상태가 다리기 때문에 언젠가는 디스크로 기록돼야 한다. 하지만 더티 페이지는 버퍼 풀에 무한정   
  머무를 수있는 것은 아니다.

### 4.2.7.4.1 플러시 리스트 플러시
- 스토리지 엔진은 주기적으로 플러시 리스트 플러시 함수를 호출해서 플러시 리스트에서 오래전에 변경된 데이터 페이지를 순서대로 디스크에 동기화 하는 작업을 수행한다.
- 디때 언제부터 얼마나 많은 더티 페이지를 한 번에 디스크로 기록하느냐에 따라 사용자의 쿼리 처리가 악영향을 받지 않으면서 부드럽게 처리된다.
- 버퍼 풀은 클린 페이지뿐만 아니라 사용자의 DML 에 의해 변경된 더티 페이지도 함께 갖고 있다.

### 4.2.7.4.2 LRU 리스트 플러시
- LRU 리스트에서 사용 빈도가 낮은 데이터 페이지를 제거해서 새로운 페이지들을 읽어올 공간을 만들어야 하는데, 이를 위해 사용되는 함수
- InnoDB 스토리지 엔진은 LRU 리스트를 스캔하며 더티 페이지는 디스크에 동기화, 클린 페이지는 즉시 프리 리스트로 페이지를 옮긴다.

### 4.2.8 Double Writer Buffer
- 리두 로그는 리두 로그 공간의 낭비를 막기 위해 페이지의 변경된 부분만 기록한다.. 이로 인해 스토리지 엔진에서 더티 페이지를 디스크   
  파일로 플러시할 때 일부만 기록되는 현상을 파셜페이지 또는 톤 페이지라고 하는데 이런 현상은 하드웨어의 오작동이나 시스템의 비정상   
  종료 등으로 발생할 수 있다.
- 더티 페이지를 디스크에 지록하기 전에 우선 더티 페이지를 기록한 후 스토리지 엔진은 각 더티 페이지를 파일의 적당한 위치에 하나씩   
  랜덤으로 쓰기를 실행한다.
- 비정상적으로 종료된 경우, 재시작될 때 항상 DoubleWriterBuffer 의 내용과 데이터 파일의 페이지들을 모두 비교해서 다른 내용을   
  담고 있는 페이지가 있으면 DoubleWriterBuffer 의 내용을 데이터 파일의 페이지로 복사한다.

### 4.2.9 언두 로그
- 트랜잭션과 격리 수준을 보장하기 위해 DML 로 변경되기 이전 버전의 데이터를 별도로 백업하는데, 이렇게 백언된 데이터를 언두로그라고 한다.
- 언두 로그의 사용
  - 트랜잭션 보장(롤백 대비용)
    - 트랜잭션이 록백되면 트랜잭션 도중 변경된 데이터를 변경 전 데이터로 복구해야 하는데, 이때 언두 로그에 백업해둔 이전 버전의 데이터를 이용해 복구한다.
  - 격리 수준 보장(격리 수준을 유지하면서 높은 동시성을 제공)
    - 특정 커넥션에서 데이터를 변경하는 도중에 다른 커넥션에서 데이터를 조회하면 트랜잭션 격리 수준에 맞게 변경중인 레코드는 읽지 않고 언두 로그에 백업해둔 데이터를 읽어서 반환한다.

### 4.2.9.2 언두 테이블스페이스 관리
- 언두 로그가 저징되는 공간

### 4.2.10 체인지 버퍼
- 변경해야 할 인덱스 페이지가 버퍼 풀에 있으면 바로 업데이트를 수행하지만 그렇지 않고 디스크로부터 읽어와서 업데이트해야 한다면 이를   
  이를 즉시 실행하지 않고 임시 공간에 저장해 두고 바로 사용자에게 결과를 반환하는 형태로 성능을 향상시키게 되는데, 이때 사용하는    
  임지 메모리 공간을 체인지 버퍼라고 한다.

### 4.2.11 리두 로그 및 로그 버퍼
- 리두 로그는 ACID 중에서 D(Durable) 에 해당하는 영속성과 밀접하게 연관되어 있다.
- 비정상 적으로 종료됐을 때 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전장치
- 데이터베이스 서버는 데이터 변경 내용을 로그로 먼저 기록한다.
- 비정상 종료가 발생하면 리두 로그의 내용을 이용해 데이터 파일을 다시 서버가 종료되기 직전의 상태로 복구
  - 커밋됐지만 데이터 파일에 기록되지 않은 데이터 -> 리두 로그에 저장된 데이터를 데이터 파일에 그대로 복사
  - 롤백됐지만 데이터 파일에 이미 기록된 데이터 -> 변경이 커밋, 롤백 됐는지 실행 중간 상태였는지 확인하는 용도로 사용

### 4.2.12 어댑티브 해시 인덱스
- InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스
- 자주 읽히는 데이터 페이지의 키 값을 이용해 해시 인덱스를 만들고 필요할 때마다 어탭티브 해시 인덱스를 검색해서 레코드가 저장된   
  페이지를 즉시 찾아갈 수 있다.
- B-Tree 를 루트노드부터 리프 노드까지 찾아가는 비용이 없어진다.
- '인덱스 키 값'과 해당 인덱스 키 값이 저장된 '데이터 페이지 주소' 쌍으로 관리. '인덱스 키 값':'데이터 페이지 주소'
- 인덱스 키 값 = 'B-Tree 인덱스의 고유ID와 B-Tree 인덱스의 실제 키 값' 조합
- 성능 향상에 크게 도움되지 않는 경우
  - 디스크 읽기가 많은 경우
  - 특정 패턴의 쿼리가 많은 경우(조인 LIKNE 패턴 검색)
- 성능 향상에 도움되는 경우
  - 디스크의 데이터가 버퍼 풀 크기와 비슷한 경우(디스크 읽기가 많지 않은 경우)
  - 동등 조건 검색이 많은 경우(동등 비교와 IN 연산자)
  - 쿼리가 데이터 중에서 일부 데이터에만 집중되는 경우
