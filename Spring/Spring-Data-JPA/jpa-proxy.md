## 프록시

<img width="878" height="207" alt="Image" src="https://github.com/user-attachments/assets/a7849b95-4a23-434c-8375-66fd497d39e5" />

Member를 조회할 때 Team도 함께 조회해야 할까?

Member와 Team이 둘 다 필요할 때는 괜찮지만, Member만 필요할 때는 낭비이다. JPA는 이를 지연로딩으로 해결한다.

지연로딩에 대해 이해하려면 먼저 프록시에 대해 이해해야 한다.

### em.find() vs em.getReference()

JPA에서는 em.find() 말고도 em.getReference()라는 메서드가 제공된다.

- em.find(): 데이터베이스를 통해서 실제 엔티티 객체 조회
- em.getReference(): **데이터베이스 조회를 미루는 가짜(프록시) 엔티티 객체 조회**

```java
package hellojpa;

import jakarta.persistence.*;

public class JpaMain {

    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            Member member = new Member();
            member.setUsername("hello");

            em.persist(member);

            em.flush();
            em.clear();

            Member findMember = em.find(Member.class, member.getId());

            tx.commit();
        } catch (Exception e) {
            tx.rollback();
        } finally {
            em.close();

        }
        emf.close();
    }
}
```

`em.find()`를 사용하면 

<img width="702" height="486" alt="Image" src="https://github.com/user-attachments/assets/75da3369-a638-4816-a081-3a9f78660395" />

이와 같이 SELECT 쿼리가 나간다.

반면 `Member findMember = em.getReference(Member.class, member.getId());` 를 사용하면 

<img width="250" height="265" alt="Image" src="https://github.com/user-attachments/assets/21ad5c7b-c0db-4cc0-8ad8-727b830de04b" />

SELECT 쿼리가 따로 나가지 않는다. 

한가지 신기한 점은 

```java
            System.out.println("findMember.id = " + findMember.getId());
            System.out.println("findMember.username = " + findMember.getUsername());
```

이렇게 아이디와 유저네임을 호출했을 때이다.

이 경우 아이디를 조회할 때는 DB에 쿼리를 날리지 않지만, 유저네임을 조회할 때는 DB에 쿼리를 날린다. 그 이유는 ID는 findMember 객체에서 이미 알고있지만(id를 통해 프록시 객체를 생성했기 때문), username은 DB에서 가져와야 하는 값이기 때문이다. 즉, 프록시 객체는 실제 엔티티가 아니라 실제 값이 필요해지는 시점까지 로딩을 지연한다.

`System.*out*.println("findMember = " + findMember.getClass());` 로 클래스 정보를 조회해보면 

`findMember = class hellojpa.Member$HibernateProxy$bjvSNZ5E`  라고 뜬다. 즉, 하이버네이트가 만든 가짜 클래스라는 것이다.

## 프록시 특징

- 프록시는 실제 클래스를 상속 받아서 만들어진다.
- 실제 클래스와 겉 모양이 같고, 사용자 입장에서는 진짜 객체인지 프록시 객체인지 구분하지 않고 사용하면 된다.
- 프록시 객체는 실제 객체의 참조(target)을 보관한다.
- 프록시 객체를 호출하면 프록시 객체의 메서드를 호출한다.

<img width="731" height="241" alt="Image" src="https://github.com/user-attachments/assets/b36dd76f-2202-4472-8ef0-156a1a1cbbf0" />

근데 target이 처음에는 보관되있지 않을 텐데 어떻게 프록시 객체의 메서드를 호출할 수 있을까?

### 프록시 객체의 초기화

<img width="1024" height="628" alt="Image" src="https://github.com/user-attachments/assets/61cbec8c-6c7a-495e-8fa1-bceeebef7644" />

1. 처음에 getName() 메서드를 호출하면 Member Target의 값이 없기 때문에 JPA가 영속성 컨텍스트에 초기화 요청을 한다. 
2. 영속성 컨텍스트가 DB 조회를 해서 실제 Entity 객체를 생성한다.
3. Member target가 생성한 Entity 객체를 연결한다.
4. Member의 getName()이 호출된다.
5. 한 번 초기화하면 다시 DB 조회할 필요가 없다.

## 프록시 특징

