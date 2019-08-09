# 8. 경계
시스템에 들어가는 모든 소프트웨어를 직접 개발하는 경우는 드물다. 어떤 식으로든 이 외부코드를 우리 코드에 깔끔하게 통합해야 한다.  
이 장에서는 소프트웨어 경계를 깔끔하게 처리하는 기법과 기교를 살펴본다.  

### 목차
+ 외부 코드 사용하기(서드 파티 코드)
+ 경계 살피고 익히기
+ Log4j 익히기
+ 학습 테스트는 공짜 이상이다.
+ 아직 존재하지 않는 코드를 사용하기
+ 깨끗한 경계

## 외부 코드(서드 파티 코드) 사용하기
인터페이스를 **제공하는 입장**과 **사용하는 입장** 사이에는 필연적인 긴장감이 존재한다.   
제공자는 좀 더 다양한 환경에서 좀 더 많은 사용자가 사용할 수 있도록 다양한 사용성을 지향한다.  
사용자는 자신의 요구에 집중하는 인터페이스를 바란다.  
이런 긴장으로 인해 시스템 경계에서 문제가 생길 소지가 많다.  

java.util.Map을 예로 들어보자. Map은 굉장히 다양한 인터페이스로 수많은 기능을 제공한다. Map이 제공하는 기능성과 유연성은 확실히 유용하지만 그만큼 위험도 크다.  
만약 Sensor라는 객체를 담는 Map을 만들려면 다음과 같이 Map을 생성한다.
~~~java
Map sensors = new HashMap();
Sensor s = (Sensor) sensors.get(sensorId);
~~~
위와 같은 코드가 한 번이 아니라 여러 차례 나온다. 즉 Map이 반환하는 object를 올바른 유형으로 변환할 책임은 Map을 사용하는 클라이언트에 있다.  
또한 위와 같은 코드는 의도도 분명히 드러나지 않는다. 이와 같은 문제점은 `제네릭을 `사용함으로써 해결할 수 있다.
~~~java
Map<String, Sensor> sensors = new HashMap<String, Sensor>();
Sensor s = sensors.get(sensorId);
~~~
하지만 이 방법 또한 Map객체가 필요이상의 기능을 제공하는 것은 막지 못한다.  
또한 Map의 인터페이스가 바뀌거나 할 경우 의존하고 있는 코드의 많은 부분들이 바뀌어야 한다.  
결국 제일 좋은 방법은 `래핑이다.`
~~~java
public class Sensors {
    // 경계의 인터페이스(이 경우에는 Map의 메서드)는 숨겨진다.
    // Map의 인터페이스가 변경되더라도 여파를 최소화할 수 있다.
    // 예를 들어 Generic을 사용하던 직접 캐스팅하던 그건 구현 디테일이며 Sensor클래스를 사용하는 측에서는 신경쓸 필요가 없다.
    // 이는 또한 사용자의 목적에 딱 맞게 디자인되어 있으므로 이해하기 쉽고 잘못 사용하기 어렵게 된다.

    private Map sensors = new HashMap();
    
    public Sensor getById(String id) {
        return (Sensor)sensors.get(id);
    }
	...
}
~~~
Map 클래스를 사용할 때마다 위와 같이 캡슐화라는 소리가 아니다.  
Map(혹은 유사한 경계 인터페이스)을 스템 전반에 걸쳐 돌려가며 사용하지 말는 말이다. Map과 같은 경계 인터페이스를 이용할 때는 
+ 이를 이용하는 클래스나 클래스 계열 밖으로 노출되지 않도록 주의한다.
+ Map 인스턴스를 공개 API의 인수로 넘기거나 반환값으로 사용하지 않는다.

