# try-finally보다는 try-with-resources를 사용하는 것을 권장
- 자바에선 InputStream, OutputStream, java.sql.Connection 등과 같은 close 메서드를 호출해 직접 닫아줘야 하는 자원이 많다.
- 이는 예측할 수 없는 성능 문제로 이어지기도 한다.
- 전통적으로 자원이 제대로 닫힘을 보장하는 수단으로 try-finally가 있다. 예외가 발생하거나 메서드에서 반환되는 경우를 포함해서 말이다.

```java
// try-finally - 더 이상 자원을 회수하는 최선의 방책이 아님
static String firstLineOfFile(String path) throws IOException {
	BufferedReader br = new BufferedReader(new FileReader(path));
	try {
		return br.readLine();
	} finally {
		br.close();
	}
}
```
- 위 코드도 나쁘지 않지만, 자원을 하나 더 사용한다면?
```java
자원이 둘 이상이면 try-finally 방식은 너무 지저분함
    static void copy(String src, String dst) throws IOException {
        InputStream in = new FileInputStream(src);
        try {
            OutputStream out = new FileOutputStream(dst);
            try {
                byte[] buf = new byte[BUFFER_SIZE];
                int n;
                while ((n = in.read(buf)) >= 0)
                    out.write(buf, 0, n);
            } finally {
                out.close();
            }
        } finally {
            in.close();
        }
    }
    
 ```
- try-finally 문을 제대로 사용한 앞의 두 코드 에제에 조차 미묘한 결점이 있다.
- 예외는 try-블록과 finally 블록 모두에서 발생할 수 있는데, 예컨대 기기에 물리적인 문제가 생긴다면 firstLineOfFile 메서드 안의 readLine 메서드가 예외를 던지고, 같은 이유로 close 메서드도 실패할 것이다.
- 이런 상황이라면 두 번쨰 예외가 첫 번째 예외를 완전히 집어삼켜 버리고 그러면 스택 추적 내역에 첫 번째 예외에 관한 정보는 남지 않게 되어 디버깅을 몹시 힘들게 할 것이다.
- 물론 두 번째 예외 대신 첫 번째 예외를 기록하도록 코드를 수정할 수 있지만, 코드가 너무 지져분해져 실제로 그렇게까지 하는 경우는 거의 없다.
## try-with-resources
- 위의 이러한 문제들은 try-with-resources 덕에 모두 해결되었다. 이 구조를 사용하려면 해당 자원이 AutoCloseable 인터페이스를 구현해야 한다.
- 단순히 void 를 반환하는 close 메서드 하나만 덩그러니 정의된 인터페이스이다.
- 자바 라이브러리와 서드파티 라이브러리들의 수많은 클래스와 인터페이스가 이미 AutoCloseable을 구현하거나 확장해뒀다.
- 만약 반드시 닫아야 하는 자원을 뜻하는 클래스를 작성한다면 AutoCloseable을 반드시 구현해야 한다.
### try-with-resources를 적용한 코드이다.
```java
// 복수의 자원을 처리하는 try-with-resources - 짧고 매혹적
static void copy(String src, String dst) throws IOException {
	try (InputStream   in = new FileInputStream(src);
			OutputStream out = new FileOutputStream(dst)) {
		byte[] buf = new byte[BUFFER_SIZE];
		int n;
		while ((n = in.read(buf)) >= 0)
			out.write(buf, 0, n);
	}
}
```
- try-with-resources 버전이 짧고 읽기 수월할 뿐 아니라, 문제를 진단하기도 훨씬 좋다.
- 숨겨진 예외들도 그냥 버려지지 않고, 스택 추적 내역에 ‘숨겨졌다’(suppressed)는 꼬리표를 달고 출력된다.
- 자바7에서 Throwalbe에 추가된 getSuppressed 메서드를 이용하면 프로그램 코드에서 가져올 수 도 있다.
- 그리고 보통의 try-finally처럼 try-finally-resources에서도 catch 절을 쓸 수 있다.
- catch절 덕분에 try문을 더 중첩하지 않고도 다수의 예외를 처리할 수 있다.
### 아래 코드는 firstLineOfFile 메서드를 살짝 수정하여 파일을 열거나 데이터를 읽지 못했을때 예외 대신 기본값을 반환하도록 변경한 코드이다.
```java
//try-with-resources를 catch 절과 함께 쓰는 모습
static String firstLineOfFile(String path, String defaultVal) {
	try (BufferedReader br = new BufferedReader(
			new FileReader(path))) {
		return br.readLine();
	} catch (IOException e) {
		return defaultVal;
	}
}
```
