***개인적으로 찾아보고 내용을 정리하는 레포***


# @Transactional 전파유형과 적용 원리 그리고 분리방법  
  
  
## propagation 속성
- @Transactional 의 속성인 propagation으로 전파 유형을 변경할 수 있다
```java
// Default, 이미 시작되어 있는 트랜잭션이 있으면 참가(호출 메서드의 트렌잭션에 같이 묶인다), 아니면 새로 생성
Propagation.REQUIRED 

// 이미 시작되어 있는 트랜잭션이 있으면 참가, 아니면 없이 진행
Propagation.SUPPORTS

// 이미 시작되어 있는 트랜잭션이 있으면 참가, 아니면 예외 발생
Propagation.MANDATORY 

// 새로운 트랜잭션을 시작해야 하는 경우 사용. 항상 새로운 트랜잭션을 시작해야할때 사용
Propagation.REQUIRES_NEW 

// 이미 시작되어 있는 트랜잭션이 있으면 보류시킨 뒤 트랜잭션을 사용하지 않음
Propagation.NOT_SUPPORTED 

// 이미 시작되어 있는 트랜잭션이 있으면 예외 발생. 트랜잭션을 사용하지 못하도록 강제화
Propagation.NEVER 

// 이미 시작되어 있는 트랜잭션이 있으면 중첩 트랜잭션 시작(트랜잭션 안에서 트랜잭션을 다시 만듬). 
// REQUEST_NEW와는 다르게 부모 트랜잭션의 커밋 롤백 영향을 받지만 부모 트랜잭션에게 영향을 주진 않는다
Propagation.NESTED 
```  
<br>  
<br>  

## @Transactional 적용 원리 
- 아래 몇가지 예시를 보면서 설명하도록 하겠다
- 예시 1
```java
@Service
@Transactional(readonly = true)
public class TestServiceImpl implements TestService {
...
  public User firstTest(TestDto request) {
    return secondTest(request);
  }
  
  @Transactional 또는 @Transactional(propagation = Propagation.REQUIRES_NEW)
  public User secondTest(TestDto resquest) {
    return userRepository.save(request.toEntity());
  }
}
```
```
위와같은 코드가 있고 Controller에서 firstTest 메서드를 호출하였을 때
과연 secondTest 메서드에서 호출되는 save메서드를 통해 정상적으로 DB에 값이 저장될까?

정답은 저장이 되지 않는다.
```
- 예시2
```java
@Service
@Transactional(readonly = true)
public class TestServiceImpl implements TestService {
...
  
  public User firstTest(TestDto request) {
    return secondTest(request);
  }
  
  public User secondTest(TestDto resquest) {
    return userRepository.save(request.toEntity());
  }
}
```
```
그럼 만약 위와 같이 메서드에 @Transactional이 없으면 firstTest 메서드를 호출하였을 때
save 메서드호출 시 정상적으로 DB에 값이 저장 될까?

정답은 저장이 되지 않는다.
```
- 예시3
```java
@Service
@Transactional(readonly = true)
public class TestServiceImpl implements TestService {
...
  
  @Transactional // 여기 추가
  public User firstTest(TestDto request) {
    return secondTest(request);
  }
  
  public User secondTest(TestDto resquest) {
    return userRepository.save(request.toEntity());
  }
}
```
```
마지막으로 위와 같이 firstTest 메서드에 @Transactional을 붙여주면
save 메서드 호출 시 정상적으로 DB에 값이 저장될까?

정답은 저장이 된다.
```

왜? 세가지 상황이 모두 될것같은데 예시3번만 동작을 하였을까?  
이유를 알기 위해선 @Transactional이 무엇을 기반으로 동작하는지 이해를 하면 된다.

***⭐스프링의 트랜잭션 처리는 스프링AOP를 기반으로 동작하며, 스프링 AOP는 프록시를 기반으로 동작한다⭐***

