# Raw Data Source Inventory

## 범위
이 문서는 현재 프로젝트에서 사용하는 `raw 원천 데이터`만 정리한다.

- 포함:
  - 외부 upstream API, RSS/XML/ICS feed, CSV/GeoJSON feed
  - 리포지토리에 들어있는 raw 파일
  - 브라우저 직접 호출과 `/api/*` 프록시 경유 호출 둘 다
- 제외:
  - 내부 파생 데이터 (`risk-scores`, `climate-anomalies`, `worldpop-exposure`, `temporal-baseline` 등)
  - 내부 운영/진단 엔드포인트 (`service-status`, `cache-telemetry` 등)
  - 정적 큐레이션 TS 상수 (`src/config/*.ts` 대부분)

## 읽는 법
- `앱 접근 경로`: 현재 코드베이스에서 실제로 호출하는 경로
- `원천`: 실제 upstream URL 또는 원본 파일
- `포맷`: 원천 포맷 기준
- `현재 앱 반환`: 프록시가 그대로 전달하는지, 일부 필드만 축약하는지
- `코드 위치`: 실제 접속 코드가 있는 파일
- `접속 흐름`: 클라이언트 서비스에서 upstream까지 이어지는 호출 경로
- `운영 메모`: 인증, 환경변수, 캐시 TTL, fallback, rate limit 관련 메모

## 코드 추적 기본 패턴

### 브라우저 직접 호출
```text
src/services/*.ts -> 외부 API
```

### 프록시 경유 호출
```text
src/services/*.ts 또는 src/config/*.ts -> /api/*.js -> 외부 upstream
```

### 리포지토리 파일 직접 사용
```text
src/config/*.ts 또는 fetch('/data/...') -> public/data/* 또는 data/*
```

## 1. 로컬 raw 파일

### 1.1 감마 조사시설 원본
- 앱 접근 경로: 리포지토리 파일 직접 참조
- 원천: `data/gamma-irradiators-raw.json`
- 포맷: JSON
- 성격:
  - 감마 조사시설 원본 데이터
  - 가공본은 `data/gamma-irradiators.json`
- 비고:
  - 지도 렌더링에는 보통 가공본이 쓰이고, 이 파일은 원본 보존용에 가깝다.

### 1.2 국가 경계 GeoJSON
- 앱 접근 경로: `GET /data/countries.geojson`
- 원천: `public/data/countries.geojson`
- 포맷: GeoJSON `FeatureCollection`
- 주요 필드:
  - `features[]`
  - `geometry`
  - `properties`

## 2. RSS / XML / 콘텐츠 raw 소스

### 2.1 일반 RSS 프록시
- 앱 접근 경로: `GET /api/rss-proxy?url=<encoded feed url>`
- 원천:
  - `src/config/feeds.ts`
  - `src/config/variants/tech.ts`
  - 허용 도메인은 `api/rss-proxy.js`의 `ALLOWED_DOMAINS`
- 대표 원천:
  - BBC: `https://feeds.bbci.co.uk/news/world/rss.xml`
  - NPR: `https://feeds.npr.org/1001/rss.xml`
  - Guardian: `https://www.theguardian.com/world/rss`
  - Google News RSS: `https://news.google.com/rss/search?...`
  - Reuters/Politico/Defense/Tech/Think Tank 계열 다수
- 포맷: RSS/XML
- 접속 방법:
```text
GET /api/rss-proxy?url=https%3A%2F%2Ffeeds.bbci.co.uk%2Fnews%2Fworld%2Frss.xml
```
- 현재 앱 반환:
  - upstream XML을 거의 그대로 반환
  - `Content-Type: application/xml`
- 코드 위치:
  - feed 정의: `src/config/feeds.ts`, `src/config/variants/tech.ts`
  - 프록시: `api/rss-proxy.js`
  - RSS 파싱/소비: `src/services/rss.ts`
- 접속 흐름:
```text
src/config/feeds.ts -> /api/rss-proxy?url=... -> 실제 RSS/XML feed
src/config/variants/tech.ts -> /api/rss-proxy?url=... -> 실제 RSS/XML feed
```
- 제약:
  - allowlist 외 도메인은 403
  - Google News는 timeout 20초, 나머지는 12초
- 운영 메모:
  - 인증: 없음
  - 환경변수: 직접 필요 없음
  - 캐시 TTL: 300초
  - fallback: 별도 stale fallback 없음
  - rate limit: 프록시 자체 limiter 없음, upstream timeout만 적용

### 2.2 YouTube live 상태
- 앱 접근 경로: `GET /api/youtube/live?channel=SkyNews`
- 원천: `https://www.youtube.com/@<channel>/live`
- 포맷: HTML
- 수집 방식:
  - 채널 `/live` 페이지 HTML을 가져옴
  - `"videoId":"..."`, `"isLive": true` 정규식 추출
- 현재 앱 반환:
```json
{ "videoId": "xxxxxxxxxxx", "isLive": true }
```
또는
```json
{ "videoId": null, "isLive": false }
```
- 코드 위치:
  - 서비스: `src/services/live-news.ts`
  - UI 사용: `src/components/LiveNewsPanel.ts`
  - 프록시: `api/youtube/live.js`
