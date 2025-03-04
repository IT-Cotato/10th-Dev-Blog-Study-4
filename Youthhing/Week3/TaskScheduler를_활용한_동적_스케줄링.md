# TaskScheduler를 통한 동적 스케줄링


코테이토 프로젝트에서 출석 입력 기능을 개발 이후, 운영진의 출결 입력 안내가 없더라도 사용자들이 세션 시작 시간이 되면 출결 입력을 할 수 있었으면 좋겠다는 니즈가 있었다.

따라서, **출석 입력이 허용되기 시작하는 시간에 사용자들에게 출결 입력 알림을 전송**이란 기획이 추가되었다.

해당 기능 개발을 위해 2가지에 대한 고민이 필요했다.

1. 서버가 클라이언트에게 출결 입력 시작을 알리는 방법
2. 유저가 지정한 시간에 알림을 전송하는 방법

첫번째 고민에 대해서는 서버 → 클라이언트의 단방향 전송을 위한 Server Sent Event를 사용하기로 결정했다.

(구체적인 이야기는 다른 글에서 다루도록 하겠다.)

이 글에선 두 번째 고민인 유저가 지정한 시간에 동적으로 서버에서 스케줄링을 하는 과정 개발에 대한 글을 작성해보겠다.

### @Scheduled를 통한 정적 스케줄링

스프링에선 Scheduled 어노테이션을 활용해 지정된 시간에 작업을 실행하는 스케줄링이 가능하다.

`fixedRate`, `fixedDelay` , `initialDelay` , `cron` 등의 설정을 통해 원하는 시간 또는 주기마다 작업을 실행할 수 있다.

하지만, `@Scheduled`은 Spring 컨테이너가 시작될 때 설정된 스케줄을 고정하는 정적 스케줄링이기 때문에, 애플리케이션 실행 중에 스케줄링 시작 시간을 변경하는 것이 불가능하다.

기획 요구사항은 코테이토 운영진이 설정한 세션 시작 시간(= 출석 시작 시간)에 접속한 사용자들에게 출결 시작 알림을 보내는 것이다.

즉, 유저가 입력한 시간에 맞게 작업을 시작하는 것이 요구사항이다. 이를 위해 스프링에서 지원하는 동적 스케줄링 하는 방법이 필요했다.

# 동적 스케줄링

Spring 기반 어플리케이션 개발에서 동적 스케줄링을 지원하는 방법은 TaskScheduler, QuartzScheduler를 활용하는 방법으로 총 2가지가 있다.

## TaskScheduler

스프링에선 동적으로 특정 Trigger 또는 시간을 기준으로 작업을 실행하는 TaskScheduler 라는 인터페이스를 제공한다.

```java
public interface TaskScheduler {

	Clock getClock();

	ScheduledFuture schedule(Runnable task, Trigger trigger);

	ScheduledFuture schedule(Runnable task, Instant startTime);

	ScheduledFuture scheduleAtFixedRate(Runnable task, Instant startTime, Duration period);

	ScheduledFuture scheduleAtFixedRate(Runnable task, Duration period);

	ScheduledFuture scheduleWithFixedDelay(Runnable task, Instant startTime, Duration delay);

	ScheduledFuture scheduleWithFixedDelay(Runnable task, Duration delay);
```

이 중 Runnable과 Instant를 파라미터로 갇는 가장 기본적인 `schedule` 메서드를 활용하면 원하는 시간에 특정 작업을 실행하도록 스케줄을 등록할 수 있다.

해당 방식은 아래와 같이 ThreadPoolTaskScheduler를 통해 스레드의 개수를 설정한 TaskScheduler를 미리 Bean으로 등록해 원하는 작업을 비동기로 실행할 수 있다.

```java
@Configuration
@EnableScheduling
public class SchedulerConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-task-");
        scheduler.initialize();
        return scheduler;
    }
}
```

단, 이 방식은 메모리 기반으로 동작하기에 어플리케이션이 재시작하면 등록된 작업이 모두 소멸되는 문제점이 있다.

## QuartzScheduler

Quartz Scheduler는 자바 기반의 오픈소스 스케줄링 프레임워크로, 특정 시간에 작업을 자동 실행할 수 있도록 지원하는 라이브러리가 존재한다.

