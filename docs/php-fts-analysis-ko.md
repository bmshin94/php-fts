# php-fts 전수조사 & 활용 전략 정리 ✨

> 카리나(Claude Code)와 함께 진행한 php-fts 저장소 분석 · 활용 · 수익화 논의 정리본
>
> - 내 저장소(포크): https://github.com/bmshin94/php-fts
> - 원본(upstream): https://github.com/olivier-ls/php-fts
> - Packagist 패키지: https://packagist.org/packages/ols/php-fts
> - Hacker News (Show HN): https://news.ycombinator.com/item?id=48041316
>
> 작성일: 2026-09-20

---

## 1. 한 줄 요약

**php-fts는 "설치할 게 아무것도 없는, 순수 PHP로 만든 전문(full-text) 검색 엔진"이다.**

Elasticsearch·Meilisearch 같은 검색 서버를 띄울 수 없는 환경(공유 호스팅, 소형 VPS)에서
**폴더 하나**만 있으면 오타 교정 + 랭킹 + 필터 + 패싯 + 하이라이팅이 되는 검색을 붙일 수 있다.

```php
$engine = SearchEngine::open('./search_data');
$engine->search('lether shoe');  // "leather shoe"를 찾아줌 (오타 1개 허용)
$engine->search('革靴');          // 일본어도 찾아줌
```

---

## 2. 저장소 전수조사 결과

### 2.1 규모

| 항목 | 값 |
|---|---|
| 전체 파일 | 123개 (.git 제외) |
| `src/` PHP 코드 | 약 17,300줄 |
| `tests/` 테스트 코드 | 약 14,400줄 |
| 테스트 케이스 수 | **740개** (`#[Test]` 어트리뷰트 기준) |
| 런타임 의존성 | **0개** (PHP 8.1+ 만 필요) |
| 라이선스 | MIT |
| 원본 저장소 스타 | 약 141★ (조사 시점) |

> 테스트가 본 코드의 83% 분량. 이 비율 자체가 이 프로젝트의 성격을 말해준다.

### 2.2 폴더별 역할

```
php-fts/
├── src/                  ← 엔진 본체 (여기가 전부)
│   ├── SearchEngine.php      공개 API. 사용자가 만질 유일한 클래스 (348줄)
│   ├── Schema.php            필드 타입 정의 (text/keyword/number/boolean/tags)
│   ├── Filter.php            eq/gt/between/in/and/or/not 필터 빌더
│   ├── Facet.php             패싯(집계) 정의 — terms / stats
│   ├── Sort.php              정렬 정의
│   ├── Highlight.php         검색어 하이라이팅 설정
│   ├── Hit.php / SearchResult.php   결과 객체
│   ├── LockManager.php       쓰기 락 (동시 쓰기 1개 보장)
│   ├── Analysis/             ★ 텍스트 분석 (26개 문자 체계 지원)
│   │   ├── Analyzer.php          토크나이저
│   │   ├── CharacterFolder.php   대소문자/악센트/아랍어 하라카트 정규화
│   │   ├── FoldingTables.php     유니코드 폴딩 테이블 (1,539줄, 자동 생성됨)
│   │   ├── Script.php            문자 체계 판별 (라틴/한글/한자/타이…)
│   │   └── Utf8.php              확장 없이 쓰는 UTF-8 처리
│   ├── Index/               ★ 역색인 엔진 (핵심 중의 핵심)
│   │   ├── SegmentIndex.php      세그먼트 읽기 (2,249줄 — 최대 파일)
│   │   ├── SegmentIndexWriter.php 세그먼트 쓰기
│   │   ├── IndexDirectory.php    커밋/매니페스트/트랜잭션 관리
│   │   ├── SegmentMerger.php     세그먼트 병합
│   │   ├── MergePolicy.php       언제 병합할지 결정
│   │   ├── PostingsWriter/Cursor/Format  포스팅 리스트 + 스킵 테이블
│   │   ├── BlockDictionary*      용어 사전 (front-coding 압축)
│   │   ├── TermGramIndex.php     ★ 오타 교정용 n-gram 인덱스
│   │   ├── Bitset.php            매치 집합 비트셋 (패싯 계산의 기반)
│   │   ├── NumericColumn / KeywordColumn / TagColumn  컬럼 스토어
│   │   └── DocumentStore*        원문 저장소
│   ├── Query/               ★ 질의 실행
│   │   ├── QueryPlan.php         질의 계획
│   │   ├── QuerySlot.php         단어 단위 슬롯
│   │   ├── TermExpansion.php     오타/접두사 확장
│   │   ├── Scorer.php            BM25F 점수 계산
│   │   ├── CollectionStatistics.php  IDF 통계
│   │   ├── TopK.php              상위 K개 힙
│   │   └── Highlighter.php       하이라이팅 실행
│   ├── Storage/             ★ 파일 포맷 계층
│   │   ├── SegmentFormat.php     헤더/디렉터리/트레일러 레이아웃
│   │   ├── SegmentWriter/Reader  실제 바이트 읽고 쓰기
│   │   ├── Varint.php            가변 길이 정수 인코딩(압축)
│   │   └── Manifest.php          "지금 살아있는 세그먼트" 목록
│   ├── Exception/           9종 예외 (전부 FtsException 상속)
│   └── autoload.php         Composer 없이 쓸 때의 오토로더
├── tests/                740개 테스트 (단위 + 통합 + 다국어 재현율 + 보안)
├── benchmark/            45,000개 상품 벤치마크 스크립트 (콜드/웜)
├── demo/                 동작하는 예제 앱 (seed.php + search.php + index.html)
├── docs/
│   ├── v2/README.md          2.0 설계 문서
│   ├── v2/SEGMENT-FORMAT.md  ★ 온디스크 바이너리 포맷 명세서
│   └── demo.gif              데모 화면 녹화
├── tools/generate-folding.php  유니코드 폴딩 테이블 생성기
├── .github/workflows/ci.yml    PHP 8.1~8.5 × Ubuntu/Windows/macOS 매트릭스 CI
├── CHANGELOG.md          41KB — 성능 수치와 설계 근거가 전부 기록됨
├── SECURITY.md           위협 모델 + 하드닝 체크리스트
└── CLAUDE.md             (내가 추가한) 카리나 페르소나 가이드
```

