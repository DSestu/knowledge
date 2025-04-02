# Pydoll is king, use it instead of selenium

## Nice starting snippet

```python
import asyncio
import os

from pydoll.browser.chrome import Chrome
from pydoll.browser.options import Options
from pydoll.browser.page import Page
from pydoll.constants import By
from pydoll.element import WebElement

async def highlight(self, element: WebElement, time: int = 5) -> None:
    """
    Temporarily highlight a web element by adding a red border and then restoring its original style.

    Args:
        element (WebElement): The web element to highlight.
        time (int, optional): Duration of the highlight in seconds. Defaults to 5.

    Briefly draws attention to a specific web element by adding a red border with a smooth transition,
    then restores the element's original styling after a specified time interval.
    """
    original_style = element.get_attribute("style")
    await page.execute_script(
        """
    argument.style.border = '3px solid red';
    argument.style.transition = 'border 0.3s ease-in-out';
    """,
        element,
    )
    await asyncio.sleep(time)

    await page.execute_script(
        f"""
        argument.setAttribute('style', '{original_style}');
        """,
        element,
    )


Page.highlight = highlight

def get_options(
    headless: bool = False,
    chrome_user_data: str = os.path.join(os.getcwd(), "chrome_user_data"),
) -> Options:
    """
    Configure and return Chrome WebDriver options for web automation.

    Args:
        headless (bool, optional): Whether to run Chrome in headless mode. Defaults to False.
        chrome_user_data (str, optional): Path to Chrome user data directory.
            Defaults to a 'chrome_user_data' directory in the current working directory.

    Returns:
        Options: Configured Chrome WebDriver options with specific settings for web scraping.
    """
    options = Options()
    if headless:
        options.add_argument("--headless")
    options.add_argument("--mute-audio")
    options.add_argument("--disable-dev-shm-usage")

    options.add_argument(f"--user-data-dir={chrome_user_data}")
    options.add_argument("--profile-directory=Default")
    return options


browser = Chrome(options=options)
await browser.start()
page = await browser.get_page()
# or
with Chrome(options=options) as browser:
    await browser.start()
    page = await browser.get_page()

```

## Scraping multiple pages at once

```python
import itertools
from typing import Any

from tqdm import tqdm

SCRAPE_CHUNK_PAGES = 60

all_urls = [...]


async def scrape_individual_page(browser: Chrome, url: str) -> Any:
    page = await browser.get_page()
    await page.go_to(url)
    return ...


all_data = []
for i in tqdm(range(0, len(all_urls) + 1, SCRAPE_CHUNK_PAGES)):
    chunk = all_urls[i : i + SCRAPE_CHUNK_PAGES]
    data_chunk = await asyncio.gather(
        *[scrape_individual_page(browser, url) for url in chunk]
    )
    all_data.extend(data_chunk)
all_data = list(itertools.chain.from_iterable(all_data))
```

### Wrapping it up in a function

```python
from typing import Callable


async def scrap_one_page(url: str, **kwargs) -> None: ...


async def chunked_scrap(
    fn: Callable,
    chunk_iterator: list,
    chunk_key: str,
    chunk_size: int = 60,
    **kwargs,
) -> list:
    """
    Asynchronously scrape data in chunks from an iterator using a provided function.

    Args:
        fn (Callable): The async function to call for each element in the chunk.
        chunk_iterator (list): The list of elements to be processed.
        chunk_key (str): The key name to pass each element to the function.
        chunk_size (int, optional): Number of elements to process concurrently. Defaults to 60.
        **kwargs: Additional keyword arguments to pass to the scraping function.

    Returns:
        list: Aggregated results from all processed chunks.
    """
    from tqdm import tqdm

    all_data = []
    for i in tqdm(range(0, len(chunk_iterator) + 1, chunk_size)):
        all_data.extend(
            await asyncio.gather(
                *[
                    fn(**{chunk_key: element}, **kwargs)
                    for element in chunk_iterator[i : i + chunk_size]
                ]
            )
        )
    return all_data


all_results = await chunked_scrap(
    fn=scrap_one_page,  # <- scrap_one_page has "url" arg
    chunk_key="url",  # <- we pass the name of the arg on which we will parallel scrap
    chunk_iterator=all_urls,  # The list of urls that we will scrap
    browser=browser,  # <- the browser instance that we will use to scrap
    chunk_size=2,  # <- the number of urls that we will scrap at the same time
)
```

# Selenium

# Nice starting snippet

```python
import os

from selenium.webdriver.common.by import By
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


from selenium import webdriver
from selenium.webdriver.chrome.options import Options

# Either one of those 2
import chromedriver_autoinstaller
chromedriver_autoinstaller.install()
# driver = webdriver.Chrome(options=options)

# or
from selenium.webdriver.chrome.service import Service
from chromedriver_py import binary_path
service = Service(executable_path=binary_path)
# driver = webdriver.Chrome(options=options, service=service)


current_path = os.getcwd()
chrome_user_data = os.path.join(current_path, "chrome_user_data")

options = webdriver.ChromeOptions()
options.add_argument("--no-sandbox")
options.add_argument("--headless")
options.add_argument("--disable-dev-shm-usage")
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option("useAutomationExtension", False)

options.add_argument(f"--user-data-dir={chrome_user_data}")
options.add_argument("--profile-directory=Default")
options.add_argument("--disable-dev-shm-usage")
# options.add_argument("--disable-extensions")
options.add_argument("--remote-debugging-port=9222")

service = Service(executable_path=binary_path)
driver = webdriver.Chrome(options=options, service=service)
driver.set_window_size(1500, 1200, driver.window_handles[0])

home = "https://www.google.fr"
driver.get(home)

# Open chrome://inspect in your browser

```

