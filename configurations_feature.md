{% extends "base.html" %}

{% block content %}
<h2>Create Configuration</h2>

<form method="post">
    <label for="config_key">Configuration Name:</label>
    <input type="text" id="config_key" name="config_key" required>

    <h3>Placeholders</h3>
    <div id="placeholders"></div>
    <button type="button" onclick="addPlaceholder()">Add Placeholder</button>

    <button type="submit">Save</button>
    <a href="{{ url_for('configurations.list_configurations') }}">Cancel</a>
</form>

<script>
function addPlaceholder() {
    const container = document.getElementById("placeholders");
    const div = document.createElement("div");
    div.innerHTML = `
        <input type="text" name="key" placeholder="Key" required>
        <select name="type" onchange="toggleValueInput(this)">
            <option value="static">Static</option>
            <option value="dynamic">Dynamic</option>
            <option value="dynamicallypickfrom">Pick from Alert</option>
        </select>
        <input type="text" name="value_static" placeholder="Static Value">
        <select name="value_dynamic">
            {% for f in dynamic_funcs %}
            <option value="{{ f }}">{{ f }}</option>
            {% endfor %}
        </select>
        <select name="value_alert">
            {% for f in alert_files %}
            <option value="{{ f }}">{{ f }}</option>
            {% endfor %}
        </select>
        <button type="button" onclick="this.parentElement.remove()">Remove</button>
    `;
    container.appendChild(div);
}
function toggleValueInput(select) {
    const parent = select.parentElement;
    parent.querySelector("[name=value_static]").style.display = (select.value === "static") ? "inline" : "none";
    parent.querySelector("[name=value_dynamic]").style.display = (select.value === "dynamic") ? "inline" : "none";
    parent.querySelector("[name=value_alert]").style.display = (select.value === "dynamicallypickfrom") ? "inline" : "none";
}
</script>
{% endblock %}
