## Spring boot - Caching 예제
Spring 가이드 Caching Data with Spring을 공부하며 만든 프로젝트입니다.

### Caching
input에 따라 정해진 output이 있는 작업이 있다고 하자. 적당한 예시로 피보나치 함수를 생각해보자.
```java
int fibo(int n) {
    if (n == 1)
        return 1;

    return n * fibo(n - 1);
}
```
input으로 넣어준 `n`이 같다면 output은 항상 `n!`으로 일정하게 나온다. 만약 이 함수를 input `n`에 대하여 100번, 아니 10000번 호출한다고 하자.
나올 output은 `n!`이지만 모든 호출마다 재귀적으로 함수를 수행하고 output을 반환한다. 매우 불필요한 반복작업이 아닐 수 없다.

이럴때 Caching을 이용한다면 어떻게 될까? 처음 함수가 호출되면 input `n`이라는 값에 대해 output `n!`을 계산하고 캐시에 저장한다. 두번째 함수가 호출되고 input으로 `n`이라는 값이 들어오면 캐시를 확인한다.
input `n`에 대해 이전에 계산한 결과값 `n!`이 캐시에 존재함을 확인하고는 계산없이 `n!`을 반환한다.

캐싱을 적용하는 작업의 cost가 클수록 캐싱은 더 빛을 받는다. 이런 캐싱을 스프링에서는 어노테이션 기반으로 쉽게 이용할 수 있도록 도와주며 캐싱을 지원하는 다른 라이브러리도 손쉽게 사용가능하다.
이번 예제 프로젝트에서는 라이브러리가 아닌 스프링에서 기본으로 지원하는 `ConcurrentHashMap`으로 만들어진 심플캐시를 이용한다. 

