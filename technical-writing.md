# Spring Data JPA와 쿼리

## 1. Spring Data JPA의 함정

최근 몇 년 간, 많은 백엔드 개발자가 ORM 기술로 Spring Data JPA를 활용해 왔습니다. JPA는 RDB와 객체 세상의 패러다임을 일치시켜 주었고, 훨씬 편하게 개발할 수 있었습니다. 하지만, 편해진 만큼 더 신경 써야 할 부분이 많아졌습니다. 편리함에 숨겨진 함정은 생각보다 위험할 수 있습니다. 숙련된 개발자라면 그럴 일이 없겠지만, 저를 비롯한 초보 개발자들은 JPA가 어떤 쿼리를 만드는지 모른 채 사용했을 가능성이 높습니다.

제가 개발에 참여한 서비스인 “투룻”팀에서도 프로덕션 배포를 한 후에야 정확히 어떤 쿼리가 만들어지는지 확인할 정도였으니, 이제 막 학습을 시작한 백엔드 개발자는 놓치기 쉬운 부분이라고 생각합니다. 불확실함은 결국 위험을 감수한다는 이야기입니다. 우리가 예상하기에 간단한 로직임에도 매우 방대한 양의 쿼리가 생성될 수도 있습니다. 이는 자칫 서비스 운용 중 중대한 장애로 이어질 수 있습니다.

DELETE 메소드를 활용하는 API를 떠올려봅시다. 다음과 같은 엔티티를 삭제하는 기능을 구현하려면 어떻게 해야 할까요?

```java
@Entity
public class User {
	@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Column(nullable = false)
private String nickname;
…
}
```

