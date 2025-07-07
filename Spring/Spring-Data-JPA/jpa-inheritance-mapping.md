# JPA 상속관계 매핑

JPA는 객체와 관계형 데이터베이스의 테이블 간 상속관계를 매핑하기 위한 기능을 제공한다.
이번 정리에서는 **Spring Data JPA가 아닌, JPA 자체의 핵심 내용**을 다룬다.

---

## 상속관계 매핑이란?

JPA에서 상속관계 매핑은 **객체의 상속 구조를 관계형 데이터베이스의 테이블에 어떻게 표현할 것인가**를 다룬다.

상속관계 매핑의 목적은:

* 객체지향적으로는 상속 구조를 유지
* 관계형 DB에서는 이를 효율적으로 표현

하는 데 있다.

---

## 1. 상속관계 매핑 3가지 방법

| 전략                                  | 설명                       | 비고           |
| ----------------------------------- | ------------------------ | ------------ |
| 조인 전략 (JOINED)                      | 각각 테이블로 변환 후 조인          | 정석, 정규화      |
| 단일 테이블 전략 (SINGLE\_TABLE)           | 한 테이블에 모두 저장, DTYPE으로 구분 | 성능 좋음, 단순할 때 |
| 구현 클래스마다 테이블 전략 (TABLE\_PER\_CLASS) | 서브타입 테이블로 각각 저장          | 거의 사용 X, 비효율 |

---

## 2. 주요 어노테이션

* `@Inheritance(strategy = InheritanceType.전략)`
  → 상속관계 매핑 전략 설정
* `@DiscriminatorColumn`
  → DTYPE 컬럼 생성 및 구분값 관리

---

## 3. 전략별 상세 비교

### 조인 전략 (JOINED)
<img width="942" height="322" alt="Image" src="https://github.com/user-attachments/assets/34a3f83d-89ff-4326-aae6-5fd2aa8aab7f" />

* 부모, 자식 각각 테이블 생성 후 `JOIN`으로 함께 사용
* 장점

  * 테이블 정규화
  * 외래 키 무결성 제약 가능
  * 저장 공간 효율적
* 단점

  * 조회 시 `JOIN`으로 성능 저하 가능
  * INSERT 시 SQL 2번 호출
* 사용 예

  ```java
  @Inheritance(strategy = InheritanceType.JOINED)
  @DiscriminatorColumn
  ```

---

### 단일 테이블 전략 (SINGLE\_TABLE)

<img width="763" height="319" alt="Image" src="https://github.com/user-attachments/assets/bc384863-cb18-46b0-a9e5-0939f11f91ed" />

* 하나의 테이블에 모든 엔티티 데이터를 저장, DTYPE으로 구분
* 장점

  * JOIN 없이 빠르게 조회 가능
  * 쿼리 구조 단순
* 단점

  * 자식 엔티티의 컬럼이 null 허용되는 경우 많음
  * 테이블이 커지면 조회 성능 저하 가능
* 사용 예

  ```java
  @Inheritance(strategy = InheritanceType.SINGLE_TABLE)
  ```
* `@DiscriminatorColumn` 없어도 DTYPE 컬럼 자동 생성

---

### 구현 클래스마다 테이블 전략 (TABLE\_PER\_CLASS)

<img width="924" height="275" alt="Image" src="https://github.com/user-attachments/assets/57ccf141-dc5b-4358-851d-31c2d550597e" />

* 각 자식 테이블만 생성, 부모 테이블은 생성되지 않음
* 장점

  * 서브 타입 구분 명확
  * not null 제약 가능
* 단점

  * 여러 자식 테이블 조회 시 `UNION ALL` 사용으로 성능 저하
  * 통합 쿼리 작성 어려움
* 사용 예

  ```java
  @Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
  ```
* 실무 및 설계 전문가 대부분 사용하지 않음

---

## 4. 실무에서 어떤 전략을 선택하는 것이 좋을까?

* 기본적으로 **조인 전략** 사용 (정석, 유지보수 및 설계 측면에서 유리)
* 구조가 단순하고 성능이 최우선이라면 **단일 테이블 전략** 사용 가능
* **구현 클래스마다 테이블 전략은 사용하지 않는 것이 좋음**

---

## 5. @MappedSuperclass

<img width="833" height="525" alt="Image" src="https://github.com/user-attachments/assets/7563b623-1440-4b54-b93e-6a86f58c0770" />

DB에는 매핑되지 않지만, **엔티티의 공통 속성 상속용**으로 사용하는 기능.

### 특징

* 상속관계 매핑 아님
* 테이블 매핑 X, `em.find()` 불가
* 자식 엔티티에 매핑 정보만 제공
* 추상 클래스 사용 권장

### 사용 예

```java
@MappedSuperclass
public abstract class BaseEntity {
    private String createdBy;
    private LocalDateTime createdDate;
    private String lastModifiedBy;
    private LocalDateTime lastModifiedDate;
    // getter, setter
}
```

### 언제 사용?

* 생성자, 수정자, 생성일자, 수정일자 등 공통 컬럼 적용 시


