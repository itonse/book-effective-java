# [item6] 불필요한 객체 생성을 피하라 <br/><br/>

똑같은 기능의 객체를 매번 생성하기보다는 **객체 하나를 재사용**하는 편이 나을 때가 많다.

다음은 String에서 매번 인스턴스를 생성하는 코드와 재사용이 가능한 코드를 선언하는 예시이다.

```java
String s = new String("java");    // 매번 새로운 String 인스턴스를 생성
String s = "java";                // 이와 똑같은 문자열 리터럴을 사용할 경우, 재사용 보장
```

두 번째 줄의 코드는 **하나의 String 인스턴스를 사용**하게 되므로, 같은 JVM 내에서 이와 똑같은 문자열 리터럴을 사용하는 모든 코드가 **같은 객체를 재사용**함이 보장된다. <br/><br/>


## 정적 팩터리 메서드를 사용해 불필요한 객체 생성을 피하기

Item1에서 설명한 것처럼, 생성자 대신 정적 팩터리 메서드를 제공하는 불변 클래스에서는 정적 팩터리 메서드를 사용해 불필요한 객체 생성을 피할 수 있다.

```java
Boolean(String)  // 생성자: 매번 새로운 객체 생성
    
Boolean.valueOf(String)  // 정적 팩터리 메서드: 불필요한 객체 생성 방지
```
<br/>

## 생성 비용이 아주 비싼 객체

생성 비용이 매우 비싼 객체도 존재한다. 이런 비싼 객체가 반복해서 필요하다면 캐싱하여 재사용하는 것이 좋다.

하지만 사용하는 객체가 비싼 객체인지 매번 명확히 알 수 없는데, 자바에서는 다음과 같이 주의해서 사용해야 할 내장 메서드들이 있다.

다음은 정규표현식을 사용한 코드의 예시이다.

```java
static boolean isUsernameValid(String s) {
    return s.matches("^[a-zA-Z0-9]{3,15}$");
}
```
<br/>
String 의 matches 함수의 내부 코드는 다음과 같다.

```java
import java.util.regex.Pattern;

public boolean matches(String expr) {
    return Pattern.matches(expr, this);
}
```
<br/>
또 Pattern 의 matches 함수의 내부는 다음과 같다.

```java
import java.util.regex.Pattern;

public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);    // 매번 새로운 객체를 생성해서 패턴을 컴파일
    Matcher m = p.matcher(input);    // 매번 객체 생성
    return m.matches();
}
```
<br/>Pattern.matches 메서드 내에 `Pattern p` 객체는 한 번만 쓰고 버려져서 곧바로 GC 대상이 된다.

그래서 성능이 중요한 상황에서 위의 코드를 반복해 사용하기엔 적합하지 않다.
<br/><br/>

성능을 개선하려면 필요한 정규표현식을 표현하는 (불변인) `Pattern 인스턴스` 를 클래스 초기화 과정에서 직접
생성해 캐싱해 두고, 나중에 `isUsernameValid` 메서드가 호출될 때마다 이 인스턴스를 재사용하게끔 하면 된다. <br/><br/>

### 최적화된 Username Valid Check Util

- **개선 전**: Pattern 인스턴스가 매번 생성
```java
static boolean isUsernameValid(String s) {
    return s.matches("^[a-zA-Z0-9]{3,15}$");
}
``` 

- **개선 후**: Pattern 인스턴스가 한 번만 생성

```java
public class UsernameUtil {    // 유틸 클래스
    private static final Pattern USERNAME = Pattern.compile("[a-zA-Z0-9]{3,15}$");
    
    static boolean isUsernameValid(String s) {
        return USERNAME.matcher(s).matches();
    }
}
```
<br/>이렇게 개선하면 `isUsernameValid` 가 빈번히 호출되는 상황에서 성능을 상당히 끌어올릴 수 있다.

<br/><br/>

## 불필요한 객체를 만들어내는 다른 예시: 오토박싱(auto boxing)

자바에서는 Boxing type 대신 Primitive Type(기본 타입) 사용을 권장한다.
<br/><br/>

```java
public static long sum() {
    long sum = 0L;    // Primitive type
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }
    return sum;
}
// 기본 타입 수행 시간: 684185667ns

public static long sum() {
	Long sum = 0L;    // Boxing type
	for (long i = 0; i < Integer.MAX_VALUE; i++) {
		sum += i;   
	}
	return sum;
}
// 박싱 타입 수행 시간: 2482506083ns (약 3.6배 증가)
```
<br/>

위의 코드에서 두 함수의 수행 시간을 비교해 보니, 박싱 타입을 사용하면 수행 시간이 **약 3.6배** 더 증가한 결과가 나왔다.

박싱 타입을 사용했을 때 성능이 더 나쁜 이유는 다음과 같다.

1. `sum += i`  연산 시 i를 Long 타입으로 변환하는 오토박싱 발생, 다시 sum에 더하는 과정에서 언박싱 발생. 이로 인한 성능 저하.
2. 덧셈 연산이 수행될 때마다 매번 새로운 `Long 객체` 생성 (객체 생성 비용, GC 비용 증가).
3. 기본 타입은 **스택 메모리**를 사용하여 메모리 할당과 해제가 빠르고 효율적. 반면, 래퍼 클래스는 **힙 메모리**를 사용하며, 힙 메모리는 GC의 대상이 되어 메모리 관리와 성능 저하가 발생함.

따라서 `null` 값 사용 등의 정말 필요한 경우가 아니라면, Boxing type은 사용을 자제하는 것이 좋다.
