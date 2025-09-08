대규모 트래픽을 위한 서버 설계를 위해 단계별로 접근

점진적인 확장을 생각해보기

**1단계**

모든 컴포넌트를 하나의 서버에서 구현할 때

**2단계**

사용자가 점차 늘기 시작하면 사용자 요청을 처리하는 웹애플리케이션 서버와 데이터베이스를 분리

수직적 규모 확장 vs 수평적 규모 확장

**3단계**

사용자 요청 분리 - 로드밸런서

**4단계**

데이터베이스 다중화

마스터-슬레이브, 다중 마스터, 원형 다중화 등의 방식

**5단계**

캐시 도입

- 데이터 캐시
    - 특징 및 사용 이유
    - 주의점
- 콘텐츠 전송 네트워크(CDN)
    - 정적 콘텐츠, 지역 문제로 느려질 수 있는 경우 지역 캐싱
    - 고려 사항

**6단계**

무상태 웹 계층

무상태를 지향해야하는 이유

무상태는 왜 NoSQL이여야만 하는가

**7단계**

데이터 센터

고 가용성

주요 기술적 난제

**8단계**

메시지 큐를 통한 비동기 처리

**9단계**

로그, 메트릭 그리고 자동화

**10단계**

데이터베이스의 규모 확장

- 수직적 확장
- 수평적 확장
    - 마스터-슬레이브 → 샤딩

그 외의 샤딩 고려사항

- 유명인사 문제(핫스팟 키)

+proxySQL이 여기에서 같이 언급되는데, WAS가 많고, DB는 적을 때 이 커넥션을 효율적으로 관리하기 위해 중간 관리자 역할을 두는 역할
각 WAS에 DB를 접근하는 요청이 1000개가 들어왔다고 했을 때
프록시SQL이 이 DB커넥션을 먼저 받음으로써 한번에 DB 커넥션으로 가서 터지지 않도록 관리하며, 다중 DB서버를 두었을 때 효과적으로 연결시킬 수 있음.

넷플릭스 가용성 포스팅

https://netflixtechblog.com/active-active-for-multi-regional-resiliency-c47719f6685b

****추가 공부 내용**

데이터베이스 샤딩을 했을 때 어떻게 표현할 수 있을지에 대한 고민

책의 예시대로 사용자 id기준으로 샤딩을 했다 가정했을 때

게시글은 어떻게 가져올 수 있을까?에 대한 고민을 해볼 수 있음

```java
@Service
@RequiredArgsConstructor
public class PostService {

    private final List<PostRepository> shardRepositories; // 샤드별 Repo 주입
    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    private static final int PAGE_SIZE = 20;
    private static final int PER_SHARD_FETCH = 25; // 여유분 α

    public PageResult<Post> getPosts(Cursor cursor) {

        // 1. 샤드별 비동기 조회
        List<CompletableFuture<List<Post>>> futures = shardRepositories.stream()
            .map(repo -> CompletableFuture.supplyAsync(
                () -> repo.findNextPage(cursor, PER_SHARD_FETCH),
                executor
            ))
            .toList();

        // 2. 결과 모으기
        List<List<Post>> results = futures.stream()
            .map(CompletableFuture::join)
            .toList();

        // 3. k-way merge (createdAt, postId ASC)
        PriorityQueue<Post> heap = new PriorityQueue<>(
            Comparator.<Post, Instant>comparing(Post::createdAt)
                      .thenComparing(Post::postId)
        );
        results.forEach(heap::addAll);

        List<Post> merged = new ArrayList<>(PAGE_SIZE);
        while (!heap.isEmpty() && merged.size() < PAGE_SIZE) {
            merged.add(heap.poll());
        }

        // 4. 다음 커서 = 마지막 Post
        Cursor nextCursor = merged.isEmpty()
            ? cursor
            : new Cursor(
                merged.get(merged.size() - 1).createdAt(),
                merged.get(merged.size() - 1).postId()
            );

        return new PageResult<>(merged, nextCursor);
    }
}

```