- 접속 흐름:
```text
src/services/live-news.ts -> /api/youtube/live?channel=... -> https://www.youtube.com/@channel/live
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 300초
  - fallback: 실패 시 `{ videoId: null }`

### 2.3 Hacker News
- 앱 접근 경로: `GET /api/hackernews?type=top&limit=20`
- 원천:
  - 목록: `https://hacker-news.firebaseio.com/v0/topstories.json`
  - 상세: `https://hacker-news.firebaseio.com/v0/item/<id>.json`
- 포맷: JSON
- 파라미터:
  - `type`: `top|new|best|ask|show|job`
  - `limit`: 최대 60
- 현재 앱 반환:
```json
{
  "type": "top",
  "stories": [ { "...HN item fields..." } ],
  "total": 20,
  "timestamp": "2026-03-13T..."
}
```
- 비고:
  - story item은 HN Firebase raw 구조를 거의 그대로 유지한다.
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 서비스: `src/services/hackernews.ts`
  - 프록시: `api/hackernews.js`
- 접속 흐름:
```text
src/services/hackernews.ts -> /api/hackernews?type=...&limit=... -> hacker-news.firebaseio.com
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 300초
  - fallback: story 단건 fetch 실패는 부분 누락 허용, 전체 실패는 500

### 2.4 GitHub Trending
- 앱 접근 경로: `GET /api/github-trending?language=typescript&since=daily`
- 원천:
  - 1차: `https://api.gitterapp.com/repositories?...`
  - fallback: `https://gh-trending-api.herokuapp.com/repositories/<language>?since=<since>`
- 포맷: JSON
- 파라미터:
  - `language`
  - `since`: `daily|weekly|monthly`
  - `spoken_language`
- 현재 앱 반환:
  - upstream JSON을 거의 그대로 반환
- 비고:
  - 공식 GitHub API가 아니라 비공식 trending scraper 계열이다.
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 서비스: `src/services/github-trending.ts`
  - 프록시: `api/github-trending.js`
- 접속 흐름:
```text
src/services/github-trending.ts -> /api/github-trending?... -> api.gitterapp.com
                                                 fallback -> gh-trending-api.herokuapp.com
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 1800초
  - fallback: 1차 실패 시 Herokuapp API로 재시도

### 2.5 arXiv
- 앱 접근 경로: `GET /api/arxiv?category=cs.AI&max_results=10&sortBy=submittedDate`
- 원천: `https://export.arxiv.org/api/query`
- 포맷: Atom XML
- 파라미터:
  - `category`: `cs.AI`, `cs.LG`, `cs.CL` 등
  - `max_results`
  - `sortBy`: `submittedDate|lastUpdatedDate|relevance`
- 실제 upstream 예:
```text
https://export.arxiv.org/api/query?search_query=cat:cs.AI&start=0&max_results=10&sortBy=submittedDate&sortOrder=descending
```
- 현재 앱 반환:
  - XML을 그대로 반환
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 서비스: `src/services/arxiv.ts`
  - 프록시: `api/arxiv.js`
- 접속 흐름:
```text
src/services/arxiv.ts -> /api/arxiv?category=... -> export.arxiv.org/api/query
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 3600초
  - fallback: 없음

### 2.6 Tech events
- 앱 접근 경로: `GET /api/tech-events?...`
- raw 원천:
  - ICS: `https://www.techmeme.com/newsy_events.ics`
  - RSS: `https://dev.events/rss.xml`
- 포맷:
  - Techmeme: ICS calendar
  - dev.events: RSS/XML
- 비고:
  - 이 엔드포인트 안에는 `CURATED_EVENTS`도 함께 들어 있으므로, strict raw 기준에서는 curated 항목은 raw가 아니다.
  - raw만 보려면 위 두 upstream를 직접 보는 것이 맞다.
- 코드 위치:
  - 프록시/집계: `api/tech-events.js`
  - UI 사용: `src/App.ts`, `src/components/TechEventsPanel.ts`
- 접속 흐름:
```text
src/App.ts / src/components/TechEventsPanel.ts -> /api/tech-events -> Techmeme ICS + dev.events RSS
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시/유지 전략: raw upstream 외에 curated events가 코드에 내장됨
  - fallback: curated events가 사실상 일부 fallback 역할

### 2.7 FwdStart
- 앱 접근 경로: `GET /api/fwdstart`
- 원천:
  - archive: `https://www.fwdstart.me/archive`
  - 개별 글: `https://www.fwdstart.me/...`
- 포맷:
  - upstream는 HTML
  - 앱 엔드포인트는 RSS/XML로 재구성
- 비고:
  - raw 원천은 HTML 페이지다.
  - 앱 반환은 raw passthrough가 아니라 scrape 후 재구성 결과다.
- 코드 위치:
  - feed 등록: `src/config/feeds.ts`
  - 프록시: `api/fwdstart.js`
