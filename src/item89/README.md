## Item 89. 인스턴스 수를 통제해야 한다면 readResolve보다는 열거 타입을 사용하라
***

인스턴스가 오직 하나만을 보장하는 싱글톤 방식을 살펴봅시다.

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();

    private Elvis() {
        
    }
}
```

이 클래스는 선언에 `implements Serializable` 을 추가하는 순간 더 이상
싱글턴이 아니게 됩니다.

기본 직렬화를 쓰지 않더라도, 명시적인 `readObject`를 제공하더라도 소용없습니다.
어떤 `readObject`를 사용하든 이 클래스가 초기화될 때 만들어진 인스턴스와는
별개인 인스턴스를 반환하게 됩니다.

`readResolve` 기능을 이용하면 `readObject` 가 만들어낸 인스턴스를 다른 것으로
대체할 수 있습니다.

역직렬화한 객체의 클래스가 `readResolve` 메서드를 적절히 정의해뒀다면, 역직렬화 후
새로 생성된 객체를 인수로 이 메서드가 호출되고, 이 메서드가 반환한 객체 참조가 새로 생성된 
객체를 대신해 반환합니다.

대부분의 경우 이때 새로 생성된 객체의 참조는 유지 하지 않으므로 바로 가비지 컬렉션 대상이 됩니다.

```java
import java.io.Serializable;

public class Elvis implements Serializable {
    public static final transient Elvis INSTANCE = new Elvis();

    private Elvis() {

    }

    // 진짜 Elvis를 반환하고, 가짜 Elvis는 가비지 컬렉션에 맡깁니다.
    private Object readResolve() {
        return INSTANCE;
    }
}
```

이 메서드는 역직렬화한 객체는 무시하고 클래스 초기화 때 만들어진 Elvis 인스턴스를 반환합니다.

따라서, Elvis 인스턴스의 직렬화 형태는 아무런 실 데이터를 가질 이유가 없으니 모든 인스턴스 필드를 transient 로 선언해야 합니다.

**`readResolve`를 인스턴스 통제 목적으로 사용한다면 객체 참조 타입 인스턴스 필드는 모두 transient로 선언해야 합니다.**

싱글턴이 transient가 아닌 참조 필드를 가지고 있다면, 그 필드의 내용은 `readResolve` 메서드가 실행되기 전에 역직렬화 됩니다.

먼저, `readResolve` 메서드와 인스턴스 필드 하나를 포함한 도둑 클래스를 작성합니다. 이 인스턴스 필드는 도둑이 '숨길' 직렬화된 싱글턴을
참조하는 역할을합니다. 직렬화된 스트림에서 싱글턴의 비휘발성 필드를 이 도둑의 인스턴스로 교체합니다. 이제 싱글턴은 도둑을 참조하고 
도둑은 싱글턴을 참조하는 순환고리가 만들어집니다.

싱글턴이 도둑을 포함하므로 싱글턴이 역직렬화할 때 도둑의 `readResolve` 메서드가 먼저 호출됩니다.
그 결과, 도둑이 `readResolve` 메서드가 수행될 때 도둑의 인스턴스 필드에는 역직렬화 도중인 싱글턴의 참조가 담겨있게 됩니다.

도둑의 `readResolve` 메서드는 이 인스턴스 필드가 참조한 값을 정적 필드로 복사하여 `readResovle`가 끝난 후에도 참조할 수 있도록합니다.
그런 다음 이 메서드는 도둑이 숨긴 transient가 아닌 필드의 원래 타입에 맞는 값을 반환합니다.

다음의 잘못된 예를 봅시다.

```java
import java.io.Serializable;

public class Elvis implements Serializable {
    public static final transient Elvis INSTANCE = new Elvis();

    private Elvis() {

    }

    private String[] favoriteSongs = {"Hound Dog", "Heartbreak Hotel"};
    
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }

    // 진짜 Elvis를 반환하고, 가짜 Elvis는 가비지 컬렉션에 맡깁니다.
    private Object readResolve() {
        return INSTANCE;
    }
}
```

위의 예는 transient가 아닌 필드를 갖고 있습니다

```java
import java.io.Serializable;

public class ElivisStealer implements Serializable {
    static Elvis impersonator;
    private Elvis payload;
    
    private Object readResolve() {
        // resolve 가 되기 전의 Elvis 인스턴스의 참조를 저장합니다.
        impersonator = payload;
        
        // favoriteSongs 필드에 맞는 타입의 객체를 반환합니다.
        return new String[] {"A Fool Such As I"};
    }
    private static final long serialVersionUID = 0;
}
```

직렬화 가능한 인스턴스 통제 클래스를 열거 타입을 이용해 구현하면 선언한 상수외의 다른 객체는 존재하지 않음을 자바가 보장해줍니다.

```java
public enum Elvis {
    INSTANCE;
    private String[] favoriteSongs = {"Hound Dog", "Heartbreak Hotel"};
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
}
```

인스턴스 통제를 위해 `readResolve`를 사용하는 방식이 완전히 쓸모없는 것은 아닙니다. 직렬화 가능 인스턴스 통제 클래스를 작성해야 하는데,
컴파일타임에는 어떤 인스턴스들이 있는지 알 수 없는 상황이라면 열거 타입으로 표현하는 것이 불가능하기 때문입니다.

**`readResolve` 메서드의 접근성은 아주 중요합니다.**

final 클래스에서라면 `readResolve` 메서드는 private 이어야 하빈다.
protected나 public으로 선언하면 같은 패키지에 속한 하위 클래스에서만 사용할 수 있습니다.
protected 나 public이면서 하위 클래스에 재정의하지 않았다면, 하위 클래스의 인스턴스를 역직렬화하면 상위 클래스의 인스턴스를 생성하여
`ClassCastException`이 발생할 수 있습니다.