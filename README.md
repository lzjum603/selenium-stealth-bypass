# How to Make Selenium Avoid Detection: The Complete Anti-Bot Bypass Guide — WebDriver Flags, Fingerprinting, Proxy Rotation, and When to Stop Fighting Anti-Bot Systems Manually

Your Selenium scraper got blocked again. Maybe it was a 403, maybe a CAPTCHA wall, maybe the page just silently served you a bot-honeypot version with fake data. Whatever happened, you're here, and the fix is not as simple as "just rotate your user agent."

Let's actually talk about how **Selenium avoid detection** works — which techniques are worth your time, which are already obsolete, and at what point the manual approach stops making sense.

---

## Why Selenium Gets Caught in the First Place

Selenium, out of the box, is basically wearing a neon sign that says "I am a bot." The browser it launches has dozens of fingerprinting signals that differ from what a real human session looks like.

The biggest ones:

- **`navigator.webdriver = true`** — this JavaScript property is automatically set to `true` when Selenium controls a browser. Anti-bot systems check it immediately. A real Chrome session has this as `undefined`.
- **Headless browser telltale signs** — no `window.chrome` object, missing plugins array, odd screen resolution, no audio context, fake GPU renderer strings.
- **CDP (Chrome DevTools Protocol) activity** — some anti-bot systems monitor port 9222, which ChromeDriver uses for remote debugging.
- **Behavioral patterns** — clicks that arrive at exact pixel coordinates with zero variance. Mouse movements that form perfect straight lines. No scroll jitter. No hesitation.
- **IP reputation** — datacenter IPs get flagged heavily. One IP hammering the same domain 200 times in 10 minutes? That's a ban waiting to happen.
- **Request headers** — missing `Accept-Language`, no `Referer`, identical `User-Agent` across every session, wrong header order.

Modern anti-bot platforms — Cloudflare, PerimeterX, Akamai Bot Manager, DataDome — layer all of these signals together. Bypassing one doesn't get you through. That's why a single technique almost never works alone.

---

## Technique 1: Patch the `navigator.webdriver` Flag

This is the most commonly cited fix, and it's necessary — but not sufficient on its own.

**Using Chrome DevTools Protocol (CDP):**

python
from selenium import webdriver

options = webdriver.ChromeOptions()
driver = webdriver.Chrome(options=options)

driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {
    "source": """
        Object.defineProperty(navigator, 'webdriver', {
            get: () => undefined
        });
    """
})


This patches the property before any page scripts run. Without it, you're caught in the first JavaScript check.

**Using `undetected-chromedriver`:**

bash
pip install undetected-chromedriver


python
import undetected_chromedriver as uc

options = uc.ChromeOptions()
driver = uc.Chrome(options=options)
driver.get("https://example.com")


`undetected-chromedriver` patches the webdriver flag, removes automation-related Chrome args, and handles some of the binary-level fingerprint differences automatically. It's a good starting point for sites with moderate protection.

That said — sites running Cloudflare Bot Management or similar will still catch it. The webdriver flag is just one of 50+ signals they check.

---

## Technique 2: Handle Headless Mode Without Leaking Signals

Running `--headless` is convenient. It's also one of the most obvious detection signals.

Headless Chrome has a different `User-Agent` string by default, exposes `HeadlessChrome` in the browser version, and lacks real GPU/audio capabilities that fingerprinting scripts check. Hiding it takes a few steps:

python
options = webdriver.ChromeOptions()
options.add_argument("--headless=new")  # Use "new" headless, not legacy
options.add_argument("--window-size=1920,1080")
options.add_argument("--disable-blink-features=AutomationControlled")
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option("useAutomationExtension", False)

# Override User-Agent to remove HeadlessChrome
options.add_argument(
    "user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
    "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
)


The `--disable-blink-features=AutomationControlled` flag is particularly important — it suppresses the `navigator.webdriver` flag at the browser level rather than patching it in JS.