- 접속 흐름:
```text
src/config/feeds.ts -> /api/fwdstart -> fwdstart.me archive HTML -> article HTML
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 코드상 별도 캐시 없음
  - fallback: 없음

## 3. 지정학 / 분쟁 / 인도주의 raw 소스

### 3.1 ACLED protests
- 앱 접근 경로: `GET /api/acled`
- 원천: `https://acleddata.com/api/acled/read`
- 인증: `ACLED_ACCESS_TOKEN` 필요
- 포맷: JSON
- 실제 upstream 파라미터:
  - `event_type=Protests`
  - `event_date=<start>|<end>`
  - `event_date_where=BETWEEN`
  - `limit=500`
  - `_format=json`
- 현재 앱 반환:
```json
{
  "success": true,
  "count": 123,
  "data": [
    {
      "event_id_cnty": "...",
      "event_date": "...",
      "event_type": "Protests",
      "sub_event_type": "...",
      "actor1": "...",
      "actor2": "...",
      "country": "...",
      "admin1": "...",
      "location": "...",
      "latitude": ...,
      "longitude": ...,
      "fatalities": ...,
      "notes": "...",
      "source": "...",
      "tags": "..."
    }
  ],
  "cached_at": "..."
}
```
- 비고:
  - raw ACLED 응답은 더 많은 필드가 있지만, 앱은 위 필드 위주로 sanitize해서 반환한다.
- 코드 위치:
  - 서비스: `src/services/protests.ts`
  - 프록시: `api/acled.js`
- 접속 흐름:
```text
src/services/protests.ts -> /api/acled -> acleddata.com/api/acled/read
```
- 운영 메모:
  - 인증: Bearer token
  - 환경변수: `ACLED_ACCESS_TOKEN`
  - 캐시 TTL: Redis 600초, 메모리 fallback 600초
  - fallback: stale memory fallback 있음
  - rate limit: 10 req/min per IP

### 3.2 ACLED conflict
- 앱 접근 경로: `GET /api/acled-conflict`
- 원천: `https://acleddata.com/api/acled/read`
- 인증: `ACLED_ACCESS_TOKEN`
- 포맷: JSON
- 실제 upstream 파라미터:
  - `event_type=Battles|Explosions/Remote violence|Violence against civilians`
  - 나머지는 protests와 동일
- 현재 앱 반환:
  - protests와 동일한 shape
  - `event_type`만 conflict 계열
- 코드 위치:
  - 서비스: `src/services/conflicts.ts`
  - 프록시: `api/acled-conflict.js`
- 접속 흐름:
```text
src/services/conflicts.ts -> /api/acled-conflict -> acleddata.com/api/acled/read
```
- 운영 메모:
  - 인증: Bearer token
  - 환경변수: `ACLED_ACCESS_TOKEN`
  - 캐시 TTL: Redis 600초, 메모리 fallback 600초
  - fallback: stale memory fallback 있음
  - rate limit: 10 req/min per IP

### 3.3 GDELT Geo
- 앱 접근 경로: `GET /api/gdelt-geo?query=protest&format=geojson&maxrecords=250&timespan=7d`
- 원천: `https://api.gdeltproject.org/api/v2/geo/geo`
- 포맷:
  - `geojson`
  - `json`
  - `csv`
- 파라미터:
  - `query`
  - `format`: `geojson|json|csv`
  - `maxrecords`: 1~500
  - `timespan`: `1d|7d|14d|30d|60d|90d`
- 현재 앱 반환:
  - upstream payload를 그대로 반환
  - CSV 요청 시 `text/csv`
- 코드 위치:
  - 서비스: `src/services/protests.ts`
  - 프록시: `api/gdelt-geo.js`
- 접속 흐름:
```text
src/services/protests.ts -> /api/gdelt-geo?... -> api.gdeltproject.org/api/v2/geo/geo
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 300초
  - fallback: 없음
  - 제약: `maxrecords <= 500`, 허용 timespan 제한

### 3.4 GDELT Doc
- 앱 접근 경로: `GET /api/gdelt-doc?query=iran&maxrecords=10&timespan=72h`
- 원천: `https://api.gdeltproject.org/api/v2/doc/doc`
- 포맷: JSON
- upstream 고정 파라미터:
  - `mode=artlist`
  - `format=json`
  - `sort=date`
- 현재 앱 반환:
```json
{
  "articles": [
    {
      "title": "...",
      "url": "...",
      "source": "domain",
      "date": "YYYYMMDDHHMMSS",
      "image": "...",
      "language": "English",
      "tone": -1.23
    }
  ],
  "query": "iran"
}
```
- 코드 위치:
  - 서비스: `src/services/gdelt-intel.ts`
  - 프록시: `api/gdelt-doc.js`
- 접속 흐름:
```text
src/services/gdelt-intel.ts -> /api/gdelt-doc?... -> api.gdeltproject.org/api/v2/doc/doc
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 300초
  - fallback: 없음
  - 제약: `maxrecords <= 20`

### 3.5 UCDP PRIO conflict dataset
- 앱 접근 경로: `GET /api/ucdp`
- 원천: `https://ucdpapi.pcr.uu.se/api/ucdpprioconflict/24.1`
- 포맷: JSON
- 접속 방법:
```text
GET https://ucdpapi.pcr.uu.se/api/ucdpprioconflict/24.1?pagesize=100&page=0
```
- raw 주요 필드:
  - `conflict_id`
  - `location`
  - `side_a`
  - `side_b`
  - `year`
  - `intensity_level`
  - `type_of_conflict`
  - `start_date`
  - `region`
