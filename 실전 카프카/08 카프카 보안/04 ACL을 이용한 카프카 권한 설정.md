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

전체적인 권할 설정사항은 위와 같이 진행될 예정이다.  
 
* peter01 유저는 peter-test09 토픽에 대해 읽기와 쓰기 가능   
* peter02 유저는 peter-test10 토픽에 대해 읽기와 쓰기 가능   
* admin 유저는 peter-test09, peter-test10 토픽에 대해 읽기와 쓰기 가능   

카프카에서 권한 설정을 위해 제공하는 kafka-acls.sh  명령어를 이용해     
아래와 같이 peter01 유저에 대해 ACL 규칙을 제공한다.    

```shell 
/usr/local/kafka/bin/kafka-acls.sh  
--authorizer-properties zookeeper.connect-peter-zk01.foo.bar:2181
--add 
--allow-principal User:peter01
## --allow-host 127.0.0.1
--operation Read
--operation Write
--opertaion DESCRIBE
--topic peter-test09

> 결과 출력됨 
```

출력 내용을 보면    
peter01 에 대해 모든 호스트에서 토픽 peter-test09의 읽기와 쓰기 오퍼레이션이 승인됐다는 내용을 확인할 수 있다.    
또한 규칙을 추가하지는 않았지만 허용할 특정 호스트를 필요에 따라 제어할 수 있다.  
이 방식과 마찬가지로 peter02 에도 토픽 peter-test10 에 대한 권한을 부여하자.   

```shell
/usr/local/kafka/bin/kafka-acls.sh    
--authorizer-properties zookeeper.connect=peter-zk01.foo.bar:2181    
--list
```

현재 적용되어 있는 ACL 규칙의 리스트는 다음과 같이 --list 파라미터를 추가해 확인할 수 있다.      
ACL 정책이 잘 적용되었다면 이제 다음과 같이 peter01 유저의 티켓을 발급받은 후      
콘솔 프로듀서를 이용해 peter-test09 토픽에게 peter-test09 message!라는 메시지를 전송해보자.  

```shell
kinit -kt /usr/local/kafka/keytabs/peter01.user.keytab.peter01
export KAFKA_OPTS="-Djava.security.auth.login.config=/home/ec2-user/kafka_client_jaas.conf"     
/usr/local/kafka/bin/kafka-console-producer.sh     
--boostrap-server peter-kafka01.foo.bar:9094.  
--topic peter-test10   
--producer.config kerberos.config
> peter-test10 message!
```     
   
peter-test09 message! 라는 메시지가 권한 오류 등의 문제 없이 잘 전송되었다.      
이번에는 ACL 규칙의 권한 설정을 확인해보기 위해 peter-test10 토픽으로 peter-test10 message! 라는 메시지를 전송해보자.   

```
/usr/local/kafka/bin/kafka-console-producer.sh.   
--bootstrap-server peter-kafka01.foo.bar:9094   
--topic peter-test10 --producer.config kerbreos.config
> peter-test10 message!   
```

메시지 전송을 하려고 엔터 키를 누르자마자 에러 메시지들이 뜬다.       
에러 내용을 자세히 살펴보며는 `peter-topic10 에 권한이 없다.` 는 문구를 확인할 수 있다.      
즉 peter01 유저는 peter-test10 토픽으로 메시지를 보낼 수 없습니다.   
따라서 조금 전 적용한 ACL 규칙이 잘 적용됐음을 알 수 있다.   

이제 peter10 유저를 콘솔 컨슈머로 peter-test09 토픽의 메시지를 읽어보겠습니다.    

```shell
/usr/local/kafka/bin/kafka-console-consumer.sh.   
--bootstrap-server peter-kafka01.foo.bar:9094.  
--topic peter-test09   
--from-beginning.   
--consumer.config kerberos.config
```