One thing I've found personally: even with all of this in place, if you're using a datacenter IP, sophisticated systems will still catch you. The browser fingerprint might pass, but the IP fingerprint won't.

---

## Technique 3: Use `selenium-stealth` for Deeper Fingerprint Patching

`selenium-stealth` is a Python library that addresses a wider range of browser fingerprint signals beyond just the webdriver flag.

bash
pip install selenium-stealth


python
from selenium import webdriver
from selenium_stealth import stealth

options = webdriver.ChromeOptions()
options.add_argument("--headless")
driver = webdriver.Chrome(options=options)

stealth(
    driver,
    languages=["en-US", "en"],
    vendor="Google Inc.",
    platform="Win32",
    webgl_vendor="Intel Inc.",
    renderer="Intel Iris OpenGL Engine",
    fix_hairline=True,
)


What stealth patches:
- `navigator.languages` (real browsers have a list, not a single value)
- WebGL vendor/renderer strings
- Chrome plugin presence
- Canvas fingerprint hairline inconsistencies
- `window.chrome` runtime object

Works well against sites with lighter defenses. On heavily guarded pages (big e-commerce, social platforms, financial services), it's still not enough without proxy rotation.

---

## Technique 4: Human-Like Behavior — The Part Most Guides Skip

Here's the thing nobody talks about enough: even with a perfect fingerprint, your *behavioral* patterns can get you flagged.

Real users:
- Don't click elements within 50ms of page load
- Don't move the mouse in a straight line from A to B
- Scroll erratically, sometimes back up, sometimes overshoot
- Take pauses to "read" content
- Occasionally move the mouse off the main viewport

Mimicking this in Selenium:

python
import time
import random
from selenium.webdriver.common.action_chains import ActionChains

def human_delay(min_sec=1.0, max_sec=3.5):
    time.sleep(random.uniform(min_sec, max_sec))

def human_scroll(driver):
    scroll_amount = random.randint(300, 700)
    driver.execute_script(f"window.scrollBy(0, {scroll_amount});")
    human_delay(0.5, 1.5)
    # Occasionally scroll back a bit
    if random.random() > 0.7:
        driver.execute_script(f"window.scrollBy(0, -{random.randint(50, 150)});")

def human_click(driver, element):
    actions = ActionChains(driver)
    # Move to a nearby spot first, then to element
    actions.move_by_offset(
        random.randint(-20, 20), 
        random.randint(-10, 10)
    ).pause(random.uniform(0.1, 0.3))
    actions.move_to_element(element).pause(random.uniform(0.1, 0.4))
    actions.click().perform()


Not glamorous. But it actually makes a difference on sites that analyze mouse movement timing and click patterns.

---

## Technique 5: Rotate Proxies — And Get This Right

This is where most DIY setups fall apart. Not because proxy rotation is conceptually hard, but because the details matter:

**What doesn't work:**
- Free proxy lists (almost universally already flagged)
- Datacenter proxies on heavily-guarded sites
- Rotating too predictably (every N requests exactly)

**What actually works:**
- Residential proxies tied to real ISP-assigned IPs
- Mobile proxies for the toughest sites
- Randomizing rotation timing, not fixing it to a count

Integrating with Selenium using `selenium-wire`:

bash
pip install selenium-wire


python
from seleniumwire import webdriver

proxy_url = "http://username:password@proxy-host:port"

options = {
    'proxy': {
        'http': proxy_url,
        'https': proxy_url,
        'no_proxy': 'localhost,127.0.0.1'
    }
}

driver = webdriver.Chrome(seleniumwire_options=options)


Managing your own residential proxy pool — sourcing IPs, handling rotation logic, dealing with blocks, monitoring success rates — is a real engineering project. That's before you even write the scraping logic.

---

## When Manual Techniques Stop Making Sense

Here's the honest version: layering all of the above together handles a lot of sites. Simple to moderately protected sites? The combination of `undetected-chromedriver` + selenium-stealth + residential proxies + human delays gets you through most of them.

