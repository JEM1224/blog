---
title: "트랜잭션"
pubDatetime: 2024-01-06T09:52:57.000Z
full_path: "src/content/blog/transcation.md"
description: "null"
tags: 
  - "DB"
---

### TOC
## Transaction 트랜잭션

A가 B에게 20만원을 `이체`하는 작업이 있다고 할 때  

1. A 계좌로부터 20만원 인출
2. B 계좌로 20만원 입금

1번과 2번 과정이 둘 다 성공해야 `이체` 가 성공한다.   

이러한 단일 작업으로 묶어서 나눠질 수 없게 만든 것을 트랜잭션이라고 부른다.  

mysql로 트랜잭션을 구현해보기  

```sql
START TRANSATION;   // 트랜잭션 시작
UPDATE account SET balance = balance - 200000 WHERE id = 'A'; 
UPDATE account SET balance = balance + 200000 WHERE id = 'B';
COMMIT; // 트랜잭션 완료 &성공
```
`START TRANSACTION` 트랜잭션을 시작하여 로직을 수행한다.  
`COMMIT`은 트랜잭션을 종료하고 지금까지 작업한 내용을 DB에 영구적으로 저장하는것을 의미한다.  
만약 단일 작업 중 일부만 성공한다면 이체 작업은 DB에 반영되지 않고 `ROLLBACK`한다.  
`ROLLBACK`은 트랜잭션을 종료하고 지금까지 작업을 모두 취소하고 트랜잭션 이전 상태로 되돌리는것을 의미한다.  

### 일반적인 트랜잭션 사용패턴
1. 트랜잭션 시작
2. 로직 수행
3. 문제가 없다면 COMMIT
4. 문제가 발생했다면 트랜잭션을 ROLLBACK

### AUTOCOMMIT
`AUTOCOMMIT`이란 각각의 SQL문을 자동으로 트랜잭션 처리해주는것을 말한다. mysql에서는 기본적으로 활성화 되어있으며 다른 DBMS에서도 대부분 같은 기능을 제공한다.
SQL문이 성공적으로 실행되면 자동으로  `COMMIT` 하고 문제가 생기면 `ROLLBACK`한다.

mysql에서 `AUTOCOMMIT`이 활성화 되어있는지 확인해보기
```mysql
SELECT @@AUTOCOMMIT;
//1 - 활성화 
```
`START TRANSACTION` 트랜잭션이 시작되면 동시에 `AUTOCOMMIT`은 비활성화 된다.
`COMMIT/ROLLBACK`과 함께 트랜잭션이 종료되면 원래 `AUTOCOMMIT`상태로 돌아간다.


## 트랜잭션의 ACID
### Atomicity 원자성
- ALL or NOTHING : 트랜잭션은 논리적으로 쪼개질 수 없는 작업 단위이기 때문에 내부의 SQL문들이 모두 성공해야 한다.
### Consistency 일관성
- 트랜잭션의 결과는 항상 일관성이 있어야 한다. 

### Isolation 독립성
- 둘 이상의 트랜잭션이 동시에 실행될 때 하나의 트랜잭션은 다른 트랜잭션 연산에 끼어들 수 없다.
- DBMS는 여러 종류의 Isolation level을 제공한다
   - 높으면 엄격하게 격리시켜 동시성이 떨어져 DB의 성능이 줄어들게 된다.
### Durability 영구성
- 트랜잭션이 성공할 경우 결과는 영구적으로 반영되어야한다.
> 영구적으로 저장된다.
DB 시스템에 문제가 생겨도 COMMIT된 트랜잭션은 DB에 남아 있는다.(=비휘발성 메모리(HDD,SDD..)에 저장함)
### 읽기에는 트랜잭션을 안걸어도 될까?

## schedule 스케줄

ex) A계좌에서 20만원을 B에게 이체 할 때 

1. 트랜잭션 시작
2. A 계좌  read
3. A 계좌 write
4. B 계좌 read
5. B 계좌 write
6. 트랜잭션 종료

트랜잭션에서  2,3,4,5와 같은 작업 하나하나를 operation이라 부른다.
`r1(A)-w1(A)-r1(B)-w1(B)-c1` 
operation을 간소화하여 표현할 수 있다. 1는 트랜잭션 번호를 의미한다. 


ex) A가 B에게 20만원을 이체할 때 B도 본인 계좌에 30만원을 입금한다면 여러 형태의 실행이 있을 수 있다.
T1 : A가 B에게 20만원 이체
T2:  B가 본인계좌에 30만원 입금
이렇게 여러 트랜잭션들이 동시에 실행될 때 각 트랜잭션에 속한 operation들의 실행 순서를 `Schedule`이라고 한다.

schedule1: `r1(A)-w1(A)-r1(B)-w1(B)-c1-r2(B)-r2(B)-c2`  : T1 이 끝난 후  T2실행  
스케줄1은 트랜잭션이 겹치지 않고 순차적으로 실행된다.  
이렇게 트랜잭션이 겹치지 않고 순차적으로  실행되는 스케줄을 `Serial schedule`이라고 한다.  
r1(A)-w1(A) : 계좌에서 읽을 때 I/O 작업을 하는 동안  cpu는 놀고있다.  
한 번에 하나의 트랜잭션만 실행되기 때문에 좋은 성능을 낼 수 없고 현실적으로 사용할 수 없는 방식이다.  

