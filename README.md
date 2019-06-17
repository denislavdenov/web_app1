# How to create a python web app

## Prerequirements 
- python 3.7
- Flask
- Pipenv
- Vagrant

The Basic Directory Structure

Being a micro-framework, Flask doesn't have a lot of opinions about how we structure our project. Because of this, we're going to keep it as simple as we can and to start, we won't have many directories at all. Here are the things that we want to create:

- Pipfile (via pipenv install) - To specify our Python version and dependencies
- /templates - To eventually hold Jinja templates that we'll use to render HTML
- /static - To hold static assets like HTML, CSS, and JavaScript files

```
cd ~/web_app1
$ mkdir notes
$ cd notes
$ pipenv --python python3.7 install flask
...
$ pipenv shell
(notes) $ mkdir templates static
(notes) $ touch {templates,static}/.gitkeep

```

```
(notes) $ curl -o .gitignore https://raw.githubusercontent.com/github/gitignore/master/Python.gitignore
(notes) $ git init
Initialized empty Git repository in /home/cloud_user/projects/notes/.git/
(notes) $ git add --all .
(notes) $ git commit -m 'Initial commit'
[master (root-commit) ef81eca] Initial commit
 5 files changed, 218 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 Pipfile
 create mode 100644 Pipfile.lock
 create mode 100644 static/.gitkeep
 create mode 100644 templates/.gitkeep

```

Then commit add your remote GH repo and push.

###Our Application Factory

There are a few different ways that we can go about creating Flask applications, but we're going to use the "application factory" approach. This approach is shown off in the official Flask tutorial and I really think it's a great way to get started. To begin, let's create an __init__.py in our notes project that will hold our application factory method:

**notes/__init__.py:**

```
import os

from flask import Flask

def create_app(test_config=None):
    app = Flask(__name__)
    app.config.from_mapping(
        SECRET_KEY=os.environ.get('SECRET_KEY', default='dev'),
    )

    if test_config is None:
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    return app

```

To run our application, we'll use the flask CLI after setting a few environment variables:

```
(notes) $ export FLASK_ENV=development
(notes) $ export FLASK_APP='.'
(notes) $ flask run --host=0.0.0.0 --port=3000
 * Serving Flask app "." (lazy loading)
 * Environment: development
 * Debug mode: on
 * Running on http://0.0.0.0:3000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 143-698-723

```

###Creating a Database

For our note-taking application to be useful, we're going to need to be able to store data in a database. We'll be using a PostgreSQL database. If you haven't, then create a new CentOS 7 and run the following commands after it's been spun up:

`vagrant up` in `db` directory

```
curl -o db_setup.sh https://raw.githubusercontent.com/linuxacademy/content-python-for-sys-admins/master/helpers/db_setup.sh
$ chmod +x db_setup.sh
$ ./db_setup.sh

```

`docker exec -i postgres psql postgres -U notes -c "CREATE DATABASE notes;"`


Now we have a database that our new application can use.

###Configuring Database Access


**notes/config.py**
```
import os

db_host = os.environ.get('DB_HOST', default='localhost')
db_name = os.environ.get('DB_NAME', default='notes')
db_user = os.environ.get('DB_USERNAME', default='notes')
db_password = os.environ.get('DB_PASSWORD', default='')
db_port = os.environ.get('DB_PORT', default='5432')

SQLALCHEMY_TRACK_MODIFICATIONS = False
SQLALCHEMY_DATABASE_URI = f"postgres://{db_user}:{db_password}@{db_host}:{db_port}/{db_name}"

```

**notes/.env:**
```
export DB_USERNAME='notes'
export DB_PASSWORD='notes'
export DB_HOST='<PUBLIC_IP>'
export DB_PORT='80'
export FLASK_ENV='development'
export FLASK_APP='.'

```

`source .env`

`pipenv install --dev python-dotenv`

`flask run --host=0.0.0.0 --port=3000`


###Installing SQLAlchemy

