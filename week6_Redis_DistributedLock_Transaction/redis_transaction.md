## 1. Redis의 트랜잭션

### 1-1. 특징

데이터베이스의 트랜잭션은 여러 작업을 하나의 단위로 묶어 원자성, 일관성, 고립성, 지속성을 보장하는 기능이다. Redis의 트랜잭션 역시 여러 명령어를 한 번에 실행할 수 있도록 해주지만, Redis의 단순성과 성능을 유지하기 위해 트랜잭션의 롤백은 지원하지 않는다. 즉, 오류가 발생해도 이미 실행된 명령어는 되돌릴 수 없다.

Redis의 트랜잭션 내 모든 명령어는 순차적으로 실행된다. 트랜잭션이 실행되는 도중에는 다른 클라이언트의 요청이 개입하지 않으며, 모든 명령어가 하나의 독립된 작업으로 수행된다.

### 1-2. AOF (Append Only File)

AOF는 Redis에서 데이터를 디스크에 영구적으로 저장하기 위한 방식 중 하나로, 수행된 모든 쓰기 명령어를 순차적으로 로그 파일에 기록한다. 트랜잭션을 사용할 경우 Redis는 트랜잭션 안의 모든 명령어를 한 번의 `write(2)` 시스템 호출로 디스크에 기록한다. 이는 트랜잭션 전체가 하나의 원자적인 단위처럼 저장되어 중간에 잘리는 일이 없도록 하기 위함이다.

하지만 시스템이 갑작스럽게 종료되거나 Redis 프로세스가 비정상적으로 중단되면, `write` 작업이 완전히 마무리되지 못할 수도 있다. 이때 AOF 파일은 트랜잭션의 일부 명령만 저장된 불완전한 상태가 되고, Redis는 이를 감지하여 재시작 시 에러를 발생시키고 실행을 멈춘다. 이런 경우에는 `redis-check-aof` 도구를 사용해 마지막에 저장된 불완전한 명령어 또는 트랜잭션을 잘라내고 정상적인 상태로 복구할 수 있다.

## 2. 명령어

### 2-1. `MULTI`

트랜잭션을 시작하는 명령어이다. 이 명령은 항상 OK를 반환한다. 이후 여러 명령어를 입력하면 Redis는 이들을 실행하지 않고 큐에 저장해두었다가, `EXEC`이 호출되면 한 번에 실행한다.

### 2-2. `EXEC`

트랜잭션을 실행하는 명령어이다. 큐에 저장해두었던 명령어들을 한 번에 실행한다. `EXEC` 명령어가 실행되어야 트랜잭션 내 명령들이 실제로 수행된다. 만약 클라이언트가 `EXEC`을 호출하기 전에 연결이 끊기면 트랜잭션은 무효 처리되며 아무 명령도 실행되지 않는다.

### 2-3. `DISCARD`

대기 중인 명령어를 모두 취소하고 트랜잭션을 종료하는 명령어이다. 이 경우 명령어는 실행되지 않고, 연결 상태는 일반 모드로 복귀한다.

### 2-4. `WATCH`

특정 키를 감시하여, 해당 키가 `EXEC` 호출 전에 변경되면 트랜잭션을 취소하게 만드는 명령어이다. 감시 중인 키가 변경되면 `EXEC`은 Null을 반환하며 트랜잭션이 취소된다. 이때 변경에는 클라이언트의 쓰기뿐 아니라, Redis 자체의 키 만료나 삭제도 포함된다. 단, Redis 6.0.9 이전 버전에서는 만료된 키가 트랜잭션 취소를 유발하지 않는다.

`WATCH`는 여러 번 호출할 수 있으며, 이후 `EXEC`을 호출할 때까지 영향을 유지한다. `EXEC`이 호출되면 모든 감시 키는 자동으로 `UNWATCH` 처리되며, `UNWATCH` 명령을 통해 수동으로 감시 키를 해제할 수도 있다.

`WATCH` 명령어를 사용하면 낙관적 잠금을 구현할 수 있으며, CAS(Check-And-Set) 방식의 동작을 가능하게 한다. 예를 들어, 어떤 키의 값을 읽고 그 값을 +1 한 뒤 저장하려는 상황이라고 하자. 만약 값을 읽고 계산하는 사이 다른 클라이언트가 그 값을 변경했다면, 이는 낙관적 잠금이 깨진 것이며 CAS의 Check 단계가 실패한 것이다. `WATCH`는 이런 상황에서 트랜잭션 전체를 자동으로 취소한다.

## 3. 오류 처리

### 3-1. 명령어가 큐에 추가되기 전에 발생하는 오류

명령어가 문법적으로 잘못되었거나 인자 개수가 부족한 경우, 혹은 서버에 설정된 메모리 제한을 초과하는 등의 경우가 이에 해당한다.

Redis 2.6.5 이후 버전에서는 명령어를 수집하는 과정에서 오류가 발생하면, `EXEC` 시점에 트랜잭션을 거부하고 오류를 반환한다. 2.6.5 이전 버전에서는 클라이언트가 `QUEUED` 응답을 검사하여 오류 여부를 직접 확인해야 했다. 명령이 제대로 큐에 들어갔다면 `QUEUED`, 그렇지 않으면 에러를 반환했으며 대부분의 클라이언트는 이 경우 트랜잭션을 포기하고 `DISCARD`를 호출했다.

### 3-2. `EXEC` 이후 실제 실행 도중 발생하는 오류

문자열 타입의 키에 리스트 연산을 수행할 때 발생하는 타입 오류 등의 경우가 이에 해당한다.

`EXEC` 이후 발생한 오류는 트랜잭션 실행 자체를 멈추진 않는다. 즉, 오류가 발생하더라도 나머지 명령은 그대로 실행된다.

## 4. Lua 스크립트

Redis는 Lua 기반 스크립트를 통해 트랜잭션과 유사한 기능을 제공한다. Redis는 `EVAL`, `EVALSHA` 명령어를 통해 Lua 스크립트를 실행하며, 스크립트 내에서 Redis 명령어를 자유롭게 호출할 수 있다. Lua 스크립트는 Redis 서버에서 하나의 원자적 작업으로 실행되므로, 트랜잭션보다 더 간단하고 빠르게 복잡한 연산을 처리할 수 있다.


---

**참고 자료**

- https://redis.io/docs/latest/develop/interact/transactions/