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
const alertFiles = {{ alert_files|tojson }};
function addRow(data) {
  data = data || {"key":"", "type":"static", "value_static":"", "value_dynamic": dynamicFuncs[0] || "", "value_alert": alertFiles[0] || ""};
  const tbody = document.querySelector("#kvTable tbody");
  const tr = document.createElement("tr");
  tr.innerHTML = `
    <td><input class="full-input" name="key" value="${escapeHtml(data.key)}" required></td>
    <td>
      <select class="full-input" name="type" onchange="toggle(this)">
        <option value="static"${data.type==='static'?' selected':''}>Static</option>
        <option value="dynamic"${data.type==='dynamic'?' selected':''}>Dynamic</option>
        <option value="dynamicallypickfrom"${data.type==='dynamicallypickfrom'?' selected':''}>Pick from Alert</option>
      </select>
    </td>
    <td>
      <input class="full-input v_static" name="value_static" value="${escapeHtml(data.value_static || '')}">
      <select class="full-input v_dynamic" name="value_dynamic" style="display:none">
        ${dynamicFuncs.map(f => `<option value="${f}" ${f===data.value_dynamic?'selected':''}>${f}</option>`).join('')}
      </select>
      <select class="full-input v_alert" name="value_alert" style="display:none">
        ${alertFiles.map(a => `<option value="${a}" ${a===data.value_alert?'selected':''}>${a}</option>`).join('')}
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
  tr.querySelector('.v_alert').style.display = sel.value === 'dynamicallypickfrom' ? '' : 'none';
}
function escapeHtml(unsafe) {
  return (unsafe + '').replace(/[&<>"'`=\/]/g, s => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;', '/': '&#x2F;', '`': '&#x60;', '=': '&#x3D;'
  })[s]);
}

const kv = {{ kv_pairs|tojson }};
for (const [k, v] of Object.entries(kv || {})) {
  const row = {
    key: k,
    type: v.type || 'static',
    value_static: v.type==='static' ? (v.value || '') : '',
    value_dynamic: v.type==='dynamic' ? (v.value || dynamicFuncs[0]) : dynamicFuncs[0],
    value_alert: v.type==='dynamicallypickfrom' ? (v.value || alertFiles[0]) : alertFiles[0]
  };
  addRow(row);
}
</script>
{% endblock %}

```