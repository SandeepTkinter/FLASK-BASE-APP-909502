# Project code dump

This file contains the source code extracted from `flask_project.zip`.

> Generated automatically.

## `flask_project/.idea/workspace.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="AutoImportSettings">
    <option name="autoReloadType" value="SELECTIVE" />
  </component>
  <component name="ChangeListManager">
    <list default="true" id="d0db8a04-7f20-45f1-831c-06ec239637b2" name="Changes" comment="" />
    <option name="SHOW_DIALOG" value="false" />
    <option name="HIGHLIGHT_CONFLICTS" value="true" />
    <option name="HIGHLIGHT_NON_ACTIVE_CHANGELIST" value="false" />
    <option name="LAST_RESOLUTION" value="IGNORE" />
  </component>
  <component name="FileTemplateManagerImpl">
    <option name="RECENT_TEMPLATES">
      <list>
        <option value="HTML File" />
      </list>
    </option>
  </component>
  <component name="ProjectColorInfo">{
  &quot;associatedIndex&quot;: 5
}</component>
  <component name="ProjectId" id="31363ly2Ccmj8ISIQzoeqi0Gkzy" />
  <component name="ProjectViewState">
    <option name="hideEmptyMiddlePackages" value="true" />
    <option name="showLibraryContents" value="true" />
  </component>
  <component name="PropertiesComponent">{
  &quot;keyToString&quot;: {
    &quot;DefaultHtmlFileTemplate&quot;: &quot;HTML File&quot;,
    &quot;ModuleVcsDetector.initialDetectionPerformed&quot;: &quot;true&quot;,
    &quot;Python.app.executor&quot;: &quot;Run&quot;,
    &quot;RunOnceActivity.ShowReadmeOnStart&quot;: &quot;true&quot;,
    &quot;last_opened_file_path&quot;: &quot;C:/Users/Sandeep/Downloads/flask_project&quot;
  }
}</component>
  <component name="RecentsManager">
    <key name="CopyFile.RECENT_KEYS">
      <recent name="C:\Users\Sandeep\Downloads\flask_project" />
    </key>
  </component>
  <component name="RunManager">
    <configuration name="app" type="PythonConfigurationType" factoryName="Python" temporary="true" nameIsGenerated="true">
      <module name="flask_project" />
      <option name="ENV_FILES" value="" />
      <option name="INTERPRETER_OPTIONS" value="" />
      <option name="PARENT_ENVS" value="true" />
      <envs>
        <env name="PYTHONUNBUFFERED" value="1" />
      </envs>
      <option name="SDK_HOME" value="" />
      <option name="WORKING_DIRECTORY" value="$PROJECT_DIR$" />
      <option name="IS_MODULE_SDK" value="true" />
      <option name="ADD_CONTENT_ROOTS" value="true" />
      <option name="ADD_SOURCE_ROOTS" value="true" />
      <option name="SCRIPT_NAME" value="$PROJECT_DIR$/app.py" />
      <option name="PARAMETERS" value="" />
      <option name="SHOW_COMMAND_LINE" value="false" />
      <option name="EMULATE_TERMINAL" value="false" />
      <option name="MODULE_MODE" value="false" />
      <option name="REDIRECT_INPUT" value="false" />
      <option name="INPUT_FILE" value="" />
      <method v="2" />
    </configuration>
    <recent_temporary>
      <list>
        <item itemvalue="Python.app" />
      </list>
    </recent_temporary>
  </component>
  <component name="SharedIndexes">
    <attachedChunks>
      <set>
        <option value="bundled-python-sdk-4c141bd692a7-e2d783800521-com.jetbrains.pycharm.community.sharedIndexes.bundled-PC-251.26927.90" />
      </set>
    </attachedChunks>
  </component>
  <component name="TaskManager">
    <task active="true" id="Default" summary="Default task">
      <changelist id="d0db8a04-7f20-45f1-831c-06ec239637b2" name="Changes" comment="" />
      <created>1754740211276</created>
      <option name="number" value="Default" />
      <option name="presentableId" value="Default" />
      <updated>1754740211276</updated>
    </task>
    <servers />
  </component>
