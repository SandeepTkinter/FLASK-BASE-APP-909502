# Flask Project - Combined Source Code

## flask_project/app.py

```py
import os
import io
import json
import logging
from zipfile import ZipFile, ZIP_DEFLATED
from functools import wraps
from flask import (
    Flask, render_template, request, redirect, url_for,
    send_file, flash, current_app
)
from werkzeug.utils import secure_filename
from werkzeug.exceptions import RequestEntityTooLarge

from dynamic_funcs import DYNAMIC_FUNCTIONS

# ---- Config ----
class Config:
    SECRET_KEY = os.environ.get("FLASK_SECRET", "replace-me-in-prod")
    BASE_DIR = os.path.abspath(os.path.dirname(__file__))
    DATA_DIR = os.path.join(BASE_DIR, "data")
    TEMPLATE_DIR = os.path.join(BASE_DIR, "templates_data")
    MAX_CONTENT_LENGTH = 10 * 1024 * 1024   # 10 MB upload limit
    ALLOWED_TEMPLATE_EXTS = {'.txt', '.md', '.tmpl'}
    DEFAULT_COUNT = 5
    MAX_COUNT = 200

# ---- App init ----
app = Flask(__name__, static_folder="static", template_folder="templates")
app.config.from_object(Config)
os.makedirs(app.config["DATA_DIR"], exist_ok=True)
os.makedirs(app.config["TEMPLATE_DIR"], exist_ok=True)

# Basic logger
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ---- helpers ----
def allowed_template_filename(filename):
    _, ext = os.path.splitext(filename.lower())
    return ext in current_app.config["ALLOWED_TEMPLATE_EXTS"]

def safe_category_name(name):
    return secure_filename(name)

def get_json_path(category):
    return os.path.join(current_app.config["DATA_DIR"], f"{category}.json")

def get_template_path(category):
    return os.path.join(current_app.config["TEMPLATE_DIR"], f"{category}.txt")

def validate_count(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            count = int(request.values.get("count", current_app.config["DEFAULT_COUNT"]))
        except (ValueError, TypeError):
            count = current_app.config["DEFAULT_COUNT"]
        count = max(1, min(count, current_app.config["MAX_COUNT"]))
        request._validated_count = count
        return func(*args, **kwargs)
    return wrapper

# ---- Routes ----
@app.errorhandler(RequestEntityTooLarge)
def handle_file_too_large(e):
    flash("Uploaded file is too large.")
    return redirect(url_for("index")), 413

@app.route("/")
def index():
    categories = []
    for fname in os.listdir(app.config["DATA_DIR"]):
        if fname.endswith(".json"):
            categories.append(fname[:-5])
    categories.sort()
    return render_template("index.html", categories=categories)

@app.route("/create", methods=["GET", "POST"])
def create():
    if request.method == "POST":
        category = request.form.get("category", "").strip()
        uploaded = request.files.get("file")
        keys = request.form.getlist("key")
        types = request.form.getlist("type")
        static_vals = request.form.getlist("value_static")
        dynamic_vals = request.form.getlist("value_dynamic")

        if not category:
            flash("Category name is required.")
            return redirect(url_for("create"))

        category = safe_category_name(category)
        if uploaded is None or uploaded.filename == "":
            flash("Template file required.")
            return redirect(url_for("create"))

        if not allowed_template_filename(uploaded.filename):
            flash("Template file type not allowed. Use .txt/.md/.tmpl")
            return redirect(url_for("create"))

        # Save template file
        template_path = get_template_path(category)
        uploaded.save(template_path)

        # Build KV dict with type info
        kv_pairs = {}
        for i, k in enumerate(keys):
            k = k.strip()
            if not k:
                continue
            t = types[i] if i < len(types) else "static"
            if t == "static":
                val = static_vals[i] if i < len(static_vals) else ""
                kv_pairs[k] = {"type": "static", "value": val}
            else:
                dyn = dynamic_vals[i] if i < len(dynamic_vals) else ""
                kv_pairs[k] = {"type": "dynamic", "value": dyn}

        json_path = get_json_path(category)
        with open(json_path, "w", encoding="utf-8") as f:
            json.dump(kv_pairs, f, indent=2, ensure_ascii=False)

        flash(f"Category '{category}' created.")
        return redirect(url_for("index"))

    return render_template("create.html", dynamic_funcs=sorted(DYNAMIC_FUNCTIONS.keys()))

@app.route("/edit/<category>", methods=["GET", "POST"])
def edit(category):
    category = safe_category_name(category)
    json_path = get_json_path(category)
    template_path = get_template_path(category)
    if not os.path.exists(json_path):
        flash("Category not found.")
        return redirect(url_for("index"))

    if request.method == "POST":
        keys = request.form.getlist("key")
        types = request.form.getlist("type")
        static_vals = request.form.getlist("value_static")
        dynamic_vals = request.form.getlist("value_dynamic")

        kv_pairs = {}
        for i, k in enumerate(keys):
            k = k.strip()
            if not k:
                continue
            t = types[i] if i < len(types) else "static"
            if t == "static":
                val = static_vals[i] if i < len(static_vals) else ""
                kv_pairs[k] = {"type": "static", "value": val}
            else:
                dyn = dynamic_vals[i] if i < len(dynamic_vals) else ""
                kv_pairs[k] = {"type": "dynamic", "value": dyn}

        with open(json_path, "w", encoding="utf-8") as f:
            json.dump(kv_pairs, f, indent=2, ensure_ascii=False)
        flash("Saved.")
        return redirect(url_for("index"))

    with open(json_path, "r", encoding="utf-8") as f:
        kv_pairs = json.load(f)
    template_exists = os.path.exists(template_path)
    return render_template("edit.html", category=category, kv_pairs=kv_pairs, template_exists=template_exists, dynamic_funcs=sorted(DYNAMIC_FUNCTIONS.keys()))

@app.route("/delete/<category>", methods=["POST"])
def delete(category):
    category = safe_category_name(category)
    json_path = get_json_path(category)
    template_path = get_template_path(category)
    try:
        if os.path.exists(json_path):
            os.remove(json_path)
        if os.path.exists(template_path):
            os.remove(template_path)
        flash("Deleted.")
    except Exception:
        logger.exception("Failed deleting category %s", category)
        flash("Error deleting.")
    return redirect(url_for("index"))

def generate_zip_bytes(category, kv_data, template_text, count):
    mem = io.BytesIO()
    with ZipFile(mem, "w", compression=ZIP_DEFLATED) as zf:
        for i in range(1, count + 1):
            content = template_text
            for k, v in kv_data.items():
                if isinstance(v, dict) and v.get("type") == "dynamic":
                    func_name = v.get("value")
                    func = DYNAMIC_FUNCTIONS.get(func_name)
                    replacement = func() if callable(func) else ""
                else:
                    replacement = v.get("value") if isinstance(v, dict) else str(v)
                content = content.replace(k, replacement)
            zf.writestr(f"{category}_file_{i}.txt", content)
    mem.seek(0)
    return mem.read()

@app.route("/download", methods=["GET"])
@validate_count
def download():
    category = request.args.get("category", "")
    if not category:
        flash("Missing category")
        return redirect(url_for("index"))

    category = safe_category_name(category)
    json_path = get_json_path(category)
    template_path = get_template_path(category)
    if not os.path.exists(json_path) or not os.path.exists(template_path):
        flash("Category data/template missing.")
        return redirect(url_for("index"))

    with open(json_path, "r", encoding="utf-8") as f:
        kv_data = json.load(f)

    with open(template_path, "r", encoding="utf-8") as f:
        template_text = f.read()

    count = request._validated_count
    zip_bytes = generate_zip_bytes(category, kv_data, template_text, count)
    filename = f"{category}_files.zip"
    return send_file(
        io.BytesIO(zip_bytes),
        mimetype="application/zip",
        as_attachment=True,
        download_name=filename
    )

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))

```

