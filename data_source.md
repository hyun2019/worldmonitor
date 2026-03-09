# Data Source Inventory

## 개요
이 프로젝트는 `src/` 프론트엔드가 직접 외부 데이터를 가져오는 경우와, `api/`의 Vercel Edge/Serverless 프록시를 통해 수집하는 경우가 섞여 있다. 실시간 뉴스, 시장 데이터, 지정학/분쟁 데이터는 대부분 `api/`를 거치며, 지도 오버레이와 기준 데이터는 `src/config/*`, `public/data/*`, `data/*`에 정적으로 포함되어 있다.

데이터 흐름의 큰 구조는 아래와 같다.

1. `src/App.ts`와 `src/services/*`가 각 패널/레이어별 로더를 호출
2. 외부 API가 필요한 경우 `/api/*` 엔드포인트를 호출
3. 일부 소스는 Upstash Redis와 메모리 캐시를 사용
4. 지도용 기준 데이터는 로컬 정적 파일/TS 상수로 바로 사용

## 1. 로컬 정적 데이터

### 지도/기준 데이터
- `public/data/countries.geojson`
  - 국가 경계 GeoJSON.
- `src/config/geo.ts`, `src/config/ports.ts`, `src/config/pipelines.ts`, `src/config/military.ts`
  - 핫스팟, 군기지, 항만, 해저 케이블, 파이프라인, 전략 수로 등 내장 데이터.
- `src/config/tech-geo.ts`, `src/config/tech-companies.ts`, `src/config/ai-datacenters.ts`, `src/config/ai-research-labs.ts`, `src/config/startup-ecosystems.ts`
  - Tech variant용 HQ, 클라우드 리전, 액셀러레이터, 데이터센터, 연구소, 스타트업 허브.
- `data/gamma-irradiators.json`, `data/gamma-irradiators-raw.json`
  - 감마 조사시설 데이터 원본/가공본.
- `api/data/military-hex-db.js`
  - 군용 항공기 식별용 로컬 기준 DB.

### 사용법 예시
```ts
// 국가 경계 로드
const response = await fetch('/data/countries.geojson');
const geojson = await response.json();
```

```ts
// 정적 TS 상수 사용
import { UNDERSEA_CABLES, INTEL_HOTSPOTS } from '@/config';

console.log(UNDERSEA_CABLES.length, INTEL_HOTSPOTS.length);
```

## 2. 뉴스/콘텐츠 소스

### RSS 집계
- 사용 위치: `src/config/feeds.ts`, `src/config/variants/tech.ts`, `src/services/rss.ts`
- 수집 방식: 대부분 `/api/rss-proxy?url=...`
- 실제 소스:
  - Reuters, AP, BBC, Guardian, Al Jazeera, CNBC, FT, Politico
  - Defense One, Breaking Defense, Janes, CSIS, RAND, Brookings
  - TechCrunch, The Verge, Ars Technica, MIT Tech Review, Y Combinator, a16z
  - Google News RSS 검색 결과 다수
- 비고:
  - 일부 피드는 Railway RSS 프록시를 사용.
  - 새 RSS 추가 시 `api/rss-proxy.js` allowlist도 같이 수정해야 함.

### 사용법 예시
```ts
const feedUrl = '/api/rss-proxy?url=' + encodeURIComponent('https://www.reutersagency.com/feed/');
const res = await fetch(feedUrl);
const xml = await res.text();
```

```ts
import { FEEDS } from '@/config';
import { fetchCategoryFeeds } from '@/services/rss';

const items = await fetchCategoryFeeds(FEEDS.politics);
```

### 라이브 비디오/임베드
- `api/youtube/live.js`: YouTube 라이브 페이지 파싱
- `api/youtube/embed.js`: 안전한 YouTube embed URL 생성
- 사용 위치: `src/services/live-news.ts`, `src/components/LiveNewsPanel.ts`

### 사용법 예시
```ts
const live = await fetch('/api/youtube/live?channel=SkyNews').then(r => r.json());
const embedUrl = '/api/youtube/embed?channel=SkyNews';
```