---

## 3. 어떻게 동작하나 (쉽게)

### 3.1 인덱싱 = "책 맨 뒤 찾아보기"를 미리 만들어 두는 것

문서를 넣으면(`put`/`putMany`) 단어를 쪼개서
**단어 → 그 단어가 있는 문서 번호 목록**(= 역색인, inverted index)을 파일로 저장한다.
검색할 땐 문서를 다 읽지 않고 이 목록만 보면 되니까 빠르다.

### 3.2 세그먼트 = "덮어쓰지 않고 새로 쓰기"

- 쓰기 한 번 = `seg_xxxx.fts` 파일 하나 (**한 번 쓰면 절대 수정 안 함**)
- 어떤 세그먼트가 "살아있는지"는 `commit.26` 매니페스트가 결정
- 커밋 = 매니페스트 파일 **rename 한 번**(원자적 연산)

→ 쓰다가 서버가 죽어도 **깨진 인덱스가 아니라 그냥 쓰레기 파일이 남을 뿐**.
아무 매니페스트도 가리키지 않는 세그먼트는 "존재하지 않는 것"이다. (Lucene과 같은 설계)

### 3.3 오타 교정 = "문서가 아니라 단어장을 뒤진다"

1.x는 문서 전체를 트라이그램으로 색인해서 느렸다.
2.0은 **단어장(vocabulary)만** n-gram으로 색인한다.

```
"lether" 입력 → 단어장에서 편집거리 1 이내 단어 찾기 → "leather" 발견 → 그 단어로 검색
```

→ 오타 교정 비용이 **문서 수가 아니라 단어 수**에 비례. 문서가 10배 늘어도 단어장은 거의 안 늘어난다.
성능이 `couteau de cuisine inox` 기준 **2,261ms → 318ms**로 떨어진 이유가 이것.

편집 예산은 Elasticsearch `fuzziness: AUTO`와 동일:
- 1~2자: 오타 허용 안 함
- 3~5자: 1자 오타 허용
- 6자~: 2자 오타 허용
- **바이트가 아니라 글자 기준** → 키릴/아랍 문자 1자 오타 = 1편집

### 3.4 랭킹 = BM25F

- TF(단어가 문서에 몇 번 나오는가) + IDF(그 단어가 얼마나 희귀한가)
- **필드별** 빈도/길이 정규화 + 필드별 가중치(`title` boost 3.0 등)
- `minimum_should_match`를 **단어 개수가 아니라 정보량(IDF)으로 계산** →
  `de`(45,000건 중 38,555건에 등장, 정보량 1.8%)는 혼자서 문턱을 넘지 못함

