# SSL을 이용한 카프카 암호화.  

자바 기반 애플리케이션에서는 **키스토어**라고 불리는 인터페이스를 통해 `퍼블릭 키`, `프라이빗 키`, `인증서`를 추상화해 제공한다.       
카프카 또한 자바 기반 애플리케이션이므로 자바의 `keytool` 이라는 명령어를 이용해 **카프카에 SSL 적용 작업을 진행하게 된다.**         

[#](#)  

카프카 SSL 적용을 위한 전반적인 구성도를 나타냈으며 카프카 단독 서버 모드가 아닌 클러스터 환경 기준이다.       

## 브로커 키스토어 생성 

클러스터 내 각 브로커마다 프라이빗 키와 인증서를 만들어야 하므로 이를 저장하기 위한 키스토어를 먼저 생성한다.     
SSL을 설명하기에 키스토어와 트러스트스터오를 살펴보자.  

[#](#)   

키스토어와 트러스트스토어라는 용어는 클라이언트와 서버 사이에 자바 애프릴케이션을 이용해 SSL 연결을 할 때 사용한다.   
키스토어와 트러스트스토어 모두 keytool 을 이용해 관리되며, 각 스토어에 저장되는 내용은 다소 차이가 있다.    

**키스토어**
* 서버 측면에서 프라이빗 키와 인증서를 저장하며, 자격 증명을 제공한다.    
* 그외에 프라이빗하고 민감한 정보를 저장한다.     
   
**프라이빗스토어**   
* 클라이언트 측면에서 서버가 제공하는 인증서를 검증하기 위한 퍼블릭 키와 서버와 SSL 연결에서 유효성을 검사하는 서명된 인증서를 저장한다.  
* 민감한 정보는 저장하지 않는다.  

```shell
kafka01> sudo mkdir -p /usr/local/kafka/ssl
kafka01> cd /usr/local/kafka/ssl/
kafka01> export SSLPASS=peterpass
```
아제 SSL 적용에 필요한 파일들을 생성하기 전 모두 한 곳에 모아두기 위해 SSL 이라는 디렉토리를 먼저 만든다.   
키스토어와 트러스트스토어 생성 시 비밀번호를 입력하는 경우가 많다.   
따라서 반복적인 작업을 최소화하기 위해 비밀번호를 환경변수로 비밀 번호룰 한번만 등록한 후 재사용하자.   

비밀번호 변경도 가능하지만 되도록 변경하지 않는 방향으로 진행하자.   
이제 준비가 완료되었다면 키스토어를 생성하자.  

```
sudo keytool -keystore kafka.server.keystore.jks -alias localhost -keyalg RSA -validity 365 -genkey -storepass $SSLPASS -keypass $SSLPASS -dname "CN=peter-kafka01.foo.bar" -storetype pkcs12
```

|옵션이름|설명|
|-----|---|
|keytool|키스토어 이름|
|alias|별칭|
|keyalg|키 알고리즘|
|genkey|키 생성|
|validity|유효일자|
|storepass|저장소 비밀번호|
|keypass|키 비밀번호|  
|dname| 식별 이름|
|stroetype|저장 타입|  

각 호스트별로 -dname 을 달리해주는 것을 신경쓰자(CN=호스트네임)   
이후, 파일이 정상적으로 생성되었다면 키스토어의 내용을 확인해보자.   

```shell
keytool -list -v -keystore kafka.server.keystore.jks
키 저장소 비밀번호 입력: // peterpass

> 출력 : 
키 저장소 유형: PKCS12
키 저장소 제공자: SUN

키 저장소에 1개의 항목이 포함되어 있습니다.

별칭 이름: localhost
생성 날짜: 2022. 4. 30
항목 유형: PrivateKeyEntry
인증서 체인 길이: 1
인증서[1]:
소유자: CN=peter-kafka01.foo.bar
발행자: CN=peter-kafka01.foo.bar
일련 번호: 549a009
적합한 시작 날짜: Sat Apr 30 14:53:50 KST 2022 종료 날짜: Sun Apr 30 14:53:50 KST 2023
인증서 지문:
	 MD5:  88:06:C1:90:77:5F:A4:E7:8B:7C:CF:65:F6:DD:5F:AB:5C:32:11:EB
	 SHA1: 30:3C:E2:04:66:50:22:08:85:3F:19:EF:48:86:3B:1D:62:6A:CA:F2:DC:03:11:26:9F:94:C6:C8:4B:4F:6C:B3
	 SHA256: SHA256withRSA
서명 알고리즘 이름: 2048비트 RSA 키
주체 공용 키 알고리즘: 3
~~~ 생략 
*******************************************
*******************************************
```

## CA 인증서 생성  

공인 인증서를 사용하는 이유는 위조된 인증서를 방지해 클라이언트-서버 간 안전한 통신을 하기 위해서이다.   
이러한 역할을 보장해주는 곳이 인증기관-`CA`이다. (여권이라고 생각하면 된다)   
     
CA가 인증서를 서명하면 그 인증서는 비로소 공인 인증된 자격을 갖게 된다.       
따라서 서버가 공인 인증된 ㅇ니증서를 갖고 있다는 것은 클라이언트들이 신뢰할 수 있는 사실을 입증한다.   
   
일반적으로 CA에 일부 비용을 지불하고 인증서를 발급받는데,   
테스트나 개발 환경등에서 비용을 지불하고 인증서를 발급받는 것은 부담이므로   
개인이 직접 자체 서명된 CA 인증서나 사설 인증서를 생성하기도 한다.   
우리는 자체 서명된 CA 인증서를 생성한 후 사설 인증서에 자체 서명된 CA 서명을 통해 신뢰할 수 있는 내부 환경을 구축하자.     

```
sudo openssl req -new -x509 -keyout ca-key -out ca-cert -days 356 -subj "/CN=foo.bar" -nodes
```   
```
Generating a 2048 bit RSA private key
.................................................................+++
..................................................................................+++
writing new private key to 'ca-key'
-----
```
 
|옵션 이름|설명|  
|-------|--|  
|new|새로 생성 요청|  
|x509|표준 인증서 번호| 
|keyout|생성할 키 파일 이름|  
|out|생성할 인증서 파일 이름|   
|days|유효 일자|  
|subj|인중서 제목|  
|nodes|프라이빗 키 파일을 암호화하지 않음|     

<img width="394" alt="image" src="https://user-images.githubusercontent.com/50267433/166094194-10f12086-417e-4d95-bcfa-73f117b95af6.png">.  
   
새로운 ca-key 가 생성되었다.   
인증서 파일인 ca-cert dhk 프라이빗 키 파일인 ca-key 가 생성된 것을 확인할 수 있다.   
프라이빗 키는 서버측에서만 갖고 있어야하는 매우 중요한 키다.    
여담이지만 간혹 특정 웹사이트에서 인증기관으로부터 인증된 인증서를 적용했음에도 프라이빗키가 유출되어 보안사고가 발생하는 경우가 종종 있다.  
인증서 관리자는 프라이빗키가 절대로 외부에 유출되지 않도록 해야한다.    


## 트러스트스토어 생성 

일반적으로 서버가 공인 인증서를 발급받은 경우     
클라이언트는 서버측으로 접속해서 서버가 갖고 있는 인증서가 신뢰할 수 있는 것인지 여부를 확인한다.      
**서버의 인증서가 신뢰할 수 있다고 판단하면 보안 통신을 시작하게 된다.**   
하지만, 사설 인증서는 인증기관을 통해 발급 받은 인증서가 아니므로 서버에 접속한 클라이언트는 이 인증서를 신뢰할 수 없다.   
**따라서 다음과 같이 생성한 자체 서명된 CA 인증서를 클라이언트가 신뢰할 수 있도록 트러스트스토어에 추가한다**    

```shell
sudo keytool -keystore kafka.server.truststore.jks -alias CARoot -importcert -file ca-cert -storepass $SSLPASS -keypass $SSLPASS
이 인증서를 신뢰합니까? [아니오]:  y
```

|옵션 이름|설명|
|------|---|
|keytool|키스토어 이름|
|alias|별칭|
|importcert|인증서를 임포트|
|file|인증서 파일|
|storepass|저장소 비밀번호|
|keypass|키 비밀번호|    

트러스트 스토어가 생성되었다.  

```shell
keytool -list -v -keystore kafka.server.truststore.jks
```
```
키 저장소 비밀번호 입력:
키 저장소 유형: jks
키 저장소 제공자: SUN

키 저장소에 1개의 항목이 포함되어 있습니다.

별칭 이름: caroot
생성 날짜: 2022. 4. 30
항목 유형: trustedCertEntry

소유자: CN=foo.bar
발행자: CN=foo.bar
일련 번호: eab9c148b1289801
적합한 시작 날짜: Sat Apr 30 15:14:55 KST 2022 종료 날짜: Fri Apr 21 15:14:55 KST 2023
인증서 지문:
	 MD5:  23:13:18:17:C7:00:22:89:35:AC:D1:99:38:14:A9:96:F1:B7:FE:8A
	 SHA1: 5B:C6:A4:70:52:70:66:0D:2E:92:89:BB:E9:F7:00:B7:BA:44:0C:5F:A4:BB:2C:2E:87:64:15:37:DC:16:B9:E2
	 SHA256: SHA256withRSA
서명 알고리즘 이름: 2048비트 RSA 키
주체 공용 키 알고리즘: 3
버전: {10}
~~ 생략 
```  

출력 내용을 보면 인증서 생성시 설정한 CN 정보와 유효 기간등이 잘 적용되었는지 확인한다.   

## 인증서 서명 

키스토어에 저장된 모든 인증서들은 자체 서명된 CA의 서명을 받아야한다.   
그렇게 해야만 클라이언트가 인증서 요청을 보냈을 때, 해당 인증서를 신뢰할 수 있기 때문이다.  

```
sudo keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file -storepass $SSLPASS -keypass $SSLPASS
```

<img width="750" alt="image" src="https://user-images.githubusercontent.com/50267433/166095074-4bebbbf9-0320-49d5-af41-6c1042feb5bd.png">

cert-file 이 생성된 것을 확인했다면 자체 서명을 적용해보자.   

```
sudo openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out -cert-signed -days 365 -CAcreateserial -passin pass:$PASSWORD
```
```
Signature ok
subject=/CN=peter-kafka01.foo.bar
Getting CA Private Key
```

|옵션 이름|설명|
|------|---|
|x509|표준 인증서 번호|
|req|인증서 서명 요청|
|ca|인증서 파일|
|cakey|프라이빗 키 파일|
|in|인풋 파일|
|out|아웃풋 파일|
|days|dbgydlfwk|
|passin|소스의 프라이빗 키 파일|

출력 내용을 보면 서명이 완료되었음을 알 수 있다.   
이제 마지막으로 키스토어에 자체 서명된 CA 인증서인 ca-cert와 서명된 cert-signed 를 추가하자.  

```
sudo keytool -keystore kafka.server.keystore.jks -alias CARoot -importcert -file ca-cert -storepass $SSLPASS -keypass $SSLPASS
이 인증서를 신뢰합니까? [아니오]:  y
인증서가 키 저장소에 추가되었습니다.
```
```
sudo keytool -keystore kafka.server.keystore.jks -alias localhost -importcert -file -cert-signed -storepass $SSLPASS -keypass $SSLPASS
인증서 회신이 키 저장소에 설치되었습니다. 
```

키스토어에 자체 서명된 CA 인증서를 추가했으니 내용을 확인해보자.  


```
keytool -list -v -keystore kafka.server.keystore.jks
키 저장소 비밀번호 입력: peterpass 

~~~~ 
```
출력 내용을 보면 최초로 키스토어를 생성했을 때의 결과와 달라진 것을 확인할 수 있다.  
저장소에는 총 2개의 인증서가 저장되어있으며 자체 서명된 CA 인증서 내용이 포험되어 있음을 알 수 있다.    
지금까지 수행한 작업들을 클러스터 내 다른 브로커에서도 동일하게 진행해야한다.     

