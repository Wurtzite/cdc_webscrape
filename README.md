Code to scrape from CDC NNDSS
# cdc_webscrape
NDC webscrape project
import requests
from bs4 import BeautifulSoup
import os
import time
import re
from docx import Document

main_url = "https://ndc.services.cdc.gov/"

def get_disease_links():
    response = requests.get(main_url)
    if response.status_code != 200:
        print("❌ Failed to fetch main page")
        return []

    soup = BeautifulSoup(response.text, "html.parser")
    links = soup.find_all("a", href=True)

    disease_links = [
        link["href"] if link["href"].startswith("https") else f"https://ndc.services.cdc.gov{link['href']}"
        for link in links if "/conditions/" in link["href"]
    ]

    return list(set(disease_links))

disease_links = get_disease_links()

with open("disease_links.txt", "w") as file:
    for link in disease_links:
        file.write(link + "\n")

print(f"✅ Extracted {len(disease_links)} disease links. Saved to 'disease_links.txt'.")

with open("disease_links.txt", "r") as file:
    disease_links = [line.strip() for line in file.readlines()]

extracted_links = set()

def extract_links(url):
    try:
        response = requests.get(url)
        if response.status_code != 200:
            print(f"❌ Failed to fetch: {url}")
            return

        soup = BeautifulSoup(response.text, "html.parser")

        for link in soup.find_all("a", href=True):
            full_link = link["href"]
            if not full_link.startswith("http"):
                full_link = "https://ndc.services.cdc.gov" + full_link
            
            extracted_links.add(full_link)
            print(f"🔗 Found: {full_link}")

    except Exception as e:
        print(f"⚠️ Error scraping {url}: {e}")

for idx, disease_url in enumerate(disease_links):
    print(f"📌 Scraping ({idx+1}/{len(disease_links)}): {disease_url}")
    extract_links(disease_url)
    time.sleep(1)

with open("extracted_links.txt", "w") as file:
    for link in extracted_links:
        file.write(link + "\n")

print(f"✅ Extracted {len(extracted_links)} links. Saved to 'extracted_links.txt'.")

with open("extracted_links.txt", "r") as file:
    print(file.read())

input_file = "extracted_links.txt"
output_file = "filtered_gov_links.txt"

gov_links = []
with open(input_file, "r") as file:
    for line in file:
        line = line.strip()
        if ".gov" in line and ".com" not in line:
            gov_links.append(line)

with open(output_file, "w") as file:
    for link in gov_links:
        file.write(link + "\n")

print(f"Filtered .gov links (without .com) saved to: {os.path.abspath(output_file)}")

input_file = "filtered_gov_links.txt"
output_folder = "CDC_Notifiable_Diseases"

os.makedirs(output_folder, exist_ok=True)

def clean_text(text):
    return re.sub(r"[\x00-\x08\x0B-\x1F\x7F]", "", text)

def extract_text_from_url(url):
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, "html.parser")
        text = soup.get_text(separator="\n", strip=True)
        return clean_text(text)
    except requests.exceptions.RequestException as e:
        print(f"❌ Error scraping {url}: {e}")
        return None

with open(input_file, "r") as file:
    gov_links = [line.strip() for line in file if line.strip()]

for idx, url in enumerate(gov_links, start=1):
    print(f"📌 Scraping ({idx}/{len(gov_links)}): {url}")

    text_content = extract_text_from_url(url)
    
    if text_content:
        doc = Document()
        doc.add_heading(f"Extracted Content from {url}", level=1)
        doc.add_paragraph(text_content)

        file_name = url.replace("https://", "").replace("http://", "").replace("/", "_").replace(".", "_") + ".docx"
        file_path = os.path.join(output_folder, file_name)

        doc.save(file_path)
        print(f"✅ Saved: {file_path}")

print("\n🎯 All .gov pages scraped and saved successfully!")