### 3.5 패싯 = "문서를 한 개도 안 읽고 개수 세기"

브랜드별/카테고리별 개수, 가격 min/max/avg 등을 **비트셋 + 컬럼 스토어**로 계산한다.
그래서 45,000건 카탈로그에서도 사이드바 집계가 순식간.

그리고 `tag`/`exclude` 조합이 진짜 핵심:

```php
Filter::in('brand', ['Adidas'])->tag('brand')   // 필터엔 브랜드 조건이 걸림
Facet::terms(exclude: 'brand')                  // 근데 브랜드 패싯은 그 조건을 무시
```

→ 아디다스를 골라도 "Puma (2)"가 계속 보인다. 이게 없으면 **필터가 일방통행 문**이 된다.
쇼핑몰 필터 UI에서 제일 흔히 틀리는 부분을 라이브러리가 대신 해결해 준다.

---

## 4. 성능 (README / CHANGELOG 실측)

**OVH 공유 호스팅, PHP 8.3, 45,000개 상품, 콜드 스타트 100회:**

| 단계 | p50 | p95 |
|---|---|---|
| `open()` 인덱스 열기 | 8.7 ms | 13.3 ms |
| 검색 1회 (최악 질의) | 118.6 ms | 156.0 ms |
| **합계** | **127.4 ms** | **165.7 ms** |

**질의별 (개발 머신, 콜드):**

| 질의 | 매치 수 | 총 시간 |
|---|---|---|
| `steel` | 1,412 | 23 ms |
| `stel` (오타) | 1,379 | 25 ms |
| `couteau de cuisine inox` | 1,013 | 61 ms |
| `acier lame longueur` (최악) | 20,109 | 109 ms |

- 45,000건 인덱스 = 디스크 약 70MB
- 임포트 1회 = 메모리 피크 64MB (128MB 제한 안)
- `putMany()`는 제너레이터를 받아 **세그먼트 1개 분량만** 메모리에 유지 → 입력 크기와 무관하게 메모리 상한 고정

---

## 5. 언어 지원 — 그리고 한국어의 현실

### 5.1 26개 문자 체계

**단어를 띄어쓰는 18개** (오타 교정 + 접두사 완성 지원):
라틴 · 키릴 · 그리스 · 아랍 · 히브리 · 데바나가리 · 벵골 · 구르무키 · 구자라트 · 오리야 · 타밀 · 텔루구 · 칸나다 · 말라얄람 · 싱할라 · 아르메니아 · 조지아 · 에티오피아

**연속 표기 8개** (n-gram 색인, 오타 교정 **없음**):
한자 · 히라가나 · 가타카나 · **한글** · 타이 · 라오 · 크메르 · 미얀마

### 5.2 ⚠️ 한국어는 절반만 지원된다

README가 직접 인정하는 부분:

> **Korean gets no typo tolerance.** Hangul is treated as continuous, like
> Lucene's `CJKBigramFilter` does by default. ... Korean does put spaces
> between words, so this one is a trade rather than a necessity.

즉:
- ✅ "가죽구두" 검색 → 바이그램(가죽/죽구/구두)으로 매칭됨. **검색 자체는 된다.**
- ❌ "가죽구투" 같은 **오타는 못 잡는다**
- ❌ 형태소 분석 없음 → "신발이" / "신발을" / "신발" 이 서로 다른 토큰
- ❌ 초성 검색(ㄱㅈㄱㄷ) 없음, 한/영 오타 변환(rkwnr → 가죽) 없음

**→ 이게 바로 한국 시장에서 내가 파고들 빈틈이다.** (7번 항목 참고)

---

## 6. 설치 및 사용법

### 6.1 설치

**Composer (권장)**
```bash
composer require ols/php-fts
```

**Composer 없이** — `src/` 폴더를 복사하고:
```php
require '/path/to/php-fts/src/autoload.php';
```

**요구사항: PHP 8.1 이상 + 쓰기 가능한 디렉터리. 끝.**
확장 모듈 불필요. `ext-posix`는 있으면 죽은 프로세스의 락을 더 빨리 회수하는 데만 쓰인다.

### 6.2 데모 바로 돌려보기

