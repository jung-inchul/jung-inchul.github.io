# redisson trylock 내부로직 살펴보기

<figure><img src="../../.gitbook/assets/redisson.png" alt=""><figcaption><p>https://opengraph.githubassets.com/c90eb78f30bf36acd86efd3a6d4093807ca77c6f8aa26f27630af6e2e17f3f76/redisson/redisson</p></figcaption></figure>

##

## redisson 이란??

* redisson은 자바 언어로 구현된 레디스 분산락 클라이언트이다
* 레디스 분산락은 서로 다른 프로세스가 서로 베타적인 방식으로 공유 리소스와 함께 작동해야 하는 많은 환경에서 매우 유용한 기본 기능이다
* 자바 언어 이외에 ruby, python, php 등 다양한 클라이언트 라이브러리가 존재한다



<figure><img src="../../.gitbook/assets/2 (3).png" alt=""><figcaption><p><a href="https://redis.io/docs/manual/patterns/distributed-locks/">https://redis.io/docs/manual/patterns/distributed-locks/</a></p></figcaption></figure>

##

## 그렇다면 분산락을 구현하는 로직은 어떻게 될까?

* 구현 자체는 간단하다
* 획득하고자 하는 이름의 락을 정의하고 유효 시간까지 락 획득을 시도하게 된다
* 만약 유효 시간이 지나면 락 획득은 실패하게 된다
* 단 여기서 주의할 점은 unlock할 경우에, 해당 세션에서 잠금을 생성한 락인지 확인해야 한다
* 그렇지 않으면 다른 세션에서 수행중인 잠금이 해제될수도 있다
  * isLocked() : 잠금이 되었는지 확인
  * isHeldByCurrentThread() : 해당 세션에서 잠금을 생성했는지 확인

```jsx
// 특정 이름으로 락 정의 
RLock lock = redissonClient.getLock(key.toString());

try {
    // 락 획득을 시도한다(20초동안 시도를 할 예정이며 획득할 경우 1초안에 해제할 예정이다)
    boolean available = lock.tryLock(20, 1, TimeUnit.SECONDS);

    if (!available) {
        System.out.println("lock 획득 실패");
        return;
    }
    
    // 트랜잭션 로직(ex. orderService.createOrder(), stockService.increase())

} catch (InterruptedException e) {
    throw new RuntimeException(e);
} finally {
    if (lock.isLocked() && lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

## 락을 점유를 시도하는 tryRock은 어떻게 구현되어 있나?

### 전체 코드

* 전체 코드를 확인하고 아래는 각 단계별 수행하는 내용에 대해서 추가적으로 설명하려고 한다
* redisson은 일반적으로 lettuce로 동시성을 해결하기 위한 차선책으로 많이 제시된다
* 그 이유는 재시도가 필요한 경우에 lettuce는 직접 스핀락으로 구현해서 락 점유를 시도하기 때문이다
* redisson은 스핀락으로 락을 점유하지 않고 pub/sub 구조로 레디스에 부하를 줄인다고 이야기하지만 실제 내부 로직을 살펴보면 스핀락 개념이 아예 없지는 않다
* redisson 구현의 특징은 lua script와 세마포어의 사용이라고 할 수 있다

```jsx
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {
        return true;
    }
    
    time -= System.currentTimeMillis() - current;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
    
    current = System.currentTimeMillis();
    CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    try {
        subscribeFuture.get(time, TimeUnit.MILLISECONDS);
    } catch (ExecutionException | TimeoutException e) {
        if (!subscribeFuture.cancel(false)) {
            subscribeFuture.whenComplete((res, ex) -> {
                if (ex == null) {
                    unsubscribe(res, threadId);
                }
            });
        }
        acquireFailed(waitTime, unit, threadId);
        return false;
    }

    try {
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
    
        while (true) {
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
            // lock acquired
            if (ttl == null) {
                return true;
            }

            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }

            // waiting for message
            currentTime = System.currentTimeMillis();
            if (ttl >= 0 && ttl < time) {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }

            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        }
    } finally {
        unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);
    }