## 경계 살피고 익히기
+ 서드 파티 코드를 사용할 때는, (우리 자신을 위해) 우리가 사용할 코드를 테스트 하는 편이 바람직하다.
> 외부 라이브러리를 사용할 때 오류 발생 시, 우리의 버그인지? 라이브러리 버그인지? 찾아내느라 골치를 앓는다.  
> 또한 외부 코드를 익히기도 어렵고 외부 코드를 통합하기도 어렵다. 동시에 하기는 두 배나 어렵다.  
> 따라서 곧바로 우리쪽 코드를 작성해 외부 코드를 호출하는 대신 먼저 간단한 테스트 케이스를 작성해 외부 코드를 익히자!!  
> 이를 `학습 테스트`라고 부른다.  
> + 학습 테스는 프로그램에서 사용하려는 방식대로 외부 API를 호출한다.  
> 통제된 환경에서 API를 제대로 이해하는지를 확인하는 셈이다.  
> 	+ 학습 테스트는 API를 사용하려는 목적에 초점을 맞춘다.

## Log4j 익히기
1 - 화면에 "hello"를 출력하는 테스트 케이스
~~~java
@Test
public void testLogCreate() {
	Logger logger = Logger.getLogger("MyLogger");
	logger.info("hello");
}
~~~
2 - Appender가 필요하다는 오류가 발생한다. 문서를 좀 더 읽어보니 ConsoleAppender라는 클래스가 있다. 생성 후 테스트 케이스를 다시 돌린다.
~~~java
@Test
public void testLogAddApender() {
  	Logger logger = Logger.getLogger("MyLogger");
	ConsoleAppender appender = new ConsoleAppender();
	logger.addAppender(appender);
	logger.info("hello");
}
~~~
3 - 이번에는 Appender에 출력 스트림이 없다고 한다. 구글 검색 후 다음과 같이 시도한다.
~~~java
@Test
public void testLogAddAppender() {
	Logger logger = Logger.getLogger("MyLogger");
	logger.removeAllAppenders();
	logger.addAppender(new ConsoleAppender(
		new PatternLayout("%p %t %m%n"),
		ConsoleAppender.SYSTEM_OUT));
	logger.info("hello");
}
// 성공했지만 ConsoleAppender를 만들어놓고 ConsoleAppender.SYSTEM_OUT을 받는건 뭔가 이상하다.
// 그래서 제거했더니 문제가 없다.
// 하지만 PatternLayout을 제거하니 또 다시 오류가 뜬다.
// 문서를 더 자세히 보니, ConsoleAppender의 기본 생성자는 '설정되지 않은 상태'란다.
// 명백하지도 않고 실용적이지도 않다... 버그이거나, 적어도 "일관적이지 않다"고 느껴진다.
~~~
조금 더 구글링, 문서 읽기, 테스트를 거쳐 log4j의 동작법을 알아냈고 그것을 간단한 유닛테스트로 기록했다.  
이제 이 지식을 기반으로 log4j를 래핑하는 클래스를 만들수 있다. 나머지 코드에서는 log4j의 동작원리에 대해 알 필요가 없게 됐다.
~~~java
public class LogTest {
    private Logger logger;
    
    @Before
    public void initialize() {
        logger = Logger.getLogger("logger");
        logger.removeAllAppenders();
        Logger.getRootLogger().removeAllAppenders();
    }
    
    @Test
    public void basicLogger() {
        BasicConfigurator.configure();
        logger.info("basicLogger");
    }
    
    @Test
    public void addAppenderWithStream() {
        logger.addAppender(new ConsoleAppender(
            new PatternLayout("%p %t %m%n"),
            ConsoleAppender.SYSTEM_OUT));
        logger.info("addAppenderWithStream");
    }
    
    @Test
    public void addAppenderWithoutStream() {
        logger.addAppender(new ConsoleAppender(
            new PatternLayout("%p %t %m%n")));
        logger.info("addAppenderWithoutStream");
    }
}
~~~

## 학습 테스트는 공짜 이상이다.
+ 학습 테스트에 드는 비용은 없다.
	+ 어쩃든 API를 배워야 하므로, 오히려 필요한 지식만 확보하는 손쉬운 방법이다.
	+ 학습 테스트는 이해도를 높여주는 정확한 실험이다.