* 참고자료
   * [예제 프로젝트](https://spring.io/guides/gs/caching/)
   * https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache
   * https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching
   
***

### <h1>프로젝트 구조</h1>
먼저, 임의로 3초가 걸리는 작업을 만들어 반복적으로 그 작업을 수행하게 한다. 캐싱이 적용되지 않은 상태이기 때문에 결과는 최소 3초의 주기를 갖고 값을 반환한다.
그 다음 캐싱을 적용하여 같은 작업을 반복적으로 수행한다. `log`에 찍힌 시간을 보며 캐싱을 적용하지 않음과 적용됨을 느낄 수 있을 것이다.

***

### 준비
3초의 수행시간을 갖는 작업을 만들어준다. `getByIsbn(String isbn)`함수를 반복적으로 호출할 것이며 강제적으로 3초의 Delay를 넣어주었다.
`SimpleBookRepository`클래스가 상속하고있는 `BookRepository`에 대한 소스는 아래에 첨부할 것이다.
```java
@Component
public class SimpleBookRepository implements BookRepository {

  @Override
  public Book getByIsbn(String isbn) {
    simulateSlowService();
    return new Book(isbn, "Some book");
  }

  // Don't do this at home
  private void simulateSlowService() {
    try {
      long time = 3000L;
      Thread.sleep(time);
    } catch (InterruptedException e) {
      throw new IllegalStateException(e);
    }
  }

}
```
`Book`과 `BookRepository`
```java
/** Book Class */
public class Book {

  private String isbn;
  private String title;

  public Book(String isbn, String title) {
    this.isbn = isbn;
    this.title = title;
  }

  public String getIsbn() {
    return isbn;
  }

  public void setIsbn(String isbn) {
    this.isbn = isbn;
  }

  public String getTitle() {
    return title;
  }

  public void setTitle(String title) {
    this.title = title;
  }

  @Override
  public String toString() {
    return "Book{" + "isbn='" + isbn + '\'' + ", title='" + title + '\'' + '}';
  }

}

/** BookRepository */
public interface BookRepository {

  Book getByIsbn(String isbn);

}
```

`SimpleBookRepository`의 함수를 호출해줄 코드도 작성한다.
```java
@Component
public class AppRunner implements CommandLineRunner {

  private static final Logger logger = LoggerFactory.getLogger(AppRunner.class);

  private final BookRepository bookRepository;

  public AppRunner(BookRepository bookRepository) {
    this.bookRepository = bookRepository;
  }

  @Override
  public void run(String... args) throws Exception {
    logger.info(".... Fetching books");
    logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
    logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
    logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
  }

}
```

작성한 코드를 실행해보자, 로그창은 아래와 같다.
```
2020-02-13 23:18:36.150  INFO 22828 --- [           main] com.example.caching.AppRunner            : .... Fetching books
2020-02-13 23:18:39.152  INFO 22828 --- [           main] com.example.caching.AppRunner            : isbn-1234 -->Book{isbn='isbn-1234', title='Some book'}
2020-02-13 23:18:42.153  INFO 22828 --- [           main] com.example.caching.AppRunner            : isbn-1234 -->Book{isbn='isbn-1234', title='Some book'}
2020-02-13 23:18:45.154  INFO 22828 --- [           main] com.example.caching.AppRunner            : isbn-1234 -->Book{isbn='isbn-1234', title='Some book'}

Process finished with exit code 0
```
Fetching이 되고 `bookRepository.getByIsbn("isbn-1234")`의 결과가 Delay를 준 만큼 `39`, `42`, `45`초에 찍히는것을 확인할 수 있다.

***

### Caching
자 이제 캐싱을 적용해보자. 먼저 SpringApplication에 캐싱의 사용을 알려주는 `@EnableCaching`어노테이션을 붙여준다.

```java
@SpringBootApplication
@EnableCaching /* 캐싱을 사용하겠다. */
public class CachingApplication {

	public static void main(String[] args) {
		SpringApplication.run(CachingApplication.class, args);
	}

}
```

그리고 캐싱을 적용할 함수를 지정해주면 된다. 이전에 만든 `SimpleBookRepositoy`클래스의 `getByIsbn(String isbn)`함수위에 `@Cacheable`을 붙여준다.
이때 어노테이션안에 들어간 `books`라는 값은 캐시 아이디이다.
```java
@Component
public class SimpleBookRepository implements BookRepository {

	@Override
	@Cacheable("books")
	public Book getByIsbn(String isbn) {
		simulateSlowService();
		return new Book(isbn, "Some book");
	}

	// Don't do this at home
	private void simulateSlowService() {
		try {
			long time = 3000L;
			Thread.sleep(time);
		} catch (InterruptedException e) {
			throw new IllegalStateException(e);
		}
	}

}
```
`@Cacheable` 이외에도 `@CachePut`이나 `@CacheEvict`과 같이 캐싱과 관련된 어노테이션들이 있으므로 참고하자.
* Docs
    * [@Cacheable](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html)
    * [@CachePut](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CachePut.html)
    * [@CacheEvict](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CacheEvict.html)
    

이제 캐싱을 적용한 코드를 실행해보자, 로그창은 아래와 같다.
```
2020-02-13 23:30:46.475  INFO 13632 --- [           main] com.example.caching.AppRunner            : .... Fetching books
2020-02-13 23:30:49.488  INFO 13632 --- [           main] com.example.caching.AppRunner            : isbn-1234 -->Book{isbn='isbn-1234', title='Some book'}
2020-02-13 23:30:49.489  INFO 13632 --- [           main] com.example.caching.AppRunner            : isbn-1234 -->Book{isbn='isbn-1234', title='Some book'}
2020-02-13 23:30:49.489  INFO 13632 --- [           main] com.example.caching.AppRunner            : isbn-1234 -->Book{isbn='isbn-1234', title='Some book'}

Process finished with exit code 0
```
Fetch후 3초의 시간 후, 첫 `getByIsbn(String isbn)`의 로그가 찍혔다. 하지만 두번째, 세번째 로그는 캐싱이되어있기 때문에 3초의 Delay없이 바로 로그가 찍힌것을 확인할 수 있다.

***

## 정리

현재 진행한 프로젝트에서는 스프링에서 제공하는 simpleCache를 적용해보았고 이는 `ConcurrentHashMap`을 이용하여 <key, value>로 저장되어 쓰여진다.
하지만 실무에서는 `Encache`나 `Redis`를 많이 사용한다고 한다.
`Redis`같은 경우 `properties`설정과 `RedisCacheManagerBuilderCustomizer` Bean 설정만 마치면 쉽게 사용할 수 있다. 다양한 캐시라이브러리에 대해서는 [이 링크](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-caching-provider)를 따라가 확인하면된다.
주로 캐시는 반복적으로 동일한 결과를 리턴하는 작업 또는, 자원사용이 많고 시간이 오래 걸리는 작업에서 사용하면 좋은 효과를 볼 수 있다.
하지만 데이터가 자주 변경되거나 시간이 짧은 작업에 적용하면 오히려 성능을 저하시킬수 있다는 것에 유의하자.
