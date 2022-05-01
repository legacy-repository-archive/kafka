# ACL을 이용한 카프카 권한 설정  

카프카의 권한(ACL) 설정은 유저별로 특정 토픽에 접근을 허용할지 또는 거부할지 등의 설정을 하는 것이다.   
카프카의 권한 설정을 위해서는 먼저 브로커의 설정 파일을 수정하고    
그 다음으로는 kafka-acls.sj 명령어를 이용해 유저별로 권한을 설정해야한다.   

## 브로커 권한 설정 

카프카 권한 설정을 위해 `/usr/local/kafka/config/server.properties` 파일을 수정해서 브로커 설정을 변경해보자.  

```properties 
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.machanism.inter.broker.protocol=GASSAPI
sasl.enabled.machanism=GASSAPI
sasl.kerberos.service.name=kafka
# 아래내용 추가 
authorizer.class.name=kafka.security.authorizer.AclAuthroizer # 카프카 권한을 위한 클래스 지정
super.users=User:admin:User:kafka                             # 유저 설정(여기서는 슈퍼 유저)
```
`server.properties` 파일을 불러와 ACL 을 위한 위와 같은 코드를 추가하자.    
위 설정 작업을 모든 브로커에 동일하게 설정해야한다.    

권한설정에서는 차단 정책의 우선순위가 가장 높으므로 이점을 참고하자.   
이후 브로커를 재시작해보자 

```
sudo systemctl restart kafka-server 
``` 

현재 브러커는 슈퍼 유저인 kafka 유저 외에는 다른 권한을 추가하지 않은 상태이며,    
기존의 실습에서 진행했던 토픽을 생성하는 명령어를 실행하는 유저에 대한    
권한이 할당되어 있지 않으므로 토픽이 생성되지 않을 것이다.    
 
하지만 주키퍼에는 커버로스가 적용되어 있지 않으므로     
별다른 인증이나 권한 없이 주키퍼 주소를 이용해 토픽을 생성할 수 있다. 

```
unset KAFKA_OPTS
/usr/local/kafka/bin/kafka-topics.sh 
--zookeeper peter-zk01.foo.bar:2181 
--create --topic peter-test09
--partitions 1
--repolication-factor 1

unset KAFKA_OPTS
/usr/local/kafka/bin/kafka-topics.sh 
--zookeeper peter-zk01.foo.bar:2181 
--create --topic peter-test10
--partitions 1
--repolication-factor 1
```

## 유저별 권한 설정 

앞선 커버로스와 카프카의 연동 작업을 하면서 peter01, peter02, admin 3명의 유저 프린시펄을 만들었다.   
해당 유저들을 이용해서 ACL 규칙을 만들고 테스트를 수행해보자.   

[#](#)  




  