- 현재 앱 반환:
  - 국가별 최신/최고 강도 1건으로 축약
- 코드 위치:
  - 서비스: `src/services/ucdp.ts`
  - 프록시: `api/ucdp.js`
- 접속 흐름:
```text
src/services/ucdp.ts -> /api/ucdp -> ucdpapi.pcr.uu.se/api/ucdpprioconflict/...
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: Redis 24시간, 메모리 fallback 24시간
  - fallback: stale memory fallback 있음

### 3.6 UCDP GED events
- 앱 접근 경로: `GET /api/ucdp-events`
- 원천: `https://ucdpapi.pcr.uu.se/api/gedevents/<version>`
- 포맷: JSON
- 특징:
  - 코드가 API version 후보를 자동 탐색한다. 예: `25.1`, `24.1`
  - `pagesize=1000`
  - 최신 데이터 기준 trailing 365일만 유지
- raw 주요 필드:
  - `id`
  - `date_start`
  - `date_end`
  - `latitude`
  - `longitude`
  - `country`
  - `side_a`
  - `side_b`
  - `best`, `low`, `high`
  - `type_of_violence`
  - `source_original`
- 현재 앱 반환:
  - `best/low/high`는 `deaths_best/deaths_low/deaths_high`로 매핑
  - `type_of_violence`는 `state-based|non-state|one-sided` 문자열로 변환
- 코드 위치:
  - 서비스: `src/services/ucdp-events.ts`
  - 프록시: `api/ucdp-events.js`
- 접속 흐름:
```text
src/services/ucdp-events.ts -> /api/ucdp-events -> ucdpapi.pcr.uu.se/api/gedevents/<version>
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: Redis 6시간, 메모리 fallback 6시간
  - fallback: stale memory fallback 있음
  - rate limit: 15 req/min per IP

### 3.7 HDX HAPI conflict events
- 앱 접근 경로: `GET /api/hapi`
- 원천: `https://hapi.humdata.org/api/v2/coordination-context/conflict-events`
- 포맷: JSON
- 실제 upstream 예:
```text
GET https://hapi.humdata.org/api/v2/coordination-context/conflict-events?output_format=json&limit=1000&offset=0&app_identifier=<base64>
```
- raw 주요 필드:
  - `location_code`
  - `location_name`
  - `reference_period_start`
  - `event_type`
  - `events`
  - `fatalities`
- 현재 앱 반환:
  - 국가별 최신 월 기준 합산 결과
  - raw 그대로는 아님
- 코드 위치:
  - 서비스: `src/services/hapi.ts`
  - 프록시: `api/hapi.js`
- 접속 흐름:
```text
src/services/hapi.ts -> /api/hapi -> hapi.humdata.org/api/v2/coordination-context/conflict-events
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: Redis 6시간, 메모리 fallback 6시간
  - fallback: stale memory fallback 있음

### 3.8 UNHCR population
- 앱 접근 경로: `GET /api/unhcr-population`
- 원천: `https://api.unhcr.org/population/v1/population/`
- 포맷: JSON
- 실제 upstream 예:
```text
GET https://api.unhcr.org/population/v1/population/?year=2026&limit=10000&page=1
```
- raw 주요 필드:
  - `coo_iso`, `coo_name` origin country
  - `coa_iso`, `coa_name` asylum country
  - `refugees`
  - `asylum_seekers`
  - `idps`
  - `stateless`
- 현재 앱 반환:
  - `globalTotals`
  - `countries[]`
  - `topFlows[]`
  - 즉 raw aggregate를 재가공한 구조
- 코드 위치:
  - 서비스: `src/services/unhcr.ts`
  - 프록시: `api/unhcr-population.js`
- 접속 흐름:
```text
src/services/unhcr.ts -> /api/unhcr-population -> api.unhcr.org/population/v1/population/
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: Redis 24시간, 메모리 fallback 24시간
  - fallback: stale memory fallback 있음
  - rate limit: 20 req/min per IP

### 3.9 World Bank indicators
- 앱 접근 경로:
  - 목록: `GET /api/worldbank?action=indicators`
  - 데이터: `GET /api/worldbank?indicator=IT.NET.USER.ZS&countries=KR,US,JP&years=5`
- 원천: `https://api.worldbank.org/v2/country/<countries>/indicator/<indicator>?format=json...`
- 포맷: JSON array
- raw 응답 특징:
  - World Bank는 `[metadata, records]` 2원소 배열을 반환
- raw record 주요 필드:
  - `countryiso3code`
  - `country.value`
  - `indicator.value`
  - `date`
  - `value`
- 현재 앱 반환:
  - `byCountry`
  - `latestByCountry`
  - `timeSeries`
  - 즉 raw를 frontend 친화형으로 재배열
