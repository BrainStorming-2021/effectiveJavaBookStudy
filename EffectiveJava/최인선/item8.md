자바는 두 가지 객체 소멸자를 제공한다.

finalizer (finalize() 메소드)
cleaner (Cleaner 객체)


finalizer와 cleaner를 지양해야 하는 이유
1. 언제 자원을 회수할지 아무도 모른다.
finalizer와 cleaner는 즉시 수행된다는 보장이 없다. 왜냐하면 전적으로 GC알고리즘에 달렸으며, 이는 GC 구현마다 천차만별이다.

원인을 알 수 없는 OutOfMemoryError가 발생한다면, finalizer와 cleaner를 의심해보자.

또한, 자원 회수 수행시점 뿐만 아니라 수행 여부조차 보장하지 않는다. 이는 굉장히 critical 한데, 회수해야 할 자원에 대한 종료 작업을 전혀 수행하지 못한 채 
프로그램이 중단될 수도 있다.

따라서 데이터베이스 같은 공유 자원의 영구 락 해제와 같이 상태를 수정하는 작업에서는 절대 finalizer 나 cleaner에 의존하지 말고 직접 자원을 close해줘야 한다.

2. 심각한 성능 문제
finalizer가 가비지 컬렉터의 효율을 떨어뜨린다. AutoCloseable 객체를 생성하고 GC가 수거하는 시간보다 약 50배가 느리다.

3. 보안 문제
생성자나 직렬화 과정에서 예외가 발생하면, 이 생성되다 만 객체에서 악의적인 하위 클래스의 finalizer가 수행될 수 있게 된다.

finalize 메소드는 java.lang.Object.class 에 있는 메소드로, @Override 해서 사용한다. 악의적인 하위클래스에 해당 메소드를 심으면, 무엇이든지 할 수 있게 된다.
```java
@Override
public void finalize() {
    try {
        reader.close();
        System.out.println("Closed BufferedReader in the finalizer");
    } catch (IOException e) {
      //
    }
}
```
Tip : finalizer 공격으로부터 방어하려면 아무일도 하지 않는 finalize 메서드를 만들고 final로 선언하자

언제 finalizer와 cleaner를 사용할까?
1. 안전망 역할 (서브 역할)
자원의 소유자가 close 메서드를 호출하지 않는 것에 대비한 안전망 역할이다. FileInputStream, FileOutputStream, ThreadPoolExecutor 는 안전망 역할의 finalizer를 제공한다.

2. 네이티브 피어와 연결된 
네이티브 피어는 자바 객체가 아니니 GC가 그 존재를 알지 못한다. 따라서 cleaner나 finalizer를 사용해서 자원을 회수하게 한다.

※ 네이티브 객체란 Java로 작성된 것이 아닌 C나 어셈블러와 같이 다른 언어로 작성된 객체를 뜻한다.

3. Cleaner 
Cleaner 객체는 JAVA 9에 추가된 기능이므로, JAVA 9이상 버전을 사용해야 한다.
```java
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // (1) Runnable 인터페이스
    private static class State implements Runnable {
        @Override public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // 방의 상태. cleanable과 공유한다.
    private final State state;

    // cleanable 객체. 수거 대상이 되면 방을 청소한다.
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
        // (2) Cleaner에 자원회수로직 runnable 인터페이스 등록
    }

    @Override public void close() { // (3) AutoCloseable 인터페이스 구현메소드
        cleanable.clean();
    }
}
```
1.  Runnable 인터페이스 : Clenaer가 clean을 할때 run()이 호출된다. 절대 Room을 참조해서는 안 된다!
2. Cleanable register(Object obj, Runnable action) : 자원회수로직 runnable 인터페이스를 등록한다.
3. close : AutoCloseable 인터페이스의 구현 메소드로 try-with-resources 문으로 관리되는 객체일 때 close() 메서드가 자동으로 호출된다. (Cleaner를 믿지 않고,, try-with-resources 블록으로 감싸서 close 메소드가 수행되게 하자)
자원회수로직인 State 인스턴스는 절대로 Room 인스턴스를 참조해서는 안된다. 순환참조가 생길경우, GC가 Room인스턴스를 회수해갈 기회가 오지 않는다. State 가 정적 중첩 클래스인 이유가 여기 있다.
정적이 아닌 중첩 클래스는 자동으로 바깥 객체의 참조를 갖게 되기 떄문이다.