+ 학습 테스트는 공짜 이상이다.
	+ 투자하는 노력보다 얻는 성과가 더 크다.
	+ 패키지 새 버전이 나온다면 학습 테스트를 돌려 차이가 있는지 확인한다.
		+ 새 버전이 우리 코드와 호환되지 않으면 학습 테스트가 이 사실을 곧바로 밝혀낸다.
+ 학습 테스트를 이용한 학습이 필요하든 그렇지 않든, 실제 코드와 동일한 방식으로 인터페이스를 사용하는 테스트 케이스가 필요하다.
	+ 이런 경계 테스트가 있다면 패키지의 새 버전으로 이전하기 쉬워진다.   
	그렇지 않다면 낡은 버전을 필요 이상으로 오랫동안 사용하려는 유혹에 빠지기 쉽다.

## 아직 존재하지 않는 코드를 사용하기
아직 개발되지 않은 모듈이 필요한데, 기능은커녕 인터페이스조차 구현되지 않은 경우가 있을 수 있다.  
> 저자는 무선통신 시스템을 구축하는 프로젝트를 하고 있었다.
  그 팀 안의 하부 팀으로 "송신기"를 담당하는 팀이 있었는데 나머지 팀원들은 송신기에 대한 지식이 거의 없었다.
  "송신기"팀은 인터페이스를 제공하지 않았다. 하지만 저자는 "송신기"팀을 기다리는 대신 "원하는" 기능을 정의하고 인터페이스로 만들었다. [지정한 주파수를 이용해 이 스트림에서 들어오는 자료를 아날로그 신호로 전송하라]
  이렇게 인터페이스를 정의함으로써 메인 로직을 더 깔끔하게 짤 수 있었고 목표를 명확하게 나타낼 수 있었다.
~~~java
public interface Transimitter {
    public void transmit(SomeType frequency, OtherType stream);
}

public class FakeTransmitter implements Transimitter {
    public void transmit(SomeType frequency, OtherType stream) {
        // 실제 구현이 되기 전까지 더미 로직으로 대체
    }
}

// 경계 밖의 API
public class RealTransimitter {
    // 캡슐화된 구현
    ...
}

public class TransmitterAdapter extends RealTransimitter implements Transimitter {
    public void transmit(SomeType frequency, OtherType stream) {
        /* 
			RealTransimitter(외부 API)를 사용해 실제 로직을 여기에 구현.
	        Transmitter의 변경이 미치는 영향은 이 부분에 한정된다.
         */
    }
}

public class CommunicationController {
    // Transmitter팀의 API가 제공되기 전에는 아래와 같이 사용한다.
    public void someMethod() {
        Transmitter transmitter = new FakeTransmitter();
        transmitter.transmit(someFrequency, someStream);
    }
    
    // Transmitter팀의 API가 제공되면 아래와 같이 사용한다.
    public void someMethod() {
        Transmitter transmitter = new TransmitterAdapter();
        transmitter.transmit(someFrequency, someStream);
    }
}
~~~

## 깨끗한 경계
소프트웨어 설계가 우수하다면 변경하는데 많은 투자와 재작업이 필요하지 않는다. 통제하지 못하는 코드를 사용할 때는 너무 많은 투자를 하거나 향후 변경 비용이 지나치게 커지지 않도록 각별히 주의해야 한다.  
+ 경계에 위치하는 코드는 깔끔히 분리한다.
+ 또한 기대치를 정의하는 테스트 케이스도 작성한다.  

통제가 불가능한 외부 패키지에 의존하는 대신 통제가 가능한 우리 코드에 의존하는 편이 훨씬 좋다.  
`외부 패키지를 호츨하는 코드를 가능한 줄여 경계를 관리하자.`
> Map에서 봤듯이, 새로운 클래스로 경계를 감싸거나 Adapter 패턴을 사용해 우리가 원하는 인터페이스를 패키지가 제공하는 인터페이스로 변환하자  
> + 어느 방법이든 코드 가독성이 높아진다.
> + 경계 인터페이스를 사용하는 일관성도 높아진다.
> + 외부 패키지가 변했을 떄 변경할 코드도 줄어든다.