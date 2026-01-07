# Cloudflare 우회: 모범 사례

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/) 

이 가이드는 Cloudflare의 보안을 우회하고 차단되지 않으면서 웹사이트를 성공적으로 スクレイピング하는 방법을 설명합니다.

- [프로キ시 솔루션 사용](#using-proxy-solutions)
- [HTTP 헤ッダー 스푸핑](#spoofing-http-headers)
- [CAPTCHA 해결 서비스 구현](#implementing-captcha-solving-services)
- [강화된 헤드리스 브라우저 사용](#using-a-fortified-headless-browser)
- [Cloudflare 솔버 사용](#using-cloudflare-solvers)
- [고급 기법](#advanced-techniques)
- [Bright Data 솔루션 통합](#incorporating-bright-data-solutions)

## Cloudflare 메커니즘 이해

Cloudflare의 [web application firewall](https://www.cloudflare.com/application-services/products/waf/) (WAF)은 웹 앱을 DDoS 및 제로데이 공격으로부터 보호합니다. 전 세계 네트워크에서 실행되며 실시간으로 공격을 차단하고, 독자적인 알고리즘을 사용하여 여러 특성을 기반으로 악성 봇을 식별하고 차단합니다.

- [**TLS fingerprints**](https://brightdata.co.kr/blog/web-data/tls-fingerprinting): JA3 fingerprints는 클라이언트 및 해당 기능/구성 정보를 식별하고 클라이언트가 정품인지 검증합니다.
- **[HTTP/2 fingerprints](https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf)**: HTTP/2 파라メータ를 사용하여 알려진 봇 시그니처와 매칭합니다.
- **HTTP 세부 정보**: 봇과 유사한 구성 여부를 확인하기 위해 헤ッダー와 Cookie를 검사합니다.
- **JavaScript fingerprints**: 브라우저, OS, 하드웨어 세부 정보를 수집하여 봇을 구분합니다.
- **행동 분석**: 머신러닝으로 リクエスト 속도, 마우스 움직임, 유휴 시간 등을 모니터링하여 봇을 탐지합니다.

봇과 유사한 활동이 감지되면 Cloudflare는 백그라운드 JavaScript 챌린지를 발급하며, 실패 시 CAPTCHA로 이어집니다.

## Cloudflare 우회 기법

Cloudflare의 독자적인 봇 탐지는 완벽하지 않으므로, 요구 사항에 가장 적합한 접근 방식을 찾기 위해 실험이 필요합니다.

### Using Proxy Solutions

Cloudflare는 단일 IPアドレス에서 너무 많은 리クエスト가 발생하면 봇으로 플래그를 지정하여 탐지합니다. 이를 피하려면 [프리미엄 レジデンシャルプロキシ](https://brightdata.co.kr/proxy-types/residential-proxies)를 사용하십시오. 다만 user-agent 검사가 적용되는 경우 user agent를 스푸핑해야 합니다.

### Spoofing HTTP Headers

[HTTP 헤ッダー](https://brightdata.co.kr/blog/web-data/http-headers-for-web-scraping)는 클라이언트 세부 정보를 드러냅니다. Cloudflare는 이를 확인하여, 많은 헤ッダー를 보내는 실제 브라우저와 소수의 헤ッダー만 보내는 스크레이퍼를 구분합니다. 대부분의 스크레이퍼 도구는 진짜 브라우저를 모방하도록 헤ッダー를 수정할 수 있습니다. 일반적인 헤ッダー는 다음과 같습니다.

#### User-Agent Header

`User-Agent` 헤ッダー는 브라우저와 OS를 드러냅니다. Cloudflare는 봇과 유사한 User-Agent를 차단할 수 있으므로, 실제 브라우저(예: Chrome, Firefox, Safari)를 모방하도록 스푸핑하면 탐지를 피하는 데 도움이 됩니다. 예를 들어 Python의 [`requests` library](https://pypi.org/project/requests/)에서는 다음과 같이 설정할 수 있습니다.

```python
import requests
 
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
}
 
response = requests.get('http://httpbin.org/headers', headers=headers)
 
print(response.status_code)
print(response.text)
```

#### Referrer Header

Cloudflare는 리クエ스트의 소스를 검증하기 위해 `Referer` 헤ッダー를 확인합니다. 유효한 URL로 이를 스푸핑하면 리クエ스트가 신뢰되는 것처럼 보이게 할 수 있습니다.

```python
import requests
 
headers = {
    'Referer': 'https://trusted-website.com'
}
 
response = requests.get('http://httpbin.org/headers', headers=headers)
 
print(response.status_code)
print(response.text)
```

#### Accept Headers

`Accept` 헤ッダー는 클라이언트가 처리할 수 있는 콘텐츠 타입을 지정합니다. 실제 브라우저의 상세한 `Accept` 헤ッダー 형태를 모방하면 탐지를 피하는 데 도움이 될 수 있습니다.

```python
import requests
 
headers = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8'
}
 
response = requests.get('http://httpbin.org/headers', headers=headers)
 
print(response.status_code)
print(response.text)
```

Cloudflare는 또한 헤ッダー 불일치 및 오래된 헤ッダー도 확인합니다. 예를 들어 Firefox user agent를 사용하면서 `Sec-CH-UA-Full-Version-List`를 함께 사용하면, Firefox는 이를 지원하지 않기 때문에 차단될 수 있습니다.

### Implementing CAPTCHA-Solving Services

Cloudflare는 다른 탐지 방법이 충분하지 않을 때 의심스러운 클라이언트에 CAPTCHA를 표시할 수 있습니다. Turnstile 시스템은 가볍고 비대화형 챌린지를 실행하지만, 스크레이퍼에 도전하는 대화형 CAPTCHA로 전환될 수 있습니다. 또한 많은 서비스는 이러한 CAPTCHA를 우회하기 위해 사람 솔버를 고용합니다. 사용 사례에 맞는 서비스를 찾으려면 [the best CAPTCHA Solvers]( https://brightdata.co.kr/blog/web-data/best-captcha-solvers)에 관한 당사 글을 읽어보십시오. 

### Using a Fortified Headless Browser

Cloudflare의 JavaScript 챌린지를 우회하려면, 스크레이퍼가 실제 브라우저 동작을 모방해야 합니다—JavaScript 실행, Cookie 처리, 스크롤/마우스 이동/클릭과 같은 동작 시뮬레이션이 필요합니다. [Selenium](https://www.selenium.dev/) 같은 도구로 이를 수행할 수 있지만, [headless browsers](https://brightdata.co.kr/blog/web-data/best-headless-browsers)는 종종 자신을 노출합니다(예: `navigator.webdriver`를 통해). [undetected_chromedriver](https://github.com/ultrafunkamsterdam/undetected-chromedriver) 및 [puppeteer-extra-plugin-stealth](https://brightdata.co.kr/blog/how-tos/avoid-getting-blocked-with-puppeteer-stealth) 같은 플러그인은 이러한 특성을 마스킹하는 데 도움이 됩니다.

아래는 undetected_chromedriver를 사용하는 예시입니다.

```python
import undetected_chromedriver.v2 as uc
driver = uc.Chrome()
with driver:
    driver.get('https://example.com')
```

Cloudflare에 더 탄력적으로 대응하기 위해 헤드리스 브라우저를 [고품질 プロキ시 서비스](https://brightdata.co.kr/proxy-types)와 함께 사용할 수 있습니다.

```python
chrome_options = uc.ChromeOptions()

proxy_options = {
    'proxy': {
        'http': 'HTTP_PROXY_URL',
        'https': 'HTTPS_PROXY_URL'
    }
}

driver = uc.Chrome(
    options=chrome_options,
    seleniumwire_options=proxy_options
)
```

브라우저는 자주 업데이트되어 헤드리스 시그니처가 노출되며, Cloudflare의 진화하는 알고리즘은 이를 악용할 수 있습니다. 따라서 플러그인은 정기적으로 유지보수하지 않으면 작동이 중단될 수 있습니다.

### Using Cloudflare Solvers

전용 [Cloudflare solver services](https://github.com/luminati-io/cloudflare-captcha-solver)는 기본 보호를 일시적으로 우회할 수 있습니다. 예를 들어 cloudscraper는 JavaScript 엔진을 사용해 브라우저 지원을 시뮬레이션하지만, 업데이트가 오래되어 효과가 떨어질 수 있습니다.

### Advanced Techniques

Cloudflare는 여러 봇 탐지 방법을 사용하므로, 단일 기법만으로는 충분하지 않습니다. 대신 실제 사용자를 모방하기 위해 접근 방식을 결합하십시오. 예를 들어 강화된 헤드리스 브라우저를 사용하고, 사람의 마우스 움직임(예: [B-spline curves](https://stackoverflow.com/a/48690652))을 시뮬레이션하며, IP 차단을 피하기 위해 レジデンシャルプロキシ를 ローテーティング하고, [Hazetunnel](https://github.com/daijro/hazetunnel) 같은 도구로 진짜 브라우저 ブラウザフィンガープリント를 재현하십시오. CAPTCHA solver를 추가하면 Cloudflare 방어를 우회할 가능성이 더욱 높아집니다.

## Incorporating Bright Data Solutions

[Bright Data’s Web Unlocker](https://github.com/luminati-io/web-unlocker-api)는 AI를 사용해 アンチボット 조치(예: ブラウザフィンガープリント, CAPTCHA 해결, IPローテーティング, リクエスト リトライ)를 극복함으로써 Cloudflare의 봇 탐지 우회를 단순화하며 99.99%의 성공률을 제공합니다. 최적의 プロキ시를 자동으로 선택하고 간단한 자격 증명을 제공하여, 표준 プロキ시 서버처럼 사용할 수 있습니다. 다른 プロキ시 서버와 동일하게 사용할 수 있습니다.

```python
import requests


host = 'brd.superproxy.io'
port = 22225

username = 'brd-customer-<customer_id>-zone-<zone_name>'
password = '<zone_password>'

proxy_url = f'http://{username}:{password}@{host}:{port}'

proxies = {
    'http': proxy_url,
    'https': proxy_url
}


url = "http://lumtest.com/myip.json"
response = requests.get(url, proxies=proxies)
print(response.json())
```

[Bright Data's Scraping Browser](https://github.com/luminati-io/scraping-browser)는 여러 プロキ시를 사용해 사이트를 언락하는 원격 브라우저에서 코드를 실행함으로써 Cloudflare를 우회합니다. [Puppeteer](https://brightdata.co.kr/products/scraping-browser/puppeteer), [Selenium](https://brightdata.co.kr/products/scraping-browser/selenium), [Playwright](https://brightdata.co.kr/products/scraping-browser/playwright)와 통합되며, 완전한 헤드리스 경험을 제공합니다.

## Conclusion

Cloudflare를 회피하는 것은 복잡할 수 있으며, 성공률도 다양한 편입니다. 임시방편으로 솔루션을 덕지덕지 붙이기보다는, Web Unlocker, Scraping Browser, Web Scraper API와 같은 [Bright Data’s offerings](https://brightdata.co.kr/products) 사용을 고려하십시오. 몇 줄의 코드만으로 복잡한 솔루션을 관리할 필요 없이 더 높은 성공률을 얻을 수 있습니다.