(notes) $ `pipenv install psycopg2-binary Flask-SQLAlchemy Flask-Migrate`


###Modeling a User

**notes/models.py:**

```
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password = db.Column(db.String(200))
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, server_default=db.func.now(), server_onupdate=db.func.now())
    notes = db.relationship('Note', backref='author', lazy=True)

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    body = db.Column(db.Text)
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, server_default=db.func.now(), server_onupdate=db.func.now())
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

```

###Registering our Database with Our Application

edit **notes/__init__.py:** to look like this:

```
import os

from flask import Flask
from flask_migrate import Migrate

def create_app(test_config=None):
    app = Flask(__name__)
    app.config.from_mapping(
        SECRET_KEY=os.environ.get('SECRET_KEY', default='dev'),
    )

    if test_config is None:
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    from .models import db

    db.init_app(app)
    migrate = Migrate(app, db)

    return app

```

`flask db init`
`flask db migrate`
`flask db upgrade`


###Creating Our First View

edit **notes/__init__.py:** to look like this:

```
import os

from flask import Flask, render_template
from flask_migrate import Migrate

def create_app(test_config=None):
    # previous code omitted
    db.init_app(app)
    migrate = Migrate(app, db)

    @app.route('/sign_up')
    def sign_up():
        return render_template('sign_up.html')
    return app

```


###The Sign Up Template

```
(notes) $ cd /tmp
(notes) $ git clone https://github.com/linuxacademy/content-intro-to-python-development.git
...
(notes) $ cd content-intro-to-python-development
(notes) $ git checkout use-case-web-app
...
(notes) $ cp -R starter_templates ~/<PathToCode>/notes/templates
(notes) $ cp -R starter_static ~/<PathToCode>/notes/static

```

###Handling form submission

edit **notes/__init__.py:** to look like this:

```
import os

from flask import Flask, render_template, redirect, url_for, request, flash
from flask_migrate import Migrate
from werkzeug.security import generate_password_hash

def create_app(test_config=None):
    # Initial setup omitted

    from .models import db, User

    db.init_app(app)
    migrate = Migrate(app, db)

    @app.route('/sign_up', methods=('GET', 'POST'))
    def sign_up():
        if request.method == 'POST':
            username = request.form['username']
            password = request.form['password']
            error = None

            if not username:
                error = 'Username is required.'
            elif not password:
                error = 'Password is required.'
            elif User.query.filter_by(username=username).first():
                error = 'Username is already taken.'

            if error is None:
                user = User(username=username, password=generate_password_hash(password))
                db.session.add(user)
                db.session.commit()
                flash("Successfully signed up! Please log in.", 'success')
                return redirect(url_for('log_in'))

            flash(error, 'error')

        return render_template('sign_up.html')

    @app.route('/log_in')
    def log_in():
        return "Login"

    return app

```

Now try to sing up at `http://localhost:3000/sign_up`


###Logging In

With our user in the database, we're ready to log in. Let's start by creating the view. We're going to duplicate the sign_up.html as log_in.html and then modify a few things:

**templates/log_in.html:**

```
{% extends "base.html" %}

{% block content %}

<div class="columns is-desktop">
  <div class="column"></div>
  <div class="column is-half-desktop">
    <h2 class="is-size-3">Log In</h2>

    <form method="post" action="{{ url_for('log_in') }}">
      <div class="field">
        <label class="label" for="username">Username</label>
        <div class="control">
          <input name="username" type="input" class="input" required></input>
        </div>
      </div>

      <div class="field">
        <label class="label" for="password">Password</label>
        <div class="control">
          <input name="password" type="password" class="input" required></input>
        </div>
      </div>

      <div class="field is-grouped">
        <div class="control">
          <input type="submit" value="Log In" class="button is-link" />
        </div>
        <div class="control">
          <a href="{{ url_for('sign_up') }}" class="button is-text">
            Don't have an account? Sign Up.
          </a>
        </div>
      </div>
    </form>
  </div>
  <div class="column"></div>
</div>

{% endblock %}

```

