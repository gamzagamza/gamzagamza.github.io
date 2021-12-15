---
title: "Mysql Master/Slave"    
layout: single    
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: Mysql Master/Slave구조 적용
last_modified_at: 2021-12-15     
---

## Replication을 구성하는 이유

만약 데이터베이스 서버가 하나만 존재할 경우 아래 이미지 처럼 애플리케이션은 하나의 데이터베이스에서 읽기와 쓰기 작업을 모두 수행하게 될 것입니다.

![1](/assets/image/mysql_master_slave/1.png)

이렇게되면 당연히 데이터베이스의 부담이 증가할 수 밖에 없고 이는 병목현상을 불러올 수 있는 원인이 될 수 있습니다.

이러한 문제를 복제 데이터베이스를 구성하는 방식을 통해 해결할 수 있는데 다음의 이미지처럼 Master DB를 복제하는 Replication DB를 구성하고

애플리케이션에서 데이터의 추가, 수정, 삭제는 Master DB에서 검색은 Replication DB에서 하도록 변경하는 것 입니다.

![2](/assets/image/mysql_master_slave/2.png)

Slave는 Master의 데이터를 복제함으로 써 데이터베이스의 데이터를 보관하는데 안정성을 높이는 역할과 부하분산의 역할까지 하게 됩니다.



## MySQL Master/Slave 구성

우선 Master와 Slave역할을 할 서버 두 개가 필요한데 제가 구성한 환경은 다음과 같습니다.

>Google Cloud Platform - 2vCPU, 1GB Memory, 20GB DISK
>
>MySQL 8.0.23
>
>Linux Ubunti 18.04 LTS

만약 MySQL 버전을 다르게 할 경우 Slave의 버전이 Master의 버전보다 높아야 합니다.



**Master에 새로운 데이터베이스 생성**

```mysql
CREATE DATABASE demo;
```



**Master에 생성한 데이터베이스의 권한을 가진 유저 생성**

```mysql
GRANT ALL PRIVILEGES ON demo.* TO '{아이디}'@'%' IDENTIFIED BY '{비밀번호}';
FLUSH PRIVILEGES;
```



**Master에 접근가능한 Replication용 유저 생성**

```mysql
GRANT ALL REPLICATION SLAVE ON *.* TO '{아이디}'@'%' IDENTIFIED BY '{비밀번호}';
```



**Master의 my.cnf 수정**

```shell
# vi /etc/mysql/conf.d/my.cnf

[mysqld]
server-id=1
log-bin=mysql-bin
```



**복제할 데이터베이스 dump파일 생성**

```shell
mysqldump -u root -p demo > dump.sql
```

생성한 덤프파일을 slave용 db가 설치된 인스턴스로 옮겨주면 됩니다.



**Master 정보 확인**

```mysql
# mysql -u root -p

mysql> SHOW MASTER STATUS;
```

위 명령어를 입력하면 다음과 같은 정보가 콘솔에 출력되는데 File과 Position에 출력된 정보를 기억해야 합니다.

![3](/assets/image/mysql_master_slave/3.png)



이제부터는 slave용 데이터베이스가 설치된 서버에서 작업을 진행해야 합니다.



**Slave에 데이터베이스 생성**

```mysql
CREATE DATABASE demo;
```



**덤프파일 실행**

```shell
mysql -u root -p demo < dump.sql
```



**Slave에 Master정보 입력**

```mysql
mysql> CHANGE MASTER TO MASTER_HOST='{마스터 아이피}', MASTER_PORT=3306, MASTER_USER='{Replication용 유저 아이디}', MASTER_PASSWORD='{Replication용 유저 비밀번호}', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=1819;
```



**연동 확인**

```mysql
mysql> SHOW SLAVE STATUS\G;
```

```
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 10.178.0.21
                  Master_User: svuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 1819
               Relay_Log_File: slave-mysql-instance-relay-bin.000002
                Relay_Log_Pos: 1233
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1819
```

위와 같은 정보가 출력되면 정상적으로 연동이 완료된 것 입니다.

만약 연동 중 에러가 발생하면 Last_Error: 부분에 에러 메시지가 출력될 것 입니다.



MySQL 8.0버전 이상을 사용하면 다음과 연동 중 다음과 같은 에러가 발생할 수 있습니다.

```
Authentication plugin 'caching_sha2_password' cannot be loaded: 
dlopen(/usr/local/mysql/lib/plugin/caching_sha2_password.so, 2): image not found
```

만약 위와같은 에러가 발생하면 Master DB에 접속해 replication용 유저의 비밀번호를 다음과 같은 명령어를 통해 변경해주면 됩니다.

```
mysql> ALTER USER '{아이디}'@'%' IDENTIFIED WITH mysql_native_password BY '{비밀번호}';
```

변경을 완료한 후 Master의 정보가 변경되었을 수 있기 때문에

다시 SHOW MASTER STATUS 명령어를 통해 정보를 확인한 후 Slave에서 연동을 시도해야 합니다.