- 프록시 객체는 처음 사용할 때 한 번만 초기화
- 프록시 객체를 초기화 할 때, 프록시 객체가 실제 엔티티로 바뀌는 것은 아님, 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근 가능
- 프록시 객체는 원본 엔티티를 상속받음, 따라서 타입 체크시 주의해야함 (== 비교 실패, 대신 instance of 사용)
- 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 em.**getReference()**를 호출해도 실제 엔티티 반환
- 영속성 컨텍스트의 도움을 받을 수 없는 준영속 상태일 때, 프록시를 초기화하면 문제 발생(하이버네이트는 org.hibernate.LazyInitializationException 예외를 터트림) → 실무에서 자주 나옴

### 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 em.**getReference()**를 호출해도 실제 엔티티 반환

이 부분을 조금 더 자세히 알아보자

```java
            Member member1 = new Member();
            member1.setUsername("member1");
            em.persist(member1);

            em.flush();
            em.clear();

            Member refMember = em.getReference(Member.class, member1.getId());
            System.out.println("refMember = " + refMember.getClass());

            Member findMember = em.find(Member.class, member1.getId());
            System.out.println("findMember = " + findMember.getClass());

            System.out.println("refMember == findMember: " + (refMember == findMember));

            tx.commit();
```

이 코드에서 `System.out.println("refMember == findMember: " + (refMember == findMember));` 실행 결과는 어떻게 나올까? JPA는`==`비교를 할 때 항상 true가 나오도록 보장해야 한다. em.find()는 실제 객체를 가져오는 메서드이기 때문에`System.out.println("findMember = " + findMember.getClass());`  의 출력 결과는 실제 클래스가 나올 것으로 예측했는데 프록시 클래스가 나왔다.

 그 이유는 proxy 객체가 한 번이라도 조회되면 em.find()를 조회해도 proxy 객체가 나온다. `==` 을 true로 보장하기 위해서.. 조금 더 자세히 들어가면 `em.find()`로 같은 PK를 조회하면, JPA는 영속성 컨텍스트를 먼저 확인하고**,** 이미 해당 ID의 **프록시 객체가 영속성 컨텍스트에 존재하므로, 새로 DB 쿼리를 날리지 않고 프록시 객체를 그대로 반환하는 것이다.**

### 왜 JPA는 `==` 비교 시 항상 `true`가 나오도록 보장해야 할까?

객체 동일성(Identity) 보장

- JPA는 **영속성 컨텍스트 단위로 동일성 보장(Identity Guarantee)** 을 제공한다.
- 동일한 트랜잭션(동일 영속성 컨텍스트) 내에서 **같은 엔티티의 PK를 기준으로 가져온 객체는 모두 동일한 인스턴스(== true)여야 한다.**

### 이유 1: 객체의 일관성 유지

- 만약 동일한 PK의 엔티티를 여러 번 조회할 때 매번 새로운 객체가 반환되면,
    - 필드가 수정된 객체
    - 수정되지 않은 다른 객체가 혼재되며 **영속성 컨텍스트의 변경 감지(Dirty Checking)가 깨진다.**
- 동일성을 보장해야,
    - 하나의 객체만 관리
    - 변경된 데이터 추적 가능
    - flush 시 한 번만 업데이트가 가능하다.

### 이유 2: 컬렉션 및 비교의 일관성

- `Set`, `Map`에 엔티티를 저장하여 관리할 때 **동일성 보장 필요**.
- `a.getMember() == b.getMember()` 비교가 의미를 가지기 위해 필요하다.

### 이유 3: 프록시(Proxy)와의 일관성 유지

- `em.getReference()`로 프록시가 먼저 로드되면,
- 이후 `em.find()` 호출 시 프록시가 아닌 실제 객체를 반환하면,
- `==` 비교 시 false가 되어 혼란이 발생한다.

**따라서 프록시가 먼저 로드되었으면 `em.find()` 호출 시에도 프록시를 그대로 반환하여 `==` 비교 시 항상 true가 되도록 보장한다.**

## 프록시 확인

- **프록시 인스턴스의 초기화 여부 확인**

PersistenceUnitUtil.isLoaded(Object entity)

- **프록시 클래스 확인 방법**

entity.getClass().getName() 출력(..javasist.. orHibernateProxy…)

- **프록시 강제 초기화**

org.hibernate.Hibernate.initialize(entity);

- 참고: JPA 표준은 강제 초기화 없음

강제 호출: **member.getName()**
