# All-in-One Flask App Source Breakdown

### File: `final_dark_flask_app_updated/app.py`
```python

from flask import Flask, render_template, request, redirect, url_for, send_file, flash
import os
import json
import io
from zipfile import ZipFile
from werkzeug.utils import secure_filename

app = Flask(__name__)
app.secret_key = 'secret'
BASE_DIR = os.path.abspath(os.path.dirname(__file__))
DATA_DIR = os.path.join(BASE_DIR, 'data')
TEMPLATE_DIR = os.path.join(BASE_DIR, 'templates_data')
os.makedirs(DATA_DIR, exist_ok=True)
os.makedirs(TEMPLATE_DIR, exist_ok=True)

@app.route('/')
def index():
    categories = []
    for fname in os.listdir(DATA_DIR):
        if fname.endswith('.json'):
            name = fname[:-5]
            categories.append(name)
    return render_template('index.html', categories=categories)

@app.route('/create', methods=['GET', 'POST'])
def create():
    if request.method == 'POST':
        category = request.form['category']
        file = request.files['file']
        keys = request.form.getlist('key')
        values = request.form.getlist('value')

        if not category or not file:
            flash("Category and file are required")
            return redirect(url_for('create'))

        category = secure_filename(category)
        file_path = os.path.join(TEMPLATE_DIR, category + '.txt')
        file.save(file_path)

        kv_data = dict(zip(keys, values))
        json_path = os.path.join(DATA_DIR, category + '.json')
        with open(json_path, 'w') as f:
            json.dump(kv_data, f, indent=4)

        return redirect(url_for('index'))

    return render_template('create.html')

@app.route('/edit/<category>', methods=['GET', 'POST'])
def edit(category):
    json_path = os.path.join(DATA_DIR, category + '.json')
    if not os.path.exists(json_path):
        flash("Category not found")
        return redirect(url_for('index'))

    if request.method == 'POST':
        keys = request.form.getlist('key')
        values = request.form.getlist('value')
        kv_data = dict(zip(keys, values))
        with open(json_path, 'w') as f:
            json.dump(kv_data, f, indent=4)
        return redirect(url_for('index'))

    with open(json_path, 'r') as f:
        kv_data = json.load(f)

    return render_template('edit.html', category=category, kv_pairs=kv_data)

@app.route('/delete/<category>')
def delete(category):
    json_path = os.path.join(DATA_DIR, category + '.json')
    template_path = os.path.join(TEMPLATE_DIR, category + '.txt')
    if os.path.exists(json_path):
        os.remove(json_path)
    if os.path.exists(template_path):
        os.remove(template_path)
    return redirect(url_for('index'))

@app.route('/download')
def download():
    category = request.args.get('category')
    try:
        count = int(request.args.get('count', 1))
    except:
        count = 1
    count = max(1, min(count, 100))

    json_path = os.path.join(DATA_DIR, category + '.json')
    template_path = os.path.join(TEMPLATE_DIR, category + '.txt')

    if not os.path.exists(json_path) or not os.path.exists(template_path):
        flash("Missing data for category")
        return redirect(url_for('index'))

    with open(json_path, 'r') as f:
        kv_data = json.load(f)

    with open(template_path, 'r') as f:
        template = f.read()

    mem_file = io.BytesIO()
    with ZipFile(mem_file, 'w') as zf:
        for i in range(1, count + 1):
            content = template
            for k, v in kv_data.items():
                content = content.replace(k, v)
            zf.writestr(f"{category}_file_{i}.txt", content)

    mem_file.seek(0)
    return send_file(mem_file, as_attachment=True, download_name=f"{category}_files.zip")

if __name__ == '__main__':
    app.run(debug=True)

```

### File: `final_dark_flask_app_updated/static/style.css`
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
}

.card {
    background-color: var(--card-bg);
    border-radius: 10px;
    padding: 20px;
    width: 240px;
    box-shadow: 0 0 10px rgba(0,0,0,0.5);
    transition: transform 0.2s ease;
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
    margin-bottom: 15px;
    border-radius: 6px;
    border: none;
    background-color: #fff;
    color: #000;
    font-size: 16px;
}

