<h1>Property Scrapper Presents The ability to have a property listing site be scoured
and generate an output that fits your filtered requests within in. </h1>




<p> This workflow was created to provide a streamlined process for Realtors or investor and home buyers to do an auto search in their preffered area, in this case Texas. There are a couple of code lines that include price range, minimum and maximum price. Zip codes can be configured as the code is modified. </p>



<p> Whats to come? The idea is to automate the search in different areas more easily. At the current moment the code has to be adjusted to modify the location, price range, bed and bath quantities. The idea is to make it user friendly without having to edit Python code.</p>

<p style="background-color:Tomato;"> Sona S.</p>








#!/usr/bin/env python3
"""
Zillow Real Estate Scraper
Scrapes residential, multifamily, and commercial properties in Texas.
Filters by price, condition, and days on market. Generates CSV reports.
"""

import json
import csv
import logging
from datetime import datetime, timedelta
from pathlib import Path
from apscheduler.schedulers.background import BackgroundScheduler
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
import time

# Setup logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

# Configuration
SEARCH_CONFIG = {
    'residential': {
        'min_price': 250000,
        'max_price': 450000,
        'max_beds': 4,
        'max_baths': 3,
        'search_type': 'homes'
    },
    'multifamily': {
        'min_price': 250000,
        'max_price': 600000,
        'search_type': 'homes'
    },
    'commercial': {
        'min_price': 1000000,
        'max_price': 5000000,
        'search_type': 'commercial'
    },
    'min_days_on_market': 35,
    'conditions': ['Minor cosmetic changes', 'Turnkey', 'As-is', 'Needs some work']
}

# Output directory
OUTPUT_DIR = Path('realtor_reports')
OUTPUT_DIR.mkdir(exist_ok=True)

# History file
HISTORY_FILE = OUTPUT_DIR / 'listing_history.json'

