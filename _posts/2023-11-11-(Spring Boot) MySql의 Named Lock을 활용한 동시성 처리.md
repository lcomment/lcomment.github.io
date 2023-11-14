---
title: [(Spring Boot) MySql의 Named Lock을 활용한 동시성 이슈 해결]
date: 2023-11-14 09:33:44 +09:00
categories: [Spring, 트러블 슈팅]
tags: [스프링, mysql, named lock, 동시성 처리]
# image: /assets/img/posts/retrospect-title.jpeg

---

> 포스팅에 첨부된 코드는 예시 코드임을 명시합니다.

## 문제 상황

&nbsp; 회사에서 운영하는 이벤트 앱은 응모하면 당첨자를 선발하는 간단한 이벤트만을 대상으로 하고 있었다. 하지만 외부에서 쿠폰 관련 이벤트 사업이 들어왔고, 내가 담당하게 되었다. 대략적인 로직은 다음과 같다.

1. 어드민 서버로 핀 번호 리스트를 포함한 요청을 보내 쿠폰 발행
2. 유저가 앱을 통해 쿠폰 발급

&nbsp; 하지만 유저가 앱을 통해 쿠폰을 발급할 때 동시성 문제가 발생할 수 있다. 두 유저가 동시 접근하여 `경쟁 상태`(race condition)이 발생할 수 있고, 한 유저가 어뷰징을 통해 발급 가능한 쿠폰 개수보다 많은 쿠폰을 발급 받아갈 수도 있다. 이를 해결하기 위해 여러 `Lock`을 고려하게 되었는데, 배포 서버가 `elastic beanstalk`임을 고려했을 때 MySql의 `Named Lock`을 활용한 `분산락`을 구현하기로 결정했다.

ps. 사실 이와 같은 문제 상황에서 `Redis`나 `Zookeeper`를 활용하여 분산락 환경을 구성해주는 경우가 많다고 한다. 하지만 처음부터 MySQL을 사용하고 있던 점, 인프라를 추가 구성했을 시 시간 및 비용 등의 리소스가 우려되는 점을 고려했을 때 Named Lock으로 구현하는 것이 맞다고 판단 되었다.

## MySql의 Named Lock

&nbsp; Named Lock은 말 그대로 이름(name)을 통해 Lock을 건다. 이름을 통해 Lock을 걸고, 작업을 완료한 후 Lock을 해제하는 식으로 동작한다. MySql 5.7 미만의 버전에서는 동시에 하나의 잠금만 획득 가능했지만, 5.7 이상부터는 여러 개의 잠금 획득이 가능하다. 

```markdown
✅ MySQL 5.7 미만 : 동시에 `하나`의 잠금만 획득 가능, 잠금 이름 글자수 `무제한`
✅ MySQL 5.7 이상 : 동시에 `여러`개 잠금 획득 가능, 잠금 이름 글자수 `60자 제한`
```

> ### GET_LOCK(str, timeout)

- 입력받은 이름(`str`)으로 `timeout`초 동안 Lock 획득 시도
- `timeout`이 음수 → Lock을 획득할 때까지 무한 대기
- 한 세션에서만이 잠금 점유 가능
- Transaction의 커밋이나 롤백의 영향을 받지 않음
- Return
    - 1(성공)
    - 0(실패)
    - null(에러)

> ### RELEASE_LOCK(str)

- 입력받은 이름(`str`)의 Lock 해제
- Return
    - 1(해제 성공)
    - 0(현재 스레드에서 획득한 Lock이 아님)
    - null(Lock이 존재하지 않음)

## NamedLockTemplate.java

```java
@Component
@RequiredArgsConstructor
public class NamedLockTemplate {

	private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;
	private static final String GET_LOCK = "SELECT GET_LOCK(:userLockName, :timeoutSeconds)";
	private static final String RELEASE_LOCK = "SELECT RELEASE_LOCK(:userLockName)";

	public <T> T executeWithLock(String userLockName, int timeoutSeconds, Supplier<T> supplier) {
		try {
			getLock(userLockName, timeoutSeconds);
			return supplier.get();
		} finally {
			releaseLock(userLockName);
		}
	}

	private void getLock(String userLockName, int timeoutSeconds) {
		Map<String, Object> params = Map.of(
			"userLockName", userLockName,
			"timeoutSeconds", timeoutSeconds
		);

		Integer result = namedParameterJdbcTemplate.queryForObject(GET_LOCK, params, Integer.class);
		checkResult(result);
	}

	private void releaseLock(String userLockName) {
		Map<String, Object> params = Map.of("userLockName", userLockName);

		Integer result = namedParameterJdbcTemplate.queryForObject(RELEASE_LOCK, params, Integer.class);
		checkResult(result);
	}

	private void checkResult(Integer result) {
		if (isNull(result) || result != 1) {
			throw new LockException("locked.resource");
		}
	}
}
```

&nbsp; 처음엔 `JpaRepository`에서 `@Query` 어노테이션을 통해 구현했는데, Jpa를 활용하는 패러다임에 맞지 않은 것 같아 고민하다가 [우아한 기술 블로그](https://techblog.woowahan.com/2631/)를 보고 수정하게 되었다.

## CouponEntryService.java

```java
@Service
@RequiredArgsConstructor
public class UserUseCouponService {
    . . .
    private final NamedLockTemplate namedLockTemplate;
    . . .

    @Transactional
	public void issueCouponWithNamedLock(User user, Event event) {
		namedLockTemplate.executeWithLock(
			COUPON_ISSUE_LOCK_NAME,
			COUPON_ISSUE_LOCK_TIMEOUT,
			() -> issueCoupon(user, event)
		);
	}
    . . .
}
```

&nbsp; 쿠폰을 발급할 때 발급 가능한 쿠폰이 있는지 `조회` 후 엔트리를 `생성`하기 때문에 두 쿼리를 묶어 issueCoupon() 메서드를 구현하고, 쿼리 시작 전에 Lock 획득, 종료 시 Lock 해제를 하였다.

## 테스트 코드

```java
. . .
	@Test
	void concurrencyTest() throws InterruptedException {
		// given
        List<User> users = userRepository.findAll();

        int threadCount = 10000;
        ExecutorService executorService = Executors.newFixedThreadPool(32);
		CountDownLatch latch = new CountDownLatch(threadCount);
		
		// when
		AtomicInteger j = new AtomicInteger(1);
		for (int i = 0; i < threadCount; i++) {
			executorService.submit(() -> {
				try {
					couponEntryService.issueCouponWithNamedLock(users.get(j.getAndIncrement()), event);
				} finally {
					latch.countDown();
				}
			});
		}

		latch.await();

		// then
		List<CouponEntry> entries = couponEntryRepository.findAll();

		assertThat(entries.size()).isEqualTo(50);
	}
. . .
```

&nbsp; 10,000명의 유저가 50개의 쿠폰을 두고 경쟁하는 상황을 예시로 테스트를 진행하였고, 결과는 다음과 같다.

![20231114-1](/assets/img/posts/20231114-1.png)

<br>

> ### References

- [우아한 기술블로그](https://techblog.woowahan.com/2631/)
- [velog@this-is-spear](https://velog.io/@this-is-spear/MySQL-Named-Lock)
