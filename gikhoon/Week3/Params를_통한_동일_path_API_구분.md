코테이토 프로젝트에서 이름, 포지션, 합격 기수별 멤버를 필터링 API가 있다.
또한, 이번 스프린트를 통해 멤버 상태(승인, 불승인, OM)에 따라 필터링 하는 같은 경로(path)의 API를 추가했다. 
이번 글에서는 Controller에서 같은 path + 다른 파라미터의 API를 구분하게 하는 방법과 Swagger에서 구분할 수 있는지 확인해보겠다.

### 같은 URL, 같은 HTTP 메서드의 엔드포인트가 존재하면?
1. 이름, 포지션, 합격 기수별 멤버 필터링 API
```java
@RequestMapping("/v1/api/member")
public class MemberController {

    @Operation(summary = "기수별 멤버에 추가 가능한 멤버 반환 API")
    @RoleAuthority(MemberRole.ADMIN)
    @GetMapping
    public ResponseEntity<AddableMembersResponse> findAddableMembersForGenerationMember(
            @RequestParam(name = "passedGenerationNumber", required = false) @Parameter(description = "멤버 합격 기수") Integer generationNumber,
            @RequestParam(name = "position", required = false) @Parameter(description = "멤버 포지션") MemberPosition position,
            @RequestParam(name = "name", required = false) @Parameter(description = "멤버 이름") String name
    ) {
        log.info("기수별 멤버에 추가 가능한 멤버 반환 API 호출");
        return ResponseEntity.ok()
                .body(memberService.findAddableMembers(generationId, generationNumber, position, name));
    }
}
```
2. 멤버 상태에 따른 멤버 필터링 API
```java
@Operation(summary = "회원 상태에 따른 조회 요청 API")
    @RoleAuthority(MemberRole.ADMIN)
    @GetMapping
    public ResponseEntity<List<MemberResponse>> findMembersByStatus(@RequestParam("status") MemberStatus status) {
        log.info("회원 상태에 따른 조회 요청 API 호출");
        return ResponseEntity.ok().body(memberService.getMemberByStatus(status));
    }
```

이 떄, Spring Application 실행 시 다음과 같은 문제가 발생한다.
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbSro1K%2FbtsMi2dSt94%2FS349p4KgXLQDqPiKRreLLK%2Fimg.png)

Spring Application이 실행되면

1. RequestMappingHandlerMapping이 실행되면서 모든 컨트롤러의 경로(Path) 정보를 가져옴

2. 매핑 정보를 HandlerMethodMapping에 저장

이 때 같은 URL, 같은 HTTP 메서드의 엔드포인트가 발견되면 충돌(Ambiguous Mapping)이 발생한다.

### 쿼리 파라미터로 구분 (params 사용)
Spring에서 다음과 같은 기준으로 API을 구분한다.


1️⃣ HTTP 메서드 (GET, POST 등)

2️⃣ 요청 경로 (@RequestMapping("/members") 등)

3️⃣ Path Variable (/members/{id} vs /members/active)

4️⃣ 쿼리 파라미터 (params 속성 사용)

따라서 동일 경로, 동일 HTTP 메소드의 API를 구분하기 위해 쿼리 파라미터를 이용해 구분하게 설정했다.

### 변경한 코드
```java
@Operation(summary = "회원 상태에 따른 조회 요청 API")
@RoleAuthority(MemberRole.ADMIN)
@GetMapping(params = "status")
public ResponseEntity<List<MemberResponse>> findMembersByStatus(@RequestParam("status") MemberStatus status) {
    log.info("회원 상태에 따른 조회 요청 API 호출");
    return ResponseEntity.ok().body(memberService.getMemberByStatus(status));
}
```

기존 GetMapping에서 params를 추가해 두 Spring에서 해당 부분을 구분할 수 있게 됐다.

### Postman 테스트
postman을 통해 Spring에서 두 API를 구분할 수 있는지 확인했다.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbfuCgH%2FbtsMjAVf3tV%2FMqgkxH8qKUa8yIKtKFzpE1%2Fimg.png)
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FzMJS0%2FbtsMktnkvJE%2Fq6OrUtF4FHSeTVElKkObK1%2Fimg.png)

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fz8uzE%2FbtsMihvuK3G%2FcPJYH6SjjjTQ76V5vzZ50k%2Fimg.png)
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fbb7VHd%2FbtsMi3Dlbre%2FE6Tu7EF05Sn0LzFNh8lEK0%2Fimg.png)
두 API가 서로 다른 메소드로 mapping된다는 것을 로그를 통해 확인할 수 있었다.

### Swagger에서 구분이 불가능한 이유
하지만 Swagger에서는 params를 통해 API 구분이 불가능하고 두 API가 합쳐져서 나왔다.

![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FvATFq%2FbtsMj7L1t7E%2Fm4wldVmwzRXCWOKRZvsVUk%2Fimg.png)

Swagger 스펙에서는 Spring과 다르게 API를 path(경로)와 HTTP 메서드를 기준으로 구분해 Params를 통해서는 구분하지 않는다.

따라서 같은 path를 사용하는 API(@GetMapping, @GetMapping(params = "status"))는 Swagger에서는 하나로 인식된다.


실제 요청에는 문제가 없지만 프로젝트 특성상 프론트가 Swagger에 나와있는 정보를 가져고와 요청을 보내기 때문에 
API Path를 수정하는 방식으로 설정할 수 밖에 없었다.

```java
@Operation(summary = "기수별 멤버에 추가 가능한 멤버 반환 API")
    @RoleAuthority(MemberRole.ADMIN)
    @GetMapping("/addable")
    public ResponseEntity<AddableMembersResponse> findAddableMembersForGenerationMember(
            @RequestParam(name = "generationId") @Parameter(description = "추가하고 싶은 기수의 Id") Long generationId,
            @RequestParam(name = "passedGenerationNumber", required = false) @Parameter(description = "멤버 합격 기수") Integer generationNumber,
            @RequestParam(name = "position", required = false) @Parameter(description = "멤버 포지션") MemberPosition position,
            @RequestParam(name = "name", required = false) @Parameter(description = "멤버 이름") String name
    ) {
        log.info("기수별 멤버에 추가 가능한 멤버 반환 API 호출");
        return ResponseEntity.ok()
                .body(memberService.findAddableMembers(generationId, generationNumber, position, name));
    }
```

Path를 /addable로 수정해 문제를 해결했다.

결론적으로 Swagger를 통해 동일한 두 API를 구분시키지 못했다. 
하지만 Spring에서 params를 이용한 파라미터가 다른 두 API를 구분하고 Swagger에서 구분하지 못하는 이유를 찾는 과정에서 결정에 대한 근거와 확신을 가질 수 있게 된 기회가 됐다.