# Usefull driver informations

2 ways of getting the webdriver easily

1. Full automatic

```bash
pip install chromedriver_autoinstaller
```

then:

```python
chromedriver_autoinstaller.install()
```

2. Using pypi

* Get your chrome version

* Install the correct version with `pip install chromedriver-py==120.0.6099.109` (remember to change the version)

* In python:

```python
from selenium.webdriver.chrome.service import Service
from chromedriver_py import binary_path

service = Service(executable_path=binary_path)
(...)
driver = webdriver.Chrome(..., service=service)
```

# Interact with an headless chrome

This can be useful for interacting with headless chrome which is started from a WSL environment.

**As getting a selenium window from wsl is tricky and needs a setup which is independant from the conda environment, this method may be preferable for setup stability.**

Add the remote debugging port argument:

```python
from chromedriver_py import binary_path
from selenium.webdriver.chrome.options import Options

options = webdriver.ChromeOptions()
options.add_argument("--headless")
options.add_argument("--remote-debugging-port=9222") # 9222 is a default port

driver = webdriver.Chrome(options=options, ...)
```

Then, in windows, go in chrome to `chrome://inspect`.

If you use the default remote debugging port, the session should show up.

If you use an other port, you have to configure the listener in: `Discover network target > Configure > localhost:{yourport}`.

If everything went correcly, the session will show up in the appearing "Remote targets" section.

Click "inspect", and the browser will show up, fully interactive.

You may also want to change the resolution of the browser:

```python
driver.set_window_size(1500, 1200, driver.window_handles[0])
```

# Obfuscation

In order to not get blocked by anti robots, you may want to hide the fact that chrome is controlled by software:

```python
from selenium.webdriver.chrome.options import Options

options = webdriver.ChromeOptions()
options.add_argument("--no-sandbox")
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option("useAutomationExtension", False)
driver = webdriver.Chrome(options=options, ...)
```

# User profile

Some websites can ask for cookies, to log in etc etc. For this reason you may want to store cookies like an actual browser.

For an unknown reason, I personally can't use one of my existing chrome profile.

However, you can create a custom one for your project by using these flags:

```python
from selenium.webdriver.chrome.options import Options

options = webdriver.ChromeOptions()
options.add_argument("--user-data-dir=C:\\...path to your working directory...\\user_data")
options.add_argument("--profile-directory=Default")
driver = webdriver.Chrome(options=options, ...)
```

# Selenium snippets

## Wait until an element appear, with timeout

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

element = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.ID, "myDynamicElement"))
)
```

## Highight element for 5 seconds with red border

```python
from selenium.webdriver.remote.webdriver import WebDriver
from selenium.webdriver.remote.webelement import WebElement

def highlight_element(driver: WebDriver, element: WebElement) -> None:
    """
    Highlights an element in the browser for 5 seconds with a red border.

    Args:
    driver: WebDriver instance
    element: WebElement to highlight
    """
    import threading
    import time

    original_style = element.get_attribute("style")

    # Apply the highlight style
    driver.execute_script(
        """
        arguments[0].style.border = '3px solid red';
        arguments[0].style.transition = 'border 0.3s ease-in-out';
    """,
        element,
    )

    def restore_style():
        # Wait for 5 seconds
        time.sleep(5)

        # Restore the original style
        driver.execute_script(
            f"arguments[0].setAttribute('style', '{original_style}');", element
        )

    # Start a new thread to handle the delay and style restoration
    threading.Thread(target=restore_style, daemon=True).start()
```

## Scroll to bottom

```python
driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
```

## Get network requests

Can be useful to gather a stream url for example, or many other things.

```python
JS_get_network_requests = "var performance = window.performance || window.msPerformance || window.webkitPerformance || {}; var network = performance.getEntries() || {}; return network;"
network_requests = driver.execute_script(JS_get_network_requests)
```

## Extract all m3u8 (stream) urls from a webpage

Using the snippet above.

You can target a more specific stream name if you already know it in advance.

You can use the chrome extension "download HLS Streams" to list streams for exploration before scripting.

```python
def extract_m3u8(driver: webdriver.Chrome, match_str: str = ".m3u8") -> list[str]:
    JS_get_network_requests = "var performance = window.performance || window.msPerformance || window.webkitPerformance || {}; var network = performance.getEntries() || {}; return network;"
    network_requests = driver.execute_script(JS_get_network_requests)
    return [n["name"] for n in network_requests if match_str in n["name"]]
```

## Download m3u8 stream as an mp4

Prequisites:

```shell
pip install m3u8
pip install m3u8_To_MP4
```

```python
# Get the stream
stream = extract_m3u8(driver, match_str=".m3u8")[0]
# (...) or a different way of filtering the needed stream than ...[0]

# Parse m3u8
stream = m3u8.load(stream)
# Use 480p stream
playlist = [p for p in stream.playlists if str(p.stream_info.resolution[1]) == "480"][0]
try:
    os.remove(driver.title + ".mp4")
except:
    pass
# Download to file
m3u8_To_MP4.multithread_download(
    m3u8_uri=playlist.absolute_uri,
    mp4_file_name=driver.title + ".mp4",
)
```