peter-test09 토픽에 peter01 유저의 권한 설정을 했으므로 메시지를 가져올 것이라고 예상했지만     
예상과 달리 peter-test09 토픽의 메시지를 읽지 못했다.       
출력 내용을 유심히 살펴보니 콘솔 컨슈머 그룹 네임인 console-consumer-54292 에 대한 접근 권한이 없다는 에러 메시지가 보인다.   
  
카프카에서는 리소스 타입으로 권한을 설정하도록 구분되어 있고,       
컨슈머 그룹에 대한 권한 설정은 그룹 리소스 타입의 권한이 필요하다.  

앞서 적용한 ACL 규칙의 대상은 리소스 타입이 토픽이었고,   
그룹 리소스 타입에 대해서는 ACL 규칙을 추가하지 않았기 때문에 peter01 유저가 메시지를 읽을 수 없었다.      

이제 리소스 그룹에 대한 내용을 알았으니 그룹에 대한 ACL 규칙을 추가해보자.     
특정 컨슈머 그룹 이름으로 한정할 수도 있지만, 테스트를 위해 모든 컨슈머 그룹이 가능하도록 ACL 규칙을 추가하자.   

```
/usr/local/kafka/bin/kafka-acls.sh   
--authorizer-properties zookeeper.connect=pete-zk01.foo.bar:2181.   
--add
--allow-principal User:peter01
--operation Read --group '*'
```
```
/usr/local/kafka/bin/kafka-acls.sh    
--authorizer-properties zookeeper.connect=peter-zk01.foo.bar:2181    
--list
```
실행이 완료된 후 ACL 리스트를 보았을 때 그룹 리소스 타입이 추가된 것을 확인할 수 있다.   

```shell
/usr/local/kafka/bin/kafka-console-consumer.sh.   
--bootstrap-server peter-kafka01.foo.bar:9094.  
--topic peter-test09   
--from-beginning.   
--consumer.config kerberos.config
```
다시 컨슈머를 이용해 peter-test09 토픽에 접근하면 메시지를 가져오는 것을 알 수 있다.  

  
이외에 peter02 유저를 이용한다면     
peter-test10 토픽의 메시지 전송은 성공하고    
peter-test09 토픽의 메시지 전송은 실패한다.  

superadmin의 경우 별도의 ACL을 설정하지 않아도 모든 토픽으로 메시지 전송 및 가져오기가 가능하다.    

## 참고 

대부분 기업에서는 보안을 설정하지 않은 환경에서 카프카 클러스터를 운영한다.    
이때 보안이 설정되지 않은 클러스터가 돌아가고 있는 상태에서 보안 구성을 적용하는 것도 위험이 매우 큰 작업이다.   
하지만 이번 장에서 살펴본 SSL이나 SASL 을 적용하면서    
listeners 나 advertised.listeners에 대한 포트를 추가하는 형태로 작업을 진행한다면 큰 부담 없이 보안을 적용할 수 있다.   
일반 카프카 클러스터에서 SSL을 적용하는 카프카 클러스터로 마이그레이션하는 순서는 다음과 같다.  

1. 인증서 준비 및 적용
2. 브로커의 server.properties에 SSL을 위한 9093 listners, advertised.listeners 추가  
3. 클라이언트와 브로커 간 SSL 통신 테스트 
4. 클라이언트와 브로커 접속 포트를 9092에서 9093으로 변경 
5. 브로커의 server.properties 에 security.inter.broker.protocol=SSL 추가   
6. 브로커의 server.properties 에 9092 listneres, advertised.listeners 삭제

마이그레이션 순서를 간략하게 나열했지만,   
중간 중간 설정 확인 및 SSL 통신 이상 유무 테스트등을 충분히 수행해야한다.    
또한 이러한 작업을 진행하기에 앞서 반드시 운영환경과 동일한 환경을 미리 구성한 후 예상 시나리오대로 잘 동작하는지 테스트를 해보고 나서 적용하자.    
