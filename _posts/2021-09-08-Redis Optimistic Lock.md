---
title: "재고가 한 개 남은 물건을 동시에 여러명이 장바구니에 담으면? (feat. Redis)" 
layout: post
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: Redis Optimistic Lock을 활용한 재고관리
last_modified_at: 2021-09-07     
---



## 비관적 잠금과 낙관적 잠금



### 비관적 잠금(Pessimistic Lock) 

비관적 잠금이란 동일한 데이터를 동시에 수정 할 가능성이 높다는 비관적인 전제로 잠금을 거는 방식을 말합니다.

지난 포스팅에서 MySql의 User Level Lock을 활용한 동시성 제어에 대해 알아보았었는데 해당 방식을 예로들 수 있습니다.

MySql에서 문자열을 통해 잠금을 획득하고 해당 잠금이 해제될 때 까지 대기해야하는 User Level Lock은 사용하기에 편리하고 추가적인 인프라 구성이 필요없는 점이 장점이지만 코드가 Block된다는 점이 개발자에게 부담으로 다가올 수 있습니다.



### 낙관적 잠금(Optimistic Lock)

낙관적 잠금은 동시에 데이터를 수정하지 않을 것이라고 낙관적으로 판단하여 잠금을 거는 방식을 말합니다.

데이터베이스에 락을 거는 방식이 아닌 Timestamp등을 통해 Version을 확인할 수 있는 정보를 가지고 충돌을 감지하는 방식입니다.

스레드1이 A상품의 재고수량을 증가시키는  작업을 진행중인데 도중에 스레드2가 A상품의 재고를 변경시키면 스레드1에서 충돌을 감지하여 아무것도 변경시키지 않는 경우를 예로들 수 있습니다.

이번 포스팅에선 Redis를 이용한 낙관적 잠금 방식을 통해 재고관리를 해보겠습니다.



## Redis

Redis는 In-Memory 기반의 Key/Value Store로써 Single-thread 기반으로 데이터를 처리합니다.

String, List, Hash, Set, Sorted Set등 여러 형식의 자료구조를 지원합니다.



Redis를 사용해 재고량을 변경시키기 위해서는 Redis Transaction을 이용해 여러 연산을 하나의 작업으로 만들어 실행시켜야 합니다.

Redis에서는 트랜잭션을 위한 명령어가 존재하는데 다음과 같습니다.

- MULTI
  - Redis 트랜잭션을 시작하는 명령어 해당 명령어로 트랜잭션을 시작하면 이후 명령은 실행되지 않고 Queue에 적재됩니다.
- EXEC
  - 정상적으로 처리되어 Queue에 적재되어있는 명령어를 일괄 실행합니다.
- DISCARD
  - Queue에 적재되어있는 명령어를 폐기합니다.
- WATCH
  - 낙관적 락(Optimistic Lock)을 거는 명령어 입니다.
  - WATCH를 통해 Lock이 걸린 데이터를 다른 트랜잭션에서 수정하면 해당 트랜잭션은 아무것도 변경하지 않습니다.



Spring에서 위의 명령어를 사용해 트랜잭션을 수행하려면 동일한 커넥션에서 명령을 입력해야합니다.

RedisTemplate은 동일한 커넥션을 유지하며 명령을 입력하기위해 SessionCallback 이라는 클래스를 제공합니다.

위 명령어와 SessionCallback을 사용해 코드를 작성하고 실행시켜 보겠습니다.

```java
public StockResult increaseStock(Long productId) {
    if(validationQuantity(productId)) {
        Object txResult = redisCartTemplate.execute(new SessionCallback<Object>() {

            @Override
            public Object execute(RedisOperations redisOperations) throws DataAccessException {
                redisOperations.watch(String.valueOf(productId));
                redisOperations.multi();
                redisOperations.opsForValue().increment(String.valueOf(productId), 1);

                return redisOperations.exec();
            }
        });
        if(txResult.toString().equals("[]")) {
            return StockResult.RESULT_TX;
        }
    } else {
        return StockResult.RESULT_QUANTITY;
    }
    return StockResult.RESULT_SUCCESS;
}

private boolean validationQuantity(Long productId) {
    Integer totalQuantity = productMapper.getProductQuantity(productId);
    Integer useQuantity = (Integer) Optional.ofNullable(redisCartTemplate.opsForValue().get(String.valueOf(productId))).orElse(0);

    return (totalQuantity - useQuantity) > 0;
}


public enum StockResult {
    RESULT_SUCCESS,
    RESULT_TX,
    RESULT_QUANTITY
}
```

![1](/assets/image/redis_optimistic_lock/1.png)

10개의 요청에 대해 한건의 주문이 성공했습니다.

현재 구매가능한 수량은 3개이지만 한개의 재고만 감소되었습니다 이러한 결과가 나온 이유는 동시에 여러개의 요청이 한 개의 상품에 대해 접근해 재고를 변경시키다보니 낙관적 잠금에 의해 경합이 발생된 모든 트랜잭션이 DISCARD 되었기 때문입니다.



주문에 대한 트랜잭션이 취소되고 다시 Retry를 해주어야할지 아니면 사용자에게 fail message를 전송해야할지 등을 결정해서 추가적인 작업을 해주어야하기 때문에 동시에 많은 경합이 발생할 수 있는 환경에서는 비관적 잠금을 사용하는것이 더 안정적인 서비스 제공에 도움이 될 것으로 예상됩니다.