edit **notes/__init__.py:** to look like this:

```
import os

from flask import Flask, render_template, redirect, url_for, request, session, flash
from flask_migrate import Migrate
from werkzeug.security import generate_password_hash, check_password_hash

def create_app(test_config=None):
    # Previous code omitted

    @app.route('/log_in', methods=('GET', 'POST'))
    def log_in():
        if request.method == 'POST':
            username = request.form['username']
            password = request.form['password']
            error = None

            user = User.query.filter_by(username=username).first()

            if not user or not check_password_hash(user.password, password):
                error = 'Username or password are incorrect'

            if error is None:
                session.clear()
                session['user_id'] = user.id
                return redirect(url_for('index'))

            flash(error, category='error')
        return render_template('log_in.html')

    @app.route('/')
    def index():
        return 'Index'

    return app

```

###Logging Out

edit **notes/__init__.py:** to look like this:

```
# imports omitted

def create_app(test_config=None):
    # Previous code omitted

    @app.route('/log_out', methods=('GET', 'DELETE'))
    def log_out():
        session.clear()
        flash('Successfully logged out.', 'success')
        return redirect(url_for('log_in'))

    return app

```

Test logging in now both with incorrect and correct password.
After you see the index page do:
`http://localhost:3000/log_out`


###Populating user Based on the Session

edit **notes/__init__.py:** 

```
import os
import functools

from flask import Flask, render_template, redirect, url_for, request, session, flash, g
from flask_migrate import Migrate
from werkzeug.security import generate_password_hash, check_password_hash

def create_app(test_config=None):
    # Ommit initial setup

    db.init_app(app)
    migrate = Migrate(app, db)

    def require_login(view):
        @functools.wraps(view)
        def wrapped_view(**kwargs):
            if not g.user:
                return redirect(url_for('log_in'))
            return view(**kwargs)
        return wrapped_view

    @app.before_request
    def load_user():
        user_id = session.get('user_id')
        if user_id:
            g.user = User.query.get(user_id)
        else:
            g.user = None

    # Remaining code omitted

```

(notes) $ `cp /tmp/content-intro-to-python-development/note_templates/* ~/<PathToProject>/notes/templates/`

###Creating a Note

edit **notes/__init__.py:** 

```
# Imports omitted

def create_app(test_config=None):
    # Initial setup omitted

    from .models import db, User, Note

    db.init_app(app)
    migrate = Migrate(app, db)

    # Earlier views omitted

    @app.route('/notes')
    @require_login
    def note_index():
        return 'Note Index'

    @app.route('/notes/new', methods=('GET', 'POST'))
    @require_login
    def note_create():
        if request.method == 'POST':
            title = request.form['title']
            body = request.form['body']
            error = None

            if not title:
                error = 'Title is required.'

            if not error:
                note = Note(author=g.user, title=title, body=body)
                db.session.add(note)
                db.session.commit()
                flash(f"Successfully created note: '{title}'", 'success')
                return redirect(url_for('note_index'))

            flash(error, 'error')

        return render_template('note_create.html')

    return app

```

###Parsing Markdown

(notes) $ `pipenv install mistune`

edit **notes/models.py:**

```
from flask_sqlalchemy import SQLAlchemy
from mistune import markdown

# db and User omitted

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    body = db.Column(db.Text)
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, server_default=db.func.now(), server_onupdate=db.func.now())
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

    @property
    def body_html(self):
        return markdown(self.body)

```

###Rendering Notes


edit **notes/__init__.py:**

Finally your `init.py` file should look like it is in the repo.


Copy now all the html files from `/tmp/content-intro-to-python-development/notes/templates/` to your templates folder because of the new references to ne

If you visit `http://localhost:3000/notes` now you are supposed to be required to log in since we previously logged out.

After login you will be able to play with notes.

```
<p>You haven't created any notes! <a href="{{ url_for('note_create') }}">Create your first note.</a>
```