```

### File: `final_dark_flask_app_updated/templates/create.html`
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Create Category</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <style>
        .form-group {
            display: flex;
            flex-direction: column;
            margin-bottom: 20px;
            align-items: flex-start;
        }

        .kv-pair {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-bottom: 10px;
        }

        .kv-pair input {
            width: 200px;
        }

        .kv-controls {
            display: flex;
            justify-content: space-between;
            width: 100%;
            margin-top: 20px;
        }

        .form-actions {
            display: flex;
            gap: 10px;
            margin-top: 30px;
        }

        input[type="text"], input[type="file"] {
            padding: 10px;
            width: 300px;
            border-radius: 6px;
            border: none;
            background-color: #fff;
            color: #000;
        }

        .button.danger {
            background-color: #cc3b3b;
        }

        .button.danger:hover {
            background-color: #b12d2d;
        }

        h1 {
            margin-bottom: 30px;
        }
    </style>
</head>
<body>
<div class="container">
    <h1>Create New Category</h1>
    <form method="POST" enctype="multipart/form-data">
        <div class="form-group">
            <label>Category Name</label>
            <input type="text" name="category" required>
        </div>

        <div class="form-group">
            <label>Upload Template File</label>
            <input type="file" name="file" required>
        </div>

        <div class="form-group">
            <label>Key-Value Pairs</label>
            <div id="kv-container">
                <div class="kv-pair">
                    <input type="text" name="key" placeholder="Find" required>
                    <input type="text" name="value" placeholder="Replace" required>
                    <button type="button" class="button danger" onclick="removePair(this)">🗑️</button>
                </div>
            </div>
            <button type="button" class="button" onclick="addKV()">➕ Add More</button>
        </div>

        <div class="form-actions">
            <button type="submit" class="button">💾 Save</button>
            <a href="/" class="button danger">↩️ Cancel</a>
        </div>
    </form>
</div>

<script>
    function addKV() {
        const div = document.getElementById("kv-container");
        const html = `
            <div class="kv-pair">
                <input type="text" name="key" placeholder="Find" required>
                <input type="text" name="value" placeholder="Replace" required>
                <button type="button" class="button danger" onclick="removePair(this)">🗑️</button>
            </div>`;
        div.insertAdjacentHTML("beforeend", html);
    }

    function removePair(button) {
        button.parentElement.remove();
    }
</script>
</body>
</html>

```

### File: `final_dark_flask_app_updated/templates/edit.html`
```html

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Edit Category</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
<div class="container">
    <h1>Edit Category: {{ category }}</h1>
    <form method="POST">
        <div id="kv-container">
            {% for k, v in kv_pairs.items() %}
            <div class="kv-pair">
                <input type="text" name="key" value="{{ k }}" required>
                <input type="text" name="value" value="{{ v }}" required>
                <button type="button" onclick="removePair(this)">🗑️</button>
            </div>
            {% endfor %}
        </div>
        <button type="button" onclick="addKV()">➕ Add</button><br><br>
        <button type="submit" class="button">Save</button>
        <a href="/" class="button danger">Cancel</a>
    </form>
</div>
<script>
    function addKV() {
        const div = document.getElementById("kv-container");
        const html = `
            <div class="kv-pair">
                <input type="text" name="key" placeholder="Find" required>
                <input type="text" name="value" placeholder="Replace" required>
                <button type="button" onclick="removePair(this)">🗑️</button>
            </div>`;
        div.insertAdjacentHTML("beforeend", html);
    }

    function removePair(button) {
        button.parentElement.remove();
    }
</script>
</body>
</html>

```

### File: `final_dark_flask_app_updated/templates/index.html`
```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Dark Mode Categories</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
    <div class="container">
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
                <form method="GET" action="/download">
                    <input type="hidden" name="category" value="{{ cat }}">
                    <input type="hidden" name="count" id="{{ cat }}_hidden" value="5">
                    <button class="button" type="submit" onclick="syncCount('{{ cat }}')">Download</button>
                </form>
                <br>
                <a href="/edit/{{ cat }}" class="button">Edit</a>
                <a href="/delete/{{ cat }}" class="button danger">Delete</a>
            </div>
            {% endfor %}
        </div>
        <a href="/create" class="button">➕ Create New Category</a>
    </div>

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
</body>
</html>

```
