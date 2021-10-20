---
title: "Redis 서버 구성"    
layout: single    
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: Redis 서버 구성
last_modified_at: 2021-10-20     
---

## Persistance

Redis는 In-Memory기반의 데이터베이스 이기 때문에 서버가 비정상적으로 종료될 경우 모든 데이터가 날아갈 위험성이 존재합니다.

Redis도 이러한 위험성을 알고있기 때문에 다음과 같은 데이터 저장 방법을 제공하고있습니다.



RDB

- RDB 방식은 특정 시점 혹은 명령을 받은 시점에 메모리에 저장되어있는 데이터를 디스크에 작성합니다.
- 특정 시점에 저장이 되기 때문에 저장된 이후 변경된 데이터는 복구할 수 없습니다.

AOF

- Append Only File의 약자로 appendonly.aof라는 파일에 입력/수정/삭제 명령이 실행될 때마다 해당 명령이 기록됩니다.
- Redis 서버가 시작될 때 해당 파일을 로드하여 데이터를 복구합니다.
- 명령이 쌓일수록 파일의 크기가 커져 기록이 중단될 수 있습니다.



## Replication

Redis 서버가 비정상적으로 종료되어 메모리에 저장된 데이터가 초기화 되어도 복구할 수 있는 방법에 RDB와 AOF가 있는것을 알았습니다.

그런데 Redis서버가 컴퓨터에 장애가 생겨 비정상적으로 종료되어버리면 그 컴퓨터의 디스크에 복구할 수 있는 파일이 존재해도 컴퓨터의 장애를 해결하는 동안 Redis서버에 접근해서 로직을 처리해야하는 애플리케이션 서비스가 동작하지 않게 될것입니다.



예를들어 쇼핑몰에서 다음과 같이 유저가 상품 구매 요청을 보내면 Redis에 존재하는 해당 상품의 재고를 차감시키는 로직을 수행하는 서버가 존재합니다.

![1](/assets/image/redis_server_config/1.png)

그런데 갑자기 Redis서버가 존재하는 건물에 정전이 발생해서 App에서 Redis로 요청을 보내지 못하게 되었습니다.

![2](/assets/image/redis_server_config/2.png)

이러한 경우 Redis가 설치되어있는 데이터를 복구할 수 있는 파일이 존재하여도 해당 건물의 정전문제를 해결하기 전까지 유저는 쇼핑몰에서 어떤 상품도 구매하지 못하게 됩니다.

이는 서비스를 운영중인 기업입장에서 굉장히 큰 손실로 이어질 수 있는 문제입니다.



이러한 문제를 해결하기 위해 Redis는 HA(High Availability)를 지원합니다.

(HA란 고가용성으로 서버와 네트워크, 프로그램 등의 정보 시스템이 상당히 오랜 기간 동안 지속적으로 정상 운영이 가능한 성질을 말합니다.)



Redis가 지원하는 HA는 다음과 같은 Master - Replica 형태 입니다.

![3](/assets/image/redis_server_config/3.png)

한 개의 마스터에 N개의 Replica 혹은 한 개의 마스터에 한 개의 Replica와 그 Replica에 연결된 Replica로 이루어진 형태로 구성이 가능합니다.

이렇게 연결된 Replica는 연결된 상위 노드의 데이터를 복사하여 가지고 있게 됩니다.



그런데 이렇게 마스터노드에서 복제 노드로 데이터를 전송하면 마스터노드의 성능에 문제가 생기지 않을까 걱정이 들 수 있습니다.

하지만 걱정과는 달리 마스터 노드는 자식 프로세스(Child Process)를 만들어 백그라운드에서 덤프 파일을 생성하고 생성한 파일을 Replica에 전송하는데 다음과 그림과 같이 동작합니다.

![4](/assets/image/redis_server_config/4.png)

Replica에서 Master노드로 replicaof 명령을 전송하면 마스터노드는 자식 프로세스를 생성해 덤프파일을 만들고, 만들어진 덤프파일을 Replica로 전송하게 됩니다.



