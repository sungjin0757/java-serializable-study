## Item87. 커스텀 직렬화 형태를 고려해보라
***

**고민해보고, 괜찮다고 판단될 때만 기본 직렬화 형태를 사용하라**

기본 직렬화 형태는 유연성, 성능, 정확성 측면에서 신중히 고민한 후 합당할 때만 사용해야 합니다.

어떤 객체의 기본 직렬화 형태는 그 객체를 루트로 하는 객체 그래프의 물리적 모습을
나름 효율적으로 인코딩 합니다.

객체가 포함한 데이터들과 그 객체에서부터 시작해 접근할 수 있는 모든 객체를 담아내며, 심지어
이 객체들이 연결된 위상까지 기술합니다. 

이상적인 직렬화 형태라면 물리적인 모습과 독립된 논리적인 모습만을 표현해야합니다.

**객체의 물리적 표현과 논리적 내용이 같다면 기본 직렬화 형태라도 무방합니다.**

```java
import java.io.Serializable;

public class Name implements Serializable{
    private final String lastName;
    private final String firstName;
    private final String middleName;
}
```

성명은 논리적으로 이름, 성, 중간이름이라는 3개의 문자열로 구성되며, 앞 코드의 인스턴스 필드들은
이 논리적 구성요소를 정확히 반영했습니다.

**기본 직렬화 형태가 적합하다고 결정했더라고 불변식 보장과 보안을 위해 `readObject` 메서드를 제공해야할 때 가 많흣비낟.**

앞의 `Name` 클래스의 경우는 `readObject` 메서드가 `lastName`과 `firstName`필드가 `null`이 아님을 보장해야 합니다.

다음 클래스는 직렬화 형태에 적합하지 않은 예로, 문자열 리스트를 표현하고 있습니다.

```java
import java.io.Serializable;
import java.util.Map;

public final class StringList implements Serializable {
    private int size = 0;
    private Entry head = null;
    
    private static class Entry implements Serializable{
        String data;
        Entry next;
        Entry previous;
    }
}
```

논리적으로 이 클래스는 일련의 문자열을 표현하며, 물리적으로는 문자열들을 이중연결 리스트로 연결 했습니다.

이 클래스에 기본 직렬화 형태를 사용하면 각 노드의 양방향 연결 정보를 포함해 모든 엔트리를 철두철미하게 기록해야합니다.

**객체의 물리적 표현과 논리적 표현의 차이가 클 때 기본 직렬화 형태를 사용하면 크게 네가지 측면에서 문제가 생깁니다.**

1. 공개 API가 현재의 내부 표현 방식에 영구히 묶입니다. 앞의 예에서 private 클래스인 StringList.Entry가 공개 API가 되어 버립니다.
다음 릴리스에서 내부 표현 방식을 바꾸더라도 StringList 클래스는 여전히 연결리스트로 표현된 입력도 처리할 수 있어야합니다.
2. 너무 많은 공간을 차지할 수 있습니다. 앞 예의 직렬화 형태는 연결 리스트의 모든 엔트리와 연결 정보 까지 기록했지만, 엔트리와 연결 정보는 내부 구현에 
해당하니 직렬화 형태에 포함할 가치가 없습니다. 
3. 시간이 너무 많이 걸릴 수 있습니다. 직렬화 로직은 객체 그래프의 위상에 관한 정보가 없으니 그래프를 직접 순회해볼 수 밖에 없습니다.
4. 스택 오버플로를 일으킬 수 있습니다.
기본 직렬화 과정은 객체 그래프를 재귀 순회하는데, 이 작업은 중간 정도 크기의 객체 그래프에서도 스택오버플로를 일으킬 수 있습니다.

StringList를 위한 합리적인 직렬화 형태는 무엇일까요? StringList의 물리적인 상세 표현은 배제한 채 논리적인 구성만 담는 것입니다. 
`writeObject`와 `readObject`가 직렬화 형태를 처리합니다. `transient` 한정자는 해당 인스턴스가 기본 직렬화 형태에 포함되지 않는다는 뜻입니다.

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public final class StringList implements Serializable {
    private transient int size = 0;
    private transient Entry head = null;

    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }

    public final void add(String s) {

    }

    private void writeObject(ObjectOutputStream s)
            throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);

        for (Entry e = head; e != null; e = e.next) {
            s.writeObject(e.data);
        }
    }

    private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException{
        s.defaultReadObject();
        int numElements = s.readInt();
        
        for(int i = 0; i < numElements; i ++){
            add((String)s.readObject());
        }
    }
}
```

신버전 인스턴스를 직렬화한 후 구버전으로 역직렬화하면 새로 추가된 필드들은 무시될 것입니다.
구버전 `readObject` 메서드에서 `defaultReadObject`를 호출하지 않는다면 역직렬화할 때
`StreamCorruptedException`이 발생할 것입니다.

기본 직렬화를 수용하든 하지 않든 `defaultWriteObject`메서드를 호출하면 `transient`로 선언하지 않은 모든 인스턴스 필드가
직렬화 됩니다. 따라서 `transient` 로 선언해도 되는 인스턴스 필드에는 모두 `transient`한정자를 붙여야합니다.

**해당 객체의 논리적 상태와 무관한 필드라고 확신할 때만 `transient` 한정자를 생략해야합니다**

기본 직렬화를 사용한다면 `transient` 필드들은 역직렬화될 때 기본값으로 초기화 됩니다.

기본 직렬화 사용 여부와 상관없이 객체의 전체 상태를 읽는 메서드에 적용해야 하는 동기화 메커니즘을 직렬화에도 적용해야 합니다.

**어떤 직렬화 형태를 택하는 직렬화 가능 클래스 모두에 직렬 버전 UID를 명시적으로 부여합시다.**

**구버전으로 직렬화된 인스턴스들과의 호환성을 끊으려는 경우르 제외하고는 직렬 버전 UID를 절대 수정하지 맙시다.**