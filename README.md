# riverview-list-crawler (.1)
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.firefox.options import Options
from bs4 import BeautifulSoup
import re
import time
import csv

def initialize_driver():
    driver_path = r'C:\Users\Phatboi\Downloads\geckodriver-v0.34.0-win32\geckodriver.exe'
    binary_path = r'C:\Program Files\Mozilla Firefox\firefox.exe'
    service = Service(executable_path=driver_path)
    options = Options()
    options.binary_location = binary_path
    driver = webdriver.Firefox(service=service, options=options)
    print("Driver initialized.")
    return driver

def search_and_extract(driver, search_term, base_url):
    try:
        driver.get(base_url)
        WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.NAME, "q")))
        search_box = driver.find_element(By.NAME, "q")
        search_box.clear()
        search_box.send_keys(search_term)
        search_box.send_keys(Keys.RETURN)
        time.sleep(2)
        links = WebDriverWait(driver, 20).until(
            EC.presence_of_all_elements_located((By.PARTIAL_LINK_TEXT, "Visit Website"))
        )
        return [result.get_attribute('href') for result in links]
    except Exception as e:
        print(f"Error retrieving results for {search_term}: {e}")
        return []

def extract_info_from_websites(driver, websites):
    email_regex = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,7}\b'
    for url in websites:
        try:
            driver.get(url)
            soup = BeautifulSoup(driver.page_source, 'html.parser')
            emails = set(re.findall(email_regex, soup.get_text(), re.IGNORECASE))
            email = '; '.join(emails) if emails else 'No email found'
            print(f"Visiting {url}: Found emails: {email}")
            yield (url, email)
        except Exception as e:
            print(f"Error visiting {url}: {e}")
            yield (url, 'Error visiting site')

def main():
    base_url = "https://www.riverviewchamber.com/list/"
    driver = initialize_driver()
    driver.get(base_url)

    with open('results.csv', 'w', newline='') as file:
        writer = csv.writer(file)
        writer.writerow(['URL', 'Email'])

        try:
            for char in 'ABCDEFGHIJKLMNOPQRSTUVWXYZ':
                print(f"Processing {char}")
                links = search_and_extract(driver, char, base_url)
                for data in extract_info_from_websites(driver, links):
                    writer.writerow(data)
        except KeyboardInterrupt:
            print("Interrupted by user, saving data collected so far...")
        except Exception as e:
            print(f"General error during processing: {e}")
        finally:
            driver.quit()
            print("Data collection completed. Check results.csv for the output.")

if __name__ == "__main__":
    main()