### 기타 콘텐츠
- `api/hackernews.js`: Hacker News
- `api/github-trending.js`: GitHub Trending
- `api/arxiv.js`: arXiv
- `api/tech-events.js`: Techmeme ICS + `dev.events` RSS
- `api/fwdstart.js`: FwdStart 뉴스레터 스크래핑

### 사용법 예시
```ts
const hn = await fetch('/api/hackernews?type=top&limit=20').then(r => r.json());
const trending = await fetch('/api/github-trending?language=typescript&since=daily').then(r => r.json());
const papers = await fetch('/api/arxiv?category=cs.AI&max_results=10').then(r => r.json());
const events = await fetch('/api/tech-events?days=90&limit=50').then(r => r.json());
```

## 3. 지정학/분쟁/인도주의 데이터

- `api/acled.js`, `api/acled-conflict.js`, `api/risk-scores.js`
  - ACLED 시위/분쟁 이벤트.
- `api/gdelt-geo.js`, `api/gdelt-doc.js`
  - GDELT 지오 이벤트/문서 검색.
- `api/ucdp.js`, `api/ucdp-events.js`
  - UCDP 분쟁 분류 및 지리 이벤트.
- `api/hapi.js`
  - HDX HAPI / UN OCHA conflict-events.
- `api/unhcr-population.js`
  - UNHCR population/displacement 데이터.
- `api/worldbank.js`
  - World Bank indicator API.
- `api/worldpop-exposure.js`
  - 이름과 달리 외부 WorldPop 호출이 아니다.
  - 프로젝트 내부의 우선 국가 목록과 밀도 근사치로 노출 인구를 계산한다.

사용 위치는 주로 `src/services/protests.ts`, `src/services/ucdp.ts`, `src/services/ucdp-events.ts`, `src/services/hapi.ts`, `src/services/unhcr.ts`, `src/services/worldbank.ts`, `src/services/population-exposure.ts`다.

### 사용법 예시
```ts
const protests = await fetch('/api/acled?limit=200').then(r => r.json());
const gdelt = await fetch('/api/gdelt-geo?query=protest&timespan=1day').then(r => r.json());
const ucdp = await fetch('/api/ucdp').then(r => r.json());
const ucdpEvents = await fetch('/api/ucdp-events').then(r => r.json());
const hapi = await fetch('/api/hapi').then(r => r.json());
const unhcr = await fetch('/api/unhcr-population').then(r => r.json());
const wb = await fetch('/api/worldbank?indicator=IT.NET.USER.ZS&countries=KR,US,JP').then(r => r.json());
```

```ts
// 내부 근사 기반 노출 인구 계산
const exposure = await fetch('/api/worldpop-exposure?mode=exposure&lat=37.5&lon=127.0&radius=50')
  .then(r => r.json());
```

## 4. 군사/항공/해상/재난 데이터

- `api/opensky.js`, `api/theater-posture.js`
  - OpenSky 기반 군용 항공 추적.
- `api/wingbits/*`
  - Wingbits 비행 상세/영역 조회.
- `api/ais-snapshot.js`
  - AIS relay snapshot 기반 선박 데이터.
- `api/nga-warnings.js`
  - NGA broadcast warnings.
- `api/cloudflare-outages.js`
  - Cloudflare Radar outage annotations.
- `api/earthquakes.js`
  - USGS 4.5+ day feed.
- `src/services/eonet.ts`
  - NASA EONET 직접 호출.
- `src/services/gdacs.ts`
  - GDACS 직접 호출.
- `api/firms-fires.js`
  - NASA FIRMS 화재 데이터.

### 사용법 예시
```ts
const flights = await fetch('/api/opensky?lamin=33&lomin=124&lamax=39&lomax=132').then(r => r.json());
const vessels = await fetch('/api/ais-snapshot?includeCandidates=true').then(r => r.json());
const posture = await fetch('/api/theater-posture').then(r => r.json());
const outages = await fetch('/api/cloudflare-outages').then(r => r.json());
const quakes = await fetch('/api/earthquakes').then(r => r.json());
const fires = await fetch('/api/firms-fires').then(r => r.json());
```