```bash
git clone https://github.com/bmshin94/php-fts
cd php-fts
composer install          # 개발용(phpunit/phpstan). 런타임엔 불필요
php demo/seed.php         # ./demo/search_data 생성 (약 194개 상품)
php -S localhost:8000 -t demo
# 브라우저에서 http://localhost:8000
```

```bash
composer test      # phpunit (740 테스트)
composer analyse   # phpstan level 6
composer check     # 둘 다
```

### 6.3 기본 사용 흐름

```php
use Ols\PhpFts\{SearchEngine, Schema, Filter, Facet, Sort, Highlight};

// 1) 스키마 (선택 — 없으면 첫 커밋에서 타입 자동 추론)
$schema = Schema::make()
    ->text('title', boost: 3.0)
    ->text('description')
    ->keyword('brand')
    ->tags('tags')
    ->number('price')
    ->number('stock')
    ->boolean('active');

// 2) 인덱스 열기 (웹루트 바깥에!)
$engine = SearchEngine::open('/var/app/search_data', $schema);

// 3) 색인 — DB 커서에서 바로 흘려보내기 (메모리 상한 고정)
$engine->putMany((function () use ($pdo) {
    foreach ($pdo->query('SELECT * FROM products') as $row) {
        yield $row['sku'] => $row;
    }
})());

// 4) 검색
$result = $engine->search(
    '가죽 구두',
    limit: 20,
    offset: 0,
    filters: Filter::all(
        Filter::eq('active', true),
        Filter::gt('stock', 0),
        Filter::between('price', 50000, 200000),
        Filter::in('brand', ['Adidas'])->tag('brand'),
    ),
    facets: [
        'brand'    => Facet::terms(exclude: 'brand'),
        'category' => Facet::terms(),
        'price'    => Facet::stats(exclude: 'price'),
    ],
    sort: Sort::desc('stock'),
    highlight: Highlight::fields(['title', 'description'])->excerpt(3),
);

echo $result->total;      // 전체 매치 수 (페이지 크기 아님)
echo $result->took;       // ms
$result->unknown;         // 인덱스에 비슷한 것도 없는 단어들
foreach ($result as $hit) {
    echo $hit->id, $hit->score, $hit->document['title'], $hit->highlights['title'];
}

// 5) 유지보수 (크론)
if ($engine->stats()['needsOptimize']) {
    $engine->optimize();
}
```

### 6.4 전체 공개 API (`SearchEngine` 13개 메서드가 전부)

| 메서드 | 설명 |
|---|---|
| `open($dir, ?$schema)` | 인덱스 열기 (static, 없으면 생성) |
| `put($id, $doc)` | 1건 추가/교체 (멱등) |
| `insert($doc)` | ID 자동 생성 후 반환 |
| `putMany($iterable)` | 배치 — 전부 성공 or 전부 미반영 |
| `delete($id)` | 논리 삭제 |
| `clear()` | 인덱스 비우기 |
| `search(...)` | 검색 |
| `get($id)` / `has($id)` / `count()` | 문서 조회 / 존재 확인 / 개수 |
| `schema()` / `directory()` / `stats()` | 메타 정보 |
| `optimize()` | 세그먼트 병합 + 삭제 문서 정리 |

`close()`는 **없다.** 요청이 열고, 읽고, 끝난다.

### 6.5 꼭 지켜야 할 것 (SECURITY.md)

1. **인덱스 디렉터리를 웹루트 바깥에 둘 것.** 원문이 그대로 들어있어서 HTTP로 노출되면 전체 데이터가 다운로드된다. 라이브러리가 강제할 수 없는 부분.
2. **요청 입력으로 바로 필터를 만들지 말 것.** `Filter::fromArray()`가 구조는 검증하지만 "이 사용자가 이 필드를 필터할 권한이 있는가"는 모른다. 화이트리스트 필수.
3. 비교는 **엄격(strict)**: `'42' !== 42`. 요청값은 캐스팅해서 넣을 것.
4. 하이라이팅은 기본 HTML 이스케이프됨. `->raw()`는 뒤에서 직접 이스케이프할 때만.

---

## 7. 자주 묻는 것들 정리

### Q. 플러그인? 스킬? MCP?
**전부 아니다. 순수 PHP 라이브러리(Composer 패키지)다.**

