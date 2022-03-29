---
title: "재고가 한 개 남은 물건을 동시에 여러명이 장바구니에 담으면? (feat. MySql User Level Lock)"    
layout: post
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: MySQL User Level Lock을 활용한 재고관리
last_modified_at: 2021-09-07     
---



## 재고 차감 동시성 문제

A라는 상품이 현재 창고에 3개가 남아있는데 이 상품을 동시에 10명이 구매하려고 장바구니에 추가하면 개발자는 먼저 주문한 3명의 장바구니에만 상품이 담기는 결과를 원할 것 입니다.



그러면 원하는 결과가 나오는지 한번 테스트 해보겠습니다.



아래는 테스트를 위한 테이블 입니다. 상품과 주문 테이블은 1:N의 구조입니다.

![1](/assets/image/mysql_user_level_lock/1.png)



동시에 10명이 1이라는 id를 가진 상품을 장바구니에 담으려고 시도할 때 아무런 조치를 취하지 않고 단순히 상품의 재고와 현재 주문된 상품의 차이를 통해 재고를 차감시켰을때 결과 입니다.

```java
public int addOrder(Long productId) {

    if(validationQuantity(productId)) {
        orderMapper.addOrder(productId);
    }

    return orderMapper.getProductOrderQuantity(productId);
}

private boolean validationQuantity(Long productId) {
    int productQuantity = productMapper.getProductQuantity(productId);
    int productOrderQuantity = orderMapper.getProductOrderQuantity(productId);

    System.out.println(productQuantity + " " + productOrderQuantity);

    return productQuantity - productOrderQuantity > 0;
}
```

![2](/assets/image/mysql_user_level_lock/2.png)

재고가 3개가 남아있기 때문에 주문량도 3개이길 바라지만 총 10개의 주문이 생성되었습니다. 이러한 문제가 발생하는 이유는 스레드가 다음과 같이 동작하기 때문입니다.

![3](/assets/image/mysql_user_level_lock/3.png)

정상적으로 재고가 관리되려면 스레드1에서 재고를 조회한 후 재고를 차감시키고 스레드2로 넘어가 스레드 2에서 재고를 조회한 후 재고를 차감시키는 순서로 실행이 되어야 합니다. 하지만 실제로 스레드는 번갈아가며 실행되기 때문에 스레드1에서 재고를 차감시키기전에 스레드2에서 재고를 조회를 할 수 있습니다.

이렇게 되면 정상적으로 남은 상품의 재고를 파악할 수 없기 때문에 위 결과처럼 재고 이상의 주문이 생성되게 됩니다.



이를 간단히 해결하는 방법도 있습니다. 재고조회와 차감을 수행하는 메소드에 synchronized 키워드를 붙여주면 되면 해당 메소드를 실행한 스레드가 종료되기 전까지 다른 스레드는 접근하지 못하게 되어 정상적으로 원하는 동작을 수행합니다. synchronized 메소드를 붙여 테스트를 진행 해보겠습니다.

```java
public synchronized int addOrder(Long productId)
```

![4](/assets/image/mysql_user_level_lock/4.png)

정상적으로 3개의 주문만 증가한것을 확인할 수 있습니다.

synchronized 키워드를 붙여주어 동시성문제를 해결할 수 있는것을 보았지만 이 방식은 서버가 여러개 구성되어있는 다중 서버 환경에서는 적합하지 않습니다.

결국 동시성 문제를 자바 코드만으로는 해결하기 부족하기에 데이터베이스에서 제어하는 방식을 생각했습니다.

MySQL의 분산락을 이용하는 방식입니다.



## MySql User Level Lock

User Level Lock 이란 데이터베이스를 이용하는 사용자가 특정 "문자열"에 Lock을 걸수있는 기능을 말하며 다음과 같은 함수를 제공합니다.

- GET_LOCK(str, timeout)
  - 입력받은 문자열로 timeout초 동안 잠금획득을 시도합니다.

- RELEASE_LOCK(str)
  - 입력받은 문자열의 잠금을 해제합니다.

- RELEASE_ALL_LOCKS()
  - 모든 잠금을 해제하고 해제한 잠금의 개수를 리턴합니다.

- IS_FREE_LOCK(str)
  - 입력받은 문자열로 잠금획득이 가능한지 확인합니다.

- IS_USED_LOCK(str)
  - 입력받은 문자열의 잠금이 존재하는지 확인합니다.



User Level Lock의 함수들에 대해 알아봤으니 이제 구현을 해보겠습니다. 기존에 구매가능 여부 체크 후 구매 로직 사이에 LOCK을 걸어주면 됩니다.

```java
public int addOrder(Long productId) {

    long start = System.currentTimeMillis();

    try(Connection connection = dataSource.getConnection()) {
      	// 상품 ID로 락 획득
        getLock(String.valueOf(productId), 3, connection);

      	// 구매가능 여부 조회 후 구매
        if(validationQuantity(productId)) {
            orderMapper.addOrder(productId);
        }
				
      	// 락 해제
        releaseLock(String.valueOf(productId), connection);
    } catch (SQLException e) {
        throw new RuntimeException(e);
    } finally {
        int result = orderMapper.getProductOrderQuantity(productId);

        log.info("time : " + String.valueOf(System.currentTimeMillis() - start));
        return result;
    }
}

private void getLock(String lockStr, int timeout, Connection connection) {
    String sql = "SELECT GET_LOCK(?, ?)";

    try(PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
      
        preparedStatement.setString(1, lockStr);
        preparedStatement.setInt(2, timeout);

        try(ResultSet resultSet = preparedStatement.executeQuery()) {
            resultSet.next();

            int result = resultSet.getInt(1);
            if(result != -1) {
                log.info("LOCK 수행 중 오류가 발생했습니다.");
            }
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
}

private void releaseLock(String lockStr, Connection connection) {
    String sql = "SELECT RELEASE_LOCK(?)";

    try(PreparedStatement preparedStatement = connection.prepareStatement(sql)) {

        preparedStatement.setString(1, lockStr);

        try(ResultSet resultSet = preparedStatement.executeQuery()) {
            resultSet.next();

            int result = resultSet.getInt(1);
            if(result != -1) {
                log.info("LOCK 수행 중 오류가 발생했습니다.");
            }
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
}
```

![5](/assets/image/mysql_user_level_lock/5.png)

10개의 요청을 전송하면 정상적으로 3개의 주문건만 들어온것을 확인할 수 있습니다.