</project>
```

## `flask_project/__pycache__/config.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/__pycache__/dynamic_funcs.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/__pycache__/extensions.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/alerts/__pycache__/routes.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/alerts/__pycache__/utils.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/alerts/routes.py`

```python
from flask import Blueprint, render_template, request, redirect, url_for, flash
import os
import json

bp = Blueprint("alerts", __name__, template_folder="../templates")

ALERTS_DIR = os.path.join("data", "alerts")  # Adjust if your alerts are stored elsewhere

@bp.route("/manage_alerts", methods=["GET", "POST"])
def manage_alerts():
    if request.method == "POST":
        key = request.form["alert_key"].strip()
        lines = request.form["alert_text"].splitlines()
        os.makedirs(ALERTS_DIR, exist_ok=True)
        with open(os.path.join(ALERTS_DIR, f"{key}.json"), "w", encoding="utf-8") as f:
            json.dump(lines, f, indent=2)
        flash(f"Alert '{key}' saved successfully!", "success")
        return redirect(url_for("alerts.manage_alerts"))

    alerts = []
    if os.path.exists(ALERTS_DIR):
        for fname in os.listdir(ALERTS_DIR):
            if fname.endswith(".json"):
                with open(os.path.join(ALERTS_DIR, fname), "r", encoding="utf-8") as f:
                    lines = json.load(f)
                alerts.append({"key": fname[:-5], "lines": lines})

    return render_template("manage_alerts.html", alerts=alerts)

@bp.route("/alerts/delete/<key>", methods=["POST"])
def delete_alert(key):
    file_path = os.path.join(ALERTS_DIR, f"{key}.json")
    if os.path.exists(file_path):
        os.remove(file_path)
        flash(f"Alert '{key}' deleted successfully!", "success")
    else:
        flash("Alert not found", "error")
    return redirect(url_for("alerts.manage_alerts"))

@bp.route("/alerts/edit/<key>", methods=["GET", "POST"])
def edit_alert(key):
    file_path = os.path.join(ALERTS_DIR, f"{key}.json")
    if not os.path.exists(file_path):
        flash("Alert not found", "error")
        return redirect(url_for("alerts.manage_alerts"))

    if request.method == "POST":
        lines = request.form["alert_text"].splitlines()
        with open(file_path, "w", encoding="utf-8") as f:
            json.dump(lines, f, indent=2)
        flash(f"Alert '{key}' updated successfully!", "success")
        return redirect(url_for("alerts.manage_alerts"))

    with open(file_path, "r", encoding="utf-8") as f:
        lines = json.load(f)

    alert = {"key": key, "lines": lines}
    return render_template("edit_alert.html", alert=alert)

```

## `flask_project/alerts/utils.py`

```python
import os
import json
from werkzeug.utils import secure_filename
from extensions import logger
from config import Config

ALERTS_DIR = Config.ALERTS_DIR

def load_alerts():
    alerts = []
    if not os.path.exists(ALERTS_DIR):
        return alerts
    for fname in sorted(os.listdir(ALERTS_DIR)):
        if fname.endswith(".json"):
            key = fname[:-5]
            fpath = os.path.join(ALERTS_DIR, fname)
            try:
                with open(fpath, "r", encoding="utf-8") as f:
                    data = json.load(f)
                    lines = data.get("lines", []) if isinstance(data, dict) else []
                    alerts.append({"key": key, "lines": lines})
            except Exception:
                logger.exception("Failed reading alert file %s", fname)
    return alerts

def save_alert(key, text):
    fname = secure_filename(key) + ".json"
    if not os.path.exists(ALERTS_DIR):
        os.makedirs(ALERTS_DIR, exist_ok=True)
    fpath = os.path.join(ALERTS_DIR, fname)
    lines = [line.rstrip() for line in text.splitlines() if line.strip()]
    with open(fpath, "w", encoding="utf-8") as f:
        json.dump({"lines": lines}, f, indent=2, ensure_ascii=False)

def delete_alert_file(key):
    fname = secure_filename(key) + ".json"
    fpath = os.path.join(ALERTS_DIR, fname)
    if os.path.exists(fpath):
        os.remove(fpath)

```

## `flask_project/app.py`

```python
import os
from flask import Flask, render_template
from config import Config
from extensions import logger