## flask_project/dynamic_funcs.py

```py
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

## flask_project/data/cat.json

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

## flask_project/static/style.css

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

## flask_project/templates/base.html

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

## flask_project/templates/create.html

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

## flask_project/templates/edit.html

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

## flask_project/templates/index.html

```html
{% extends "base.html" %}
{% block content %}
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

        <form method="GET" action="{{ url_for('download') }}">
            <input type="hidden" name="category" value="{{ cat }}">
            <input type="hidden" name="count" id="{{ cat }}_hidden" value="5">
            <button class="button" type="submit" onclick="syncCount('{{ cat }}')">Download</button>
        </form>

        <br>
        <a href="{{ url_for('edit', category=cat) }}" class="button">Edit</a>

        <form action="{{ url_for('delete', category=cat) }}" method="post" style="display:inline">
          <button class="button danger" type="submit">Delete</button>
        </form>
    </div>
    {% else %}
    <div class="card">
      <h3>No categories yet.</h3>
    </div>
    {% endfor %}
</div>

<a href="{{ url_for('create') }}" class="button">➕ Create New Category</a>

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

## flask_project/templates_data/cat.txt

```txt
In the Bollywood industry, Tabu, whose certified name is Tabassum Fatima Hashmi, has long been delivering her performances with grace, nuance, and bold decisions all her life. Throughout her more than three-decade generational career, she has moved between Bollywood and South Indian movies with simplicity, leaving a legacy of standout performances and still is. However, her portrayal of three different roles—mother, wife, and lover—to the same actor, Nandamuri Balakrishna, is among the most intriguing aspects of her journey.
```

## flask_project/tests/test_app.py

```py
import io
import os
import json
import tempfile
from app import app

def test_create_and_download(tmp_path, monkeypatch):
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

