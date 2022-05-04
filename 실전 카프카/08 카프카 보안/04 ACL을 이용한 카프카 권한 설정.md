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


ㅇㅣ용해 peter01
ㅇㅣ