def create_app():
    app = Flask(__name__, static_folder='static', template_folder='templates')
    app.config.from_object(Config)

    # ensure directories
    os.makedirs(Config.DATA_DIR, exist_ok=True)
    os.makedirs(Config.TEMPLATE_DIR, exist_ok=True)
    os.makedirs(Config.ALERTS_DIR, exist_ok=True)

    # register blueprints
    from alerts.routes import bp as alerts_bp
    from categories.routes import bp as categories_bp

    app.register_blueprint(alerts_bp)
    app.register_blueprint(categories_bp)

    # small index route to list categories (keeps original behavior)
    @app.route('/')
    def index():
        categories = []
        for fname in os.listdir(app.config['DATA_DIR']):
            if fname.endswith('.json'):
                categories.append(fname[:-5])
        categories.sort()
        return render_template('index.html', categories=categories)

    return app

if __name__ == '__main__':
    create_app().run(host='0.0.0.0', port=int(os.environ.get('PORT', 5000)))

```

## `flask_project/categories/__pycache__/routes.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/categories/__pycache__/utils.cpython-311.pyc`

_Binary or non-text file — skipped inclusion._

## `flask_project/categories/routes.py`

```python
import os
import io
import json
from zipfile import ZipFile, ZIP_DEFLATED
from functools import wraps
from flask import Blueprint, render_template, request, redirect, url_for, flash, current_app, send_file
from config import Config
from dynamic_funcs import DYNAMIC_FUNCTIONS
from .utils import safe_category_name, allowed_template_filename, get_json_path, get_template_path

bp = Blueprint('categories', __name__)