Controller에서 TestServiceImpl이 아닌 TestService 인터페이스를 의존하고 있고 스프링 DI를 통해 프록시 객체를 주입받게 된다.  
이후 firstTest 메서드를 호출하게 되면 프록시 내에서 모든 메서드 호출에 대한 interceptor들을 위임받아 설정한 Advice를 실행시킨다  
그 이후에 실제로 firstTest 메서드가 호출되게 되는 것 이다

**결국 프록시 객체를 통하여 실행이 되어야 @Transactional이 적용되는 것 이다.**

그럼 위 예시들을 하나씩 풀어보면 아래와 같다
- 예시1
```
예시1 에선 클래스 상단에 @Transactional(readonly = true)와 
secodeTest 메서드 상단에 @Transactional 또는 @Transactional(propagation = Propagation.REQUIRES_NEW)가 있다

간략하게 순서를 정리해보자면 아래와 같다
컨트롤러에서 firstTest 호출 시도 -> @Transactional(readonly = true) 적용 -> firstTest 호출
-> secondTest 호출 -> save 호출 시도 -> @Transactional 적용 -> save 호출 -> 이후 생략 -> 종료

컨트롤러에서 firstTest 메서드를 호출할 때 별도의 트랜잭션 처리가 없어서 
클래스 상단에 있는 @Transactional(readonly = true)가 적용된다
이후 secondTest 메서드를 호출하게되면 프록시를 통해서 호출되는것이 아닌 내부에서 호출하는 것이기 때문에 
별도의 트랜잭션이 생성되지 않는다

그리고 save 메서드를 호출할땐 프록시 객체를 통하여 호출이 되어 @Transactional이 적용되긴 하나,
@Transactional의 propagation속성 기본값이 REQUIRED라 부모 트랜잭션에 같이 묶이게 되어 Insert가 발생하지 않는 것이다
```
- 예시2
```
예시2도 예시1과 똑같은 맥락이다

컨트롤러에서 프록시를 통해 firstTest가 호출되는데 이때 클래스 상단의 @Transactional(readonly = true)가 적용된다
이후 save 메서드 호출 시 @Transactional이 적용되나 부모 트랜잭션에 같이 묶이게 되어 Insert가 발생하지 않는 것이다
```
- 예시3
```
예시3은 조금 다르다

앞서 firstTest가 호출될 때 클래스 상단의 @Transactional(readonly = true)가 적용되었지만,
firstTest 메서드 상단에 @Transactional이 존재하므로 해당 어노테이션이 우선순위를 갖게되어 적용된다.

결국 현재 readonly=true가 아닌 readonly=false 속성을 가진 트랜잭션이 생성되었고,
이후 save 메서드가 호출되면서 부모 트랜잭션에 같이 묶이게 되어 Insert가 정상적으로 발생하게 되는 것이다.
```  
<br>  
<br>  

# 그럼 트랜잭션을 분리하고 싶을 땐 어떻게 해야할까?
- 지금까지 찾아본 내용은 2가지 방법이 있다
- 1.내가 내 자신을 주입받아 호출하는 방법
```java
  @Autowired
  private final Testservice self;
  
  public User firstTest(TestDto request) {
    return self.secondTest(request);
  }

  @Transactional 또는 @Transactional(propagation = Propagation.REQUIRES_NEW)
  public User secondTest(TestDto resquest) {
    return userRepository.save(request.toEntity());
  }
```
```
대략적으로 표현해보자면 위와 같이 나 자신을 @Autowired로 주입받아 호출하는 방법이다
이렇게 될 경우 프록시 객체를 통하여 실행되기 때문에 트랜잭션이 생성되게 된다

다만 @Autowired 필드주입 방법은 순환참조에 있어 좋지 않다는 평가를 받기때문에 추천하지 않는다.
```
- 2.클래스 분리
```
만약 트랜잭션을 분리해야하는 경우
firstTest와 secondTest를 별도의 클래스로 분리하여 호출할 땐 DI를 통해 
주입받은 프록시 객체를 통하여 호출하는 방향으로 설계하면 된다.
```
