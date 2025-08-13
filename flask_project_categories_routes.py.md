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
from .utils import safe_category_name, allowed_template_filename, get_json_path, get_template_path, list_alert_files, pick_from_alert

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
        alert_vals = request.form.getlist('value_alert')

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
            elif t == 'dynamicallypickfrom':
                val = alert_vals[i] if i < len(alert_vals) else ''
                kv_pairs[k] = {'type': 'dynamicallypickfrom', 'value': val}
            else:
                dyn = dynamic_vals[i] if i < len(dynamic_vals) else ''
                kv_pairs[k] = {'type': 'dynamic', 'value': dyn}

        json_path = get_json_path(category)
        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(kv_pairs, f, indent=2, ensure_ascii=False)

        flash(f"Category '{category}' created.")
        return redirect(url_for('index'))

    return render_template('create.html', dynamic_funcs=sorted(DYNAMIC_FUNCTIONS.keys()), alert_files=sorted(list_alert_files()))

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
        alert_vals = request.form.getlist('value_alert')

        kv_pairs = {}
        for i, k in enumerate(keys):
            k = k.strip()
            if not k:
                continue
            t = types[i] if i < len(types) else 'static'
            if t == 'static':
                val = static_vals[i] if i < len(static_vals) else ''
                kv_pairs[k] = {'type': 'static', 'value': val}
            elif t == 'dynamicallypickfrom':
                val = alert_vals[i] if i < len(alert_vals) else ''
                kv_pairs[k] = {'type': 'dynamicallypickfrom', 'value': val}
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
    return render_template('edit.html', category=category, kv_pairs=kv_pairs, template_exists=template_exists, dynamic_funcs=sorted(DYNAMIC_FUNCTIONS.keys()), alert_files=sorted(list_alert_files()))

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
                if isinstance(v, dict):
                    t = v.get('type')
                    if t == 'dynamic':
                        func = DYNAMIC_FUNCTIONS.get(v.get('value'))
                        replacement = func() if callable(func) else ''
                    elif t == 'dynamicallypickfrom':
                        replacement = pick_from_alert(v.get('value'))
                    else:
                        replacement = v.get('value', '')
                else:
                    replacement = str(v)
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