def validate_count(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            count = int(request.values.get("count", current_app.config.get('DEFAULT_COUNT', Config.DEFAULT_COUNT)))
        except (ValueError, TypeError):
            count = current_app.config.get('DEFAULT_COUNT', Config.DEFAULT_COUNT)
        count = max(1, min(count, current_app.config.get('MAX_COUNT', Config.MAX_COUNT)))
        request._validated_count = count
        return func(*args, **kwargs)
    return wrapper

@bp.route('/create', methods=['GET', 'POST'])
def create():
    if request.method == 'POST':
        category = request.form.get('category', '').strip()
        uploaded = request.files.get('file')
        keys = request.form.getlist('key')
        types = request.form.getlist('type')
        static_vals = request.form.getlist('value_static')
        dynamic_vals = request.form.getlist('value_dynamic')

        if not category:
            flash('Category name is required.')
            return redirect(url_for('categories.create'))

        category = safe_category_name(category)
        if uploaded is None or uploaded.filename == '':
            flash('Template file required.')
            return redirect(url_for('categories.create'))

        if not allowed_template_filename(uploaded.filename):
            flash('Template file type not allowed. Use .txt/.md/.tmpl')
            return redirect(url_for('categories.create'))

        template_path = get_template_path(category)
        uploaded.save(template_path)

        kv_pairs = {}
        for i, k in enumerate(keys):
            k = k.strip()
            if not k:
                continue
            t = types[i] if i < len(types) else 'static'
            if t == 'static':
                val = static_vals[i] if i < len(static_vals) else ''
                kv_pairs[k] = {'type': 'static', 'value': val}
            else:
                dyn = dynamic_vals[i] if i < len(dynamic_vals) else ''
                kv_pairs[k] = {'type': 'dynamic', 'value': dyn}

        json_path = get_json_path(category)
        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(kv_pairs, f, indent=2, ensure_ascii=False)

        flash(f"Category '{category}' created.")
        return redirect(url_for('index'))

    return render_template('create.html', dynamic_funcs=sorted(DYNAMIC_FUNCTIONS.keys()))

@bp.route('/edit/<category>', methods=['GET', 'POST'])
def edit(category):
    category = safe_category_name(category)
    json_path = get_json_path(category)
    template_path = get_template_path(category)
    if not os.path.exists(json_path):
        flash('Category not found.')
        return redirect(url_for('index'))

    if request.method == 'POST':
        keys = request.form.getlist('key')
        types = request.form.getlist('type')
        static_vals = request.form.getlist('value_static')
        dynamic_vals = request.form.getlist('value_dynamic')

        kv_pairs = {}
        for i, k in enumerate(keys):
            k = k.strip()
            if not k:
                continue
            t = types[i] if i < len(types) else 'static'
            if t == 'static':
                val = static_vals[i] if i < len(static_vals) else ''
                kv_pairs[k] = {'type': 'static', 'value': val}
            else:
                dyn = dynamic_vals[i] if i < len(dynamic_vals) else ''
                kv_pairs[k] = {'type': 'dynamic', 'value': dyn}

        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(kv_pairs, f, indent=2, ensure_ascii=False)
        flash('Saved.')
        return redirect(url_for('index'))

    with open(json_path, 'r', encoding='utf-8') as f:
        kv_pairs = json.load(f)
    template_exists = os.path.exists(template_path)
    return render_template('edit.html', category=category, kv_pairs=kv_pairs, template_exists=template_exists, dynamic_funcs=sorted(DYNAMIC_FUNCTIONS.keys()))

@bp.route('/delete/<category>', methods=['POST'])
def delete(category):
    category = safe_category_name(category)
    json_path = get_json_path(category)
    template_path = get_template_path(category)
    try:
        if os.path.exists(json_path):
            os.remove(json_path)
        if os.path.exists(template_path):
            os.remove(template_path)
        flash('Deleted.')
    except Exception:
        flash('Error deleting.')
    return redirect(url_for('index'))

def generate_zip_bytes(category, kv_data, template_text, count):
    mem = io.BytesIO()
    with ZipFile(mem, 'w', compression=ZIP_DEFLATED) as zf:
        for i in range(1, count + 1):
            content = template_text
            for k, v in kv_data.items():
                if isinstance(v, dict) and v.get('type') == 'dynamic':
                    func_name = v.get('value')
                    func = DYNAMIC_FUNCTIONS.get(func_name)
                    replacement = func() if callable(func) else ''
                else:
                    replacement = v.get('value') if isinstance(v, dict) else str(v)
                content = content.replace(k, replacement)
            zf.writestr(f"{category}_file_{i}.txt", content)
    mem.seek(0)
    return mem.read()

@bp.route('/download', methods=['GET'])
@validate_count
def download():
    category = request.args.get('category', '')
    if not category:
        flash('Missing category')
        return redirect(url_for('index'))

    category = safe_category_name(category)
    json_path = get_json_path(category)
    template_path = get_template_path(category)
    if not os.path.exists(json_path) or not os.path.exists(template_path):
        flash('Category data/template missing.')
        return redirect(url_for('index'))

    with open(json_path, 'r', encoding='utf-8') as f:
        kv_data = json.load(f)

    with open(template_path, 'r', encoding='utf-8') as f:
        template_text = f.read()

    count = request._validated_count
    zip_bytes = generate_zip_bytes(category, kv_data, template_text, count)
    filename = f"{category}_files.zip"
    return send_file(
        io.BytesIO(zip_bytes),
        mimetype='application/zip',
        as_attachment=True,
        download_name=filename
    )

```

## `flask_project/categories/utils.py`

```python
import os
from werkzeug.utils import secure_filename
from config import Config

def safe_category_name(name):
    return secure_filename(name)

def allowed_template_filename(filename):
    _, ext = os.path.splitext(filename.lower())
    return ext in Config.ALLOWED_TEMPLATE_EXTS

def get_json_path(category):
    return os.path.join(Config.DATA_DIR, f"{category}.json")

def get_template_path(category):
    return os.path.join(Config.TEMPLATE_DIR, f"{category}.txt")

```

## `flask_project/config.py`

```python
import os

class Config:
    SECRET_KEY = os.environ.get("FLASK_SECRET", "replace-me-in-prod")
    BASE_DIR = os.path.abspath(os.path.dirname(__file__))
    DATA_DIR = os.path.join(BASE_DIR, "data")
    TEMPLATE_DIR = os.path.join(BASE_DIR, "templates_data")
    ALERTS_DIR = os.path.join(BASE_DIR, "alerts_data")
    MAX_CONTENT_LENGTH = 10 * 1024 * 1024   # 10 MB upload limit
    ALLOWED_TEMPLATE_EXTS = {'.txt', '.md', '.tmpl'}
    DEFAULT_COUNT = 5
    MAX_COUNT = 200

```

## `flask_project/data/alerts/1.json`

```json
[
  "1"
]
```

## `flask_project/data/cat.json`

```json
{
  "Bollywood": {
    "type": "static",
    "value": "Sandeep"
  },
  "industry": {
    "type": "dynamic",
    "value": "random_number"
  }
}
```

## `flask_project/dynamic_funcs.py`

```python
import uuid
import random
from datetime import datetime

def random_number():
    return str(random.randint(1000, 9999))

def current_date():
    return datetime.now().strftime("%Y-%m-%d")

def random_uuid():
    return str(uuid.uuid4())

def sequential_counter_factory():
    counter = {"v": 0}
    def gen():
        counter["v"] += 1
        return str(counter["v"])
    return gen

sequential_counter = sequential_counter_factory()

DYNAMIC_FUNCTIONS = {
    "random_number": random_number,
    "current_date": current_date,
    "random_uuid": random_uuid,
    "sequential_counter": sequential_counter
}

```

## `flask_project/extensions.py`

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

```

## `flask_project/static/style.css`

```css
:root {
    --bg: #1e1e1e;
    --fg: #ffffff;
    --card-bg: #2c2c2c;
    --accent: #10a37f;
    --button-bg: #10a37f;
    --button-fg: #ffffff;
    --danger-bg: #d9534f;
}

body {
    margin: 0;
    font-family: 'Segoe UI', sans-serif;
    background-color: var(--bg);
    color: var(--fg);
}

.container {
    max-width: 900px;
    margin: 40px auto;
    padding: 20px;
    text-align: center;
}

.cards {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 20px;
    margin-bottom: 24px;
}

.card {
    background-color: var(--card-bg);
    border-radius: 10px;
    padding: 20px;
    width: 240px;
    box-shadow: 0 0 10px rgba(0,0,0,0.5);
    transition: transform 0.2s ease;
    text-align: center;
}

.card:hover {
    transform: scale(1.03);
}

.counter {
    display: flex;
    justify-content: center;
    align-items: center;
    margin: 10px 0;
}

.counter input {
    width: 60px;
    text-align: center;
    font-size: 16px;
    padding: 5px;
    border-radius: 6px;
    border: none;
    background-color: #fff;
    color: #000;
}

.counter button {
    background-color: var(--button-bg);
    color: var(--button-fg);
    border: none;
    padding: 6px 10px;
    font-size: 18px;
    border-radius: 6px;
    cursor: pointer;
    margin: 0 6px;
}

.button {
    display: inline-block;
    margin-top: 10px;
    background-color: var(--button-bg);
    color: var(--button-fg);
    padding: 10px 16px;
    border-radius: 8px;
    text-decoration: none;
    font-weight: bold;
    cursor: pointer;
    border: none;
    transition: background 0.3s ease;
}

.button:hover {
    background-color: #0d8a6c;
}

.button.danger {
    background-color: var(--danger-bg);
}

.kv-pair {
    display: flex;
    gap: 10px;
    align-items: center;
    justify-content: center;
    margin-bottom: 10px;
    flex-wrap: wrap;
}

.kv-pair input {
    padding: 8px;
    border-radius: 6px;
    border: none;
    font-size: 15px;
    flex: 1 1 150px;
}

.kv-pair button {
    background-color: var(--danger-bg);
    color: #fff;
    border: none;
    padding: 6px 10px;
    border-radius: 6px;
    cursor: pointer;
}

.full-input {
    width: 100%;
    padding: 10px;
    margin-bottom: 5px;
    border-radius: 6px;
    border: none;
    background-color: #fff;
    color: #000;
    font-size: 16px;
    box-sizing: border-box;
}

.form-group {
    display: flex;
    flex-direction: column;
    margin-bottom: 20px;
    align-items: flex-start;
}

.form-actions {
    display: flex;
    gap: 10px;
    margin-top: 30px;
    justify-content: center;
}

/* New unified table style for create/edit */
.form-table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 15px;
}
.form-table th, .form-table td {
    padding: 6px;
    text-align: left;
}
.form-table th {
    background-color: var(--card-bg);
    font-weight: bold;
}
.form-table input.full-input,
.form-table select.full-input {
    width: 100%;
    box-sizing: border-box;
}

/* Flash messages */
.flashes {
    list-style: none;
    padding: 0;
    color: #9bd6b3;
    margin-bottom: 12px;
}

```

## `flask_project/templates/base.html`

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Template Generator</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
  <div class="container">
    <h1>Template Generator</h1>
    {% with messages = get_flashed_messages() %}
      {% if messages %}
        <ul class="flashes">
        {% for m in messages %}
          <li>{{ m }}</li>
        {% endfor %}
        </ul>
      {% endif %}
    {% endwith %}
    {% block content %}{% endblock %}
  </div>
</body>
</html>
```

## `flask_project/templates/create.html`

```html
{% extends "base.html" %}
{% block content %}
<h1>Create New Category</h1>

<form method="post" enctype="multipart/form-data" id="createForm">
  <div class="form-group">
    <label>Category Name</label>
    <input class="full-input" type="text" name="category" required>
  </div>

  <div class="form-group">
    <label>Upload Template File</label>
    <input class="full-input" type="file" name="file" required>
  </div>

  <h3>Placeholders</h3>
  <table id="kvTable" class="form-table">
    <thead>
      <tr>
        <th style="width: 30%">Key</th>
        <th style="width: 20%">Type</th>
        <th style="width: 40%">Value</th>
        <th style="width: 10%">Action</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
  <button type="button" class="button" onclick="addRow()">➕ Add row</button>

  <div class="form-actions">
    <button type="submit" class="button">💾 Save</button>
    <a href="{{ url_for('index') }}" class="button danger">↩️ Cancel</a>
  </div>
</form>

<script>
const dynamicFuncs = {{ dynamic_funcs|tojson }};
function addRow(data) {
  data = data || {"key":"", "type":"static", "value_static":"", "value_dynamic": dynamicFuncs[0] || ""};
  const tbody = document.querySelector("#kvTable tbody");
  const tr = document.createElement("tr");
  tr.innerHTML = `
    <td><input class="full-input" name="key" value="${escapeHtml(data.key)}" required></td>
    <td>
      <select class="full-input" name="type" onchange="toggle(this)">
        <option value="static"${data.type==='static'?' selected':''}>Static</option>
        <option value="dynamic"${data.type==='dynamic'?' selected':''}>Dynamic</option>
      </select>
    </td>
    <td>
      <input class="full-input v_static" name="value_static" value="${escapeHtml(data.value_static || '')}">
      <select class="full-input v_dynamic" name="value_dynamic" style="display:none">
        ${dynamicFuncs.map(f => `<option value="${f}" ${f===data.value_dynamic?'selected':''}>${f}</option>`).join('')}
      </select>
    </td>
    <td><button type="button" class="button danger" onclick="this.closest('tr').remove()">🗑️</button></td>
  `;
  tbody.appendChild(tr);
  toggle(tr.querySelector('select[name="type"]'));
}
function toggle(sel) {
  const tr = sel.closest('tr');
  tr.querySelector('.v_static').style.display = sel.value === 'static' ? '' : 'none';
  tr.querySelector('.v_dynamic').style.display = sel.value === 'dynamic' ? '' : 'none';
}
function escapeHtml(unsafe) {
  return (unsafe + '').replace(/[&<>"'`=\/]/g, s => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;',
    '/': '&#x2F;', '`': '&#x60;', '=': '&#x3D;'
  })[s]);
}
addRow();
</script>
{% endblock %}

