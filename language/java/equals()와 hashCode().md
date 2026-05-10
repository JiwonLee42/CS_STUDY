## 개념

`equals()` : 
- 두 객체 비교, true면 hashCode()도 같아야 함
- 객체 주소가 같으면 True
- == 는 값만 판단 가능, 같은 객체인지 판단하려면 equals 사용 필수
- String 객체에서는 주소가 같으면 True, 오버라이딩된 equals 함수를 사용하기 때문


`hashCode()` : 
- hashCode()가 같다고 해서 equals()가 true일 필요는 없음
- 동일한 객체들이 해시 기반 컬렉션에서 같은 버켓에 가도록 보장

## Java

equals():
- Object.equals()는 참조 비교
- 같은 타입 객체여도 같은 인스턴스가 아니라면 false

equals()를 오버라이드할 때 중요한 점
- hasCode() 오버라이드 필수: hashCode()를 오버라이드하지 않는다면 기본 Object.hashCode()가 사용됨 -> 이럴 경우 equals에서 같다고 판정이 나더라도 해시값이 다를수가 있음 -> hashSet에서 contain 여부를 조회할 때, false가 반환될 수 있음
- 자기 자신 비교, 타입 체크, 필드 비교, hashCode도 같은 필드 기준 계산 필요
```java
import java.util.Objects;

public class Member {
    private final String name;
    private final int age;

    public Member(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Member)) return false;
        Member member = (Member) o;
        return age == member.age &&
                Objects.equals(name, member.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}

```


단점
- equals만 오버라이드하고, hashCode를 만들지 않는다면 버그 발생하기 때문에 번거로움
- equalsI()와 hashCode()가  다른 필드를 보게 되면 계약이 깨짐
- mutable 필드를 기준으로 hashCode()를 생성하게 되면 문제가 생길 수 있음
	- 추후 HashSet에 해당 객체를 넣고, mutable 필드를 수정하면 hash값이 넣었을 때와 변경 이후가 다르기 때문에 문제가 생길 수 있음
	- 해시 기반 컬렉션의 key가 되는 객체는 불변이 안전함

```java
class Member {
    String name;
}


Set<Member> set = new HashSet<>();
Member m = new Member("kim");
set.add(m);

m.name = "lee";

set.contains(m); // 이상 동작 가능

```


### Kotlin

equals보다는 `==` , `===` 를 사용한 equality 를 위한 연산자 문법 사용이 권장된다. 

- == : equals() 기반 비교, 내부적으로 equals 사용
	- Kotlin의 모든 클래스는 기본적으로 Any를 상속하는데, 자바 Object처럼..
	- Any에는 이미 equals 함수가 있는데, 같은 인스턴스면 true, 다르면 false
	```kotlin
	// 기본 equals 함수
	open operator fun equals(other: Any?): Boolean
	
	// 오버라이드 equals 함수
	override fun equals(other: Any?): Boolean

	```
- === : 참조 동일성 비교, 같은 인스턴스인지 비교 
- ` Any.equals()`: 참조 동일성 성격
- `data class`는` equals()`와 `hashCode()` 자동 생성
	- 같은 타입이라면 true

- data class의 경우

```kotlin
data class Member(
    val name: String,
    val age: Int
)

val m1 = Member("kim", 20)
val m2 = Member("kim", 20)

println(m1 == m2)   // true
println(m1 === m2)  // false

```


- 일반 클래스에서 오버라이드할  경우 - hashCode() 오버라이드 필수
```java
class Money(val amount: Int) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Money) return false
        return this.amount == other.amount
    }

    override fun hashCode(): Int {
        return amount
    }
}

```




### 주의할 점
- 일반 클래스는 자동 값 비교가 아님, 데이터클래스만 해당
- 배열 비교의 경우 `equals()`보다는 `contentEquals()`로 비교

