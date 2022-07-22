## Item 90. 직렬화된 인스턴스 대신 직렬화 프록시 사용을 검토하라
***

`Serializable` 을 구현하기로 결정한 순간 언어의 정상 메커니즘은 생성자 이외의 방법으로
인스턴스를 생성할 수 있게 됩니다. 버그와 보안 문제가 일어날 가능성이 커집니다.

이 위협을 줄여줄 기법이 직렬화 프록시 기법입니다.

직렬화 프록시 패턴은 복잡하지 않습니다. 먼저, 바깥 클래스의 논리적 상태를 정밀하게
표현하는 중첩 클래스를 설계해 private static으로 선언합니다.

이 중첩 클래스가 바로 바깥 클래스의 직렬화 프록시입니다. 중첩 클래스의 생성자는 단 하나 이어야하며,
바깥 클래스를 매개변수로 받아야합니다.

이 생성자는 단순히 인수로 넘어온 인스턴스의 데이터를 복사합니다.
일관성 검사나 방어적 복사도 필요 없습니다. 설계상, 직렬화 프록시의 기본 직렬화 형태는 바깥 클래스의 직렬화 형태로 쓰기에 이상적입니다.
그리고 바깥 클래스와 직렬화 프록시 모두 `Serializable` 을 구현한다고 선언해야합니다.

```java
import java.io.InvalidObjectException;
import java.io.Serializable;

public final class Period implements Serializable {
    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        this.start = start;
        this.end = end;
        if (this.start.compareTo(this.end) > 0) {
            throw new IllegalArgumentException(
                    start + "가 " + end + "보다 높다."
            );
        }
    }

    private static class SerializationProxy implements Serializable {
        private final Date start;
        private final Date end;

        SerializationProxy(Period p) {
            this.start = p.start;
            this.end = p.end;
        }
        
        private Object readResolve() {
            return new Period(start, end);
        }

        private static final long serialVersionUID = 2323l;
    }

    public Date start() {
        return new Date(start.getTime());
    }

    public Date end() {
        return new Date(end.getTime());
    }

    private void readObject(ObjectInputStream s)
            throws IOException, ClassNotFoundException {
        throw new InvalidObjectException("Proxy Necessary!");
    }

    private Object writeReplace() {
        return new SerializationProxy(this);
    }
}
```

`writeReplace` 메서드는 
자바의 직렬화 시스템이 바깥 클래스의 인스턴스 대신 `SerializationProxy`의 인스턴스를
반환하게 하는 역할을 합니다. 달리 말해, 직렬화가 이뤄지기 전에 바깥 클래스의 인스턴스를 직렬화 프록시로 반환해 줍니다.

이 덕분에 직렬화 시스템은 결코 바깥 클래스의 직렬화된 인스턴스를 생성해낼 수 있습니다.

마지막으로 바깥 클래스와 논리적으로 동일한 인스턴스를 반환하는 `readResovle` 메서드를 추가하면 됩니다. 이 메서드는 역직렬화 시에
직렬화 시스템이 직렬화 프록시를 다시 바깥 클래스의 인스턴스로 변환하게 해줍니다.

이의 이점은 직렬화는 생성자를 이용하지 않고도 인스턴스를 생성하는 기능을 제공하는데, 이 패턴은 직렬화의 이런 언어도단적 특성을 상당부분 제거합니다.
즉, 일반 인스턴스를 만들 때와 똑같은 생성자, 정적 팩터리, 혹은 다른 메서드를 사용해 역직렬화된 인스턴스를 생성하는 것입니다.

따라서 역직렬화된 인스턴스가 해당 클래스의 불변식을 검사할 또 다른 수단을 강구하지 않아도 됩니다. 그 클래스의 정적 팩터리나 생성자가 불변식을 확인해주고 인스턴스
메서드들이 불변식을 작 지켜준다면, 따로 더 해둬야 할 일이 없는 것입니다.

직렬화된 프록시 패턴에는 한계가 두 가지 있습니다.
1. 클라이언트가 멋대로 확장할 수 있는 클래스에는 적용할 수 없습니다.
2. 객체 그래프에 순환이 있는 클래스에도 적용할 수 없습니다. 이런 객체의 메서드를 직렬화 프록시의 `readResolve` 안에서 호출하려 하면 `ClassCastException`
이 발생할 것입니다.
3. 성능이 느려집니다.