```

## `flask_project/templates/edit.html`

```html
{% extends "base.html" %}
{% block content %}
<h1>Edit Category: {{ category }}</h1>

<form method="post" id="editForm">
  <h3>Placeholders</h3>
  <table id="kvTable" class="form-table">
    <thead>
      <tr>
        <th style="width: 30%">Key</th>
        <th style="width: 20%">Type</th>
        <th style="width: 40%">Value</th>
        <th style="width: 10%">Action</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
  <button type="button" class="button" onclick="addRow()">➕ Add row</button>

  <div class="form-actions">
    <button type="submit" class="button">Save</button>
    <a href="{{ url_for('index') }}" class="button danger">Cancel</a>
  </div>
</form>

<script>
const dynamicFuncs = {{ dynamic_funcs|tojson }};
const kv = {{ kv_pairs|tojson }};
function addRow(data) {
  data = data || {"key":"", "type":"static", "value_static":"", "value_dynamic": dynamicFuncs[0] || ""};
  const tbody = document.querySelector("#kvTable tbody");
  const tr = document.createElement("tr");
  tr.innerHTML = `
    <td><input class="full-input" name="key" value="${escapeHtml(data.key)}" required></td>
    <td>
      <select class="full-input" name="type" onchange="toggle(this)">
        <option value="static"${data.type==='static'?' selected':''}>Static</option>
        <option value="dynamic"${data.type==='dynamic'?' selected':''}>Dynamic</option>
      </select>
    </td>
    <td>
      <input class="full-input v_static" name="value_static" value="${escapeHtml(data.value_static || '')}">
      <select class="full-input v_dynamic" name="value_dynamic" style="display:none">
        ${dynamicFuncs.map(f => `<option value="${f}" ${f===data.value_dynamic?'selected':''}>${f}</option>`).join('')}
      </select>
    </td>
    <td><button type="button" class="button danger" onclick="this.closest('tr').remove()">🗑️</button></td>
  `;
  tbody.appendChild(tr);
  toggle(tr.querySelector('select[name="type"]'));
}
function toggle(sel) {
  const tr = sel.closest('tr');
  tr.querySelector('.v_static').style.display = sel.value === 'static' ? '' : 'none';
  tr.querySelector('.v_dynamic').style.display = sel.value === 'dynamic' ? '' : 'none';
}
function escapeHtml(unsafe) {
  return (unsafe + '').replace(/[&<>"'`=\/]/g, s => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;',
    '/': '&#x2F;', '`': '&#x60;', '=': '&#x3D;'
  })[s]);
}
for (const [k, v] of Object.entries(kv || {})) {
  const row = {
    key: k,
    type: v.type || 'static',
    value_static: v.type==='static' ? (v.value || '') : '',
    value_dynamic: v.type==='dynamic' ? (v.value || dynamicFuncs[0]) : dynamicFuncs[0]
  };
  addRow(row);
}
</script>
{% endblock %}