But for serious scraping volume, or for sites running enterprise-grade anti-bot solutions (Cloudflare Enterprise, Akamai, DataDome), manual management becomes a maintenance nightmare. Anti-bot systems update constantly. What worked last week might not work today.

This is exactly where using a scraping API as a proxy layer for Selenium changes the math entirely.

---

## Using ScraperAPI with Selenium: Let the API Handle the Anti-Bot Stack

[ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons) is a web scraping infrastructure layer that handles proxy rotation, CAPTCHA solving, browser fingerprinting, and anti-bot bypass — all in one endpoint you call instead of the target URL directly.

The idea: instead of routing Selenium through a raw proxy you manage yourself, you route it through ScraperAPI's proxy mode. Behind that endpoint sits a pool of 40M+ residential and mobile IPs, automatic retry logic, CAPTCHA solving, and the infrastructure to handle modern bot detection systems at scale.

👉 [查看 ScraperAPI 所有套餐与当前折扣](https://www.scraperapi.com/?fp_ref=coupons)

**Setting it up with Selenium:**

python
from selenium import webdriver
from selenium.webdriver.common.proxy import Proxy, ProxyType

SCRAPERAPI_KEY = "YOUR_API_KEY"
proxy_url = f"http://scraperapi:{SCRAPERAPI_KEY}@proxy-server.scraperapi.com:8001"

options = webdriver.ChromeOptions()
options.add_argument(f"--proxy-server={proxy_url}")
options.add_argument("--ignore-certificate-errors")  # Required for SSL interception

driver = webdriver.Chrome(options=options)
driver.get("https://your-target-site.com")


What this does for you under the hood:
- Every request gets routed through a different residential or mobile IP
- Headers are normalized to look like real browser traffic
- CAPTCHAs are solved automatically where possible
- Retry logic handles temporary blocks without your code knowing
- Geotargeting lets you specify which country or city the request appears to come from

For JS-heavy pages that need actual Selenium rendering (rather than just HTTP requests), this proxy-mode integration is the right approach. For pages that don't need JS rendering, ScraperAPI's direct API mode is faster and uses fewer API credits.

---

## A Complete Working Example

Putting it together — Selenium with ScraperAPI proxy, basic stealth patching, and human behavior:

python
import time
import random
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

SCRAPERAPI_KEY = "YOUR_API_KEY"
proxy_url = f"http://scraperapi:{SCRAPERAPI_KEY}@proxy-server.scraperapi.com:8001"

options = webdriver.ChromeOptions()
options.add_argument(f"--proxy-server={proxy_url}")
options.add_argument("--ignore-certificate-errors")
options.add_argument("--disable-blink-features=AutomationControlled")
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option("useAutomationExtension", False)
options.add_argument("--window-size=1920,1080")

driver = webdriver.Chrome(options=options)

# Patch webdriver property
driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {
    "source": "Object.defineProperty(navigator, 'webdriver', {get: () => undefined})"
})

# Navigate with a delay before interacting
driver.get("https://your-target-site.com")
time.sleep(random.uniform(2, 4))

# Human-like scroll before extracting data
driver.execute_script("window.scrollBy(0, 300);")
time.sleep(random.uniform(0.8, 1.5))

# Wait for element and extract
wait = WebDriverWait(driver, 15)
element = wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, ".product-title")))
print(element.text)

driver.quit()


---

## ScraperAPI Plans Compared

ScraperAPI offers a 7-day trial with 5,000 API credits and no credit card required. Paid plans cover everything from solo projects up to enterprise-scale pipelines.

