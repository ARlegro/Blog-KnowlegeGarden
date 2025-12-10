---
{"dg-publish":true,"permalink":"/프로젝트/나만무/데이터수집-가공/Crawl4AI 공부 - LLM · RAG 시대의 웹 크롤링 올인원 도구/","noteIcon":"","created":"2025-12-03T14:53:06.816+09:00","updated":"2025-12-10T11:56:23.429+09:00"}
---


## 1.  Crawl4AI - 오픈소스 웹 크롤러

> Crawl4AI는 웹 페이지를 빠르게 크롤링하고, LLM이 바로 활용할 수 있는 **Markdown/텍스트 형태로 정제**해주는 오픈소스 크롤러이다. 복잡한 HTML → **AI 친화적 데이터로 자동 변환**해주는 것이 가장 큰 가치

주소 : https://github.com/unclecode/crawl4ai

## 2.  역할 - LLM, RAG용 데이터 수집 도구 

> 웹사이트를 아주 빠르고 효율적으로 긁어와서 AI가 이해하기 쉬운 형태로 바꿔줌 

*✅HTML → Markdown 자동 변환* 
- 복잡한 DOM을 구조화된 마크다운 문서로 변환
- LLM이 이해하기 쉬운 구조 생성 (헤더, 문단, 리스트, 이미지 alt 등)

*✅노이즈 제거(Pruning)*
- 광고/네비게이션/사이드바/`<style>` 같은 더러운 것들 버림 by 내부 알고리즘
- 즉, **BeautifulSoup의 텍스트 정리 + 후처리 ➡ 한 번에 해줌** 
- 결과물을 “LLM이 읽기 좋은 문서”로 만들어줌

즉, **크롤링 + 텍스트 정제 + 후처리를 한 번에 수행**

## 3.  장점 (Feat. RAG에 좋은 이유)

> 웹 사이트 전체의 내용을 아주 쉽게 모아 RAG에 적용할 수 있다.

웹 ➡ 마크다운/텍스트 ➡ 바로 RAG에 넣을 수 있음 

### 3.1.  RAG 파이프라인 최적화 
RAG는 보통 아래 흐름을 갖는다:
1. **데이터 수집** (크롤링)
2. **정제 / 분할(Chunking)**
3. **임베딩 → 벡터DB 적재**

>[!tip] Crawl4AI는 1,2번을 통합해서 한다.


> **웹 → Markdown → Chunk 리스트**까지의 과정이 매우 깔끔해진다.

### 3.2.  비동기 고성능 크롤링 
- `AsyncWebCrawler` 기반
- 여러 URL을 병렬로 처리
- JS 렌더링 필요한 페이지도 대응 가능




## 4.  All-In-One 기능 (크롤링 ~ VectorDB 저장)
> Crawl4AI는 CLI 기반으로 크롤링 + 정제 + 임베딩 - 벡터 DB 저장을 한번에 할 수 있음 

```bash
crawl4ai crawl <URL or sitemap> \
  --format json \
  --embed \
  --db pgvector \
  --model all-MiniLM-L6-v2
```




## 5.  사용 

*공식 문서*
1. [Crawl4ai 깃허브 링크](https://github.com/unclecode/crawl4ai)
2. [PydanticAI 공식문서](https://ai.pydantic.dev/)


### 5.1.  설치법 

```BASH
pip install crawl4ai
```




### 5.2.  사이트의 모든 페이지 주소 크롤링

각 사이트는 `sitemap.xml`에서 페이지들을 갖고 있음
이를 활용해서 여러 페이지 내용을 한 번에 긁어올 수 있음




### 5.3.  사용 예시 1. 블로그 리뷰 여러 개 긁고 + RAG

> 시나리오 : 서울 ○○동 맛집에 대한 블로그 리뷰 100개를 긁어서,  **유저가 질문 던지면 RAG로 답해주는 시스템**

*과정*
1. **URL 수집**
	- 네이버/다음/구글 검색 결과에서 해당 맛집 관련 블로그 URL 리스트를 만든다.
	  
2. **Crawl4AI로 여러 URL을 비동기로 크롤링**
    - 여러 URL을 비동기로 돌려 마크다운으로 받는다.
      
3. **Markdown결과 ➡ Chunking**
    - 문단/섹션 단위로 잘라 chunk 리스트 생성  
      
4. **임베딩 & 벡터DB 저장**
5. **질문 들어오면 → 관련 chunk 검색 → LLM에 넣어 답변 생성**


```python
import asyncio
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig
from crawl4ai.markdown_generation_strategy import DefaultMarkdownGenerator
from crawl4ai.content_filter_strategy import PruningContentFilter

urls = [
    "https://blog.example.com/review1",
    "https://blog.example.com/review2",
    # ...
]

async def crawl_many(urls):
    content_filter = PruningContentFilter(
        threshold=0.5,
        threshold_type="fixed",
        min_word_threshold=20,
    )
    md_generator = DefaultMarkdownGenerator(content_filter=content_filter)
    run_config = CrawlerRunConfig(markdown_generator=md_generator)

    results = []

    async with AsyncWebCrawler() as crawler:
        for url in urls:
            result = await crawler.arun(url=url, config=run_config)
            if result.success:
                results.append(
                    {
                        "url": url,
                        "title": result.markdown.title,
                        "markdown": result.markdown.fit_markdown
                        or result.markdown.raw_markdown,
                    }
                )
    return results

if __name__ == "__main__":
    crawled_docs = asyncio.run(crawl_many(urls))
    # 이걸 이제 chunk → embedding → 벡터DB로
```

### 5.4.  사용 예시 2. 특정 사이트의 전체 크롤링 + RAG 
> [!INFO] 시스템 예시 
> 특정 여행 블로그 사이트 전체를 지식베이스로 만들어서 ‘제주도 렌트/숙소/맛집 추천해줘’ 같은 질문에 답하게 만들기

이 때 사용할 것 기능들
1. `BFSDeepCrawlStrategy` 깊이 크롤 전략
2. URL 필터(`URLPatternFilter`, `ContentTypeFilter`)를 사용해서 “이 도메인 아래, 특정 path만 크롤” 하는 식으로 세팅

과정
1. **도메인 루트 또는 sitemap URL**에서 시작
2. 내부 링크를 BFS로 따라가면서
    - HTML만 수집
    - 쓸모없는 `/login`, `/cart` 같은 URL은 패턴으로 제외
      
3. 각 페이지를 마크다운으로 저장 → RAG용 인덱싱

## 6.  요약 

- Crawl4AI는 웹 페이지를 빠르고 효율적으로 수집해 **LLM이 이해 가능한 Markdown 형태로 자동 정제**해주는 오픈소스 크롤러
- 비동기 기반의 고속 크롤링, JS 렌더링 지원, 콘텐츠 필터링, Markdown 생성 전략 등을 제공
- RAG 파이프라인(수집 → 정제 → 임베딩)의 초기 단계를 크게 단순화
- 여행 리뷰, 블로그 글, 문서 모음 등의 대규모 데이터셋 구축에 매우 적합