//        return get(tryLockAsync(waitTime, leaseTime, unit));
}
```

### 1. lock 획득을 시도한다

* 획득을 시도하려는 lock의 유지시간을 확인한다
* 아래에서 tryAcuiqre 로직을 더 살펴보겠지만 우선 ttl값이 null이면 lock을 점유했다고 간주한다

```jsx
// 1. 락 획득을 시도한다
// (락 점유시간이 null이라면 획득이 가능하다고 판단하여 true를 리턴한다)
Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
// lock acquired
if (ttl == null) {
    return true;
}
```

* tryAcquire 내부 로직을 살펴보면 lua script를 사용해서 setnx를 실행하는 것을 확인할 수 있다

```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return nil; " +
                    "end; " +
                    "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return nil; " +
                    "end; " +
                    "return redis.call('pttl', KEYS[1]);",
            Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
            redis.call('exists', KEYS[1]) == 0
}
```

1. ```
   if (redis.call('exists', KEYS[1]) == 0) then 
   redis.call('hincrby', KEYS[1], ARGV[2], 1);
   redis.call('pexpire', KEYS[1], ARGV[1]);
   return nil;
   end;

   // LOCK KEY가 존재하는지 확인한다(없으면 0, 있으면 1)
   // LOCK KEY가 존재하지 않으면 LOCK KEY와 현재 쓰레드 아이디를 기반으로 값을 1 증가시켜준다
   // LOCK KEY에 유효시간을 설정한다
   // null 값을 리턴한다
   ```
2. ```
   if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
   redis.call('hincrby', KEYS[1], ARGV[2], 1);
   redis.call('pexpire', KEYS[1], ARGV[1]);
   return nil;
   end;

   // 해시맵 기반으로 LOCK KEY와 쓰레드 아이디로 존재하면 0이고, 존재하지 않으면 저장하고 1을 리턴한다
   // LOCK KEY가 존재하지 않으면 LOCK KEY와 현재 쓰레드 아이디를 기반으로 값을 1 증가시켜준다
   // LOCK KEY에 유효시간을 설정한다
   // null 값을 리턴한다
   ```
3. ```
   return redis.call('pttl', KEYS[1]);

   // 위의 조건들이 모두 false 이면 현재 존재하는 LOCK KEY의 TTL 시간을 리턴한다
   ```



### 2. waitTime이 초과되었는지 확인한다

* lock 에 대한 점유시간이 아직 남아있다면 다시 lock에 대한 획득을 시도하기 이전에 waitTime(lock 획득 시간)이 초과되지는 않았는지 확인한다
* 만약 이미 초과되었다면 lock 획득은 실패로 리턴한다

```jsx
// 2. 락 획득 대기 시간보다 초과되었는지 확인하고 초과되었으면 false를 리턴한다
time -= System.currentTimeMillis() - current;
if (time <= 0) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}
```



### 3. 고유 Thread Id를 채널로 구독하여 lock이 available할때까지 대기한다

* CompleteFuture.get() 메서드를 호출하여 thread id로 구독한 채널로 lock 획득이 유효할때까지 대기한다
* 만약에 사용자가 설정한 waitTime을 초과할 경우 TimeoutException이 발생하여 lock 획득에 실패한다
* 여기서 중요한 점은 subscribe 내부에는 세마포어를 사용해서 공유자원에 대한 점유를 수행한다는 것이다
* 세마포어를 사용하여 공유자원을 점유하기 때문에 스핀락 보다는 레디스 I/O에 대한 부하를 줄일 수 있다

```jsx
// 3. threadId를 채널로 구독하여 waitTime까지 대기한다.(block 처리)
// (만약 설정한 waitTime 보다 구독에 대한 응답이 없을 경우엔 TimeoutException이 발생하여 락 획득 결과를 false로 리턴한다)
CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
try {
    subscribeFuture.get(time, TimeUnit.MILLISECONDS);
} catch (ExecutionException | TimeoutException e) {
    if (!subscribeFuture.cancel(false)) {
        subscribeFuture.whenComplete((res, ex) -> {
            if (ex == null) {
                unsubscribe(res, threadId);
            }
        });
    }
    acquireFailed(waitTime, unit, threadId);
    return false;
}
```

* subscribe 내부 로직을 간단히 살펴보면 세마포어를 사용하는 것을 확인할 수 있다

```java
public CompletableFuture<E> subscribe(String entryName, String channelName) {
    AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
    CompletableFuture<E> newPromise = new CompletableFuture<>();

    semaphore.acquire(() -> {
        if (newPromise.isDone()) {
            semaphore.release();
            return;
        }

        E entry = entries.get(entryName);
        if (entry != null) {
            entry.acquire();
            semaphore.release();
            entry.getPromise().whenComplete((r, e) -> {
                if (e != null) {
                    newPromise.completeExceptionally(e);
                    return;
                }
                newPromise.complete(r);
            });
            return;
        }

        E value = createEntry(newPromise);
        value.acquire();
		
				...
}
```

* 아래는 CompletableFuture.get 메소드에서 exception이 발생하는 케이스를 확인한 내용이다

```jsx
/**
 * Waits if necessary for at most the given time for this future
 * to complete, and then returns its result, if available.
 *
 * @param timeout the maximum time to wait
 * @param unit the time unit of the timeout argument
 * @return the result value
 * @throws CancellationException if this future was cancelled
 * @throws ExecutionException if this future completed exceptionally
 * @throws InterruptedException if the current thread was interrupted
 * while waiting
 * @throws TimeoutException if the wait timed out
 */
@SuppressWarnings("unchecked")
public T get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    long nanos = unit.toNanos(timeout);
    Object r;
    if ((r = result) == null)
        r = timedGet(nanos);
    return (T) reportGet(r);
}
```



{% hint style="info" %}
**redisson은 왜 세마포어를 사용했는가?**

공식문서를 살펴 보면 세마포어를 사용한 이유를 파악할 수 있다. 레디스는 싱글 쓰레드로 동작하기 때문에 공유 자원에 대해서 쓰레드 세이프하게 동작하기 위해 동기화 매커니즘을 수행하기 위한 용도로 사용되었다고 설명하고 있다

공식문서 참고 : [https://redisson.org/glossary/java-semaphore.html](https://redisson.org/glossary/java-semaphore.html)
{% endhint %}



### 4. waitTime 이전까지 무한루프를 수행하면서 lock 점유시간을 한번 더 확인한다

* lock을 획득하여 아직 점유 유효시간이 남아있는지 한번 더 체크한다

```jsx
while (true) {
  long currentTime = System.currentTimeMillis();

	// 6. 락 획득을 시도한다
	// (락 점유시간이 null이라면 획득이 가능하다고 판단하여 true를 리턴한다)
  ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
  // lock acquired
  if (ttl == null) {
      return true;
  }
	...
```

### 5. thread id로 구독한 객체로 유효시간 또는 남은시간까지 lock이 avaliable한지 구독한다

* ttl은 이전에 lock이 점유되어 남아있던 시간을 의미한다
* time은 시도할 수 있는 남은 시간을 의미한다

```jsx
// waiting for message
currentTime = System.currentTimeMillis();
if (ttl >= 0 && ttl < time) {
    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
} else {
    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
}
```

### 6. lock을 시도할 수 있는 시간이 남아있는지 체크한다

```jsx
time -= System.currentTimeMillis() - currentTime;
if (time <= 0) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}
```

### 7. 그 이후에는 유효시간 동안 while문으로 4\~6번 과정을 반복하게 된다

##

## 결론

* 처음엔 redisson은 spin lock 로직이 없이 내부적으로 pub/sub 구조만 가지고 있는줄 알았다
* 하지만 살펴보니 spin lock 개념은 완전히 걷어내지는 못했지만 그래도 lettuce로 spin lock을 구현하는 것보단 훨씬 적은 부하로 락을 획득할 수 있다고 보여진다

## 참고

* [https://redis.io/docs/manual/patterns/distributed-locks/](https://redis.io/docs/manual/patterns/distributed-locks/)
