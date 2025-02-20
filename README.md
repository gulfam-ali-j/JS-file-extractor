
import re
import requests
import hashlib
import difflib
from bs4 import BeautifulSoup
import json
import os




Step 1 : To install this tool just git clone 

Step 2 : Make it excutable : 

        #chmod +x api_monitor.py

Step 3 : Run with arguments 
         
     # Configurationpython3 api_monitor.py -u https://target.com -o old_endpoints.json -t 10

TARGET_URL = "https://target.com"
OLD_DATA_FILE = "old_endpoints.json"
HEADERS = {"User-Agent": "Mozilla/5.0"}

# Step 1: Fetch HTML page and extract JS file
response = requests.get(TARGET_URL, headers=HEADERS)
soup = BeautifulSoup(response.text, 'html.parser')

# Extract JavaScript file (assumes it follows a pattern like main.[hash].js)
js_files = [script["src"] for script in soup.find_all("script") if "main" in script.get("src", "")]
if not js_files:
    print("No JS files found!")
    exit()

js_url = TARGET_URL + js_files[0]
print(f"Extracted JS file: {js_url}")

# Step 2: Download and parse JavaScript file
js_response = requests.get(js_url, headers=HEADERS)
js_content = js_response.text

# Step 3: Extract API Endpoints, GraphQL Queries, and Feature Flags
endpoints = set(re.findall(r'"(https?://[^"]+)"', js_content))
graphql_queries = set(re.findall(r'query\s+([a-zA-Z0-9_]+)', js_content))
graphql_mutations = set(re.findall(r'mutation\s+([a-zA-Z0-9_]+)', js_content))
feature_flags = set(re.findall(r'FEATURE_FLAG_[A-Z_]+', js_content))

data = {
    "endpoints": list(endpoints),
    "graphql_queries": list(graphql_queries),
    "graphql_mutations": list(graphql_mutations),
    "feature_flags": list(feature_flags)
}

# Step 4: Compare with old data
if os.path.exists(OLD_DATA_FILE):
    with open(OLD_DATA_FILE, "r") as f:
        old_data = json.load(f)
else:
    old_data = {"endpoints": [], "graphql_queries": [], "graphql_mutations": [], "feature_flags": []}

def diff_lists(old, new):
    return list(set(new) - set(old))

new_endpoints = diff_lists(old_data["endpoints"], data["endpoints"])
new_graphql_queries = diff_lists(old_data["graphql_queries"], data["graphql_queries"])
new_graphql_mutations = diff_lists(old_data["graphql_mutations"], data["graphql_mutations"])
new_feature_flags = diff_lists(old_data["feature_flags"], data["feature_flags"])

if new_endpoints or new_graphql_queries or new_graphql_mutations or new_feature_flags:
    print("\n🔔 ALERT: New Changes Detected! 🔔")
    if new_endpoints:
        print(f"New Endpoints: {new_endpoints}")
    if new_graphql_queries:
        print(f"New GraphQL Queries: {new_graphql_queries}")
    if new_graphql_mutations:
        print(f"New GraphQL Mutations: {new_graphql_mutations}")
    if new_feature_flags:
        print(f"New Feature Flags: {new_feature_flags}")
else:
    print("No new changes detected.")

# Step 5: Save new data for future comparison
with open(OLD_DATA_FILE, "w") as f:
    json.dump(data, f, indent=4)
