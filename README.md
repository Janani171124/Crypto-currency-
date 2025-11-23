from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
from selenium_stealth import stealth
import pandas as pd
from datetime import datetime
import time
import os
import webbrowser
import sys

# System Encoding Fix (
sys.stdout.reconfigure(encoding='utf-8')


# Configuration

URL = "https://coinmarketcap.com/"
CSV_FILE = "crypto_prices.csv"
TOP_N = 10  # Number of top cryptocurrencies to extract

# Chrome Options Setup

options = webdriver.ChromeOptions()
options.add_argument("--start-maximized")
options.add_argument("--disable-blink-features=AutomationControlled")
options.add_argument("--disable-gpu")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")
options.add_argument("--ignore-certificate-errors")
options.add_argument("--user-agent=Mozilla/5.0")

# Run in background (set True to hide browser)
HEADLESS = False
if HEADLESS:
    options.add_argument("--headless=new")

# ---------------------------------------------------
# Launch Browser
# ---------------------------------------------------
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)

# Enable Stealth Mode
stealth(
    driver,
    languages=["en-US", "en"],
    vendor="Google Inc.",
    platform="Win32",
    webgl_vendor="Intel Inc.",
    renderer="Intel Iris OpenGL Engine",
    fix_hairline=True,
)


# Load Website

print("Loading CoinMarketCap homepage...")
driver.get(URL)

# Wait for the crypto table to fully load
try:
    WebDriverWait(driver, 30).until(
        EC.presence_of_all_elements_located((By.CSS_SELECTOR, "tbody tr"))
    )
    time.sleep(5)  # extra wait for JS rendering
except Exception as e:
    print(" Could not load table:", e)
    driver.quit()
    exit()
# Extract Data
print(" Extracting cryptocurrency data...")

rows = driver.find_elements(By.CSS_SELECTOR, "tbody tr")
print(f"Found {len(rows)} rows on the page")

data = []
for row in rows[:TOP_N]:
    try:
        cols = row.find_elements(By.TAG_NAME, "td")
        if len(cols) > 4:
            name_symbol = cols[2].text.split("\n")
            name = name_symbol[0] if len(name_symbol) > 0 else ""
            symbol = name_symbol[1] if len(name_symbol) > 1 else ""
            price = cols[3].text
            change_24h = cols[4].text
            market_cap = cols[7].text if len(cols) > 7 else ""

            if name:
                data.append({
                    "Name": name,
                    "Symbol": symbol,
                    "Price": price,
                    "24h Change": change_24h,
                    "Market Cap": market_cap,
                    "Timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                })
    except Exception as e:
        print(" Error reading row:", e)
        continue

# Convert to DataFrame and Display
df = pd.DataFrame(data)
print("\nExtracted Data:")
print(df)


# Save Data to CSV
if not df.empty:
    file_exists = os.path.exists(CSV_FILE)
    df.to_csv(CSV_FILE, mode='a', index=False, header=not file_exists)
    print(f"\nData saved successfully to {CSV_FILE}")
else:
    print("No data extracted — please retry or refresh the page.")

# Automatically Open CSV File (Optional)

if os.path.exists(CSV_FILE):
    print(" Opening CSV file...")
    webbrowser.open(CSV_FILE)


# Close Browser

time.sleep(5)  # Keep browser open for viewing
driver.quit()
print("Browser closed. Script finished successfully.")
