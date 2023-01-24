# 메모리 누수는 언제 일어나는가?
1. 스택 pop()
- 스택이 커졌다가 줄어들었을 때 스택에서 꺼내진 객체들을 가비지 컬렉터가 회수하지 않는다.
- 이 객체들을 더 이상 사용하지 않더라도...
- 해법: 해당 참조를 다 썻을 때 null 처리를 한다.
'''java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    /**
     * 원소를 위한 공간을 적어도 하나 이상 확보한다.
     * 배열 크기를 늘려야 할 때마다 대략 두 배씩 늘린다.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
  }
  '''
  
- Stack에서 pop을 하게 되면, element 상의 포인터는 한칸 내려온다.
- 하지만 그 이전에 element에서 참조하고 있는 객체는 그대로 있다.

'''java
public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }
'''

2. 캐시
- 객체 참조를 캐시에 넣고 나서, 객체를 다쓰고 나서도 참조를 그대로 둘때.
- 데이터는 바뀌었는데 캐시에는 그대로 남아 있는 경우가 있다.
- WeakHashMap을 사용 -> key를 사용하지 않으면 value를 삭제한다.

3. 리스너 혹은 콜백
- 콜백을 약한 참조로 저장하면 GC가 즉시 수거함.
