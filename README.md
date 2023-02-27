# Redis-TroubleShooting 학습

레디스 백업은 RDB , AOF를 사용한 백업 두가지 종류가 있다.

---

# 1. RDB(Redis Database)를 사용한 백업

- 특정 시점의 스냅샷으로 데이터 저장
- 재시작 시 RDB 파일이 있으면 읽어서 복구

![image](https://user-images.githubusercontent.com/40031858/221501912-1fb5da9b-00a5-4ae5-b42d-14bca442aaed.png)

## RDB 사용의 장점

- 작은 파일 사이즈로 백업 파일 관리가 용이(원격지 백업, 버전 관리 등)
- fork를 이용해 백업하므로 서비스 중인 프로세스는 성능에 영향 없음
- 데이터 스냅샷 방식이므로 빠른 복구가 가능

## RDB 사용의 단점

- 스냅샷을 저장하는 시점 사이의 데이터 변경사항은 유실될 수 있음
- fork를 이용하기 때문에 시간이 오래 걸릴 수 있고, CPU와 메모리 자원을 많이 소모
- 데이터 무결성이나 정합성에 대한 요구가 크지 않은 경우 사용 가능 (마지막 백업 시 에러 발생 등의 문제)

### RDB 설정

- 설정파일이 없어도 기본값으로 RDB가 활성화되어 있음
- 설정 파일 만드려면 템플릿을 받아서 사용 (https://redis.io/docs/management/config/)

`저장 주기 설정 (Ex: 60초마다 10개 이상의 변경이 있을 때 수행)`

    save 60 10

`스냅샷을 저장할 파일 이름`

    dbfilename dump.rdb

`수동으로 스냅샷 저장`

    bgsave

### Docker를 사용해 Redis 설정 파일 적용하기

- docker run 사용 시 -v 옵션을 이용해 디렉토리 또는 파일을 마운팅할 수 있음
- redis 이미지 실행 시 redis-server에 직접 redis 설정 파일 경로 지정 가능

`Docker 컨테이너 실행 시 설정 파일 적용하기`

    docker run -v /my/redis.conf:/redis.conf --name my-redis redis redis-server /redis.conf

---

# 2. AOF(Append Only File)를 사용한 백업

- 모든 쓰기 요청에 대한 로그를 저장
- 재시작 시 AOF에 기록된 모든 동작을 재수행해서 데이터를 복구

![image](https://user-images.githubusercontent.com/40031858/221508964-862b9d3d-c4c2-426d-98d7-7d2c5bfd16d4.png)

## AOF 사용의 장점

- 모든 변경사항이 기록되므로 RDB 방식 대비 안정적으로 데이터 백업 가능
- AOF 파일은 append-only 방식이므로 백업 파일이 손상될 위험이 적음
- 실제 수행된 명령어가 저장되어 있으므로 사람이 보고 이해할 수 있고 수정도 가능


![image](https://user-images.githubusercontent.com/40031858/221509330-91501d40-3c35-4408-9e50-ccf59f12d8c2.png)

## AOF 사용의 단점

- RDB 방식보다 파일 사이즈가 커짐
- RDB 방식 대비 백업 & 복구 속도가 느림 (백업 성능은 fsync 정책에 따라 조절 가능)

### AOF 설정

`AOF 사용 (기본값은 no)`

    appendonly yes

`AOF 파일 이름`

    appendfilename appendonly.aof

`fsync 정책 설정(always, everysec, no)`

    appendfsync everysec

### fsync 정책(appendfsync 설정 값)

- fsync() 호출은 OS에게 데이터를 디스크에 쓰도록 함
- 가능한 옵션과 설명
  - always: 새로운 커맨드가 추가될 때마다 수행. 가장 안전하지만 가장 느림
  - everysec: 1초마다 수행. 성능은 RDB 수준에 근접
  - no: OS에 맡김. 가장 빠르지만 덜 안전한 방법(커널마다 수행 시간이 다를 수 있음)

### AOF 관련 개념

- Log rewriting: 최종 상태를 만들기 위한 최소한의 로그만 남기기 위해 일부를 새로 씀

    ex) 1개의 key값을 100번 수정해도 최종 상태는 1개이므로 SET 1개로 대체 가능

- Multi Part AOF: Redis 7.0 부터 AOF가 단일 파일에 저장되지 않고 여러 개가 사용됨
  - base file : 마지막 rewrite 시의 스냅샷을 저장
  - incremental file : 마지막으로 base file이 생성된 이후의 변경사항이 쌓임
  - manifest file : 파일들을 관리하기 위한 메타 데이터를 저장 

---

# Redis replication(복제)

- 백업만으로는 장애 대비에 부족함 (백업 실패 가능성, 복구에 소요되는 시간)
- Redis도 복제를 통해 가용성을 확보하고 빠른 장애조치가 가능
- master가 죽었을 경우 replica 중 하나를 master로 전환해 즉시 서비스 정상화 가능
- 복제본(replica)은 read-only 노드로 사용 가능하므로 traffic 분산도 가능


![image](https://user-images.githubusercontent.com/40031858/221554951-9bf95224-ef77-4800-b6dc-39ab9fdd011d.png)

### Redis 복제 사용

- Replica 노드에서만 설정을 적용해 master-replica 복제 구성 가능

`Replica로 동작하도록 설정`

    replicaof 127.0.0.1 6379

`Replica는 read-only로 설정`

    replica-read-only

>> `Master 노드에는 RDB나 AOF를 이용한 백업 기능 활성화가 필수! (재시작 후에 비어있는 데이터 상태가 복제되지 않도록)`



<img width="1139" alt="image" src="https://user-images.githubusercontent.com/40031858/221558072-bd8d9ff2-2592-43ea-999c-d95e81e6ff82.png">



<img width="838" alt="image" src="https://user-images.githubusercontent.com/40031858/221558539-ad218e32-f00f-4e04-8c9c-98ddab415344.png">

마스터에 키를 입력했을 때 replica에서 잘 적용됨을 볼 수 있음.

</details>

<details>
<summary> 참고로 이 레포지토리에 `docker-compose`를 작성했으니 `docker-compose up --build` 명령어를 수행하면댐.
</summary>
<img width="987" alt="image" src="https://user-images.githubusercontent.com/40031858/221560476-3bb2f1bc-2f7a-44ae-9340-5cf6be49d20f.png">
</details>


