프로젝트를 진행하면서 다양한 조건을 통한 필터링을 구현해야 했다. 

이 때 조건이 있을 수도 있고 "전체"로 찾는 경우도 있다. 

기존의 방식대로라면 각 조건이 있는지 없는 지 여부에 따라 다양한 API를 구성해야 했다.
이를 한 API로 해결하고자 QueryDSL을 통한 동적 쿼리로 구현하였다.

### QueryDSL이란?
QueryDSL은 정적 타입을 이용해서 SQL과 같은 쿼리를 생성할 수 있도록 해주는 프레임워크이다.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fbyubt6%2FbtsL4ex0srk%2FVpVfxz1HAc70WdQPsZvEM1%2Fimg.png)

왜 사용하는지 설명하기 위해 다른 쿼리 생성 방식을 설명하겠습니다.

### Spring Data JPA

스프링을 사용하면 기본적인 CRUD는 보통 JPA를 통해 구현한다.

이는 repository.findAllByName(String name) 처럼 Repository에서 정해진 간단한 네이밍 룰을 사용해서 메서드를 작성하여 특정 조건에 해당하는 쿼리를 간편하게 실행할 수 있다. 

### 네이티브 쿼리(Native Query)
하지만 JPA는 간단한 대신 복잡한 조건의 데이터를 가져오기 어렵다. 실제 프로젝트를 진행해보면 join을 통해 두개 이상의 테이블을 join해야 할 때 등 
필연적으로 네이티브 쿼리를 사용할 수 밖에 없는 상황이 존재한다는 것을 알고 있을 것이다.

```java
@Query("select gm from GenerationMember gm join fetch gm.member m where gm.generation = :generation")
    List<GenerationMember> findAllByGenerationWithMember(@Param("generation") Generation generation);
```

그럴 때는 위와 같이 실제 sql문을 직접 입력하는 방식으로 진행한다. 
하지만 이는 오타가 나기 쉽고 sql문의 길이가 길어졌을 경우 가독성이 떨어지는 문제가 발생한다.

### QueryDSL의 장점
QueryDSL은 쿼리를 문자가 아니라 진짜 자바 코드로 JPQL 빌더 역할을 수행한다. 이를 통해 자바 코드로 쿼리를 작성할 수 있게 도와준다.다음은 QueryDSL 코드의 예시이다

```java
@Repository
@RequiredArgsConstructor
public class SessionRepositoryCustomImpl  implements SessionRepositoryCustom {
    private final JPAQueryFactory queryFactory;
    
    @Override
    public List<Session> findAllByFetchJoin() {
        QSession session = QSession.session;
        return queryFactory.selectFrom(session)
                .join(session.generation)
                .fetchJoin()
                .fetch();
    }
}
```

이를 통해 Java 코드를 이용해 쿼리를 작성할 수 있어 컴파일 레벨에서 잘못된 쿼리 파라미터 타입까지 확인할 수 있다.

또한 메소드 체이닝을 통해 가독성 좋은 코드를 짤 수 있다.


### Q클래스

QueryDSL이 쿼리를 Java 코드를 이용할 수 있는 이유는 무엇일까? 

그것은 Q클래스에 있다. QueryDSL은 기존에 @Entity로 선언된 클래스들을 탐색해 Q클래스를 생성한다.
그리고 생성된 Q클래스는 기존 엔티티에 더해 쿼리 작성을 위한 구조가 존재한다.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fd6Tj70%2FbtsL37lsFMQ%2FajorO1Kj7j7lU49j6Lcx00%2Fimg.png)

Q클래스 안에는 선언된 필드에 대한 필드가 존재해 Type-safe하게 쿼리를 작성할 수 있다.

```java
@Generated("com.querydsl.codegen.DefaultEntitySerializer")
public class QMember extends EntityPathBase<Member> {
    private static final long serialVersionUID = 1502383728L;
    private static final PathInits INITS = PathInits.DIRECT2;
    public static final QMember member = new QMember("member1");
    public final StringPath email = createString("email");
    public final NumberPath<Long> id = createNumber("id", Long.class);
    public final StringPath introduction = createString("introduction");
    public final StringPath name = createString("name");
    public final NumberPath<Integer> passedGenerationNumber = createNumber("passedGenerationNumber", Integer.class);
    public final StringPath password = createString("password");
    public final StringPath phoneNumber = createString("phoneNumber");
}
```

QueryDSL의 메소드 체이닝을 이용해 가독성 좋은 동적 쿼리를 짜는데 많이 이용된다.

### 동적 쿼리
서론에 말했던 것처럼 진행하는 프로젝트에는 member를 찾는 다양한 필터링이 존재한다.

```text
- 파라미터 목록
    - passedGenerationNumber (선택)
        - 멤버 합격 기수 필터링(9기면 9을 입력해야 함)
    - position (선택)
        - 멤버 포지션 필터링(선택)
        - BE, FE, DESIGN, PM
    - name (선택)
        - 멤버 이름 필터링
```

모든 필터링 항목은 선택적이다. 필터링이 걸릴수도 있고 없을 수 있다. 

이를 한 API에 처리하기 위해 QueryDSL을 이용한다.

```java
@Override
    public List<Member> findAllWithFilters(Integer passedGenerationNumber, MemberPosition memberPosition, String name) {
        QMember qMember = QMember.member;
        BooleanBuilder builder = new BooleanBuilder();

        if (passedGenerationNumber != null) {
            builder.and(qMember.passedGenerationNumber.eq(passedGenerationNumber));
        }

        if (memberPosition != null) {
            builder.and(qMember.position.eq(memberPosition));
        }

        if (name != null && !name.isEmpty()) {
            builder.and(qMember.name.containsIgnoreCase(name));
        }

        return queryFactory.selectFrom(qMember)
                .where(builder)
                .fetch();
    }
```

BooleanBuilder를 이용하면 동적으로 생성한 where절을 builder에 삽입할 수 있다. 각 파라미터의 null여부에 따라 where절에 넣을 쿼리를 다르게 생성할 수 있다.

또한 Q클래스에 있는 "eq" 등 쿼리 생성 메소드를 통해 자바 코드로 쿼리를 간편하게 작성할 수 있다.





QueryDSL은 동적 쿼리를 쉽게 작성할 수 있도록 도와주고 그 과정에서 타입 안전성과 가독성을 동시에 확보할 수 있기 때문에 많이 사용된다. 또한 추가되는 필터(파라미터)에 유연하게 대처할 수 있기 때문에 유지보수성에도 유리하다.



혹시라도 동적쿼리를 이용하게 된다면 한 번 시도해보는 것도 나쁘지 않은 것 같다.



 

 