이렇게 Replica를 통해 Redis 서버를 구성하면 문제없이 안정적인 운영을 할 수 있을까요? 그렇지 않습니다.

애플리케이션 서버는 192.168.0.100의 주소를 가진 Master노드를 통해 Redis 서버에 접근하여 사용중 입니다.

그런데 Master노드가 비정상적으로 종료되면 Replica노드를 Master 노드로 승격시킨 후

애플리케이션 서버를 종료한 다음 192.168.0.200의 주소를 가진 노드로 접근하도록 접속 정보를 변경한 후 다시 실행시켜야 합니다.



결국 Master노드가 종료되면 Replica 노드를 승격시키고 접속정보를 변경해 다시 실행시키는데 필요한 시간만큼 서버의 운영이 중지되는 것 입니다.

이러한 문제를 단일 장애점(single point of failure, SPOF) 혹은 단일 실패점 이라고 부르는데, 이는 시스템 구성 요소 중에서 하나가 동작하지 않으면 전체 시스템이 중단되는 요소를 말합니다.

이 문제를 해결하기위해 2가지 기술이 존재하는데 sentinel과 cluster입니다.



## Sentinel

sentinel을 번역하면 보초라는 뜻 인데 말 그대로 보초를 서는 보초를 통해 동작중인 노드를 감시하는 것을 말합니다.

sentinel을 통해 서버를 구성하면 다음과 같은 형태의 노드가 구성이 됩니다. Sentinel노드는 최소 3개 이상이어야 하며, 무조건 홀수 개 이어야 합니다.

![5](/assets/image/redis_server_config/5.png)

Master노드에 문제가 생겨 종료되면 이를 감시하고 있는 Sentinel노드는 Master노드가 진짜 문제가 생겨 종료된 것인지 Sentinel노드끼리 투표를 진행하여 결론을 내립니다.

(투표진행에 있어 Sentinel노드의 수가 짝수이면 동률이 나와 어떻게 해야할지 판단하지 못하는 경우가 생기므로 홀수 개의 노드를 구성하는 것 입니다.)

Master노드에 문제가 생겨 종료된 것으로 판단되면 Replica노드 중 하나를 선택해 Master노드로 승격시킵니다.

Application은 Sentinel노드를 통해 Master노드에 접근하므로 접근정보 변경을 위해 서버를 종료할 필요가 없어지게 됩니다.



## Cluster

Cluster방식은 Sentinel방식처럼 감시를 위한 노드를 따로 두는 것이 아닌 모든 노드가 서로를 감시하도록 하는 방식입니다.

![6](/assets/image/redis_server_config/6.png)

기존의 다른 구성 방식들과는 차이가 존재하는데 바로 Master노드가 여러개 존재하는 것 입니다. Cluster를 사용하기 위해서는 최소 3개 이상의 Master노드가 필요합니다.

이렇게 여러개로 구성된 Master노드는 각각 데이터를 저장하기위한 슬롯을 가지는데 총 16384개의 슬롯을 N개의 Master노드가 나누어 가지게 됩니다.

![7](/assets/image/redis_server_config/7.png)

이렇게 나누어진 슬롯에 Key값을 해싱하여 나온 결과를 통해 데이터를 저장할 슬롯의 위치를 파악하고 데이터를 저장합니다.



만약 Master노드가 비정상적으로 종료되면 해당 Master노드와 연결된 Replica노드가 Master노드로 승격됩니다.

또한, 1개의 Master노드에 연결된 Replica노드의 수는 무조건 1개일 필요는 없습니다.

만약 A라는 Master노드에 2개의 Replica노드가 연결되어있고 B라는 Master노드에는 1개의 Replica노드가 연결되어있을때 B라는 Master노드에 연결된 Replica노드가 종료되어버리면 A의 Master노드에 연결되어있는 2개의 Replica노드 중 하나의 노드가 B라는 Master노드의 Replica노드로 변경됩니다.