- 코드 위치:
  - 서비스: `src/services/worldbank.ts`
  - 프록시: `api/worldbank.js`
- 접속 흐름:
```text
src/services/worldbank.ts -> /api/worldbank?... -> api.worldbank.org/v2/...
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: HTTP 3600초, 클라이언트 메모리 캐시 1시간
  - fallback: 빈 결과 객체 반환 가능
  - 비고: Edge에서 403 이슈가 있어 Node serverless 사용

### 3.10 PizzINT raw
- 앱 접근 경로:
  - `GET /api/pizzint/dashboard-data`
  - `GET /api/pizzint/gdelt/batch?...`
- 원천:
  - `https://www.pizzint.watch/api/dashboard-data`
  - `https://www.pizzint.watch/api/gdelt/batch`
- 포맷: JSON
- 비고:
  - 앱 프록시는 raw를 거의 그대로 전달한다.
  - 클라이언트 `src/services/pizzint.ts`에서 후처리한다.
- 코드 위치:
  - 서비스: `src/services/pizzint.ts`
  - 프록시: `api/pizzint/dashboard-data.js`, `api/pizzint/gdelt/batch.js`
- 접속 흐름:
```text
src/services/pizzint.ts -> /api/pizzint/dashboard-data -> pizzint.watch/api/dashboard-data
src/services/pizzint.ts -> /api/pizzint/gdelt/batch -> pizzint.watch/api/gdelt/batch
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: dashboard 60초, gdelt batch 300초
  - fallback: 없음

## 4. 항공 / 해상 / 군사 / 재난 raw 소스

### 4.1 OpenSky states
- 앱 접근 경로: `GET /api/opensky?lamin=33&lomin=124&lamax=39&lomax=132`
- 원천: `https://opensky-network.org/api/states/all`
- 포맷: JSON
- raw 응답 주요 구조:
```json
{
  "time": 1710000000,
  "states": [
    [
      "icao24",
      "callsign",
      "origin_country",
      "time_position",
      "last_contact",
      "longitude",
      "latitude",
      "baro_altitude",
      "on_ground",
      "velocity",
      "true_track",
      "vertical_rate",
      "sensors",
      "geo_altitude",
      "squawk",
      "spi",
      "position_source",
      "category"
    ]
  ]
}
```
- 비고:
  - 앱 프록시는 raw를 거의 그대로 반환한다.
- 코드 위치:
  - 서비스: `src/services/military-flights.ts`
  - 프록시: `api/opensky.js`
  - 서버 집계: `api/theater-posture.js`
- 접속 흐름:
```text
src/services/military-flights.ts -> /api/opensky -> opensky-network.org/api/states/all
api/theater-posture.js -> Railway relay /opensky -> OpenSky 계열 raw flight state
```
- 운영 메모:
  - 인증: 현재 `api/opensky.js`는 무인증 호출
  - 환경변수: `WS_RELAY_URL`은 별도 relay 경로에서 사용
  - 캐시 TTL: 30초
  - fallback: 429/오류 시 오류 payload 반환
  - 비고: cloud IP 차단 이슈 때문에 relay 경로가 병행 사용됨

### 4.2 Wingbits flights / details
- 앱 접근 경로:
  - `GET /api/wingbits/flights?la=...&lo=...&w=500&h=500&unit=nm`
  - `GET /api/wingbits/details/<icao24>`
  - `POST /api/wingbits/details/batch`
- 원천: `https://customer-api.wingbits.com/v1/...`
- 인증: `WINGBITS_API_KEY`
- 포맷: JSON
- raw 엔드포인트:
  - `GET /v1/flights?by=box&la=<lat>&lo=<lon>&w=<width>&h=<height>&unit=nm`
  - `GET /v1/flights/details/<icao24>`
- 현재 앱 반환:
  - upstream JSON 거의 그대로
- 코드 위치:
  - 서비스: `src/services/wingbits.ts`
  - 프록시: `api/wingbits/[[...path]].js`
- 접속 흐름:
```text
src/services/wingbits.ts -> /api/wingbits/flights -> customer-api.wingbits.com/v1/flights
src/services/wingbits.ts -> /api/wingbits/details/:icao24 -> customer-api.wingbits.com/v1/flights/details/:icao24
```
- 운영 메모:
  - 인증: API key
  - 환경변수: `WINGBITS_API_KEY`
  - 캐시 TTL: flights 30초, details 24시간
  - fallback: 미설정 시 `{ configured: false }`

### 4.3 AIS snapshot
- 앱 접근 경로: `GET /api/ais-snapshot?candidates=true`
- 원천:
  - `WS_RELAY_URL` 기반 relay의 `/ais/snapshot`
- 포맷: JSON
- 현재 앱이 기대하는 raw shape:
  - `status` object
  - `disruptions[]`
  - `density[]`
- 비고:
  - 실제 relay upstream는 이 저장소 밖에 있다.
  - 여기서는 앱이 기대하는 raw contract만 확인 가능하다.
- 코드 위치:
  - 서비스: `src/services/ais.ts`
  - 프록시: `api/ais-snapshot.js`
