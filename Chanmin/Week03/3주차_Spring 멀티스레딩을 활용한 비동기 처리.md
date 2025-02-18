지난번에 스프링의 멀티스레딩 활용 방식을 알아봤다. 

여러 가지 방식 중에서 멀티스레딩을 활용한 비동기 처리 방식을 내 프로젝트에 적용시켜보고자 한다. 동기 처리를 했을 때와 비동기 처리를 했을 때의 성능 차이를 분석하여 비동기 처리 방식이 어떻게 활용될 수 있는지 살펴보자.

## 스프링의 비동기 처리 적용 방법

우선 스프링에서 비동기 처리를 적용시키기 위한 간단한 방법을 알아보자.

```
@Configuration
@EnableAsync
public class AsyncConfig {}
```

스프링에서 비동기 처리를 적용하려면 먼저 비동기 설정 클래스를 생성하고, @EnableAsync 어노테이션을 사용해 비동기 기능을 활성화해야 한다.

그 후, 비동기 처리를 적용할 메서드에 @Async 어노테이션을 추가하면 해당 메서드가 별도의 스레드에서 실행된다.

## 비동기 처리를 활용할 경우 성능 확인

비동기 처리를 내 프로젝트에 적용시켜 보자.

현재 개발 중인 경제 학습 애플리케이션에서는 뉴스 데이터를 저장하는 로직이 다음과 같이 동작한다.

1\. **매일 자정(00:00)에 뉴스 크롤링 실행**

-   Jsoup을 활용하여 네이버 뉴스의 각 카테고리를 순차적으로 크롤링한다.
-   예를 들어, 네이버 뉴스에 금융, 주식, 부동산과 같은 카테고리가 존재할 경우, 각 카테고리를 차례대로 크롤링하여 데이터를 수집한다.

2. **데이터 가공 및 저장**

-   크롤링된 뉴스 데이터를 뉴스 엔터티(Entity)로 변환한 후, JdbcTemplate을 사용하여 데이터베이스에 저장한다.

이 과정은 현재 카테고리 순으로 순차적으로 실행되기 때문에, 뉴스 데이터의 양이 많아질수록 크롤링과 저장 과정에서 시간이 오래 걸릴 수 있다. 이를 개선하기 위해 **비동기 처리**를 적용하여 여러 카테고리를 동시에 크롤링하고 저장할 수 있게 하여 성능이 향상되도록 해보자.

### 동기 처리 성능 확인

먼저, 동기 방식으로 뉴스 데이터를 크롤링할 때 소요되는 시간을 측정해 보자.

이를 위해 테스트 코드를 작성하여 크롤링과 데이터 저장 과정의 실행 시간을 확인한다.

```
@Test
public void testFetchAndSaveAllNews_Performance() {
    StopWatch stopWatch = new StopWatch();

    // 성능 측정 시작
    stopWatch.start();

    // 크롤링 실행 (비동기)
    newsService.fetchAndSaveAllNews();

    // 성능 측정 종료
    stopWatch.stop();

    // 실행 시간 출력
    System.out.println("🚀 fetchAndSaveAllNews() 실행 시간: " + stopWatch.getTotalTimeMillis() + " ms");
}
```

아래는 동기 방식으로 뉴스 데이터를 크롤링하는 코드이다.

예를 들어, 금융 카테고리의 뉴스를 크롤링하는 동안에는 부동산 카테고리의 뉴스를 크롤링할 수 없다. 즉, 크롤링 과정이 순차적으로 진행되며, 각 카테고리가 완료된 후에야 다음 카테고리를 처리할 수 있다.

```
@Transactional
public void fetchAndSaveAllNews() {
    for (NewsCrawler crawler : crawlers) {
        List<NewsDTO> newsList = crawler.crawl();

        List<News> filteredNews = newsCrawlerService.filterOutDuplicateUrls(newsList);

        newsCrawlerService.saveNewsBatch(filteredNews);
    }
}
```

