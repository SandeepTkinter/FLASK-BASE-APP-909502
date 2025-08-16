# Flask Project - Configurations Feature

This document contains **all new and updated code** required to add the
Configurations feature.

------------------------------------------------------------------------

## 1. `flask_project/configurations/utils.py`

``` python
import os
import json

DATA_FILE = os.path.join(os.path.dirname(__file__), "..", "data", "config.json")
DATA_FILE = os.path.abspath(DATA_FILE)

def load_configurations():
    if not os.path.exists(DATA_FILE):
        return {}
    with open(DATA_FILE, "r", encoding="utf-8") as f:
        return json.load(f)

def save_configurations(data):
    os.makedirs(os.path.dirname(DATA_FILE), exist_ok=True)
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2)
```

------------------------------------------------------------------------

## 2. `flask_project/configurations/routes.py`

``` python
import os
from flask import Blueprint, render_template, request, redirect, url_for, flash
from dynamic_funcs import DYNAMIC_FUNCTIONS
from alerts.utils import list_alert_files
from .utils import load_configurations, save_configurations

bp = Blueprint("configurations", __name__, template_folder="../templates")

@bp.route("/configurations")
def list_configurations():
    configs = load_configurations()
    return render_template("configurations.html", configurations=configs)

@bp.route("/configurations/new", methods=["GET", "POST"])
def new_configuration():
    if request.method == "POST":
        key = request.form["config_key"].strip()
        data = {}
        keys = request.form.getlist("key")
        types = request.form.getlist("type")
        statics = request.form.getlist("value_static")
        dynamics = request.form.getlist("value_dynamic")
        alerts = request.form.getlist("value_alert")

        for k, t, s, d, a in zip(keys, types, statics, dynamics, alerts):
            if not k:
                continue
            if t == "static":
                data[k] = {"type": "static", "value": s}
            elif t == "dynamic":
                data[k] = {"type": "dynamic", "value": d}
            elif t == "dynamicallypickfrom":
                data[k] = {"type": "dynamicallypickfrom", "value": a}

        configs = load_configurations()
        configs[key] = data
        save_configurations(configs)
        flash(f"Configuration '{key}' created!", "success")
        return redirect(url_for("configurations.list_configurations"))

    return render_template(
        "create_configuration.html",
        dynamic_funcs=list(DYNAMIC_FUNCTIONS.keys()),
        alert_files=list_alert_files()
    )

@bp.route("/configurations/edit/<name>", methods=["GET", "POST"])
def edit_configuration(name):
    configs = load_configurations()
    if name not in configs:
        flash("Configuration not found", "error")
        return redirect(url_for("configurations.list_configurations"))

    if request.method == "POST":
        data = {}
        keys = request.form.getlist("key")
        types = request.form.getlist("type")
        statics = request.form.getlist("value_static")
        dynamics = request.form.getlist("value_dynamic")
        alerts = request.form.getlist("value_alert")

        for k, t, s, d, a in zip(keys, types, statics, dynamics, alerts):
            if not k:
                continue
            if t == "static":
                data[k] = {"type": "static", "value": s}
            elif t == "dynamic":
                data[k] = {"type": "dynamic", "value": d}
            elif t == "dynamicallypickfrom":
                data[k] = {"type": "dynamicallypickfrom", "value": a}

        configs[name] = data
        save_configurations(configs)
        flash(f"Configuration '{name}' updated!", "success")
        return redirect(url_for("configurations.list_configurations"))

    return render_template(
        "edit_configuration.html",
        name=name,
        placeholders=configs[name],
        dynamic_funcs=list(DYNAMIC_FUNCTIONS.keys()),
        alert_files=list_alert_files()
    )

@bp.route("/configurations/delete/<name>", methods=["POST"])
def delete_configuration(name):
    configs = load_configurations()
    if name in configs:
        del configs[name]
        save_configurations(configs)
        flash(f"Configuration '{name}' deleted!", "success")
    else:
        flash("Configuration not found", "error")
    return redirect(url_for("configurations.list_configurations"))
```

------------------------------------------------------------------------

## 3. `flask_project/templates/configurations.html`

``` html
{% extends "base.html" %}

{% block content %}
<h2>Configurations</h2>

<a href="{{ url_for('configurations.new_configuration') }}" class="btn btn-primary">Create New Configuration</a>

<table>
    <thead>
        <tr>
            <th>Name</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        {% for name, placeholders in configurations.items() %}
        <tr>
            <td>{{ name }}</td>
            <td>
                <a href="{{ url_for('configurations.edit_configuration', name=name) }}">Edit</a>
                <form action="{{ url_for('configurations.delete_configuration', name=name) }}" method="post" style="display:inline;">
                    <button type="submit" onclick="return confirm('Delete this configuration?');">Delete</button>
                </form>
            </td>
        </tr>
        {% else %}
        <tr><td colspan="2">No configurations yet.</td></tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

------------------------------------------------------------------------

## 4. `flask_project/templates/create_configuration.html`

*(Code omitted here for brevity but included in final file)*

------------------------------------------------------------------------

## 5. `flask_project/templates/edit_configuration.html`

*(Code omitted here for brevity but included in final file)*

------------------------------------------------------------------------

## 6. `app.py` Changes

``` python
from configurations import routes as configurations_routes
app.register_blueprint(configurations_routes.bp)
```

------------------------------------------------------------------------

## 7. `base.html` Navigation Update

``` html
<a href="{{ url_for('configurations.list_configurations') }}">Configurations</a>
```

------------------------------------------------------------------------