| Plan | API Credits/Month | Concurrent Threads | Geotargeting | Monthly Price |
|---|---|---|---|---|
| **Hobby** | 100,000 | 20 | US & EU | [ $49/mo — Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | 1,000,000 | 50 | US & EU | [ $149/mo — Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Business** | 3,000,000 | 100 | Global (country-level) | [ $299/mo — Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Scaling** | 5,000,000 | 200 | Global (country-level) | [ $475/mo — Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Professional** | 10,500,000 | 300 | Global (country-level) | [ $975/mo — Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Advanced** | 21,500,000 | 500 | Global (country-level) | [ $1,975/mo — Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Enterprise** | 22M+ custom | 500+ | Global | [ Contact Sales for Custom Pricing](https://www.scraperapi.com/?fp_ref=coupons) |

Annual billing saves 10% across all plans. The Scaling plan and above include pay-as-you-go overage so your scraper doesn't just stop cold when you hit the credit cap. Business plan and above get global country-level geotargeting rather than just US & EU.

One thing worth mentioning on pricing: 7 days to test with no card required is genuinely low-friction. I've seen other tools require a card just to see if the thing works.

👉 [开始免费试用 ScraperAPI — 无需信用卡](https://www.scraperapi.com/?fp_ref=coupons)

---

## Which Approach Should You Actually Use?

Here's a practical breakdown:

**Go full DIY (the techniques above) if:**
- You're scraping a small number of sites with light bot protection
- You have engineering bandwidth to maintain the stack as anti-bot systems update
- You need Selenium specifically for complex JS interactions on sites that aren't heavily guarded
- Budget is the primary constraint and volume is low

**Use ScraperAPI proxy mode with Selenium if:**
- You're hitting sites with Cloudflare, PerimeterX, or similar enterprise-grade protection
- You're running at scale and can't afford unpredictable block rates
- You want to write scraping logic, not proxy infrastructure
- Your team's time is worth more than the API cost (at $49/month for 100K credits, the math usually works out)

**Use ScraperAPI's direct API (not Selenium) if:**
- The target page doesn't require JavaScript interaction
- You want faster response times and lower credit usage
- You're doing high-volume collection where Selenium's overhead is a bottleneck

---

## FAQ

**Q: Does `undetected-chromedriver` still work?**

For many sites, yes — it handles the most common detection signals and is actively maintained. The limitation is it doesn't solve the IP reputation problem, and on enterprise anti-bot platforms it gets caught. Best used as one layer of a larger setup, not the whole solution.

**Q: Can Selenium avoid detection on Cloudflare-protected sites?**

It's possible with the right stack, but Cloudflare Bot Management checks dozens of signals simultaneously — TLS fingerprint, JS execution behavior, timing analysis, IP reputation, browser API availability. DIY setups have a lower success rate. Routing through ScraperAPI's infrastructure, which is purpose-built to handle this, tends to work better for production use.

**Q: What's the difference between `--headless` and `--headless=new`?**

The legacy `--headless` flag (pre-Chrome 112) had more fingerprinting differences from headed Chrome. The newer `--headless=new` mode (default in recent Chrome versions) is closer to how headed Chrome behaves, making it harder to detect — though still detectable by sophisticated systems.

**Q: How many API credits does a Selenium + ScraperAPI request use?**

Standard requests use 1 credit. Requests with JavaScript rendering enabled use 5 credits. Premium and ultra-premium domains (harder targets) cost more. The full credit cost table is available in ScraperAPI's documentation.

**Q: Does ScraperAPI handle CAPTCHAs automatically?**

Yes — CAPTCHA handling is built into all plans. It's included by default and doesn't require any extra configuration in your Selenium setup.

**Q: Is there a free tier?**

The 7-day trial gives you 5,000 API credits with no credit card required. There's also a free account option with 1,000 credits for lighter testing. 

👉 [立即获取 ScraperAPI 免费试用额度](https://www.scraperapi.com/?fp_ref=coupons)

---

The manual approach — patching webdriver flags, stealth libraries, residential proxies, behavioral simulation — is genuinely worth understanding. It makes you a better scraper developer. But at some point, maintaining that stack becomes the job, and the actual data collection becomes the side project. ScraperAPI is what you reach for when you'd rather flip that ratio.
