## 지연 로딩(Lazy Loading)

### 개념

- **지연 로딩**은 연관된 엔티티를 즉시 조회하지 않고, 실제로 해당 객체가 필요한 시점에 DB에서 조회하는 전략이다.
- JPA에서는 연관관계에 `fetch = FetchType.LAZY`를 지정하여 설정할 수 있다.

---

### 예시 코드

```java
@Entity
public class Member extends BaseEntity {
    @Id
    @GeneratedValue
    private Long Id;

    private String username;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn
    private Team team;
}

```

```java
public class JpaMain {

    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            Team team = new Team();
            team.setName("teamA");
            em.persist(team);

            Member member1 = new Member();
            member1.setUsername("member1");
            member1.setTeam(team);
            em.persist(member1);

            em.flush();
            em.clear();

            Member m = em.find(Member.class, member1.getId());
            
            System.out.println("m = " + m.getTeam().getClass());
            System.out.println("==========");
            m.getTeam().getName();
            System.out.println("==========");

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

---

### 실행 시 쿼리

```bash
Hibernate: 
    select
        m1_0.Id,
        m1_0.createdBy,
        m1_0.createdDate,
        m1_0.lastModifiedBy,
        m1_0.lastModifiedDate,
        m1_0.team_Id,
        m1_0.username 
    from
        Member m1_0 
    where
        m1_0.Id=?
m = class hellojpa.Team$HibernateProxy$pTHrLfOY
==========
Hibernate: 
    select
        t1_0.Id,
        t1_0.name 
    from
        Team t1_0 
    where
        t1_0.Id=?
==========
```

처음 `em.find()`로 `Member`를 조회할 때는 `Team`에 대한 조회 쿼리가 나가지 않은 것을 통해, `team` 필드가 **프록시 객체로 주입되었음**을 확인할 수 있다.

이후 `m.getTeam().getName()`을 호출하면, `getTeam()`은 여전히 **프록시 객체를 반환**하지만, `getName()`을 호출하는 시점에 **실제 Team 데이터가 필요하므로**, 프록시 객체가 **초기화되면서 DB에서 Team 데이터를 조회**하게 되고, 이때 쿼리가 실행된다.

## 즉시로딩(Eager Loading)

### 개념

- **즉시 로딩**은 연관된 엔티티를 조회할 때, 해당 연관 객체도 **함께 즉시 조회**하는 전략이다.
- JPA에서는 연관관계에 `fetch = FetchType.EAGER`를 지정하여 설정할 수 있다.

---

### 예시 코드

```java
@Entity
public class Member extends BaseEntity {
    @Id
    @GeneratedValue
    private Long Id;

    private String username;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn
    private Team team;
}

```

```java
public class JpaMain {

    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            Team team = new Team();
            team.setName("teamA");
            em.persist(team);

            Member member1 = new Member();
            member1.setUsername("member1");
            member1.setTeam(team);
            em.persist(member1);

            em.flush();
            em.clear();

            Member m = em.find(Member.class, member1.getId());
            
            System.out.println("m = " + m.getTeam().getClass());
            System.out.println("==========");
            System.out.println("teamName = " + m.getTeam().getName());
            System.out.println("==========");

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

---

### 실행 시 쿼리

```bash
Hibernate: 
    select
        m1_0.Id,
        m1_0.createdBy,
        m1_0.createdDate,
        m1_0.lastModifiedBy,
        m1_0.lastModifiedDate,
        t1_0.Id,
        t1_0.name,
        m1_0.username 
    from
        Member m1_0 
    left join
        Team t1_0 
            on t1_0.Id=m1_0.team_Id 
    where
        m1_0.Id=?
m = class hellojpa.Team
==========
teamName = teamA
==========
```

처음 `em.find()`로 `Member`를 조회할 때, **LEFT JOIN을 통해 연관된 `Team` 엔티티도 함께 조회되는 것**을 확인할 수 있다.

또한 `System.out.println("teamName = " + m.getTeam().getName());` 호출 시에는 **이미 `Team` 데이터가 로딩된 상태이므로 별도의 쿼리가 추가로 실행되지 않는다.**

## 주의할 점

### **가급적 지연 로딩만 사용해야 한다.**

**즉시 로딩은 JPQL에서 N+1 문제를 일으킨다.**

```java
package hellojpa;

import jakarta.persistence.*;
import java.util.List;

public class JpaMain {

    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            Team teamA = new Team();
            teamA.setName("teamA");
            em.persist(teamA);

            Team teamB = new Team();
            teamB.setName("teamB");
            em.persist(teamB);

            // 멤버 2명 생성
            Member member1 = new Member();
            member1.setUsername("member1");
            member1.setTeam(teamA);
            em.persist(member1);

            Member member2 = new Member();
            member2.setUsername("member2");
            member2.setTeam(teamB);
            em.persist(member2);

            em.flush();
            em.clear();

            List<Member> members = em.createQuery("select m from Member m", Member.class)
                    .getResultList();

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

### 실행 시 쿼리

```bash
Hibernate: 
    /* select
        m 
    from
        Member m */ select
            m1_0.Id,
            m1_0.createdBy,
            m1_0.createdDate,
            m1_0.lastModifiedBy,
            m1_0.lastModifiedDate,
            m1_0.team_Id,
            m1_0.username 
        from
            Member m1_0
Hibernate: 
    select
        t1_0.Id,
        t1_0.name 
    from
        Team t1_0 
    where
        t1_0.Id=?
Hibernate: 
    select
        t1_0.Id,
        t1_0.name 
    from
        Team t1_0 
    where
        t1_0.Id=?
```

각 `Member`는 서로 다른 `Team`에 소속되어 있기 때문에, 각기 다른 팀 정보를 조회하려면 **`Team` 테이블에 대한 쿼리가 두 번 발생**한다.

즉, `Member` 수만큼 연관된 `Team`을 조회하는 쿼리가 반복적으로 실행되며, 이는 **연관된 팀의 개수만큼 쿼리가 추가로 발생한다는 의미**다.

이와 같은 방식은 데이터가 많아질수록 성능 저하를 초래할 수 있으며, 대표적인 **N+1 문제**로 이어진다.

반면, **지연 로딩(LAZY)** 으로 설정하면 아래처럼 `Member`에 대해서만 한 번의 쿼리만 실행된다:

```bash
Hibernate: 
    /* select
        m 
    from
        Member m */ select
            m1_0.Id,
            m1_0.createdBy,
            m1_0.createdDate,
            m1_0.lastModifiedBy,
            m1_0.lastModifiedDate,
            m1_0.team_Id,
            m1_0.username 
        from
            Member m1_0

```

`Team`에 실제 접근하지 않는 한, 추가 쿼리는 발생하지 않는다. 이 덕분에 **불필요한 연관 객체 로딩을 방지**할 수 있어 성능 면에서 유리하다.

### 정리

- **실무에서는 지연 로딩(LAZY)을 기본으로 설정**한다. → 거의 반드시..?
- 그리고 연관 객체가 반드시 필요할 때는 `fetch join` 등을 활용하여 **명시적으로 즉시 로딩을 유도**한다.
- 즉시 로딩(EAGER)을 무분별하게 사용하면, JPQL·페이징 등 다양한 곳에서 **예상치 못한 N+1 문제와 성능 저하**가 발생할 수 있다.