```ts
// 프론트에서 직접 호출하는 자연재해 소스
import { fetchNaturalEvents } from '@/services/eonet';

const naturalEvents = await fetchNaturalEvents();
```

## 5. 시장/거시/암호화폐 데이터

- `api/fred-data.js`: FRED
- `api/finnhub.js`: Finnhub 주가/시세
- `api/yahoo-finance.js`, `api/stock-index.js`, `api/etf-flows.js`, `api/macro-signals.js`
  - Yahoo Finance 계열 시계열과 ETF/거시 지표
- `api/coingecko.js`, `api/stablecoin-markets.js`
  - CoinGecko 가격/스테이블코인 상태
- `api/polymarket.js`
  - Polymarket events/markets
- `api/eia/[[...path]].js`
  - 미국 EIA 에너지 데이터 프록시

### 사용법 예시
```ts
const fred = await fetch('/api/fred-data?series_id=DGS10').then(r => r.json());
const finnhub = await fetch('/api/finnhub?symbols=AAPL,MSFT,NVDA').then(r => r.json());
const yahoo = await fetch('/api/yahoo-finance?symbol=QQQ').then(r => r.json());
const gecko = await fetch('/api/coingecko?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true')
  .then(r => r.json());
const stablecoins = await fetch('/api/stablecoin-markets').then(r => r.json());
const polymarket = await fetch('/api/polymarket?closed=false&order=volume&ascending=false&limit=20')
  .then(r => r.json());
const macro = await fetch('/api/macro-signals').then(r => r.json());
const etfFlows = await fetch('/api/etf-flows').then(r => r.json());
```

## 6. AI/요약/분류/캐시

- `api/groq-summarize.js`, `api/openrouter-summarize.js`
  - 뉴스 요약
- `api/classify-event.js`, `api/classify-batch.js`, `api/country-intel.js`
  - LLM 기반 이벤트 분류/국가 인텔 생성
- 캐시:
  - `api/_upstash-cache.js`
  - `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`
- 주요 시크릿:
  - `GROQ_API_KEY`
  - `OPENROUTER_API_KEY`
  - `ACLED_ACCESS_TOKEN`
  - `FRED_API_KEY`
  - `FINNHUB_API_KEY`
  - `EIA_API_KEY`
  - `WS_RELAY_URL`, `VITE_WS_RELAY_URL`
  - `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET`

### 사용법 예시
```ts
const summary = await fetch('/api/groq-summarize', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ headlines: ['Oil prices jump after shipping disruption'] }),
}).then(r => r.json());
```

```ts
const classified = await fetch('/api/classify-batch', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ headlines: ['Missile strike reported near port city'] }),
}).then(r => r.json());
```

## 7. 소스 추가/수정 시 체크리스트

### RSS 추가
1. `src/config/feeds.ts` 또는 `src/config/variants/tech.ts`에 feed 추가
2. `/api/rss-proxy`를 쓸 경우 `api/rss-proxy.js` allowlist에 도메인 추가
3. 막히는 도메인이면 Railway RSS 프록시 경로 사용 검토

### 신규 외부 API 추가
1. 브라우저에서 직접 호출하지 말고 가능하면 `api/*.js` 프록시 생성
2. CORS, rate limit, timeout, stale cache 전략 같이 추가
3. `src/services/data-freshness.ts`에 source 상태 반영 검토
4. 필요한 시크릿은 `runtime-config`와 README/문서에 함께 기록

## 8. 현재 판단

- 현재 핵심 데이터 소스는 `RSS + GDELT + ACLED + UCDP + OpenSky/Wingbits/AIS + FRED/CoinGecko/Polymarket + UNHCR/HAPI/WorldBank` 조합이다.
- 지도 자체의 많은 레이어는 외부 API가 아니라 `src/config/*`에 내장된 정적 데이터로 렌더링된다.
- `api/worldpop-exposure.js`처럼 이름상 외부 데이터처럼 보이지만 실제로는 내부 근사 계산인 엔드포인트가 있으므로, 신규 기능 추가 시 “실제 원천 데이터”와 “내부 파생 데이터”를 구분해서 봐야 한다.