- 접속 흐름:
```text
src/services/ais.ts -> /api/ais-snapshot -> WS_RELAY_URL 기반 /ais/snapshot
```
- 운영 메모:
  - 인증: relay 측 정책에 따름
  - 환경변수: `WS_RELAY_URL`
  - 캐시 TTL: Redis 8초, 메모리 8초, stale memory 최대 60초
  - fallback: memory stale fallback 있음

### 4.4 FAA NAS status
- 앱 접근 경로: `GET /api/faa-status`
- 원천: `https://nasstatus.faa.gov/api/airport-status-information`
- 포맷: XML
- 현재 앱 반환:
  - XML passthrough
- 클라이언트 사용 포인트:
  - `Ground_Delay_List`
  - `Ground_Stop_List`
  - `Arrival_Departure_Delay_List`
  - `Airport_Closure_List`
- 코드 위치:
  - 서비스: `src/services/flights.ts`
  - 프록시: `api/faa-status.js`
- 접속 흐름:
```text
src/services/flights.ts -> /api/faa-status -> nasstatus.faa.gov/api/airport-status-information
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 프록시 무캐시, 클라이언트 메모리 캐시 5분
  - fallback: 클라이언트는 실패 시 빈 결과/시뮬레이션 로직 사용

### 4.5 NOAA / NWS active alerts
- 앱 접근 경로: 브라우저 직접 `https://api.weather.gov/alerts/active`
- 원천: 동일
- 포맷: GeoJSON 유사 JSON
- raw 주요 구조:
  - `features[]`
  - `features[].properties.event`
  - `severity`
  - `headline`
  - `description`
  - `areaDesc`
  - `onset`
  - `expires`
  - `geometry`
- 비고:
  - 프록시 없이 브라우저 직접 호출
- 코드 위치:
  - 서비스: `src/services/weather.ts`
  - 로드 트리거: `src/App.ts`
- 접속 흐름:
```text
src/App.ts -> src/services/weather.ts -> api.weather.gov/alerts/active
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 서비스 내부 circuit breaker 캐시 사용
  - fallback: 오류 시 빈 배열

### 4.6 NASA EONET
- 앱 접근 경로: 브라우저 직접 `https://eonet.gsfc.nasa.gov/api/v3/events?status=open&days=30`
- 원천: 동일
- 포맷: JSON
- raw 주요 필드:
  - `events[]`
  - `events[].categories[]`
  - `events[].sources[]`
  - `events[].geometry[]`
- 비고:
  - 앱은 EONET과 GDACS를 합쳐 자연재해 레이어를 만든다.
- 코드 위치:
  - 서비스: `src/services/eonet.ts`
  - 호출 트리거: `src/App.ts`
- 접속 흐름:
```text
src/App.ts -> src/services/eonet.ts -> eonet.gsfc.nasa.gov/api/v3/events
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 코드상 별도 서버 캐시 없음
  - fallback: 오류 시 빈 배열

### 4.7 GDACS
- 앱 접근 경로: 브라우저 직접 `https://www.gdacs.org/gdacsapi/api/events/geteventlist/MAP`
- 원천: 동일
- 포맷: JSON / GeoJSON 유사
- raw 주요 필드:
  - `features[]`
  - `properties.eventtype`
  - `properties.eventid`
  - `properties.name`
  - `properties.alertlevel`
  - `properties.country`
  - `properties.fromdate`
  - `properties.url.report`
  - `geometry.coordinates`
- 코드 위치:
  - 서비스: `src/services/gdacs.ts`
  - 호출 트리거: `src/services/eonet.ts`, `src/App.ts`
- 접속 흐름:
```text
src/services/eonet.ts -> src/services/gdacs.ts -> gdacs.org/gdacsapi/api/events/geteventlist/MAP
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 서비스 내부 circuit breaker 캐시
  - fallback: 오류 시 빈 배열

### 4.8 USGS earthquakes
- 앱 접근 경로: `GET /api/earthquakes`
- 원천: `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/4.5_day.geojson`
- 포맷: GeoJSON
- 현재 앱 반환:
  - raw passthrough
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 서비스: `src/services/earthquakes.ts`
  - 프록시: `api/earthquakes.js`
- 접속 흐름:
```text
src/services/earthquakes.ts -> /api/earthquakes -> earthquake.usgs.gov/.../4.5_day.geojson
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 300초
  - fallback: 없음

### 4.9 NASA FIRMS fires
- 앱 접근 경로:
  - `GET /api/firms-fires?days=1`
  - `GET /api/firms-fires?region=Ukraine&days=1`
- 원천: `https://firms.modaps.eosdis.nasa.gov/api/area/csv/<apiKey>/<source>/<bbox>/<days>`
- 인증: `NASA_FIRMS_API_KEY` 또는 `FIRMS_API_KEY`
- 포맷: CSV
- 현재 원천 source:
  - `VIIRS_SNPP_NRT`
- raw CSV 주요 컬럼:
  - `latitude`
  - `longitude`
  - `bright_ti4`
  - `scan`
  - `track`
  - `acq_date`
  - `acq_time`
  - `satellite`
  - `confidence`
  - `bright_ti5`
  - `frp`
  - `daynight`
