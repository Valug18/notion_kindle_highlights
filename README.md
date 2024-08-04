
### `main.py`

```python
import requests
import json
import re

def load_config():
    with open('config.json', 'r') as config_file:
        return json.load(config_file)

def get_existing_entries(token, database_id):
    url = f'https://api.notion.com/v1/databases/{database_id}/query'
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Notion-Version": "2022-06-28"
    }

    response = requests.post(url, headers=headers, json={})
    response.raise_for_status()
    data = response.json()

    entries = {}
    for result in data.get('results', []):
        book_title = result['properties'].get('Book', {}).get('title', [{}])[0].get('text', {}).get('content', '')
        page_number = result['properties'].get('Page', {}).get('number', None)
        date = result['properties'].get('Date', {}).get('date', {}).get('start', '')
        
        key = (book_title, page_number, date)
        entries[key] = result['id']
    
    return entries

def create_notion_page(token, database_id, book, highlights, page, date):
    url = 'https://api.notion.com/v1/pages'
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Notion-Version": "2022-06-28"
    }

    rich_text_highlights = []
    for highlight in highlights:
        rich_text_highlights.append({
            "type": "text",
            "text": {
                "content": highlight
            }
        })

    data = {
        "parent": {"database_id": database_id},
        "properties": {
            "Book": {
                "title": [
                    {
                        "type": "text",
                        "text": {
                            "content": book
                        }
                    }
                ]
            },
            "Date": {
                "date": {
                    "start": date
                }
            },
            "Page": {
                "number": page if page is not None else 0
            },
            "Highlight": {
                "rich_text": rich_text_highlights
            }
        }
    }

    print(f"Sending data to Notion: {json.dumps(data, indent=2)}")

    try:
        response = requests.post(url, headers=headers, data=json.dumps(data))
        response.raise_for_status()
        print(f"Response Status Code: {response.status_code}")
        print(f"Response Text: {response.text}")
        return response.json().get('id')
    except requests.exceptions.HTTPError as err:
        print(f"HTTP error occurred: {err}")
        print(f"Response Status Code: {response.status_code}")
        print(f"Response Text: {response.text}")
    except Exception as err:
        print(f"Other error occurred: {err}")

def read_clippings(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        content = file.read()

    entries = content.split('==========')
    parsed_entries = []

    for entry in entries:
        lines = entry.strip().split('\n')
        if len(lines) >= 3:
            book = lines[0].strip()
            page_info = lines[1].strip()
            highlight_lines = lines[2:]
            date = "2024-08-03T00:00:00Z"

            page_match = re.search(r'\b\d+\b', page_info)
            page = int(page_match.group(0)) if page_match else None

            date_match = re.search(r'\d{2}/\d{2}/\d{4}', entry)
            if date_match:
                date = date_match.group(0)
                date = f"{date[-4:]}-{date[3:5]}-{date[:2]}T00:00:00Z"

            highlights = [line.strip() for line in highlight_lines]
            parsed_entries.append((book, highlights, page, date))

    return parsed_entries

if __name__ == "__main__":
    config = load_config()
    token = config['token']
    database_id = config['database_id']
    file_path = config['file_path']
    
    existing_entries = get_existing_entries(token, database_id)
    clippings = read_clippings(file_path)
    
    for clipping in clippings:
        book, highlights, page, date = clipping
        key = (book, page, date)

        if key not in existing_entries:
            page_id = create_notion_page(token, database_id, book, highlights, page, date)
            print(f"Created page with ID: {page_id}")
        else:
            print(f"Entry already exists for book: {book}, page: {page}, date: {date}")
