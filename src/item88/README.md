## Item 88. readObject 메서드는 방어적으로 작성하라

***

다음의 클래스를 봐봅시다.

```java
public final class Period {
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
    
    public Date start(){
        return new Date(start.getTime());
    }
    
    public Date end() {
        return new Date(end.getTime());
    }
}
```

이 클래스를 직렬화하기로 결정했다고 해봅시다.
Period 객체의 물리적 표현이 논리적 표현과 부합하므로 기본 직렬화 형태를 사용해도 나쁘지 않습니다.

하지만, 이렇게 해서는 이 클래스의 주요한 불변식을 더는 보장하지 못합니다.

문제는 `readObject` 메서드가 실질적으로 또 다른 public 생성자이기 때문입니다.
보통의 생성자처럼 `readObject` 메서드에서도 인수가 유효한지 검사해야하고 필요하다면 매개변수를 방어적으로 복사해야합니다.

`readObject` 는 매개변수로 바이트 스트림을 받는 생성자라 할 수 있습니다.
보통의 경우 바이트 스트림은 정상적으로 생성된 인스턴스를 직렬화해 만들어집니다.
하지만 불변식을 깨뜨릴 의도로 임의 생성한 바이트 스트임을 건네면 문제가 생깁니다. 정상적인 생성자로는 만들어낼 수 없는 객체를 생성해낼 수도 있기 때문입니다.

이러한 문제를 고치려면 `Period`의 `readObject` 메서드가 `defaultReadObject`를 호출한 다음 역직렬화된 객체가 유효한지 검사해야합니다.

이의 작업으로 허용되지 않은 Period 인스턴스를 생성하는일을 막을 수 있지만, 정상 `Period` 인스턴스에서 시작된 바이트 스트림 끝에 private Date 필드로의
참조를 추가하면 가변 `Period` 인스턴스를 만들어낼 수 있습니다.

이제 이 참조로 얻은 Date 인스턴스들을 수정할 수 있으니, `Period` 인스턴스는 더는 불변이 아니게 됩니다.

```java
import java.io.*;
import java.time.Period;
import java.util.Date;

public class MutablePeriod {
    public final Period period;
    public final Date start;
    public final Date end;

    public MutablePeriod() {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream out = new ObjectOutputStream();

            out.writeObject(new Date(), new Date());

            /**
             * 악의적인 '이전 객체 참조', 즉 내부 Date 필드로의 참조를 추가
             */
            byte[] ref = {0x71, 0, 0x7e, 0, 5};
            bos.write(ref);
            ref[4] = 4;
            bos.write(ref);

            // Period 역직렬화 후 Date 참조를 훔침
            ObjectInputStream in = new ObjectInputStream(
                    new ByteArrayInputStream(bos.toByteArray())
            );
            period = (Period) in.readObject();
            start = (Date) in.readObject();
            end = (Date) in.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new AssertionError(e);
        }
    }

    public static void main(String[] args) {
        MutablePeriod mp = new MutablePeriod();
        Period p = mp.period;
        Date pEnd = mp.end;
        
        pEnd.setYear(78);
        pEnd.setYear(69);
    }
}
```

이 예에서 Period 인스턴스는 불변식을 유지한 채 생성됐지만, 의도적으로 내부의 값ㅇ르 수정할 수 있었습니다.
이처럼 변경할 수 있는 Period 인스턴스를 획득한 공격자는 이 인스턴스가 불변이라고 가정하는 캘르에 넘겨 엄청난 보안 문제를 일으킬 수 있습니다.

이 문제의 근원은 Period의 `readObject` 메서드가 방어적 복사를 충분히 하지 않은데 있습니다.
객체를 역직렬화할 때는 클라이언트카 소유해서는 안되는 객체 참조를 갖는 필드를 모두 반드시 방어적으로 복사해야합니다.

따라서 `readObject` 에서는 불변 클래스 안의 모든 private 가변 요소를 방어적으로 복사해야합니다. 다음의 `readObject` 메서드라면 
Period 불변식과 불변 성질을 지켜내기에 충분합니다.

```java
private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException{
    s.defaultReadObject();    
    
    start = new Date(start.getTime());
    end = new Date(end.getTime());

    if (this.start.compareTo(this.end) > 0) {
        throw new IllegalArgumentException(
                start + "가 " + end + "보다 높다."
        );
    }
}
```

이 메서드를 사용하려면 `start` 와 `end` 필드에서 `final`한정자를 제거해아합니다. 
이 한정자를 제거하고도 불변식을 지킬 수 있습니다.

기본 `readObject` 메서드를 써도 좋을지를 판단하는 방법은
`transient` 필드를 제외한 모든 필드의 값을 매개변수로 받아 유효성 검사없이 필드에 대입하는 `public`생성자를 추가해도 괜찮아야 합니다.

그렇지 않으면 `readObject`를 커스텀해야합니다.