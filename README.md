# Aspectos destacados de Notion Kindle

Este proyecto permite leer subrayados de Kindle desde un archivo `My Clippings.txt` y cargarlos en una base de datos de Notion.

## Requisitos

- Python 3.6 o superior
- Una cuenta de Notion
- Un token de integración de Notion

## Instalación

1. Clonar el repositorio:

    ```bash
    git clone https://github.com/tu-usuario/notion-kindle-highlights.git
    cd notion-kindle-highlights
    ```

2. Crea y activa un entorno virtual (opcional pero recomendado):

    ```bash
    python -m venv venv
    source venv/bin/activate  # En Windows usa `venv\Scripts\activate`
    ```

3. Instala las dependencias:

    ```bash
    pip install -r requirements.txt
    ```

4. Coloca tu archivo `My Clippings.txt` en la carpeta `clippings`.

## Configuración

5. Crea un archivo `config.json` en la raíz del proyecto con el siguiente contenido:

    ```json
    {
      "token": "tu_token_de_notion",
      "database_id": "tu_id_de_base_de_datos",
      "file_path": "clippings/My Clippings.txt"
    }
    ```

## Obtener un token de integración de Notion

1. Ve a [Notion Integrations](https://www.notion.so/my-integrations).
2. Crea una nueva integración y copia el token de integración.
3. Invita a la integración a tu base de datos de Notion.

## Uso

Ejecuta el script principal:

```bash
python main.py


```bash
### `main.py`

```python
import requests
import json
import re

def create_notion_page(token, database_id, book, highlights, page, date, author=None, status=None):
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

    properties = {
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

    if author:
        properties["Author"] = {
            "rich_text": [
                {
                    "type": "text",
                    "text": {
                        "content": author
                    }
                }
            ]
        }

    if status:
        properties["Status"] = {
            "select": {
                "name": status
            }
        }

    data = {
        "parent": {"database_id": database_id},
        "properties": properties
    }

    print(f"Sending data to Notion: {json.dumps(data, indent=2)}")

    try:
        response = requests.post(url, headers=headers, data=json.dumps(data))
        response.raise_for_status()
        print(f"Response Status Code: {response.status_code}")
        print(f"Response Text: {response.text}")
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
            date = "2024-08-03T00:00:00Z"  # Cambia esto si es necesario

            # Extraer el número de página usando expresión regular
            page_match = re.search(r'\b\d+\b', page_info)
            page = int(page_match.group(0)) if page_match else None

            # Extraer la fecha real si está presente en el archivo
            date_match = re.search(r'\d{2}/\d{2}/\d{4}', entry)
            if date_match:
                date = date_match.group(0)
                # Convertir la fecha a formato ISO 8601
                date = f"{date[-4:]}-{date[3:5]}-{date[:2]}T00:00:00Z"

            highlights = [line.strip() for line in highlight_lines]
            parsed_entries.append((book, highlights, page, date))

    return parsed_entries

if __name__ == "__main__":
    import os

    token = os.getenv('NOTION_API_KEY')
    database_id = os.getenv('DATABASE_ID')
    file_path = 'clippings/My Clippings.txt'
    
    clippings = read_clippings(file_path)
    
    for clipping in clippings:
        book, highlights, page, date = clipping
        create_notion_page(token, database_id, book, highlights, page, date)