| | php-fts |
|---|---|
| Claude/Cursor 플러그인? | ❌ |
| Claude Skill? | ❌ (`CLAUDE.md`가 있는 건 이 포크에 내가 넣은 페르소나 가이드일 뿐) |
| MCP 서버? | ❌ (**하지만 감싸서 만들 수는 있다** — 아래 참고) |
| 실체 | PHP 8.1+ 라이브러리. `composer require ols/php-fts` |

### Q. API 토큰이 필요해?
**전혀 필요 없다.** 네트워크 통신 자체가 0이다.
- 외부 서비스 호출 없음 → 토큰/키/계정 없음
- `composer.json`의 `require`는 `{"php": ">=8.1"}` 뿐
- 그래서 **오프라인/폐쇄망/에어갭 환경에서도 100% 동작**하고, 데이터가 밖으로 안 나간다

### Q. 왜 GitHub에서 유명할까?
1. **Hacker News "Show HN"에 올라가서 터졌다** → 원본 저장소 141★
2. **해결하는 고통이 아주 흔하다** — 공유 호스팅/소규모 VPS에서 검색 붙이기. 기존 선택지는 전부 뭔가를 요구했다:

   | | php-fts | TNTSearch | Scout DB | Meilisearch |
   |---|---|---|---|---|
   | Composer 의존성 | **없음** | `predis/predis` | Laravel | HTTP 클라이언트 |
   | PHP 확장 | **없음** | `pdo`, `mbstring` | `pdo` | `curl`, `json` |
   | 띄울 서비스 | **없음** | 없음 | DB | 서버/SaaS |
   | 공유호스팅 | **가능** | `pdo_sqlite` 있으면 | 가능 | 불가 |
   | 패싯/정확한 total | **있음** | 없음 | 없음 | 있음 |

3. **"의존성 0"이라는 극단적 제약이 이야깃거리다.** 확장 없이 유니코드 폴딩, 확장 없이 바이너리 포맷, 확장 없이 BM25F.
4. **문서와 정직함의 수준이 비정상적으로 높다.**
   - README에 **"Limits"** 섹션을 직접 씀 — "한국어는 오타 교정 안 됨", "티베트어는 아예 색인 안 됨", "`shoe`는 `snowshoes`를 못 찾음(의도적)"
   - CHANGELOG 41KB에 **"틀렸던 수치까지" 기록**
   - "Elasticsearch 쓸 수 있으면 그거 쓰세요"라고 README 초반에 써 놓음
   - 온디스크 바이너리 포맷 **명세서**를 따로 씀
5. **CI가 진지하다.** PHP 8.1~8.5 × Ubuntu/Windows/macOS, 그리고 `ext-intl`/`ext-posix`를 **일부러 끄고 돌리는 전용 잡**까지.
6. 코드 17,300줄 vs 테스트 14,400줄 / 740 케이스.

> 정리: **"기능이 신기해서"가 아니라 "엔지니어링 태도가 신뢰를 줘서" 유명해진 케이스.**

### Q. 로컬 에이전트 구축에 도움이 될까?
**"로컬 RAG의 키워드 검색 레이어"로는 아주 좋다. 단, 스택 궁합을 봐야 한다.**

👍 좋은 점
- **완전 오프라인.** 임베딩 API도, 벡터 DB 서버도 필요 없음 → 데이터가 로컬 밖으로 안 나감
- 인덱스가 그냥 **파일 폴더** → 통째로 복사/백업/배포 가능
- **BM25 키워드 검색**은 RAG에서 벡터 검색과 상호보완(하이브리드). 제품 코드, 에러 코드, 고유명사처럼 임베딩이 약한 쿼리에 특히 강함
- 콜드 오픈 9ms → CLI 도구로 매번 새 프로세스를 띄워도 부담 없음

👎 걸리는 점
- **PHP다.** 에이전트 생태계(LangChain, LlamaIndex, Claude Agent SDK)는 Python/TS 중심
- 벡터 검색이 아니라 **의미 기반 검색은 못 한다** ("환불 방법" ↔ "반품 절차" 매칭 불가)
- **한국어 형태소 분석 없음** → 한국어 문서 RAG 품질이 떨어짐
- 수백만 문서 / 실시간 동시 쓰기 부적합

✅ 현실적인 통합 방법
```
Claude Code / 에이전트
        ↓ MCP (stdio)
 Node/TS MCP 서버 (search_docs 툴)
        ↓ exec
  php cli.php search "질의"
        ↓
     php-fts
```
`SearchEngine::search()` 하나만 감싸면 되니까 MCP 서버는 100줄 안쪽으로 나온다.
**이게 우리가 가장 먼저 만들어 볼 만한 것이다.**