def load_history():
    """Load previous listings."""
    if HISTORY_FILE.exists():
        with open(HISTORY_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_history(history):
    """Save listings to history."""
    with open(HISTORY_FILE, 'w') as f:
        json.dump(history, f, indent=2)

def setup_driver():
    """Setup Selenium WebDriver with Chrome."""
    options = Options()
    options.add_argument('--start-maximized')
    options.add_argument('--disable-blink-features=AutomationControlled')
    options.add_argument('user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36')
    driver = webdriver.Chrome(options=options)
    return driver

def scrape_zillow(property_type, min_price, max_price, search_type='homes'):
    """Scrape Zillow for listings."""
    driver = setup_driver()
    listings = []
    try:
        # Build Zillow search URL
        url = f"https://www.zillow.com/{search_type}/for_sale/?searchQueryState=%7B%22pagination%22:%7B%7D,%22usersSearchTerm%22:%22Texas%22,%22mapBounds%22:%7B%7D,%22regionSelection%22:%5B%7B%22regionId%22:25,%22regionType%22:2%7D%5D,%22isMapVisible%22:false,%22filterState%22:%7B%22price%22:%7B%22min%22:{min_price},%22max%22:{max_price}%7D,%22daysOn%22:%7B%22value%22:{SEARCH_CONFIG['min_days_on_market']}%7D%7D,%22isListVisible%22:true%7D"
        logger.info(f"Scraping {property_type} from Zillow...")
        driver.get(url)
        # Wait for listings to load
        time.sleep(3)
        WebDriverWait(driver, 10).until(
            EC.presence_of_all_elements_located((By.CLASS_NAME, 'search-results-list'))
        )
        # Extract listings
        listing_cards = driver.find_elements(By.CLASS_NAME, 'property-card-data')
        logger.info(f"Found {len(listing_cards)} listings")
        for card in listing_cards:
            try:
                address_elem = card.find_element(By.CLASS_NAME, 'property-address')
                address = address_elem.text if address_elem else 'N/A'
                price_elem = card.find_element(By.CLASS_NAME, 'property-price')
                price_text = price_elem.text.replace('$', '').replace(',', '') if price_elem else '0'
                try:
                    price = int(float(price_text.split()[0]))
                except:
                    price = 0
                # Get details
                details = card.text.split('\n')
                beds = baths = 'N/A'
                days_on_market = 0
                condition = 'Unknown'
                for detail in details:
                    if 'bd' in detail:
                        beds = detail.split()[0]
                    if 'ba' in detail:
                        baths = detail.split()[0]
                    if 'day' in detail.lower():
                        try:
                            days_on_market = int(detail.split()[0])
                        except:
                            pass
                # Get listing link
                link_elem = card.find_element(By.TAG_NAME, 'a')
                url = link_elem.get_attribute('href') if link_elem else 'N/A'
                listing = {
                    'address': address,
                    'price': price,
                    'beds': beds,
                    'baths': baths,
                    'days_on_market': days_on_market,
                    'condition': condition,
                    'url': url,
                    'property_type': property_type,
                    'listed_date': datetime.now().isoformat()
                }
                listings.append(listing)
            except Exception as e:
                logger.warning(f"Error extracting listing: {e}")
                continue
        return listings
    except Exception as e:
        logger.error(f"Error scraping Zillow: {e}")
        return []
    finally:
        driver.quit()

def filter_listings(listings, property_type, history):
    """Filter listings by criteria and history."""
    filtered = []
    for listing in listings:
        address_key = listing.get('address', '').lower()
            # Skip duplicates
        if address_key in history:
            continue
        # Filter by days on market
        if listing.get('days_on_market', 0) < SEARCH_CONFIG['min_days_on_market']:
            continue
        # Filter by price
        if property_type == 'residential':
            config = SEARCH_CONFIG['residential']
            if not (config['min_price'] <= listing.get('price', 0) <= config['max_price']):
                continue
        elif property_type == 'multifamily':
            config = SEARCH_CONFIG['multifamily']
            if not (config['min_price'] <= listing.get('price', 0) <= config['max_price']):
                continue
        elif property_type == 'commercial':
            config = SEARCH_CONFIG['commercial']
            if not (config['min_price'] <= listing.get('price', 0) <= config['max_price']):
                continue
        filtered.append(listing)
    return filtered

def generate_csv_report(listings, property_type):
    """Generate CSV report."""
    if not listings:
        logger.info(f"No new {property_type} listings found.")
        return
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    filename = OUTPUT_DIR / f"{property_type}_{timestamp}.csv"
    fieldnames = ['Address', 'Price', 'Beds', 'Baths', 'Days on Market', 'Condition', 'Listed Date', 'URL']
    with open(filename, 'w', newline='') as csvfile:
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
            for listing in listings:
            writer.writerow({
                'Address': listing.get('address', ''),
                'Price': f"${listing.get('price', 0):,.0f}",
                'Beds': listing.get('beds', 'N/A'),
                'Baths': listing.get('baths', 'N/A'),
                'Days on Market': listing.get('days_on_market', ''),
                'Condition': listing.get('condition', ''),
                'Listed Date': listing.get('listed_date', ''),
                'URL': listing.get('url', '')
            })
    logger.info(f"Generated report: {filename}")

def run_search():
    """Execute the property search."""
    logger.info("=" * 50)
    logger.info("Starting Zillow property search...")
    logger.info("=" * 50)
    history = load_history()
    timestamp = datetime.now().isoformat()
    # Search residential
    logger.info("\n[1/3] Searching residential properties...")
    residential_listings = scrape_zillow(
        'residential',
        SEARCH_CONFIG['residential']['min_price'],
        SEARCH_CONFIG['residential']['max_price']
    )
    filtered_residential = filter_listings(residential_listings, 'residential', history)
    if filtered_residential:
        for listing in filtered_residential:
            history[listing.get('address', '').lower()] = timestamp
        generate_csv_report(filtered_residential, 'residential')
        logger.info(f"✓ Found {len(filtered_residential)} new residential listings")
    else:
        logger.info("✗ No new residential listings")
    time.sleep(2)  # Be respectful to Zillow
    # Search multifamily
    logger.info("\n[2/3] Searching multifamily properties...")
    multifamily_listings = scrape_zillow(
        'multifamily',
        SEARCH_CONFIG['multifamily']['min_price'],
        SEARCH_CONFIG['multifamily']['max_price']
    )
    filtered_multifamily = filter_listings(multifamily_listings, 'multifamily', history)
    if filtered_multifamily:
        for listing in filtered_multifamily:
            history[listing.get('address', '').lower()] = timestamp
        generate_csv_report(filtered_multifamily, 'multifamily')
        logger.info(f"✓ Found {len(filtered_multifamily)} new multifamily listings")
    else:
        logger.info("✗ No new multifamily listings")
    time.sleep(2)
    # Search commercial
    logger.info("\n[3/3] Searching commercial properties...")
    commercial_listings = scrape_zillow(
        'commercial',
        SEARCH_CONFIG['commercial']['min_price'],
        SEARCH_CONFIG['commercial']['max_price'],
        'commercial'
    )
    filtered_commercial = filter_listings(commercial_listings, 'commercial', history)
    if filtered_commercial:
        for listing in filtered_commercial:
            history[listing.get('address', '').lower()] = timestamp
        generate_csv_report(filtered_commercial, 'commercial')
        logger.info(f"✓ Found {len(filtered_commercial)} new commercial listings")
    else:
        logger.info("✗ No new commercial listings")
    # Save history
    save_history(history)
    logger.info("\n" + "=" * 50)
    logger.info("Search complete. Reports saved to realtor_reports/")
    logger.info("=" * 50 + "\n")

def start_scheduler():
    """Start scheduler for 3.5 day intervals."""
    scheduler = BackgroundScheduler()
    scheduler.add_job(run_search, 'interval', hours=84, id='zillow_scrape')
    scheduler.start()
    logger.info("Scheduler started. Search will run every 3.5 days.")
    logger.info("Press Ctrl+C to stop.\n")
    try:
        while True:
            pass
    except KeyboardInterrupt:
        scheduler.shutdown()
        logger.info("\nScheduler stopped.")

if __name__ == '__main__':
    # Run search immediately
    run_search()
     # Start scheduler
    start_scheduler()
