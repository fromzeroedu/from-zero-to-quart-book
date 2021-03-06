

# QuartFeed, an SSE appplication using MySQL <!-- 5 -->


## Introduction <!-- 5.1 -->
Server Sent Events, or SSEs, or EventSource in JavaScript, are an extension to HTTP that allow a client to keep a connection open to a server, thereby allowing the server to send events to the client as it chooses.

By default, the server sends updates with a `data` payload. You can also have an `event` type, which by default is `message`, but could be things like `add` or `remove`. Additionally it has an `id` parameter that allows the client to continue where it left off if the connection was lost.

We are going to build a lightweight version of the popular FriendFeed website, one of the pioneers in the social media space. using Quart and SSE.

For our FriendFeed clone we’ll have the event type to be either `post`, which is a new post, `like` if some one liked the post and `comment` if it’s a comment to a `post`.

For a more complex version or exercise to students, we could also have `groups`, which could be distinct `/sse` endpoints and `like` events for comments.

## Setup (step-0) <!-- 5.2 -->
- Git clone/fork [https://github.com/esfoobar/quart-mysql-boilerplate](https://github.com/esfoobar/quart-mysql-boilerplate)

- Pipenv install requirements: `pipenv install`

- Install MySQL, create database and user
	- Start Mysql: `mysql.server start`
		Login to MySQL with your root user and password:
		`mysql -uroot -prootpass`
		Create the database:
		`CREATE DATABASE quartfeed;`
		And now create the user and password:
		`CREATE USER 'qf_user'@'%' IDENTIFIED BY 'qf_password';`
		Allow the user full access to the database:
		`GRANT ALL PRIVILEGES ON quartfeed.* TO 'qf_user'@'%';`
		And reload the privileges:
		`FLUSH PRIVILEGES;`
	- Update `.quartenv`

- Create User Module
	- Rename `counter` to `user`
	- Edit `user/models`
	- Edit `user/views`
	- Update `application.py` blueprint imports on line 16 and 20 to `user_app`
	- Erase the `conftest` and `test_counter` files. We’ll create testing later.
	- 
	
- Start the app to see if things are working so far
	
- `pipenv run quart run` and open `/registration`
	
- Alembic
	
	- Note: If you are using Docker, you do ` docker-compose run --rm web pipenv run alembic revision --autogenerate -m "create user cursor table"`
		
	- Remove existing `migrations`
	- `pipenv run alembic init migrations`
	- Edit the `alembic.ini` script and add the `sqlalchemy.url` from `settings` on line 38. These setting vars will come from the `env.py` we’ll write later:
	
	  - `sqlalchemy.url = mysql+pymysql://%(DB_USERNAME)s:%(DB_PASSWORD)s@%(DB_HOST)s:3306/%(DATABASE_NAME)s`
	- Edit the `env.py` in migrations on line 27:
	  ```python
	  	section = config.config_ini_section
	  	config.set_section_option(section, "DB_USERNAME", os.environ.get("DB_USERNAME"))
	  	config.set_section_option(section, "DB_PASSWORD", os.environ.get("DB_PASSWORD"))
	  	config.set_section_option(section, "DB_HOST", os.environ.get("DB_HOST"))
	  	config.set_section_option(section, "DATABASE_NAME", os.environ.get("DATABASE_NAME"))
	  ```
	- Add the following at the top (before `logging.config`:
	  ```python
	  	import os, sys
	  	from dotenv import load_dotenv
	  	from pathlib import Path
	  ```
	- Then add this under `from alembic import context`:
	  ```python
	  	# Path ops
	  	parent = Path(__file__).resolve().parents[1]
	  	load_dotenv(os.path.join(parent, ".quartenv"))
	  	sys.path.append(str(parent))
	  ```
	- Finally we need to add a metadata object on line 29 (where it says `target_metadata=None`).  The import needs to happen here after the `sys.path.append` above:
	  ```python
	  	from user.models import metadata as UserMetadata
	  	target_metadata = [UserMetadata]
	  ```
	
- We’re ready now to create the application structure
	
- Migration Execution
	- Then create the first commit with `pipenv run alembic revision --autogenerate -m "create user table"`
	- Check that the versions file was created properly, and then,
	- Run the first migration with `pipenv run alembic upgrade head `
	- Log into the mysql and check the table as well as `alembic_version`.
		- `describe user;`

- Finally run the server:
	- `pipenv run quart run`
	- Go to `localhost:5000/registration`. You shouldn’t get any errors.

## User Registration - Initial Setup (step-1) <!-- 5.3 -->

We’re going to create a templates folder with a base, navbar templates using bootstrap. We’ll also create the `user` templates folder and create `register.html` template on it.

Then let’s modify the `user/views` and add the template rendering. Notice the format: `return await render_template` and not `await return render_template`.



## User Registration - Parsing the Form (step-2) <!-- 5.4 -->
Notice the `form = await request.form`,  and not `await form = request.form`.

Let’s check the user enters all the fields and setup an error variable




## User Registration - CSRF, check existing user and Password Hashing (step-3) <!-- 5.5 -->

Normally CSRF is included in libraries like Flask-WTForms, which don’t work in Quart, since they use the Flask request object, so we’ll do a quick implementation for it.

We’ll generate a UUID and store it in the session, and then check that the token on the form matches it.

We’ll then check that the username hasn’t been used earlier.

Finally, before we register the user, we don’t want to store clear passwords. Normally I would use Werkzeug, but Quart doesn’t include it, so we’ll install the passlib library. So do: `pipenv install passlib==1.7.1 `

Now let’s encode the password for the database.

And now we’ll save the user on the database.

## User Login (step-4) <!-- 5.6 -->

Implement Login and Logout using sessions and update navbar.

### Testing User Registration and User Login (step-5) <!-- 5.7 -->

Add the `create_all` method from the `test_counter` file and make it use the UserMetadata from the user model

Let’s first test the initial response and see that we get the registration page

We’re going to modify the login page and add a flash message if the user is coming from a successful registration. We’ll use this string to test if users are being registered.

We’ll also check in the database (e2e testing)

Then we’ll test registering a user without email or password

I notice that the test :
```python
    # missing password
    response = await create_test_client.post(
        "/register", form={"username": "testuser", "password": ""}
    )
    body = await response.get_data()
    assert "Please enter username and password" in str(body)
```

Is not passing. Why? Because the database lookup returns a result when passing an existing user (testuser) with no password, so I will add a “if not error” on the user exists block

```python
        # check if the user exists
        if not error:
            conn = current_app.sac
            stmt = user_table.select().where(
                user_table.c.username == form.get("username")
            )
            result = await conn.execute(stmt)
            row = await result.fetchone()
            if row and row.id:
                error = "Username already exists"
```

When doing the test with an unknown user:

```python
    response = await create_test_client.post(
        "/login", form={"username": "testuser2", "password": "test123"}
    )
```

I notice that it’s not passing because I’m assuming there’s a row returned:
```python
if not pbkdf2_sha256.verify(password, row.password):
            error = "User not found"
```

So I change it to an `elif`:
```python
        if not row:
            error = "User not found"
        # check the password
  >>    elif not pbkdf2_sha256.verify(password, row.password):
            error = "User not found"
```

Which goes to show you, testing makes your application better and safer.

Run the tests using `pipenv run pytest`

# Relationship Module (step-6)

- Relationship: set relationships `fm_userid -> to_userid`

We’ll create the relationship between users. We can enforce bi-directional (like Facebook friends) or unidirectional (like Twitter followers). We’ll do the second to simplify the code.

Create the model:

```python
from sqlalchemy import Column, Table, Integer, ForeignKey

from user.models import metadata as UserMetadata

relationship_table = Table(
    "relationship",
    UserMetadata,
    Column("id", Integer, primary_key=True),
    Column("fm_user_id", Integer, ForeignKey("user.id")),
    Column("to_user_id", Integer, ForeignKey("user.id")),
)
```
We need to use the `UserMetadata` since we are going to establish a foreign key to the user table.

Add the `relationship_table` to migrations/env.py
```python
from user.models import metadata as UserMetadata
from relationship.models import relationship_table

target_metadata = [UserMetadata]
```
We import it so that the auto-discovery can “see” the new table. However, notice we don’t add anything to the `target_metadata`_ 
Migration Execution
- Create the commit with `pipenv run alembic revision --autogenerate -m "create relationship table"`
- Check that the versions file was created properly, and then,
- Run the first migration with `pipenv run alembic upgrade head `

Create a simple profile user view so that you can follow other users:
- Created a `profile` route function, where we fetch the username and if we don’t find him, return a 404, and then fetch the relationship status
- Created a basic `profile.html` with a follow/unfollow/edit profile button
- Create `relationship.views` with the routes
- Add `relationshp` blueprint to `application`
- Add `login_required` decorator

# Follow/Unfollow users (step-7)

- Added  common methods to get users and check relationships to start a DRY pattern
- Created relationship add and remove
- Write tests
	- Need to change she scope of the `event_loop` scope to session, otherwise the relationship module runs and closes the loop


# Profile Edit (step-8)

# Profile edit
 - Setup form
- Views processing

# Add a profile image (step-9)

- Add a profile image
	- [Profile template](https://github.com/esfoobar/flaskbook/blob/master/templates/user/profile.html), [view processing](https://github.com/esfoobar/flaskbook/blob/master/user/views.py#L144-L149), [resize utility](https://github.com/esfoobar/flaskbook/blob/68fbd6ebd5344ff5a9a45dc2b607187a39490562/utilities/imaging.py)

- added imagemagick-dev to Dockerfile
- added wand to Pipfile
- Added enctype to form and image section
- Added UPLOADS\_FOLDER, IMAGES\_FOLDER, IMAGE\_URL to settings, .quartenv, docker-compose.yml
- Added saving functionality on user/views.py:profile\_edit
- added utilities/imaging.py
- -Added profile image\_url dict on get user by username
- added image field to user , did a migration:
	- `docker-compose run --rm web pipenv run alembic revision --autogenerate -m "added new image field user"` and then `docker-compose run --rm web pipenv run alembic upgrade head`
- Added image to profile edit and to profile page
- Added empty profile image to repo

# Create a Post  (step-10)

- created the post blueprint with the basic view controller
- created post model
	- Added server side datetiime using `server_defaault` per [instructions here]((https://stackoverflow.com/questions/13370317/sqlalchemy-default-datetime)
- created home page post form
- add the new model to migrations/env.py
- did a migration


# Friends Posts on Homepage using SSE (step-11)

- added broadcast.js
- added broadcast.js on home page
- created followers method on relationship
- add the post to the followers feed 
- added updated field to feed table
- get the latest 10 posts from feed order by desc updated
- user image url method (`image_url_from_image_ts`)
- IMPORTANT: This step just prints out the events to the JS Console, no templating yet. The initial render does work on the home page

# Template Literals (step-12)

- Create template literal for the whole section
- Pass the full post information on the context
- To print the actual statement: `print(stmt.compile(compile_kwargs={"literal_binds": True}))`
- NOTE: I've noticed that if you do code changes while the Quart application is running and the SSE page is open, erratic behavior can appear. Make sure to do the code changes, save, then stop and start Docker and do a HARD RELOAD of the browser page

# Databases Migration (step-13)

- Moved database connectivity over to [databases](https://www.encode.io/databases/)
- Testing and migrations updates

# Comments (step-14)

- Setup comments form toggle display and entry
- Post comment to backend
- Get comments query and add to context
- Render new comments from SSE

# Likes (step-15)

- Post likes to backend
- Finish frontend

- Test posts. comments and likes