### Q. 우리가 React나 PHP로 만들 수 있어?

**세 가지 층으로 나눠 봐야 한다.**

| 층 | 가능? | 난이도 |
|---|---|---|
| ① php-fts를 **가져다 쓰기** (PHP 백엔드 + React 프론트) | ✅ 당장 가능 | ⭐ |
| ② php-fts를 **확장하기** (한국어 애드온, MCP 래퍼, 프레임워크 연동) | ✅ 가능 | ⭐⭐⭐ |
| ③ 같은 엔진을 **처음부터 다시 만들기** | ⚠️ 가능하지만 비추 | ⭐⭐⭐⭐⭐ |

**① 이게 정답 — 아키텍처:**
```
React (Vite) ──fetch──> PHP API (search.php) ──> php-fts (파일 인덱스)
   UI/패싯/무한스크롤        1 요청 = 1 search() 호출
```
`demo/index.html` + `demo/search.php`가 이미 이 구조의 바닐라 버전이다.
React로 갈아끼우는 건 UI 작업일 뿐. 응답 JSON에 `total`, `hits`, `facets`가 이미 다 들어있다.

**③을 React(JS)로 재구현하는 건 왜 비추인가:**
- 브라우저에서 돌리면 인덱스 70MB를 클라이언트가 전부 다운받아야 함
- JS 생태계엔 이미 성숙한 대안이 있음 — MiniSearch, Lunr.js, Orama, FlexSearch
- php-fts의 진짜 가치는 "알고리즘"이 아니라 **"3년치 엣지케이스 + 740개 테스트"**. 그걸 다시 만드는 건 낭비

**결론: 엔진은 php-fts를 쓰고, 우리는 그 위에 한국어 레이어 + UI + 통합을 만든다.**

---

## 8. 수익화 아이디어 (우선순위 순)

> 공통 전제: 라이선스가 **MIT**라서 상업적 이용/수정/재배포/유료 판매 전부 합법.
> 저작권 고지만 유지하면 된다. (그리고 원작자 크레딧은 예의상 꼭 남기자 💖)

### 🥇 Tier 1 — 가장 현실적 (지금 당장)

**1. 한국어 애드온 `php-fts-korean` (오픈코어 모델)**

README가 **스스로 인정한 구멍**을 메우는 것. 가장 명확한 가치 제안.
- 무료(OSS): 한/영 오타 변환(`rkwnr` → `가죽`), 초성 검색(`ㄱㅈ` → `가죽`), 자모 분해 유사도
- 유료 PRO (₩99,000~₩490,000 1회 or 연 구독): 형태소 분석기 연동(은전한닢/Kiwi), 동의어 사전, 불용어 셋, 한국어 랭킹 튜닝 프리셋
- 왜 되나: 한국어 검색 품질은 **직접 체감되는 차이**다. "신발이"로 검색해서 "신발"이 안 나오면 바로 안다.
- 수익: 라이선스 판매 + GitHub Sponsors

**2. PHP 쇼핑몰/CMS 플러그인 판매**

한국 시장은 **그누보드5 / 영카트 / 카페24 / 고도몰 / 워드프레스**가 장악. 기본 검색은 `LIKE '%키워드%'`라 느리고 오타도 못 잡는다.
- 제품: "검색 강화 플러그인" — 오타 교정, 패싯 필터, 자동완성, 검색어 통계 대시보드
- 가격: 사이트당 ₩150,000~₩500,000 (평생) 또는 연 ₩100,000
- 채널: 그누보드 자료실, CodeCanyon($29~$99), WordPress.org(무료) + 프리미엄 애드온
- 왜 되나: **"서버 추가 없이"**가 공유 호스팅 쓰는 한국 중소 쇼핑몰에 그대로 꽂힌다

**3. Laravel Scout 드라이버 + 프리미엄 패키지**

```php
// config/scout.php
'driver' => 'php-fts'
```
- 무료 드라이버로 진입 → 유료 애드온(한국어, 관리 UI, 동의어 관리, 검색 분석)
- Laravel 커뮤니티는 패키지 구매에 익숙 (Spatie 모델)
- 가격: 프로젝트당 $49 / 무제한 $199