```

## `flask_project/templates/edit_alert.html`

```html
{% extends "base.html" %}
{% block content %}
<div style="position:absolute; left:20px; top:20px;">
  <a href="{{ url_for('alerts.manage_alerts') }}" class="button">↩️ Back</a>
</div>

<h1>Edit Alert: {{ alert.key }}</h1>

<form method="POST">
  <div class="form-group" style="width:100%; text-align:left;">
    <label>Alert Key</label>
    <input name="alert_key" value="{{ alert.key }}" readonly class="full-input">
  </div>

  <div class="form-group" style="width:100%; text-align:left;">
    <label>Alert Text (each line will be stored as a list element)</label>
    <textarea name="alert_text" class="full-input" rows="8" required>{{ alert.lines | join('\n') }}</textarea>
  </div>

  <div class="form-actions">
    <button type="submit" class="button">💾 Update Alert</button>
  </div>
</form>
{% endblock %}

```

## `flask_project/templates/index.html`

```html
{% extends "base.html" %}
{% block content %}

<div style="position:absolute; left:20px; top:20px;">
  <a href="{{ url_for('alerts.manage_alerts') }}" class="button">Manage Alerts</a>
</div>

<h1>📁 Categories</h1>
<div class="cards">
    {% for cat in categories %}
    <div class="card">
        <h2>{{ cat }}</h2>
        <div class="counter">
            <button onclick="decrement('{{ cat }}')">−</button>
            <input type="number" id="{{ cat }}_count" value="5" min="1">
            <button onclick="increment('{{ cat }}')">+</button>
        </div>

        <form method="GET" action="{{ url_for('categories.download') }}">
            <input type="hidden" name="category" value="{{ cat }}">
            <input type="hidden" name="count" id="{{ cat }}_hidden" value="5">
            <button class="button" type="submit" onclick="syncCount('{{ cat }}')">Download</button>
        </form>

        <br>
        <a href="{{ url_for('categories.edit', category=cat) }}" class="button">Edit</a>

        <form action="{{ url_for('categories.delete', category=cat) }}" method="post" style="display:inline">
          <button class="button danger" type="submit">Delete</button>
        </form>
    </div>
    {% else %}
    <div class="card">
      <h3>No categories yet.</h3>
    </div>
    {% endfor %}