[##_Image|kage@bHHcGf/btsMlnWcWht/W6xK9KRqRcU1LpMpGhw4xK/img.png|CDM|1.3|{"originWidth":2086,"originHeight":360,"style":"alignCenter"}_##]

약 200개의 뉴스 데이터를 저장하는 데 1,296ms(1.2초)가 소요되었다.

뉴스 데이터의 양이 증가할수록 메서드의 실행 시간도 비례하여 증가할 것으로 예상된다. 따라서, 보다 효율적인 처리를 위해 **비동기 방식**을 도입해 보자.

### 비동기 처리 성능 확인

비동기 방식으로 크롤링을 수행하면 각 카테고리의 크롤러가 **순차적으로 실행되는 것이 아니라, 동시에 실행**될 수 있다. 이를 통해 전체 실행 시간을 단축해 보자.

아래 코드에서는 @Async 어노테이션을 사용하여 크롤링을 비동기적으로 실행하고, 모든 크롤링 작업이 완료될 때까지 대기하는 방식으로 처리한다.

```
public void fetchAndSaveAllNewsAsync() {
    List<CompletableFuture<Void>> futures =
            crawlers.stream().map(newsCrawlerService::crawl).toList();

    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join(); // 모든 크롤링이 끝날 때까지 대기
}
```

```
@Async
@Transactional
public CompletableFuture<Void> crawl(NewsCrawler crawler) {

    List<NewsDTO> newsList = crawler.crawl();
    List<News> filteredNews = filterOutDuplicateUrls(newsList);
    saveNewsBatch(filteredNews);

    return CompletableFuture.completedFuture(null);
}
```

기존 **동기 방식**에서는 하나의 카테고리가 완료될 때까지 다음 카테고리를 크롤링할 수 없었다. **비동기 방식**을 적용하여 각 크롤러가 응답을 반환할 때까지 기다리지 않고, 동시에 여러 크롤링 작업을 수행할 수 있게 되었다. 

CompletableFuture.allOf()를 사용하여 **모든 크롤링 작업이 끝날 때까지 대기**하도록 처리한다.

[##_Image|kage@ZTTGK/btsMkXcyU6g/8i1zeyLbgloHi0DauK97kK/img.png|CDM|1.3|{"originWidth":2096,"originHeight":360,"style":"alignCenter"}_##]

그 결과, 약 200개의 뉴스 데이터를 저장하는 데 748ms(0.7초)가 소요되었으며, 기존보다 약 **0.5초가량 성능이 향상**되었다. 

뉴스 데이터가 증가하더라도, 각 크롤러를 **병렬로 실행**함으로써 처리 시간을 최적화할 수 있을 것으로 예상된다. 

## ThreadPool 활용하기

비동기 처리는 별도의 **스레드**를 생성하여 메서드를 실행하는 방식으로 동작한다.

스프링에서는 기본적으로 SimpleAsyncTaskExecutor를 사용하여 **새로운 스레드를 생성**하고, 이를 통해 비동기 처리를 수행한다.

아래 코드의 doExecute 메서드를 확인하면 스레드를 생성하는 것을 확인할 수 있다.

```
public class SimpleAsyncTaskExecutor extends CustomizableThreadCreator implements AsyncListenableTaskExecutor, Serializable, AutoCloseable {
	
    ...

	protected void doExecute(Runnable task) {
    	this.newThread(task).start();
	}
    
    ...
    
}
```

하지만, **매번 새로운 스레드를 생성하면 시스템 자원이 낭비되고, 스레드 생성에 따른 오버헤드가 증가**하는 문제가 발생할 수 있다.

이를 방지하기 위해 스레드 풀(ThreadPool)을 활용할 수 있다.

스레드 풀을 사용하면 **제한된 개수의 스레드를 미리 생성해 두고, 이를 재활용**하여 보다 효율적으로 비동기 작업을 수행할 수 있다.

아래는 스프링에서 스레드 풀을 설정하는 방법이다.

```
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10); // 스레드 풀의 기본 크기
        executor.setMaxPoolSize(100); // 스레드 풀의 최대 크기
        executor.setQueueCapacity(50); // 대기열 크기
        executor.initialize();
        return executor;
    }
}
```

기본적으로 **최소 10개의 스레드**를 유지하며, 필요에 따라 **최대 100개까지 동적으로 생성**할 수 있다.

또한, 실행 대기 중인 작업이 있을 경우 **최대 50개의 작업을 큐에 저장**하여 효율적으로 처리한다.

[##_Image|kage@70fe1/btsMlU7dab8/nPD98svvxbwchgvUZXTUQ0/img.png|CDM|1.3|{"originWidth":2072,"originHeight":360,"style":"alignCenter"}_##]

실행 시간이 603ms 까지 감소된 것을 확인할 수 있다.

## 결론

스프링에서 비동기 처리를 간단하게 활용하는 방법을 알아보았다.

이를 활용하면 대용량 데이터 처리, 외부 API 연동 등 다양한 작업에서 성능을 최적화할 수 있다. 앞으로 비동기 처리를 적극적으로 활용하여 여러 기능에 적용해 봐야겠다.