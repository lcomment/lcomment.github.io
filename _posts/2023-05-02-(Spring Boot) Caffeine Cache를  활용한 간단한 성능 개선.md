---
title: [(Spring Boot) Caffeine Cache를  활용한 간단한 성능 개선]
date: 2023-05-02 15:33:44 +09:00
categories: [Spring, 트러블 슈팅]
tags: [스프링, 캐시, Lovebird]

--- 

## 1. 서론

&nbsp; `Lovebird` 프로젝트를 진행하다가 개선할 필요가 있어보이는 로직이 있어서 해결 후 포스팅을 하게 되었다. Lovebird는 `커플 다이어리` 도메인의 앱인데, D-Day를 불러오는 부분에서 고민을 하게 되었다. 일반적인 방법인 DB에서 읽어오는 식으로 구현하면 어플을 실행할 때마다, 또는 해당 탭을 킬 때마다 `SELECT` 동작이 반복된다. 이는 크지만 않지만 DB 부하뿐만 아니라 READ 비용까지 증가시킨다. 따라서 이를 막기 위해 `Cache`를 활용하기로 했다.

<br>

## 2. 기존 로직

&nbsp; 유저가 HOME 화면이나 Profile 화면을 로드했을 때 Service 레이어의 `findMemberByNickname()` 메서드가 실행된다. 기존에는 Repository의 메서드를 이용해 해당 로직을 수행했다.

```java
. . .
	@Transactional(readOnly = true)
    public Member findMemberByNickname(String nickname) {
        return memberRepository.findMemberByNickname(nickname).orElseThrow(EntityNotFoundException::new);
    }
. . .
```

<br>

## 3. Cache와 Caffeine Cache

> 캐시(cache) : 데이터나 값을 미리 복사해 놓는 임시 장소

- Local Cache
	- 서버마다 캐시를 따로 저장
	- 다른 서버의 캐시를 참조하기 어려움
	- 속도 빠름
	- 로컬 서버 장비의 Resource를 이용한다. (Memory, Disk)
- Global Cache
	- 여러 서버에서 캐시 서버 접근 및 참조 가능
	- 별도의 캐시 서버 이용 → 서버 간 데이터 공유가 쉬움
	- 네트워크 트래픽을 사용해야 해서 로컬 캐시보다는 느리다.
	- 데이터를 분산하여 저장 가능

&nbsp; 캐시는 크게 로컬 캐시와 글로벌 캐시로 나뉜다. 현재 서버가 단일 노드이고, 글로벌 캐시를 구성하기 위해선 redis 등의 추가 인프라 비용이 발생하기 때문에 비용을 절약하면서 더 빠른 성능을 가져올 수 있는 로컬 캐시를 활용하기로 했다.

![20231108-1](/assets/img/posts/20230502-1.png)
<div align=center><h5>(Images: Caffeine Benchmarks)</h5></div>

&nbsp; 위 데이터는 스프링에서 제공하는 캐시들의 `초당 데이터 처리량` 비교 자료이고, 데이터에서 알 수 있듯이 Caffeine은 매우 우수한 지표를 보여주고 있다. 다른 기능이 딱히 필요가 없고, 단지 `Read`와 `Write` 측면에서 최고의 성능을 보여주고 싶었기 때문에 Caffeine Cache를 활용하기로 결정했다.

> Caffeine is a high performance, near optimal caching library.
>
> ✍️ by [Caffeine Github](https://github.com/ben-manes/caffeine)

<br>

## 4. Caffeine Cache 적용하기

> ### i) dependency 추가

```shell
# build.gradle
implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation 'com.github.ben-manes.caffeine:caffeine'
```

<br>

> ### ii) CacheType 생성

```java
@Getter
@RequiredArgsConstructor
public enum CacheType {
    MEMBER_PROFILE("member", 12, 10000);

    private final String cacheName;
    private final int expiredAfterWrite;
    private final int maximumSize;
}
```

- 현재는 하나 밖에 없지만, 이후 확장을 고려하여 enum으로 생성
- `expireAfterWrite` : 항목이 생성된 후 또는 해당 값을 가장 최근에 바뀐 후 특정 기간이 지나면 각 항목이 캐시에서 자동으로 제거되도록 지정
- `maximumSize` : 캐시에 포함할 수 있는 최대 엔트리 수 지정

<br>

> ### iii) Cache 세팅

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public List<CaffeineCache> caffeineCaches() {
        return Arrays.stream(CacheType.values())
                .map(cache -> new CaffeineCache(cache.getCacheName(), Caffeine.newBuilder().recordStats()
                                .expireAfterWrite(cache.getExpiredAfterWrite(), TimeUnit.HOURS)
                                .maximumSize(cache.getMaximumSize())
                                .build()))
                .toList();
    }
    @Bean
    public CacheManager cacheManager(List<CaffeineCache> caffeineCaches) {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(caffeineCaches);

        return cacheManager;
    }
}
```

- Parameter 
  - `recordStats`
    - 캐시에 대한 Statics 적용
  - `expireAfterWrite`
     - 항목이 생성된 후 또는 해당 값을 가장 최근에 바뀐 후 특정 기간이 지나면 각 항목이 캐시에서 자동으로 제거되도록 지정
  - `maximumSize`
     - 캐시에 포함할 수 있는 최대 엔트리 수 지정

<br>

> ### iv) 로직 수정

```java
	. . .
    @Transactional(readOnly = true)
    @Cacheable(cacheNames = "member", key = "#nickname", value = "member")
    public Member findMemberByNickname(String nickname) {
        return memberRepository.findMemberByNickname(nickname).orElseThrow(EntityNotFoundException::new);
    }
    
    @Transactional
    @CacheEvict(cacheNames = "member", key = "#nickname")
    public void delete(String nickname) {
        memberRepository.delete(findMemberByNickname(nickname));
    }
    . . .
```

<br>

## 5. Test

&nbsp; Spring에서는 AOP Proxy가 `@Cacheable`을 처리해준다. 따라서 메서드 실행 전에 프록시 객체가 요청된 메서드의 결과가 캐싱돼 있는지 확인하게 된다. 이러한 동작을 하는 객체가 `CacheInterceptor`다.

&nbsp; 그럼 테스트는 어떻게 진행할까? 캐싱 동작이 이루어졌다면 해당 메서드를 여러 번 호출하더라도 `한번만` 실행될 것이다. 코드는 다음과 같다.

```java
    . . .
    
    @Test
    @DisplayName("findMemberByNickname() 메서드를 호출하면 처음을 제외한 나머지는 Cache가 인터셉트한다.")
    void findMemberByNicknameUsedCache() throws IOException {
        // given
        Member mockMember = getMember("TEST", Gender.MALE, getLocalDate("2023-05-15"));

        given(memberRepository.findMemberByNickname(anyString()))
                .willReturn(Optional.of(mockMember));

        // when
        IntStream.range(0, 10).forEach((i) -> memberService.findMemberByNickname("TEST"));

        // then
        verify(memberRepository, times((1))).findMemberByNickname("TEST");
    }
    . . .
```

<br>

---

<br>

### Reference

- [tistory@livenow14](https://livenow14.tistory.com/56)
- [tistory@goldfishhead](https://goldfishhead.tistory.com/29)