JPA를 사용한다면 쿼리를 잘 몰라도 메소드 단 한 줄로 삭제 기능을 완성할 수 있습니다.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
void deleteByNickname(String nickname);
…
}
```

정말 놀랍지 않나요? 어떤 쿼리가 생성되고 DB로 전송되는지, 어떤 과정을 통해 User 엔티티가 삭제되는지 전혀 몰라도 DB에서 데이터를 삭제할 수 있습니다. 쿼리를 직접 작성해야 했던 과거의 기술들에 비해 매우 편리한 기능입니다. 하지만, 앞서 설명한 것처럼 어떤 쿼리가 생성되는지 실행 시간 전까지는 정확히 알 수 없습니다.

우리는 메소드처럼 쿼리도 한 줄이 생성될 것을 기대합니다. 불행하게도 이 메소드는 DELETE 쿼리 한줄을 작성하는 것에서 끝나지 않습니다. 삭제하려는 엔티티를 조건에 맞춰 DB에서 조회하기 위해 SELECT 쿼리를 생성하고, 조회된 데이터에 대해 DELETE 쿼리를 생성합니다. 메소드 그 어디에도 SELECT 쿼리가 생성된다는 단서는 없는데도 말이죠. 이것이 JPA 쿼리 생성 기능의 함정 중 하나입니다.

## 2. JPA를 맹신하면 안 되는 이유

JPA 쿼리 생성 기능에는 어떤 쿼리가 생성될지 예측할 수 없는 점 뿐만 아니라 많은 단점이 있습니다. 하지만, 메소드 이름만으로도 원하는 쿼리를 작성하는 기능은 매우 강력한 장점입니다. 실제로 간단한 쿼리를 활용할 것이 자명한 기능은 JPA의 쿼리 생성 기능을 적극 활용하는 편이 훨씬 낫습니다.

따라서 우리는 JPA의 쿼리 생성 기능은 포기할 수 없습니다. 그렇다면 JPA 쿼리 생성 기능의 함정들을 잘 알고 피해야만 합니다. 함께 JPA 쿼리 생성 기능의 함정에 대해 알아봅시다.

### 복잡한 쿼리를 작성하기 어려움

우리가 프로젝트나 서비스 유지 보수 할 때는 매우 복잡한 쿼리를 다뤄야 할 때가 필연적으로 옵니다. 수많은 도메인이 동시에 연관관계를 갖기도 하고, 복잡한 조건을 만족하는 값을 찾아와야 할 때도 있습니다. 예를 들어, 다음과 같은 요구사항이 있다고 가정해 봅시다.

> 성별이 남성이고, 특정 나이 이상이며, 거주하는 도시가 일치하고 서비스 가입일이 1년 이내인 모든 사용자를 조회합니다.

DB에서 조건에 일치하는 쿼리를 생성하기 위해 작성한 JPA 쿼리 메소드는 다음과 같습니다.

```java
List<User> findByGenderAndAgeGreaterThanAndCityAndCreatedAtAfter(
	String gender,
	int age,
	String city,
	LocalDate oneYearBefore
);
```

길고 복잡하며 이해하기 어려운 메소드 이름이 작성되었습니다. 이렇게 JPA의 쿼리 생성 기능만 활용하면 복잡한 조건의 쿼리를 작성하기 어려워집니다. 오타가 발생하기도 쉽고 유지 보수하기도 어렵습니다. 만약 조건이 추가된다면 메소드 이름은 더 길어지고 매개변수도 늘어날 겁니다.

### 컴파일 시점에 쿼리 오류를 찾을 수 없음

Spring Data JPA는 애플리케이션 실행 시점에 메소드 이름을 분석하고 쿼리를 생성합니다. 그 말인즉, 메소드 이름에 쿼리적인 오류가 있어도 컴파일 시점에 발견할 수 없다는 것입니다. 다음과 같은 메소드를 작성했다고 가정해 봅시다.

```java
List<User> findByGenderAndAgeGreaterThen(
	String gender,
	int age
);
```

이 메소드의 마지막 조건인 `GreaterThen`은 오타입니다. 하지만 컴파일 시점에 이 오타가 잘못된 쿼리인지 알 수 없습니다. 실행 시간에 발생하는 오류는 따로 테스트 코드를 작성하지 않는 이상, 사용자가 해당 기능을 사용하기 전까진 알 수 없는 지뢰와 다름없습니다. 충분한 QA와 테스트 코드를 통해 해결할 수 있는 문제라고 하지만, 사람은 항상 실수할 수밖에 없다고 생각합니다.

### 필요한 컬럼만 조회할 수 없음

쿼리 생성 기능을 활용하면 기본적으로 엔티티의 모든 컬럼을 조회합니다. 특정 컬럼만 필요한 경우에도 항상 모든 컬럼을 조회하기 때문에 성능 저하가 발생할 수 있습니다. 예를 들어, 특정 나이 이상의 사용자의 이름을 조회하는 기능을 구현하려면 다음과 같은 쿼리가 생성되어야 합니다.

```sql
SELECT name FROM user WHERE age > ?;
```

하지만 JPA 쿼리 생성 기능을 활용할 때는 다음과 같은 메소드를 작성하게 되고,

```java
List<String> findByAgeGreaterThan(int age);
```

이 메소드는 다음과 같은 쿼리를 생성합니다.

```sql
SELECT * FROM user WHERE age > ?;
```

따라서, 필요한 컬럼만 선택적으로 조회하는 기능은 JPA 쿼리 생성 기능만으로 구현하기엔 역부족입니다. 물론 모든 컬럼을 조회해온 다음에 필요한 컬럼만 따로 추출하도록 코드를 작성해도 무방합니다. 하지만, 시간이 지날수록 요구 사항에 따라 엔티티의 컬럼은 매우 많아질 수 있고 조회 API 호출 때마다 모든 컬럼을 조회한다면 분명히 성능 저하가 발생할 수밖에 없습니다.

### 조건에 따른 삭제나 수정에는 부가 쿼리가 발생함

앞서 설명했던 것처럼 쿼리 생성 기능을 사용하여 삭제나 수정 기능을 구현할 때, Spring Data JPA는 먼저 데이터를 조회합니다. 이에 따라 예상치 못한 추가 쿼리가 발생하여 성능 저하가 일어날 수 있습니다.

예를 들어, 다음과 같은 삭제 메소드를 정의했다고 가정해 봅시다.

```java
void deleteByEmail(String email);
```

```sql
SELECT * FROM user WHERE email = ?
DELETE FROM user WHERE id = ?
```

대량의 데이터를 삭제하거나 수정해야 하는 경우, 이러한 방식은 성능 문제를 일으킬 수 있습니다. “쿼리 한 줄인데 무슨 성능 저하?”라고 생각할 수도 있지만, 쿼리 한 줄의 무게는 생각보다 무겁습니다. 하나의 쿼리가 DB로 전송되기 위해선, DB 연결을 맺고, I/O 작업을 수행하는 등, 전혀 가볍지 않은 과정을 거쳐야 합니다. 즉, 불필요한 부가 쿼리는 발생시키지 않는 것이 최선입니다.

## 3. 쿼리 생성 기능을 대체하는 방법

이런 JPA의 한계를 극복하기 위해 활용할 수 있는 여러 방법이 있습니다. 이 중 가장 많이 사용되는 두 가지 방법을 살펴보겠습니다.

### @Query

Spring Data JPA의 @Query 어노테이션을 사용하면 개발자가 직접 JPQL(Java Persistence Query Language)과 네이티브 쿼리를 직접 작성할 수 있습니다. 이 방법을 사용하면 JPA 쿼리 생성 기능의 한계를 극복하고 더 복잡하고 유연한 쿼리를 작성할 수 있습니다.

예를 들어, 다음과 같이 @Query 어노테이션을 사용할 수 있습니다.

```java
@Query("SELECT new com.example.dto.UserInfoDto(u.id, u.email, u.name) FROM User u WHERE u.age > :age AND u.city = :city")
List<UserInfoDto> findUserInfoByAgeAndCity(@Param("age") int age, @Param("city") String city);
```

@Query 어노테이션을 통해 JPA 쿼리 생성 기능의 여러 문제를 해결했습니다.

1. 메소드 이름에 복잡한 조건이 포함되지 않아도 원하는 쿼리를 만들 수 있습니다.
2. 엔티티에서 원하는 컬럼만 조회할 수 있습니다.
3. 애플리케이션 로딩 시점에 JPQL을 SQL로 파싱하기 때문에 쿼리 문법 오류를 확인할 수 있습니다.

하지만 이렇게 편리한 @Query 어노테이션에도 여러 문제가 있습니다.

1. 원하는 컬럼만 조회했을 때, 데이터를 매핑하고자 하는 객체의 패키지 레퍼런스를 작성해야 합니다.
2. 동적 쿼리를 작성할 수 없습니다.

물론, 그럼에도 JPA의 쿼리 생성 기능보다는 훨씬 유지보수하기 쉽습니다. 복잡한 쿼리를 메소드 이름을 통해 생성하는 것보단 쿼리를 작성하는 편이 훨씬 이해하기 쉽기 때문입니다. 실제로 실무에서도 메소드 이름을 간단하게 작성하고 대부분 @Query 어노테이션을 활용한다고 합니다.

### QueryDSL

QueryDSL은 타입 안전한 쿼리를 작성할 수 있게 해주는 프레임워크입니다. QueryDSL을 사용하려면 먼저 프로젝트에 QueryDSL 의존성을 추가하고, Q클래스를 생성해야 합니다. 그 후, 다음과 같이 쿼리를 작성할 수 있습니다.

```java
public List<User> findUsersByEmailAndName(String email, String name) {
    QUser user = QUser.user;
    return queryFactory
        .selectFrom(user)
        .where(user.email.eq(email)
            .and(user.name.eq(name)))
        .fetch();
}
```

위의 쿼리는 이메일과 이름이 일치하는 사용자를 조회하는 쿼리입니다. JPA 쿼리 생성 기능에 비해 훨씬 복잡하다고 생각할 수 있지만, QueryDSL의 진가는 동적 쿼리 작성에서 두드러집니다.

예를 들어, 사용자를 이메일, 이름, 나이에 따라 조회하는 기능을 구현한다고 가정해 봅시다. 또한, 각 조건은 추가될 수도, 그렇지 않을 수도 있습니다. 온전히 사용자에 의해 조건이 결정된다고 했을 때, QueryDSL을 활용하면 다음과 같이 동적 쿼리를 작성할 수 있습니다.

```java
public List<User> findUsers(String email, String name, Integer age) {
    QUser user = QUser.user;
    BooleanBuilder builder = new BooleanBuilder();

    if (email != null) {
        builder.and(user.email.eq(email));
    }
    if (name != null) {
        builder.and(user.name.eq(name));
    }
    if (age != null) {
        builder.and(user.age.gt(age));
    }

    return queryFactory
        .selectFrom(user)
        .where(builder)
        .fetch();
}
```

@Query 어노테이션이나 쿼리 생성 기능을 활용하려 했으면, 각 조건에 맞춰 메소드를 여러 개 작성해야 했을 것입니다. 선택적 조건이 늘어나면 늘어날수록 메소드로 추가로 작성해야 했겠죠.

QueryDSL의 장점을 정리하자면 다음과 같습니다.

1. 컴파일 과정에서 쿼리 오류를 발견할 수 있습니다.
2. 타입 안전한 쿼리를 작성할 수 있습니다.
3. 동적 쿼리를 쉽게 작성할 수 있습니다.
4. IDE의 자동 완성 기능을 활용할 수 있습니다.

하지만 QueryDSL도 단점이 있습니다:

1. 초기 설정이 복잡합니다.
2. 추가적인 학습이 필요합니다.
3. 프로젝트 빌드 시간이 늘어날 수 있습니다.

아무래도 Spring이 기본적으로 제공해 주는 기능은 아니다보니 추가적인 학습이 필요합니다. QueryDSL만의 문법과 추가된 여러 클래스에 대해 알아야 합니다. 하지만, 이런 초기 과정만 무난하게 넘길 수 있다면 정말 강력한 장점을 가진다고 생각합니다.

## 4. 그래서 어떤 방법을 활용해야 할까?

지금까지 살펴본 각 방법은 확고한 장단점이 있습니다. 따라서 우린 상황에 맞는 적절한 방법을 선택해야 합니다. 제가 생각한 각 방법에 어울리는 상황은 다음과 같습니다.

1. 간단하고 예상이 되는 쿼리를 작성해야 할때: JPA의 쿼리 생성 기능
   - 예: `findByEmail(String email)`
   - 장점: 직관적이고 빠르게 구현 가능

2. 복잡하지만 정적인 쿼리: @Query 어노테이션
   - 예: 
     ```java
     @Query("SELECT u FROM User u WHERE u.email = :email AND u.age > :age")
     List<User> findUsersByEmailAndAgeGreaterThan(@Param("email") String email, @Param("age") int age);
     ```
   - 장점: SQL에 익숙한 개발자가 직접 쿼리를 최적화할 수 있음

3. 복잡하고 동적인 쿼리: QueryDSL
   - 예:
     ```java
     public List<User> findUsers(String email, Integer age) {
         BooleanBuilder builder = new BooleanBuilder();
         if (email != null) {
             builder.and(user.email.eq(email));
         }
         if (age != null) {
             builder.and(user.age.gt(age));
         }
         return queryFactory.selectFrom(user).where(builder).fetch();
     }
     ```
   - 장점: 타입 안전성, 높은 유연성, 가독성 좋은 코드

하지만, 모든 선택에는 정답이 없습니다. 내가 속한 팀이 현재 처한 상황이나 서비스의 복잡도, 데이터의 크기 등의 여러 요인으로 각 방법이 오히려 비효율적일 수 있습니다. 동적 쿼리가 필요하지도 않은 상황에서의 QueryDSL 도입은 팀원들의 학습 부담을 증가시킬 뿐입니다. 따라서 문제를 해결하기 위한 수단으로 각 방법이 사용되어야 하지, 목적이 되어서는 안 됩니다.

## 5. 마치며.

마지막으로, Spring Data JPA를 사용할 때는 항상 생성되는 실제 쿼리를 확인하는 습관을 들여야 합니다. 이를 통해 예상치 못한 성능 이슈를 조기에 발견하고 해결할 수 있기 때문입니다. 부가적인 쿼리가 발생하지는 않는지, 너무 많은 서브 쿼리가 작성되진 않았는지, 의도한 대로 잘 동작하는지 항상 확인해야 합니다.

Java 코드만 잘 짠다고 해서 좋은 Spring 백엔드 개발자가 될 수 없다고 생각합니다. 필연적으로 DB와 씨름하게 될 백엔드 개발자는 쿼리에 대해 잘 이해하고 있어야 하며, 쿼리의 처리 속도를 어떻게 해야 더 높일 수 있을지 고민해야 합니다. 예를 들어, 적절한 컬럼에 인덱스를 설정하면 같은 조회 쿼리여도 속도 차이는 수백 배까지 날 수 있습니다.

이 글에선 기본적으로 JPA의 편리함을 경계하고, 발생할 수 있는 문제를 해결하기 위한 방법을 설명하고 있습니다. 상세한 쿼리 튜닝을 다루진 않지만, 초보 Spring 개발자가 꼭 알아야 하는 내용이라고 생각합니다. 이 글을 읽는 모든 분이 맹목적으로 기술을 활용하지 않고, 항상 비판적인 자세로 "더 나은 방법은 없을까?"하고 고민해주셨으면 좋겠습니다.

## 출처

- 자바 ORM 표준 JPA 프로그래밍 - 김영한 저
- [김영한의 스프링 부트와 JPA 실무 완전 정복 로드맵](https://www.inflearn.com/roadmaps/149) - 김영한
- [JPA Query Methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html) - Spring 공식 문서
- [데이터베이스 인덱스의 원리와 최적화 전략](https://f-lab.kr/insight/database-index-principles-and-optimization-strategies) - F-Lab