- 현재 앱 반환:
  - region별 JSON 배열로 파싱 후 반환
- 코드 위치:
  - 서비스: `src/services/firms-satellite.ts`
  - 프록시: `api/firms-fires.js`
- 접속 흐름:
```text
src/services/firms-satellite.ts -> /api/firms-fires -> firms.modaps.eosdis.nasa.gov/api/area/csv/...
```
- 운영 메모:
  - 인증: API key
  - 환경변수: `NASA_FIRMS_API_KEY` 또는 `FIRMS_API_KEY`
  - 캐시 TTL: 600초
  - fallback: 부분 region 실패 허용, 전체 실패 시 500

## 5. 시장 / 거시 / 에너지 / 예측 raw 소스

### 5.1 FRED
- 앱 접근 경로: `GET /api/fred-data?series_id=DGS10`
- 원천: `https://api.stlouisfed.org/fred/series/observations`
- 인증: `FRED_API_KEY`
- 포맷: JSON
- 주요 파라미터:
  - `series_id`
  - `observation_start`
  - `observation_end`
  - `file_type=json`
- raw 주요 필드:
  - `observations[]`
  - `observations[].date`
  - `observations[].value`
- 코드 위치:
  - 서비스: `src/services/fred.ts`
  - 프록시: `api/fred-data.js`
- 접속 흐름:
```text
src/services/fred.ts -> /api/fred-data?series_id=... -> api.stlouisfed.org/fred/series/observations
```
- 운영 메모:
  - 인증: API key
  - 환경변수: `FRED_API_KEY`
  - 캐시 TTL: 3600초
  - fallback: 없음

### 5.2 Finnhub quotes
- 앱 접근 경로: `GET /api/finnhub?symbols=AAPL,MSFT,NVDA`
- 원천: `https://finnhub.io/api/v1/quote?symbol=<SYMBOL>&token=<APIKEY>`
- 인증: `FINNHUB_API_KEY`
- 포맷: JSON
- raw quote 필드:
  - `c` current
  - `d` change
  - `dp` percent change
  - `h` high
  - `l` low
  - `o` open
  - `pc` previous close
  - `t` timestamp
- 현재 앱 반환:
  - 각 symbol별 `{ price, change, changePercent, ... }`로 이름을 풀어준다.
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 프록시: `api/finnhub.js`
  - 소비: `src/App.ts` 및 market 관련 패널
- 접속 흐름:
```text
src/config/variants/base.ts -> /api/finnhub?symbols=... -> finnhub.io/api/v1/quote
```
- 운영 메모:
  - 인증: API key
  - 환경변수: `FINNHUB_API_KEY`
  - 캐시 TTL: 30초
  - fallback: symbol별 개별 오류 객체 반환 가능

### 5.3 Yahoo Finance chart
- 앱 접근 경로: `GET /api/yahoo-finance?symbol=QQQ`
- 원천: `https://query1.finance.yahoo.com/v8/finance/chart/<symbol>`
- 포맷: JSON
- 현재 앱 반환:
  - raw passthrough
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 프록시: `api/yahoo-finance.js`
  - 소비: `src/App.ts` 및 market 관련 패널
- 접속 흐름:
```text
src/config/variants/base.ts -> /api/yahoo-finance?symbol=... -> query1.finance.yahoo.com/v8/finance/chart/<symbol>
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 60초
  - fallback: 없음

### 5.4 CoinGecko simple price / markets
- 앱 접근 경로:
  - `GET /api/coingecko?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`
  - `GET /api/coingecko?endpoint=markets&ids=bitcoin,ethereum&vs_currencies=usd`
- 원천:
  - simple price: `https://api.coingecko.com/api/v3/simple/price`
  - markets: `https://api.coingecko.com/api/v3/coins/markets`
- 포맷: JSON
- 현재 앱 반환:
  - raw passthrough
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 서비스: `src/services/markets.ts`
  - 프록시: `api/coingecko.js`
- 접속 흐름:
```text
src/config/variants/base.ts / src/services/markets.ts -> /api/coingecko?... -> api.coingecko.com/api/v3/...
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: Redis 120초, 메모리 fallback 120초
  - fallback: 429 또는 오류 시 stale/error fallback 사용

### 5.5 Stablecoin markets
- 앱 접근 경로: `GET /api/stablecoin-markets?coins=tether,usd-coin,dai`
- raw 원천: CoinGecko `coins/markets`
- 포맷: JSON
- 비고:
  - 이 엔드포인트는 raw 원천 자체는 CoinGecko지만 앱 반환은 완전히 가공된 요약 결과다.
  - strict raw 기준에서는 upstream CoinGecko가 진짜 원천이다.
- 코드 위치:
  - UI 호출: `src/components/StablecoinPanel.ts`
  - 프록시: `api/stablecoin-markets.js`
- 접속 흐름:
```text
src/components/StablecoinPanel.ts -> /api/stablecoin-markets -> api.coingecko.com/api/v3/coins/markets
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 120초
  - fallback: cached response 또는 unavailable summary 반환

