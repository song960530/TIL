# DB Connection을 고려한 최적화 (@Transactional 전파속성, 메서드 분리, OSIV활용)

과제로 진행중이였던 프로젝트 리뷰를 진행하던 중 트랜잭션과 DB커넥션 관련하여 질문을 받은적이 있다.  
restTemplate를 사용하여 api통신을 하는 부분이 있었는데 해당 부분에서 질문이 들어왔다  

**Q. restTemplate사용해서 통신할때 통신 시간이 길어지면 DB커넥션도 응답이 올때까지 계속 물려있나요?**  
A. 😨?????  

전혀 생각하지 못했던 부분이였다. 평소에 기능구현을 위주로 개발하는 습관이 있어서 성능적인 부분에는 무지했던 것 이다  
그래서.. 열심히 찾아봤고, @Transactional의 동작원리와 전파속성을 찾아보면서 트랜잭션에 대한 기본적인 이해를 할 수 있게 되었다.  
ps. [@Transactional 전파유형과 적용 원리 그리고 분리방법](https://github.com/song960530/TIL/tree/main/spring/transaction)  

오케이. 트랜잭션은 이해했는데 이게 DB커넥션이랑도 관계가 있는건가?  
느낌상 관계가 있는것 같은데... DB 커넥션을 가져오는 시점이 언제인지 알면 위에서 받았던 질문에 대한 대응을 할 수 있지 않을까?  

그래서 열심히 열심히 찾아봤고 OSIV까지 가서야 이해를 할 수 있었다.(인강에서 그냥 흘려듣고 넘겼던 부분이였다..ㅠ😥)  
  
-------------------------------------------------------------------------------------------------------------------------------------
## OSIV
하이버네이트에선 Open Session In View, JPA에선 Open JPA In View 라고 불린다  
영속성 컨텍스트의 라이프사이클을 트랜잭션 범위로 할지? API응답 시점까지 유지할지? 설정을 통해 정하는 것 이다  

- OSIV ON  
![OSIV ON](https://user-images.githubusercontent.com/52727315/156882628-3c482364-e996-4ffb-a092-7fa6830abcf5.png)  

OSIV 전략을 사용하면 DB 커넥션 시작 시점부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 DB커넥션을 유지한다  

장점으론 DB커넥션과 영속성 컨텍스트를 계속 유지하고 있기 때문에 Lazy Loading이 가능하다는 것 이다  
장점이 있으면 단점도 있는 법 최종 응답까지 DB커넥션을 사용하기 때문에 장애로 이어질 수 있다  

- OSIV OFF  
![OSIV OFF](https://user-images.githubusercontent.com/52727315/156882625-b6cf1e6b-56b1-4159-8b1b-aecc65d63cf6.png)  

OSIV를 끄면 트랜잭션 종료 시 영속성 컨텍스트도 같이 종료가되고, DB커넥션도 반환하게 된다  

장점으론 트랜잭션 종료 시점에 영속성 컨텍스트가 종료되고 DB커넥션을 바로 반환한다는 점 이다  
반대로 단점으론 트랜잭션을 벗어나는 순간 Lazy Loading이 불가능하므로 트랜잭션 내에서 fetch join(JPQL join fetch)등으로 조회해야하는 엔티티는 미리 조회를 해야한다는 점 이다

-------------------------------------------------------------------------------------------------------------------------------------  

여기까지 이해를 하였더니 위에서 받은 질문에 대한 대응으론 굳이 OSIV 옵션까지 변경하지 않아도 된다는 결론에 도달하게 되었다.  
API를 호출하는 메서드를 별도로 분리하여 상단에 전파속성만 바꿔줘도 된다
```java
    @Override
    @Transactional(propagation = Propagation.NEVER)
    public ResponseEntity<Object> callScrapApi(User user) {
        ResponseEntity<Object> apiResponse;

        ...// restTemplate로 통신하는 부분

        return apiResponse;
    }
```
위 코드처럼 @Transactional(propagation = Propagation.NEVER)을 통해 "이 메서드는 트랜잭션에 포함되면 안된다"고 명시를 해주고 별도의 메서드로 분리하여 
Controller에서 호출한 후 이후 처리를 위한 메서드를 호출해주면 된다.  
**(✨Service내에서가 아닌 컨트롤러에서 프록시를 통하여 호출되도록 하는것이 핵심. 이거때문에 메서드를 분리하여 별도로 호출해준거임)**  

하지만 OSIV까지 찾게되었던 건 API호출 메서드부분이 아닌 그 앞단의 문제때문이였다  
앞단에서 AOP를 걸어둔 상태라 Login관련 Aspect클래스가 동작하게 되어있는데 이부분에서 서비스쪽 메서드를 한번 호출하여 조회하는 부분이 있다  

예상했던 바에선 조회가 끝난 뒤 DB커넥션을 반납할 줄 알았는데 커넥션을 반납하지 않는것이였다... restTemplate을 통해 API응답을 전달받을때까지, 그리고 그 이후까지
계속 DB커넥션 하나가 반납되지 않고 최종응답을 반환해주고 나서야 반납이 되는것이였다  
-> 이거때문에 찾아보게 된 것 이였구 OSIV까지 찾아보게 된 것이였다.  

위부분 관련해선 굳이 DB조회까지 갈 필요가 없을 것 같아 로직을 수정하여 DB조회를 하지 않도록 수정하였고, 해당 메서드 상단에
```java
@Transactional(propagation = Propagation.SUPPORTS)
```
를 붙여줘서 트랜잭션이 있으면 포함되고, 없으면 없이 진행하도록 수정해주었다. (혹시 다른부분에서 사용할수도 있어서 SUPPORTS로 지정해줬음)  

그리고 추가적으로 트랜잭션 단위로 DB커넥션을 고민하면서 개발하면 성능적으로나, 개발측면적으로나(개인적으로) 더 좋을것 같다 판단이 되어 OSIV기능을 사용하지 않도록 설정을 추가해주었다
```
// application.yml

spring:
  jpa:
    open-in-view: false
```

-------------------------------------------------------------------------------------------------------------------------------------

## 결론

최적화 부분에서 효과를 본것은 아래와 같다
1. Filter와 AOP부분에서 로그인관련 엔티티조회가 3번씩 이뤄지던것을 전파속성 변경과 로직 수정으로 1번조회로 공유가 가능하도록 최적화
2. restTemplate을 사용하여 통신할 때 계속 DB Connection이 물려있던 문제 해결. (전파속성, 로직수정으로 해결)
3. OSIV기능을 끔으로 써 트랜잭션 종료 시 DB Connection 리소스를 바로바로 반납하도록 최적화  

최적화는 정말 끝이 없는 것 같다.  
글을적고있는 지금도 수정해야할 것 같은 부분이 눈에 보이는것 같고 조금 더 나은 방법이 있을것 같다는 느낌이 든다  
그래도 이렇게 한번 고민해보고 적용해보니 기존에 신경쓰지 않았던 부분을 알게되어 한층 더 성장한 느낌이 들어 뿌듯한 시간이였다😆


🍖아참 그리고 OSIV기능을 사용하지 않을땐 Command(필수 비즈니스)와 Query(화면에 특화된 조회성 비즈니스)를 구분하여 관리해주면 좋다고 한다!  
🍖아아 그리고 HikariCp에서 connection 가져오고 반납하는부분 확인하고 싶을 땐
```java
  // Connection 가져오는 부분
  // HikariPoll.java getConnection 메서드 172 라인
  
   public Connection getConnection(final long hardTimeout) throws SQLException
   {
      suspendResumeLock.acquire();
      final long startTime = currentTime();

      try {
         long timeout = hardTimeout;
         do {
            PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
            if (poolEntry == null) {
               break; // We timed out... break and throw exception
            }
    ...
```
```java
  // Connection 반납하는 부분
  // ProxyConnection.java close 메서드 247라인

   public final void close() throws SQLException
   {
      // Closing statements can cause connection eviction, so this must run before the conditional below
      closeStatements();

      if (delegate != ClosedConnection.CLOSED_CONNECTION) {
         leakTask.cancel();

         try {
            if (isCommitStateDirty && !isAutoCommit) {
```
요기 디버그 걸면 DB 커넥션 가져오고 반납하고 하는걸 확인할 수 있다
