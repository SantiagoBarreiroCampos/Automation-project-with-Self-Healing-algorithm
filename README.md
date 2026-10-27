# 🩹 Automation Project with Self-Healing Algorithm

Web automation project built with **Selenium** that includes a **self-healing** algorithm powered by Machine Learning. When a locator (XPath) stops working because the page has changed, the system analyzes the current page and, using scikit-learn's **Nearest Neighbors** algorithm, finds the element most similar to the one that was last located successfully, returning a new "healed" locator so the test can keep running.

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)
![Selenium](https://img.shields.io/badge/Selenium-4.20-43B02A?logo=selenium&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.3.1-F7931E?logo=scikitlearn&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-2.0.1-150458?logo=pandas&logoColor=white)

---

## 📌 Table of Contents

- [Motivation](#-motivation)
- [How it works](#️-how-it-works)
- [Project structure](#-project-structure)
- [Requirements](#-requirements)
- [Installation](#-installation)
- [Usage](#️-usage)
- [Trying out self-healing](#-trying-out-self-healing)
- [Configuration](#-configuration)
- [Limitations and next steps](#-limitations-and-next-steps)
- [Author](#-author)

---

## 💡 Motivation

One of the biggest problems in UI test automation is **locator fragility**: a change to an `id`, a class or the DOM structure breaks tests even when the functionality is still intact. This project is a proof of concept showing how a test can **heal itself** at runtime, reducing the maintenance cost of automation suites.

---

## ⚙️ How it works

```
          ┌──────────────────────────┐
          │  driver.find_element()   │
          └────────────┬─────────────┘
                       │
              Found?   │
        ┌──────── Yes ─┴── No ─────────┐
        ▼                              ▼
┌───────────────────┐     ┌────────────────────────────┐
│ Save element      │     │ Scrape the current page    │
│ attributes to     │     │ → current_elements.csv     │
│ elements.csv      │     └─────────────┬──────────────┘
└───────────────────┘                   ▼
                          ┌────────────────────────────┐
                          │ One-Hot Encoding + KNN     │
                          │ (cosine similarity)        │
                          └─────────────┬──────────────┘
                                        ▼
                          ┌────────────────────────────┐
                          │ Similarity ≥ 90% → return  │
                          │ the new XPath              │
                          └────────────────────────────┘
```

1. **Learning on every successful run.** Every time an element is found, `Scraper.save_web_element_scraping` stores its fingerprint (tag, text, all its HTML attributes and URL) in `resources/elements.csv`, linked to the locator that was used.
2. **Failure detection.** If `find_element` raises an exception, the self-healing process starts.
3. **Scraping the current page.** `Scraper.scraping_current_page_elements` grabs the `page_source`, parses it with **BeautifulSoup** and extracts all relevant elements (`a`, `div`, `span`, `button`, `input`, `select`, `pre`, `textarea`, `svg`, `img`). For each one it builds an XPath with `XpathGenerator` and writes it to `resources/current_elements.csv`.
4. **Encoding.** The (categorical) attributes of the original element and of the page elements are turned into binary vectors with `OneHotEncoder`.
5. **Nearest neighbor search.** `NearestNeighbors` with the **cosine** metric finds the page element most similar to the last successfully found element.
6. **Validation.** If the similarity is above the tolerance (**90%** by default) the new XPath is returned; otherwise the element is discarded.

Console output format (illustrative values):

```
	Element with locator "//*[@id='user-name-old']" not found
	Similar element found with locator: //*[@id="user-name"]
	Element similarity: XX.XX%
	Element valid. Returning new locator
```

---

## 📁 Project structure

```
Automation-project-with-Self-Healing-algorithm/
└── self-healing/
    ├── brain/
    │   ├── generator.py        # XpathGenerator: builds XPaths from BeautifulSoup elements
    │   ├── healing.py          # Self-healing algorithm (OneHotEncoder + NearestNeighbors)
    │   └── scraping.py         # Scraper: extracts and stores element and page data
    ├── resources/
    │   ├── elements.csv         # History of successfully found elements
    │   └── current_elements.csv # Snapshot of the page at the moment of failure
    ├── saucedemo-login.py      # Example test: login on saucedemo.com
    └── requirements.txt
```

| Module | Responsibility |
|---|---|
| `brain/generator.py` | Builds an element's XPath by walking up its parents until it finds one with an `id` or reaches the document root. |
| `brain/scraping.py` | Stores the attributes of found elements and scrapes the whole page when something fails. |
| `brain/healing.py` | Reads both CSVs, encodes the data, runs KNN and decides whether the proposed locator is valid. |
| `saucedemo-login.py` | Integration example: wraps `find_element` in `_find_element`, which triggers self-healing on failure. |

---

## 🧰 Requirements

- Python **3.9+**
- Google Chrome installed (Selenium 4 manages ChromeDriver automatically via *Selenium Manager*)

Dependencies (`self-healing/requirements.txt`):

```
scikit-learn==1.3.1
pandas==2.0.1
selenium==4.20
beautifulsoup4==4.11.1
```

---

## 🚀 Installation

```bash
git clone https://github.com/SantiagoBarreiroCampos/Automation-project-with-Self-Healing-algorithm.git
cd Automation-project-with-Self-Healing-algorithm/self-healing

python -m venv venv
# Windows
venv\Scripts\activate
# Linux / macOS
source venv/bin/activate

pip install -r requirements.txt
```

---

## ▶️ Usage

> ⚠️ Run the scripts **from the `self-healing/` folder**, since the paths to `resources/` are relative.

```bash
cd self-healing
python saucedemo-login.py
```

The script opens Chrome, navigates to [saucedemo.com](https://www.saucedemo.com/), fills in the username and password and clicks the login button. Every element that is found updates `resources/elements.csv`.

### Using it in your own tests

Just replace your `driver.find_element` calls with a wrapper like the one in the example:

```python
from brain.scraping import Scraper
from brain.healing import init_healing


def find_element(driver, by, locator):
    try:
        element = driver.find_element(by, locator)
        Scraper(driver).save_web_element_scraping(element, locator)  # learn from success
        return element
    except Exception:
        Scraper(driver).scraping_current_page_elements()             # capture the current page
        new_locator = init_healing(locator)                           # find the most similar element
        return driver.find_element(by, new_locator)
```

---

## 🧪 Trying out self-healing

To simulate a page change that makes a locator obsolete:

1. Run `saucedemo-login.py` at least once with the correct locators so `elements.csv` has data.
2. Edit `resources/elements.csv` and change the `loc` value of an element, e.g. `//*[@id='user-name']` → `//*[@id='user-name-old']`.
3. Change the same locator in `saucedemo-login.py` to `//*[@id='user-name-old']`.
4. Run the script again: the locator doesn't exist on the page, self-healing kicks in and the system recovers the correct XPath for the username field by comparing its attributes.

---

## 🔧 Configuration

| Parameter | File | Default | Description |
|---|---|---|---|
| `tolerance` | `brain/healing.py` | `0.9` | Minimum similarity (0–1) required to accept the proposed element. |
| `n_neighbors` | `brain/healing.py` | `1` | Number of neighbors computed by KNN. |
| `algorithm` | `brain/healing.py` | `'auto'` | `NearestNeighbors` algorithm. |
| `TAGS` | `brain/scraping.py` | `a, div, span, button, input, select, pre, textarea, svg, img` | HTML tags analyzed during scraping. |
| `GET_ELEMENT_RECT` | `brain/scraping.py` | `False` | If `True`, also stores each element's position and size (`rect`). More accurate, but slower. |

---

## 🚧 Limitations and next steps

- If no element exceeds the tolerance, `init_healing` returns `None` and the test still fails.
- The history is stored in CSV files; a database would be a better fit for large suites.
- It only works with XPath locators generated from an `id` or from the element's position in the DOM.
- Future ideas: give more weight to stable attributes (`id`, `name`, `data-test`), package it as a **pytest** plugin, support other browsers and automatically persist the healed locator.

---

## 👤 Author

**Santiago Barreiro Campos**

[![GitHub](https://img.shields.io/badge/GitHub-SantiagoBarreiroCampos-181717?logo=github)](https://github.com/SantiagoBarreiroCampos)

If you find this project useful, don't forget to leave a ⭐ on the repository!