### 5.6 Polymarket gamma API
- 앱 접근 경로:
  - `GET /api/polymarket?closed=false&order=volume&ascending=false&limit=20`
  - `GET /api/polymarket?endpoint=events&tag=geopolitics&closed=false...`
- 원천:
  - markets: `https://gamma-api.polymarket.com/markets`
  - events: `https://gamma-api.polymarket.com/events`
- 포맷: JSON
- 파라미터:
  - `closed`
  - `order`: `volume|liquidity|startDate|endDate|spread`
  - `ascending`
  - `limit`
  - `tag_slug` for events
- 현재 앱 반환:
  - raw passthrough
- 코드 위치:
  - URL 생성: `src/config/variants/base.ts`
  - 서비스: `src/services/polymarket.ts`
  - 프록시: `api/polymarket.js`
- 접속 흐름:
```text
src/services/polymarket.ts -> /api/polymarket?... -> gamma-api.polymarket.com/markets|events
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 120초
  - fallback: 없음

### 5.7 EIA petroleum
- 앱 접근 경로:
  - health: `GET /api/eia/health`
  - data: `GET /api/eia/petroleum`
- 원천: `https://api.eia.gov/v2/seriesid/<SERIES>?api_key=<key>&num=2`
- 인증: `EIA_API_KEY`
- 포맷: JSON
- 현재 시리즈:
  - `PET.RWTC.W` WTI
  - `PET.RBRTE.W` Brent
  - `PET.WCRFPUS2.W` Production
  - `PET.WCESTUS1.W` Inventory
- raw 주요 필드:
  - `response.data[].period`
  - `response.data[].value`
  - `response.data[].unit`
- 현재 앱 반환:
  - 각 시리즈별 `{ current, previous, date, unit }`
- 코드 위치:
  - 서비스: `src/services/oil-analytics.ts`
  - 프록시: `api/eia/[[...path]].js`
- 접속 흐름:
```text
src/services/oil-analytics.ts -> /api/eia/petroleum -> api.eia.gov/v2/seriesid/<SERIES>
```
- 운영 메모:
  - 인증: API key
  - 환경변수: `EIA_API_KEY`
  - 캐시 TTL: 1800초
  - fallback: health endpoint로 설정 여부 확인 가능

### 5.8 USASpending
- 앱 접근 경로: 브라우저 직접 `https://api.usaspending.gov/api/v2/search/spending_by_award/`
- 원천: 동일
- 포맷: JSON
- 메서드: `POST`
- 현재 앱 요청 body 핵심:
```json
{
  "filters": {
    "time_period": [{ "start_date": "YYYY-MM-DD", "end_date": "YYYY-MM-DD" }],
    "award_type_codes": ["A", "B", "C", "D"]
  },
  "fields": [
    "Award ID",
    "Recipient Name",
    "Award Amount",
    "Awarding Agency",
    "Description",
    "Start Date",
    "Award Type"
  ],
  "limit": 15,
  "order": "desc",
  "sort": "Award Amount"
}
```
- 비고:
  - 프록시 없이 클라이언트 직접 호출
- 코드 위치:
  - 서비스: `src/services/usa-spending.ts`
  - 호출 트리거: `src/App.ts`
- 접속 흐름:
```text
src/App.ts -> src/services/usa-spending.ts -> api.usaspending.gov/api/v2/search/spending_by_award/
```
- 운영 메모:
  - 인증: 없음
  - 환경변수: 없음
  - 캐시 TTL: 없음
  - fallback: 서비스 레이어에서 오류 시 빈 결과 반환

## 6. 빠진다고 봐야 하는 것들
아래는 프로젝트에 존재하지만 raw inventory에는 넣지 않는 편이 맞다.

- `api/risk-scores.js`
- `api/climate-anomalies.js`
- `api/worldpop-exposure.js`
- `api/temporal-baseline.js`
- `api/service-status.js`
- `api/story.js`
- `api/og-story.js`

이들은 raw 원천이 아니라 내부 계산, 캐시, 운영성, 렌더링 결과물이기 때문이다.

## 7. 실무 메모

### 7.1 원천 그대로 가져오는 엔드포인트
- `rss-proxy`
- `earthquakes`
- `yahoo-finance`
- `coingecko`
- `polymarket`
- `opensky`
- `wingbits`

### 7.2 raw를 sanitize/축약하는 엔드포인트
- `acled`
- `acled-conflict`
- `gdelt-doc`
- `ucdp`
- `ucdp-events`
- `hapi`
- `unhcr-population`
- `finnhub`
- `firms-fires`

### 7.3 raw 원천이지만 브라우저 직접 호출하는 것
- `api.weather.gov`
- `eonet.gsfc.nasa.gov`
- `gdacs.org`
- `api.usaspending.gov`

### 7.4 raw 원천 추적이 저장소 밖에 있는 것
- AIS relay (`WS_RELAY_URL/.../ais/snapshot`)
- Railway OpenSky relay fallback

이 둘은 현재 저장소에서 contract는 확인 가능하지만, 최종 upstream 구현체 자체는 이 저장소에 없다.
