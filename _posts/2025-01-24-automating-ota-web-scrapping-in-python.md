---
layout: post
title: Automating OTA Web Scraping in Python
date: 2025-01-24 21:01:00
description: A practical guide to scraping accommodation data from Agoda using Python, Selenium, and BeautifulSoup.
tags: scrapping python automation
categories: automation
---

## 🧭 Overview

Online Travel Agencies (OTAs) like **Agoda**, **Booking.com**, and **Expedia** manage millions of property listings.  
For researchers, analysts, or automation developers, collecting structured data from these platforms is crucial — for **price monitoring**, **listing optimization**, or **competitive analysis**.

This post walks through how to **automate Agoda web scraping** using Python.  
We’ll use a mix of **Selenium**, **BeautifulSoup**, and **pandas** to extract data efficiently.

---

## ⚙️ Step 1 — Setting Up the Environment

Install the required libraries:

```bash
pip install selenium beautifulsoup4 pandas

Then download a compatible ChromeDriver

for your browser version.

We’ll also use webdriver-manager to simplify setup:

pip install webdriver-manager

🌐 Step 2 — Launching Agoda with Selenium

from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.by import By
import time

# Launch browser
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()))

# Open Agoda search page
city = "Yogyakarta"
url = f"https://www.agoda.com/search?city={city}"
driver.get(url)

time.sleep(5)  # wait for page to load

🔍 Step 3 — Extracting Hotel Listings

Once the page is loaded, we can grab hotel information using XPath or CSS selectors.

from bs4 import BeautifulSoup

soup = BeautifulSoup(driver.page_source, "html.parser")
hotels = soup.find_all("div", class_="PropertyCard__Container")

data = []

for h in hotels:
    name = h.find("h3").get_text(strip=True) if h.find("h3") else None
    price = h.find("span", class_="pd-price").get_text(strip=True) if h.find("span", class_="pd-price") else None
    rating = h.find("span", class_="ReviewScore__score").get_text(strip=True) if h.find("span", class_="ReviewScore__score") else None
    
    data.append({"name": name, "price": price, "rating": rating})

print(f"Scraped {len(data)} hotels.")

📊 Step 4 — Saving Results

import pandas as pd

df = pd.DataFrame(data)
df.to_csv("agoda_yogyakarta.csv", index=False)
print("Data saved to agoda_yogyakarta.csv")

🧠 Step 5 — Automating with Parameters

You can easily extend this scraper to multiple cities or date ranges:

cities = ["Yogyakarta", "Jakarta", "Bali"]

for city in cities:
    url = f"https://www.agoda.com/search?city={city}"
    driver.get(url)
    time.sleep(5)
    # repeat scraping logic...

Add logging or scheduling (via cron or Airflow) to make it production-ready.
🚀 Key Takeaways

✅ Selenium handles dynamic content rendered by JavaScript.
✅ BeautifulSoup simplifies HTML parsing.
✅ pandas structures data for analysis or export.
✅ You can scale this to hundreds of cities with request throttling or proxy rotation.
🧩 Next Steps

    Add proxy rotation to handle IP blocking.

    Use headless browsers for performance.

    Integrate into FastAPI for on-demand scraping.

    Store data into PostgreSQL for analysis.

💬 Conclusion

By combining Python, Selenium, and BeautifulSoup, you can automate OTA data collection with minimal manual effort.
Whether it’s for market research, listing optimization, or pricing analytics, web scraping forms the backbone of modern OTA intelligence.