Spring Boot와도 쉽게 통합 가능하며, Spring Batch, Spring Scheduler보다 정교한 스케줄링이 가능하다.

[https://www.quartz-scheduler.org/](https://www.quartz-scheduler.org/)

- Job: 실행해야되는 작업
- Trigger: Job을 실행할 조건 또는 시점
- JobStore: Job과 Trigger의 저장소
    - RAMStore: JVM에 스케줄을 저장, 따라서 어플리케이션이 종료되면 사라진다.
    - JDBCStore: JDBC 기반의 Job과 Trigger 저장 방식. DB에 내용을 저장하므로 사라지지 않는다.
        
        

### JDBCStore를 통한 DB 스케줄 관리

JPA/Hibernate를 사용하는 경우 Quartz는 자체적인 라이브러리를 통해 스케줄을 DB에 저장하고 어플리케이션이 재시작할 때 스케줄을 복구한다.

따라서, 아래와 같이 properties 파일 설정 후 별도의 테이블을 구현할 필요가 없다.

```java
# Quartz를 DB 기반 저장 방식으로 설정
spring.quartz.job-store-type=jdbc

# DB 연결 정보 설정
spring.datasource.url=jdbc:mysql://localhost:3306/quartzdb
spring.datasource.username=root
spring.datasource.password=root

# Quartz 테이블 자동 생성 (처음 실행 시 한 번만 사용)
spring.quartz.jdbc.initialize-schema=always
```

(구체적으로 실행해보고 싶은 인원은 [해당 링크(Quartz 공식 예제)](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/quick-start.html)를 통해 테스트해보면 좋을 듯 하다.)

[스프링에서도 Quatz와 통합 가이드 문서](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-quartz)가 있다. 

구체적인 활용 예제는 아래 Repository에 추후 업데이트 하도록 하겠다.

https://github.com/Youthhing/scheduler-example

# 해결

결론적으로 2가지 방법 중 TaskScheduler를 활용하기로 결정했다.

QuartzScheduler는 어플리케이션이 재시작 되었을때도 스케줄이 유지된다는 장점과 분산 환경에서 사용하기 좋다는 점, 복잡한 배치처리를 하기에 좋다는 장점이 있다.

하지만, 우리가 해야하는 작업과 서버 환경을 고려했을때 과한 스펙이라고 판단했다.

우리 서버는 분산 환경의 서버가 아닌 단일 서버를 사용하고 있고, 알림을 전송할 당시에 연결되어있는 사용자에게만 메시지를 전송하는 비교적 간단한 스케줄을 이용한다.

따라서, 외부 라이브러리 이용에 대한 부담을 고려해 우선 TaskScheduler와 알림 데이터를 DB에 저장해 어플리케이션이 재시작될 때 진행되지 않은 스케줄을 다시 로딩하는 방식으로 문제를 해결하기로 결정했다.

## ✅ TaskScheduler 와 DB 활용

어플리케이션이 재시작해도 실행되지 않은 Task를 메모리에 올리기 위해선 아래와 같은 2가지 작업이 필요하다.

1. 별도의 작업 테이블 추가
2. 어플리케이션 재시작 시 실행되지 않은 작업을 TaskScheduler에 등록

### 1. 별도의 작업 테이블 추가

아래와 같은 출석 알림 테이블을 추가한다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class AttendanceNotification {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "attendance_id")
    private Attendance attendance;

    @Column(name = "is_done", nullable = false)
    private boolean done;

    @Builder
    public AttendanceNotification(Attendance attendance, boolean done) {
        this.attendance = attendance;
        this.done = done;
    }

    public void done() {
        this.done = true;
    }
}
```

이후, 출석이 생성될 때 출석 알림 데이터를 생성 후 스케줄에 등록한다.

```java
@Transactional
    public void createAttendance(Session session, Location location, LocalDateTime attendanceDeadline, LocalDateTime lateDeadline) {
				// ... 출석 생성 로직
				
        AttendanceNotification attendanceNotification = AttendanceNotification.builder().attendance(attendance).done(false).build();
        attendanceNotificationRepository.save(attendanceNotification);

        schedulerService.scheduleAttendanceNotification(attendanceNotification);
    }