</div>

<a href="{{ url_for('categories.create') }}" class="button">➕ Create New Category</a>

<script>
function increment(category) {
    const input = document.getElementById(category + "_count");
    input.value = parseInt(input.value) + 1;
}
function decrement(category) {
    const input = document.getElementById(category + "_count");
    if (parseInt(input.value) > 1) {
        input.value = parseInt(input.value) - 1;
    }
}
function syncCount(category) {
    const visibleInput = document.getElementById(category + "_count");
    const hiddenInput = document.getElementById(category + "_hidden");
    hiddenInput.value = visibleInput.value;
}
</script>
{% endblock %}

```

## `flask_project/templates/manage_alerts.html`

```html
{% extends "base.html" %}
{% block content %}

<div style="position:absolute; left:20px; top:20px;">
  <a href="{{ url_for('index') }}" class="button">↩️ Back</a>
</div>

<h1>Manage Alerts</h1>

<form method="POST" style="margin-bottom: 20px;">
  <div class="form-group" style="width:100%; text-align:left;">
    <label>Alert Key (filename without .json)</label>
    <input name="alert_key" class="full-input" placeholder="Enter key" required>
  </div>

  <div class="form-group" style="width:100%; text-align:left;">
    <label>Alert Text (each line will be stored as a list element)</label>
    <textarea name="alert_text" class="full-input" rows="8" placeholder="Type each alert line on a new line..." required></textarea>
  </div>

  <div class="form-actions">
    <button type="submit" class="button">💾 Save Alert</button>
  </div>