### 🥈 Tier 2 — 중기 (1~3개월)

**4. "검색 붙여드립니다" 구축·컨설팅 서비스**
- 패키지 A (₩300만~): 기존 쇼핑몰/사이트에 검색 이식 + 인덱싱 크론 + 대시보드
- 패키지 B (월 ₩30만~): 유지보수, 랭킹 튜닝, 검색어 리포트
- 포트폴리오는 위의 오픈소스 애드온이 대신해 준다 → **OSS가 영업 도구**

**5. 셀프호스팅 검색 대시보드 (SaaS 아닌 "라이선스 제품")**
- React 관리자 UI: 인덱스 상태, 검색어 랭킹, 0건 검색어(= `$result->unknown` 활용!), 동의어 편집, 수동 랭킹 부스트
- **0건 검색어 리포트**는 쇼핑몰 MD가 진짜 돈 내는 기능. "고객이 찾았는데 우리가 안 판 것" 목록이니까.
- 가격: ₩500,000 (셀프호스팅 영구 라이선스)

**6. MCP 서버 제품 — "로컬 문서 검색 MCP"**
- Claude Code / Cursor에 붙이는 **완전 로컬** 문서 검색 MCP
- 셀링 포인트: 임베딩 API 비용 0원, 데이터 외부 유출 0, 폐쇄망 가능
- 무료 배포로 인지도 → PRO(팀 인덱스 동기화, 한국어, 권한 분리) 유료

### 🥉 Tier 3 — 장기 / 하이 리스크

**7. 정적 사이트 검색 SaaS**
- Algolia DocSearch의 저가 대안. 크롤링 + 인덱싱 + 검색 위젯
- 월 $9~$29. 다만 **php-fts의 최대 강점인 "서버 불필요"를 스스로 버리는** 모델이라 차별화가 약함

**8. 템플릿/스타터킷 판매**
- "React + PHP 검색 스타터킷" (Gumroad $39~$79)
- 쇼핑몰 검색 페이지, 문서 검색, 사내 위키 검색 — 3종 템플릿

### 💡 전략 요약

```
1단계: 한국어 애드온 OSS 공개  →  신뢰 + 유입
2단계: 그누보드/워드프레스 플러그인 유료화  →  첫 현금
3단계: 구축 서비스 + 대시보드 라이선스  →  객단가 상승
4단계: MCP/스타터킷으로 개발자 시장 확장
```

**핵심 인사이트: php-fts 자체를 파는 게 아니라, php-fts가 "못 하는 것"(한국어)과
"안 하는 것"(UI/관리도구/통합)을 파는 것.** 엔진은 남의 것이지만 레이어는 내 것이다. 🔥

---

## 9. 이 프로젝트에서 배울 것 (돈 안 되어도 남는 것)

- **역색인 / BM25 / 세그먼트 / 포스팅 리스트** — 검색엔진의 진짜 내부 구조를 PHP로 읽을 수 있다 (C++ Lucene보다 훨씬 읽기 쉬움)
- **원자적 커밋 설계** — rename 하나로 트랜잭션 만들기. 다른 프로젝트에도 그대로 써먹는 패턴
- **제약이 설계를 낳는다** — "확장 없음"이라는 제약이 오히려 더 좋은 설계를 만들어낸 실물 사례
- **정직한 문서 쓰는 법** — Limits 섹션, 틀렸던 수치까지 남기는 CHANGELOG. 이게 신뢰를 만든다
- **테스트 문화** — 740개 테스트, 크로스 OS/버전 CI, 확장 끄고 돌리는 전용 잡

---

## 10. 다음에 할 만한 것 (제안)

- [ ] `composer install && composer test` 돌려서 740개 테스트 통과 확인
- [ ] `php demo/seed.php` → 데모 띄워서 패싯 동작 눈으로 보기
- [ ] **한국어 문서로 인덱싱해서 현재 품질 실측** (바이그램 매칭이 실제로 얼마나 쓸만한지)
- [ ] `src/Analysis/Script.php` + `Analyzer.php` 읽고 한글 토큰화 지점 찾기
- [ ] 한/영 오타 변환 + 초성 검색 프로토타입 (엔진 수정 없이 질의 전처리로 가능!)
- [ ] MCP 래퍼 100줄 프로토타입
- [ ] React 데모 UI로 교체

---

*정리: 카리나 💖 · Claude Code*