schedule3: `r1(A)-w1(A)-r2(B)-w2(B)-c2-r1(B)-w1(B)-c1`: T1 실행 중 T2 실행  
이렇게 트랜잭션이 겹쳐서 실행되는 스케줄을 `Nonserial schedule`이라고 한다.  
w1(A)-r2(B): I/O작업을 하는 동안 다른 트랜잭션을 진행한다.  
동시성이 높아져 같은 시간동안 더 많은 트랜잭션을 처리할 수 있지만 이상한 결과가 나올 수 있다.  

## Nonserial 스케줄로 실행해도 이상한 결과가 나오지 않게 하는 법이 없을까 ?🧐  
: serial schedule과 동일한 nonserial schedule을 실행하자!   
### Conflict of two operations  
1. 서로 다른 트랜잭션 소속 
2. 같은 데이터에 접근
3. 최소 하나는 write operation
2개의 operation이 위 세가지 조건을 만족하면 conflict이다.  

`r1(A)-w1(A)-r2(B)-w2(B)-c2-r1(B)-w1(B)-c1`  
1. r2(B)-w1(B) : read-write conflict   
2. w2(B)-w1(B) : write-write conflict 
3. w2(B)-r1(B): read-write conflict  
총 3개의 conflict가 있다는 걸 확인할 수 있다.  

conflict operation은 순서가 변경되면 결과도 변경된다.  

### Conflict equivalent for two schedules  
1 두 스케줄은 같은 트랜잭션들을 가진다.  
2. 어떤 conflicting operations의 순서도 양쪽 스케줄 모두 동일하다.  
2개의 스케줄이 위 2가지조건을 만족하면 `conflict equivalent`다.  

schedule2: `r2(B)-r2(B)-c2-r1(A)-w1(A)-r1(B)-w1(B)-c1`  : T2가 끝난 후  T1실행  
schedule3: `r1(A)-w1(A)-r2(B)-w2(B)-c2-r1(B)-w1(B)-c1`: T1 실행 중 T2 실행  

두 스케줄 모두 같은 트랜잭션들을 가지고 있다.  
두 스케줄은 모두 3개의 순서가 같은 conflict operation을 가지고 있다.  
두 스케줄은 `Conflict equivalent`다.  

2번 스케줄은 serial schedule이고 3번 스케줄은 nonserial schedule이다.  
이때 nonserial scheule인 3번 스케줄은 serial schedule과 `conflict equivalent`할 때 `conflict serializable`이라고 할 수 있다.  

schedule2: `r2(B)-r2(B)-c2-r1(A)-w1(A)-r1(B)-w1(B)-c1`  : T2가 끝난 후  T1실행  
schedule4: `r1(A)-w1(A)-r1(B)-r2(B)-w2(B)-c2-w1(B)-c1`: T1 실행 중 T2 실행  
-> 스케줄4는 업데이트한 데이터가 사라지는 `Lost Update`가 일어난다.  
두 스케줄은 모두 같은 트랜잭션을 가지고 있다.  
스케줄4는 r1(B)-w2(B) conflict operaiton을 가지고 있으나 스케줄2는 순서가 변경되어있다.  
두 스케줄은 `Conflict equivalent`하지 않다.  

schedule1: `r1(A)-w1(A)-r1(B)-w1(B)-c1-r2(B)-r2(B)-c2`  : T1 이 끝난 후  T2실행  
schedule4: `r1(A)-w1(A)-r1(B)-r2(B)-w2(B)-c2-w1(B)-c1`: T1 실행 중 T2 실행  
두 스케줄 모두 같은 트랜잭션을 가지고 있으나 conflict operation 순서가 다르므로 `Conflict equivalent`하지 않다.  
스케줄4가 그 어떤 serial schedule과도 `Conflict equivalent`하지 않았으므로 `Conflict serializable`하지 않다.  


`Conflict serializable`한 `nonserial schedule`을 허용해 이상한 결과가 나오지 않고 , 여러 트랜잭션을 겹쳐서 실행하자 !  

어떤 스케줄이 하나의 serial 스케줄과 동일(equivalent)하다면 이 스케줄은 serializable하다.  



### 참고
[쉬운코드-트랜잭션을 아십니까?](https://www.youtube.com/watch?v=sLJ8ypeHGlM&list=PLcXyemr8ZeoREWGhhZi5FZs6cvymjIBVe&index=14)  
[inpa - 트랜잭션 개념 & 사용](https://inpa.tistory.com/entry/MYSQL-%F0%9F%93%9A-%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98Transaction-%EC%9D%B4%EB%9E%80-%F0%9F%92%AF-%EC%A0%95%EB%A6%AC)  
[Database transaction schedule](https://en.wikipedia.org/wiki/Database_transaction_schedule)  