</form>

<hr>

<h2>Saved Alerts</h2>

<div class="cards" style="justify-content:flex-start;">
  {% if alerts %}
    {% for alert in alerts %}
    <div class="card" style="width: 100%; max-width: 900px; text-align:left;">
      <h3>{{ alert.key }}</h3>
      <pre style="white-space:pre-wrap; color: #fff; margin: 0 0 10px 0;">{{ alert.lines | join('\n') }}</pre>

      <div style="display:flex; gap:10px;">
        <a href="{{ url_for('alerts.edit_alert', key=alert.key) }}" class="button">✏️ Edit</a>
        <form method="POST" action="{{ url_for('alerts.delete_alert', key=alert.key) }}" onsubmit="return confirm('Delete alert {{ alert.key }}?');">
          <button type="submit" class="button danger">Delete</button>
        </form>
      </div>
    </div>
    {% endfor %}
  {% else %}
    <div class="card" style="width:100%; max-width: 900px;">
      <h3>No alerts yet.</h3>
    </div>
  {% endif %}
</div>

{% endblock %}

```

## `flask_project/templates_data/cat.txt`

```
In the Bollywood industry, Tabu, whose certified name is Tabassum Fatima Hashmi, has long been delivering her performances with grace, nuance, and bold decisions all her life. Throughout her more than three-decade generational career, she has moved between Bollywood and South Indian movies with simplicity, leaving a legacy of standout performances and still is. However, her portrayal of three different roles—mother, wife, and lover—to the same actor, Nandamuri Balakrishna, is among the most intriguing aspects of her journey.
```

## `flask_project/tests/test_app.py`

```python
import io
import os
import json
import tempfile
from app import create_app

def test_create_and_download(tmp_path, monkeypatch):
    app = create_app()
    client = app.test_client()
    # point data directories to tmp
    app.config['DATA_DIR'] = str(tmp_path / "data")
    app.config['TEMPLATE_DIR'] = str(tmp_path / "templates")
    os.makedirs(app.config['DATA_DIR'], exist_ok=True)
    os.makedirs(app.config['TEMPLATE_DIR'], exist_ok=True)

    data = {
        'category': 'demo',
        'key': ['__NAME__'],
        'type': ['static'],
        'value_static': ['Sandeep']
    }
    data_file = (io.BytesIO(b"Hello __NAME__"), "tpl.txt")
    resp = client.post("/create", data={**data, 'file': data_file}, content_type='multipart/form-data', follow_redirects=True)
    assert resp.status_code in (200, 302)

    # download
    resp = client.get("/download?category=demo&count=2")
    assert resp.status_code == 200
    assert 'application/zip' in resp.headers['Content-Type']

```