```

```java
public void scheduleAttendanceNotification(final AttendanceNotification attendanceNotification) {
    Attendance attendance = attendanceNotification.getAttendance();
    Session session = sessionReader.findById(attendance.getSessionId());
    ZonedDateTime seoulTime = TimeUtil.getSeoulZoneTime(session.getSessionDateTime());

		// TaskScheduler에 등록
    ScheduledFuture<?> schedule = taskScheduler.schedule(() -> {
                log.info("schedule attendance notification: session id <{}>, time <{}>", attendance.getSessionId(), session.getSessionDateTime());
                sseSender.sendAttendanceStartNotification(attendanceNotification);
                notificationByAttendanceId.remove(session.getId());
            },
            seoulTime.toInstant());
    notificationByAttendanceId.put(session.getId(), schedule);
}
```

### 2. 어플리케이션 재시작 시 작업 등록

DB에서 실행되지 않은 작업을 @PostConstruct를 활용해 스프링 컨테이너가 시작될 때 스케줄로 등록한다.

```java
@PostConstruct
protected void restoreScheduledTasksFromDB() {
    List<AttendanceNotification> attendanceNotifications = attendanceNotificationRepository.findAllByDoneFalse();

    attendanceNotifications.forEach(
            attendanceNotification -> {
                Session session = sessionReader.findById(attendanceNotification.getAttendance().getSessionId());
                if (session.getSessionDateTime().isBefore(LocalDateTime.now())) {
                    return;
                }

                ScheduledFuture<?> schedule = taskScheduler.schedule(
                        () -> {
                            log.info("schedule attendance notification: session id <{}>, time <{}>", session.getId(), session.getSessionDateTime());
                            sseSender.sendAttendanceStartNotification(attendanceNotification);
                            notificationByAttendanceId.remove(attendanceNotification.getAttendance().getId());
                        },
                        TimeUtil.getSeoulZoneTime(session.getSessionDateTime()).toInstant()
                );
                notificationByAttendanceId.put(attendanceNotification.getAttendance().getId(), schedule);
                log.info("restored attendance notification: attendance id <{}>", attendanceNotification.getAttendance().getId());
            });
}
```

### 실행 결과

지정된 시간에 스케줄된 이벤트 전송 로그

```java
2025-02-17T22:43:30.005+09:00  INFO 76846 --- [cheduled-task-1] o.c.c.common.schedule.SchedulerService   : schedule attendance notification: session id <37>, time <2025-02-17T22:43:30>
2025-02-17T22:43:30.051+09:00  INFO 76846 --- [cheduled-task-1] org.cotato.csquiz.common.sse.SseSender   : [send attendance notification: session id <37>, time <2025-02-17T22:43:30>]
```

클라이언트에는 이렇게 이벤트를 전달 받을 수 있다.

![결과.png](image/image01.png.png)

### OutOfMemoryError의 발생 가능성

단, 이 경우는 `ScheduledFuture` 를 ConcurrentHashMap을 통해서 관리하고 있다. 즉 메모리에 올려서 관리하는 것인데 스케줄이 많아질수록 메모리 사용량이 증가하기에 OOM이 발생할 가능성이 있다. 

이는 모니터링 툴 추가 후 지속적으로 추적할 예정이다.

# 느낀점

출결 알람 전송 기능을 개발하며 똑같이 동작 하더라도 다양한 선택지들이 많았다. 그 중 무언가를 선택하는 기준ㅇ에 관해서 고민을 많이한 것 같다. 이번엔 ‘서버 환경’과 ‘개발 비용’을 우선해서 방법을 선택했다. 

그 결과 서버 → 클라이언트 이벤트 전송도, 동적 스케줄링도 비교적 간단한 방법을 선택했다. 서비스가 보다 확장되고 운영하면서 얻는 메트릭을 기반으로 더 나은 방법으로 유지보수하며 성장하고 싶다.

짧 - 생각보다 자료가 많이 없어서 공식문서 보면서 공부했는데 역시 공식 문서가 최고다.
