## `flask_project/categories/utils.py`

```python
import os
from werkzeug.utils import secure_filename
from config import Config
import json, random

def safe_category_name(name):
    return secure_filename(name)

def allowed_template_filename(filename):
    _, ext = os.path.splitext(filename.lower())
    return ext in Config.ALLOWED_TEMPLATE_EXTS

def get_json_path(category):
    return os.path.join(Config.DATA_DIR, f"{category}.json")

def get_template_path(category):
    return os.path.join(Config.TEMPLATE_DIR, f"{category}.txt")

def list_alert_files():
    if not os.path.exists(Config.ALERTS_DIR):
        return []
    return [f[:-5] for f in os.listdir(Config.ALERTS_DIR) if f.endswith('.json')]

def pick_from_alert(alert_key):
    fpath = os.path.join(Config.ALERTS_DIR, f"{alert_key}.json")
    if not os.path.exists(fpath):
        return ''
    with open(fpath, "r", encoding="utf-8") as f:
        data = json.load(f)
    if isinstance(data, dict):
        lines = data.get("lines", [])
    elif isinstance(data, list):
        lines = data
    else:
        lines = []
    return random.choice(lines) if lines else ''

```