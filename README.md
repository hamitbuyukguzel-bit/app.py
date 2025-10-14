"""
Language Learner Tracker
Simple Flask app to track learners, their levels, and notes.

Run:
    python app.py
Open:
    http://127.0.0.1:5000/

Author:
    (Your name) — ready to push to GitHub
License:
    MIT (suggested)
"""
from flask import Flask, request, redirect, url_for, render_template_string, send_file, flash
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
import csv
import io
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///learners.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['SECRET_KEY'] = os.urandom(24)
db = SQLAlchemy(app)

# ---------
# Models
# ---------
class Learner(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(120), nullable=False)
    language = db.Column(db.String(80), nullable=False, default='English')
    level = db.Column(db.String(40), nullable=True)  # e.g. A1/A2/B1 or "Beginner"
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    notes = db.relationship('Note', backref='learner', cascade="all, delete-orphan", lazy=True)

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    learner_id = db.Column(db.Integer, db.ForeignKey('learner.id'), nullable=False)
    content = db.Column(db.Text, nullable=False)
    tags = db.Column(db.String(200), nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

# ---------
# Helpers
# ---------
def init_db():
    db.create_all()

# Small, clean HTML templates using render_template_string to keep single-file
BASE_HTML = """
<!doctype html>
<title>Language Learner Tracker</title>
<link rel="stylesheet"
 href="https://cdn.jsdelivr.net/npm/water.css@2/out/water.css">
<div style="max-width:900px;margin:2rem auto;">
  <header>
    <h1>Language Learner Tracker</h1>
    <p>Keep learner levels and notes organized.</p>
    <nav>
      <a href="{{ url_for('index') }}">Home</a> |
      <a href="{{ url_for('new_learner') }}">Add Learner</a>
    </nav>
    <hr>
    {% with messages = get_flashed_messages() %}
      {% if messages %}
        <ul>
        {% for m in messages %}
          <li><strong>{{ m }}</strong></li>
        {% endfor %}
        </ul>
      {% endif %}
    {% endwith %}
  </header>
  {% block body %}{% endblock %}
  <footer style="margin-top:2rem;border-top:1px solid #ddd;padding-top:1rem;">
    <small>Export notes as CSV. Simple, no-login demo app.</small>
  </footer>
</div>
"""

# ---------
# Routes
# ---------
@app.route('/')
def index():
    q = request.args.get('q', '').strip()
    lang = request.args.get('language', '').strip()
    learners = Learner.query.order_by(Learner.name).all()
    if q:
        learners = Learner.query.filter(Learner.name.ilike(f'%{q}%')).order_by(Learner.name).all()
    if lang:
        learners = Learner.query.filter(Learner.language.ilike(f'%{lang}%')).order_by(Learner.name).all()

    return render_template_string(
        BASE_HTML + """
{% block body %}
  <form method="get">
    <label>Search name: <input name="q" value="{{ request.args.get('q','') }}"></label>
    <label>Language: <input name="language" value="{{ request.args.get('language','') }}"></label>
    <button type="submit">Filter</button>
    <a href="{{ url_for('index') }}">Clear</a>
  </form>

  <h2>Learners</h2>
  {% if learners %}
    <table>
      <thead><tr><th>Name</th><th>Language</th><th>Level</th><th>Notes</th><th>Actions</th></tr></thead>
      <tbody>
      {% for L in learners %}
        <tr>
          <td>{{ L.name }}</td>
          <td>{{ L.language }}</td>
          <td>{{ L.level or '-' }}</td>
          <td>{{ L.notes|length }}</td>
          <td>
            <a href="{{ url_for('view_learner', learner_id=L.id) }}">View</a> |
            <a href="{{ url_for('edit_learner', learner_id=L.id) }}">Edit</a> |
            <a href="{{ url_for('export_notes', learner_id=L.id) }}">Export CSV</a> |
            <a href="{{ url_for('delete_learner', learner_id=L.id) }}" onclick="return confirm('Delete learner and notes?')">Delete</a>
          </td>
        </tr>
      {% endfor %}
      </tbody>
    </table>
  {% else %}
    <p>No learners yet. <a href="{{ url_for('new_learner') }}">Add one</a>.</p>
  {% endif %}
{% endblock %}
""",
        learners=learners
    )

@app.route('/learners/new', methods=['GET', 'POST'])
def new_learner():
    if request.method == 'POST':
        name = request.form.get('name', '').strip()
        language = request.form.get('language', '').strip() or 'English'
        level = request.form.get('level', '').strip() or None
        if not name:
            flash('Name is required.')
            return redirect(url_for('new_learner'))
        L = Learner(name=name, language=language, level=level)
        db.session.add(L)
        db.session.commit()
        flash('Learner created.')
        return redirect(url_for('index'))

    return render_template_string(BASE_HTML + """
{% block body %}
  <h2>Add Learner</h2>
  <form method="post">
    <label>Name: <input name="name" required></label><br>
    <label>Language: <input name="language" placeholder="e.g. Spanish"></label><br>
    <label>Level: <input name="level" placeholder="e.g. A2, Beginner"></label><br>
    <button type="submit">Create</button>
    <a href="{{ url_for('index') }}">Cancel</a>
  </form>
{% endblock %}
""")

@app.route('/learners/<int:learner_id>')
def view_learner(learner_id):
    L = Learner.query.get_or_404(learner_id)
    return render_template_string(BASE_HTML + """
{% block body %}
  <h2>{{ L.name }} — {{ L.language }}</h2>
  <p><strong>Level:</strong> {{ L.level or '—' }}</p>
  <p><strong>Created:</strong> {{ L.created_at.strftime('%Y-%m-%d %H:%M') }}</p>

  <h3>Notes</h3>
  <form method="post" action="{{ url_for('add_note', learner_id=L.id) }}">
    <textarea name="content" rows="3" style="width:100%;" placeholder="Add a quick progress note..."></textarea><br>
    <input name="tags" placeholder="tags (comma separated)"><br>
    <button type="submit">Add Note</button>
  </form>

  {% if L.notes %}
    <ul>
    {% for n in L.notes|sort(attribute='created_at', reverse=True) %}
      <li>
        <small>{{ n.created_at.strftime('%Y-%m-%d %H:%M') }}</small>
        {% if n.tags %} <em>[{{ n.tags }}]</em>{% endif %}<br>
        {{ n.content }}<br>
        <a href="{{ url_for('delete_note', note_id=n.id) }}" onclick="return confirm('Delete note?')">Delete</a>
      </li>
    {% endfor %}
    </ul>
  {% else %}
    <p>No notes yet.</p>
  {% endif %}

  <p>
    <a href="{{ url_for('edit_learner', learner_id=L.id) }}">Edit learner</a> |
    <a href="{{ url_for('export_notes', learner_id=L.id) }}">Export notes (CSV)</a> |
    <a href="{{ url_for('index') }}">Back</a>
  </p>
{% endblock %}
""", L=L)

@app.route('/learners/<int:learner_id>/notes', methods=['POST'])
def add_note(learner_id):
    L = Learner.query.get_or_404(learner_id)
    content = request.form.get('content', '').strip()
    tags = request.form.get('tags', '').strip()
    if not content:
        flash('Note cannot be empty.')
        return redirect(url_for('view_learner', learner_id=learner_id))
    n = Note(learner=L, content=content, tags=tags or None)
    db.session.add(n)
    db.session.commit()
    flash('Note added.')
    return redirect(url_for('view_learner', learner_id=learner_id))

@app.route('/learners/<int:learner_id>/edit', methods=['GET', 'POST'])
def edit_learner(learner_id):
    L = Learner.query.get_or_404(learner_id)
    if request.method == 'POST':
        L.name = request.form.get('name', L.name).strip()
        L.language = request.form.get('language', L.language).strip()
        L.level = request.form.get('level', L.level).strip() or None
        db.session.commit()
        flash('Learner updated.')
        return redirect(url_for('view_learner', learner_id=learner_id))

    return render_template_string(BASE_HTML + """
{% block body %}
  <h2>Edit Learner</h2>
  <form method="post">
    <label>Name: <input name="name" value="{{ L.name }}" required></label><br>
    <label>Language: <input name="language" value="{{ L.language }}"></label><br>
    <label>Level: <input name="level" value="{{ L.level }}"></label><br>
    <button type="submit">Save</button>
    <a href="{{ url_for('view_learner', learner_id=L.id) }}">Cancel</a>
  </form>
{% endblock %}
""", L=L)

@app.route('/note/<int:note_id>/delete')
def delete_note(note_id):
    n = Note.query.get_or_404(note_id)
    learner_id = n.learner_id
    db.session.delete(n)
    db.session.commit()
    flash('Note deleted.')
    return redirect(url_for('view_learner', learner_id=learner_id))

@app.route('/learners/<int:learner_id>/delete')
def delete_learner(learner_id):
    L = Learner.query.get_or_404(learner_id)
    db.session.delete(L)
    db.session.commit()
    flash('Learner and notes deleted.')
    return redirect(url_for('index'))

@app.route('/learners/<int:learner_id>/export')
def export_notes(learner_id):
    L = Learner.query.get_or_404(learner_id)
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(['learner', 'language', 'level', 'note_id', 'note_created_at', 'tags', 'content'])
    for n in L.notes:
        writer.writerow([L.name, L.language, L.level or '', n.id, n.created_at.isoformat(), n.tags or '', n.content])
    output.seek(0)
    mem = io.BytesIO()
    mem.write(output.getvalue().encode('utf-8'))
    mem.seek(0)
    fname = f"{L.name.replace(' ','_')}_notes.csv"
    return send_file(mem, as_attachment=True, download_name=fname, mimetype='text/csv')

# Health / simple API endpoints (optional)
@app.route('/api/learners')
def api_learners():
    all_learners = Learner.query.order_by(Learner.name).all()
    return {
        "learners": [
            {
                "id": L.id,
                "name": L.name,
                "language": L.language,
                "level": L.level,
                "notes_count": len(L.notes)
            } for L in all_learners
        ]
    }

# ---------
# Main
# ---------
if __name__ == '__main__':
    init_db()
    app.run(debug=True)
