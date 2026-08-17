# FastAPI Project Summary

This repo is basically a small FastAPI blog app. It starts with the basics and then adds the frontend, routing, and error handling.

## 1. Project Goal

The app does a few simple but important things:
- shows a homepage with posts
- opens one post using a path parameter
- exposes the same data through an API
- renders HTML with templates
- serves static files like CSS, JS, images, and icons
- handles missing pages and validation errors

This is a good beginner project because you can see how a simple FastAPI app grows from basic routes into a real web app.

---

### Init

This was the first basic version.

Learned:
- how to create a FastAPI app with `FastAPI()`
- how `@app.get()` defines routes
- how to store temporary data in a Python list
- how FastAPI turns Python objects into JSON automatically
- how to set project dependencies in `pyproject.toml`

Example from this stage:

```python
app = FastAPI()

@app.get("/api/posts")
def get_posts():
    return posts
```

The main thing here is that FastAPI automatically turns Python dictionaries/lists into JSON responses.

Also worth noting:
- `FastAPI[standard]` gives the main app dependencies
- `jinja2` is needed for HTML templates
- `uv` is used to manage the environment and dependencies

---

### Add HTML and CSS format for the frontend

This is where the app starts looking like a real website instead of just an API.

Learned:
- `Jinja2Templates` for rendering HTML pages
- `StaticFiles` for serving CSS, images, and JS
- how to organize templates with a base layout
- `url_for()` for building safe links
- how Bootstrap helps with styling
- how template inheritance works with `{% extends "layout.html" %}` and `{% block content %}`

Basic pattern:

```python
from fastapi.templating import Jinja2Templates
from fastapi.staticfiles import StaticFiles

app = FastAPI()
app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")
```

This is the part where I realized FastAPI can do both of these at the same time:
- return JSON for APIs
- render HTML pages in the browser

Template example:

```html
{% extends "layout.html" %}
{% block content %}
  {% for post in posts %}
    <h2>{{ post.title }}</h2>
  {% endfor %}
{% endblock content %}
```

This is the core Jinja stuff I kept seeing:
- `{{ ... }}` prints values
- `{% ... %}` handles logic like loops and blocks

The `layout.html` file also shows the usual page structure:
- metadata tags
- Bootstrap CSS
- navbar
- sidebar
- footer
- dark mode toggle in JavaScript

This is the point where the backend and frontend finally started feeling connected.

---

### Path Parameter, Validation and Error Handling

This is the part that really started making FastAPI click for me.

It also sets the pattern for later updates: when a new feature gets added, I can look at the code, explain what changed, and connect it to the bigger idea.

#### A. Path parameters

A path parameter is just a variable in the URL.

```python
@app.get("/posts/{post_id}")
def get_post(post_id: int):
    for post in posts:
        if post.get("id") == post_id:
            return post
    raise HTTPException(status_code=404, detail="Post not found")
```

Why this matters:
- `/posts/1` maps to `post_id = 1`
- FastAPI automatically passes the value into the function
- type hints like `int` help with validation

#### B. Validation

FastAPI validates input automatically based on types.

Example:

```python
@app.get("/posts/{post_id}")
def get_post(post_id: int):
```

If someone calls `/posts/abc`, FastAPI will reject it before entering the function, returning a 422 validation error.

This is one of the biggest strengths of FastAPI: automatic request validation with type hints.

#### C. HTTPException

This is the standard way to raise errors in FastAPI.

```python
raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Post not found")
```

This gives:
- proper HTTP status codes
- structured error messages
- a clean API error response

#### D. Custom exception handlers

This file also demonstrates custom handlers for different error types:

```python
@app.exception_handler(StarletteHTTPException)
def general_http_exception_handler(...):
```

and

```python
@app.exception_handler(RequestValidationError)
def validation_exception_handler(...):
```

Why this matters:
- API routes can return JSON errors
- browser routes can render a friendly HTML error page

Example behavior:
- `/api/posts/abc` returns JSON with validation details
- `/posts/abc` returns a template-based error page

This is a very important concept in real FastAPI apps because users on the browser and API clients need different error types.

---

### Pydantic Schemas - Request and Response Validation

This is the commit where the app started using Pydantic models properly instead of just raw dictionaries.

This is a big step because now the app is not only returning data, it is validating the shape of the data coming in and going out.

What changed:
- a new `schemas.py` file was added
- `PostBase`, `PostCreate`, `PostUpdate`, and `PostResponse` were created
- endpoints started using `response_model=`
- a `POST /api/posts` route was added to create a new post

The model setup looks like this:

```python
class PostBase(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    content: str = Field(min_length=1)
    author: str = Field(min_length=1, max_length=50)

class PostCreate(PostBase):
    pass

class PostResponse(PostBase):
    id: int
    date_posted: str
```

This matters because Pydantic gives a structured way to define what the API expects and what it returns.

So instead of just trusting the input, FastAPI checks things like:
- title is a string
- content cannot be empty
- author has a max length
- id exists in the response model

The route for creating a post looks like this:

```python
@app.post("/api/posts", response_model=PostResponse, status_code=status.HTTP_201_CREATED)
def create_post(post: PostCreate):
    new_id = max(p["id"] for p in posts) + 1 if posts else 1
    new_post = {
        "id": new_id,
        "author": post.author,
        "title": post.title,
        "content": post.content,
        "date_posted": "April 23, 2025",
    }
    posts.append(new_post)
    return new_post
```

This is where the real validation pattern starts to become obvious:
- `post: PostCreate` validates the incoming request body
- `response_model=PostResponse` makes sure the response matches the expected format
- `status_code=201` tells the client that a new resource was created

This is one of the biggest FastAPI features in real apps. You define the schema once, and FastAPI handles a lot of the validation and serialization work for you.

---

## 3. Core FastAPI concepts learned in this repo

### A. App creation

```python
app = FastAPI()
```

This creates the web application instance that handles routing, validation, and responses.

### B. Route decorators

```python
@app.get("/")
@app.get("/posts")
```

Routes define the endpoints the application exposes. The decorator tells FastAPI which HTTP method and path to handle.

### C. Request object

```python
from fastapi import Request
```

The `Request` object gives access to the incoming HTTP request information, including:
- URL
- headers
- cookies
- path
- method

This is important for rendering templates or checking the request context.

### D. TemplateResponse

```python
return templates.TemplateResponse(
    request,
    "home.html",
    {"posts": posts, "title": "Home"},
)
```

This is how FastAPI renders an HTML page with Jinja templates.

It sends:
- the request object
- the template file name
- a context dictionary with values to display in the page

### E. Static files

```python
app.mount("/static", StaticFiles(directory="static"), name="static")
```

This makes files in the `static` folder accessible from the browser, such as:
- CSS
- images
- JS
- manifests

Example usage:

```html
<link rel="stylesheet" type="text/css" href="{{ url_for('static', path='css/main.css') }}">
```

### F. Response types

The project uses two main response styles:

1. JSON responses for APIs
2. HTML/template responses for browser pages

API example:

```python
@app.get("/api/posts")
def get_posts():
    return posts
```

HTML example:

```python
return templates.TemplateResponse(request, "home.html", {"posts": posts})
```

### G. Pydantic models and validation

This is the key idea introduced in the latest commit.

The app now has models like `PostCreate` and `PostResponse` in `schemas.py`, and FastAPI uses them to validate incoming data and shape outgoing responses.

Example:

```python
@app.post("/api/posts", response_model=PostResponse)
def create_post(post: PostCreate):
    ...
```

So the flow is basically:
- the request body is validated against `PostCreate`
- the function runs if it passes
- the result is converted to the response format defined by `PostResponse`

This is what makes FastAPI feel much more structured than a plain Python route function.

### H. In-memory data model

The app stores blog posts as a Python list of dictionaries:

```python
posts: list[dict] = [
    {
        "id": 1,
        "author": "Ari Curlind",
        "title": "FastAPI is Awesome",
        "content": "...",
        "date_posted": "August 20, 2021",
    }
]
```

This is a simple mock database for learning. In real production projects, you would usually use a database like SQLite, PostgreSQL, or MongoDB.

---

## 4. Important frontend-template concepts used in this repo

### Jinja syntax

- `{{ post.title }}`: render variable content
- `{% for post in posts %}`: loop over data
- `{% block content %}`: reusable template section
- `{% extends "layout.html" %}`: inherit from a base template

These are standard Jinja features and are used heavily in Flask and FastAPI projects.

### URL generation with `url_for`

```html
<a href="{{ url_for('post_page', post_id=post.id) }}">{{ post.title }}</a>
```

This is important because it creates URLs without hardcoding paths. It helps prevent broken links and keeps routes consistent.

---

## 5. File-by-file explanation

### `main.py`

This is the main application file. It contains:
- app setup
- route definitions
- post data
- template rendering
- custom exception handlers
- path parameter logic

This file is the heart of the project and demonstrates the core of FastAPI development.

### `templates/layout.html`

This is the base HTML structure for the website.

It includes:
- metadata
- Bootstrap CDN link
- navbar
- sidebar
- footer
- dark mode script
- main block for content

This teaches how to design reusable page templates.

### `templates/home.html`

This is the homepage template. It loops through the posts and displays a blog list.

### `templates/post.html`

This page displays a single post with:
- author
- date
- title
- content
- delete modal UI skeleton

It shows how to build a detail page from a specific post ID.

### `templates/error.html`

This is the HTML page rendered for errors like 404 or validation issues when the request is for a browser page.

### `schemas.py`

This file is where the Pydantic models live.

It defines:
- `PostBase` for the shared fields
- `PostCreate` for input validation when creating a post
- `PostUpdate` for update-style models
- `PostResponse` for the response shape returned by the API

This is the file that really shows the move from raw dicts to validated structured data.

### `static/`

This folder holds all public frontend assets:
- CSS styles
- icons
- profile images
- web manifest
- JS utilities

This is how FastAPI serves asset files in a website project.

### `pyproject.toml`

This defines the project metadata and dependencies.

It includes:
- project name/version
- Python requirement
- FastAPI dependency
- Jinja2 dependency

---

## 6. What makes FastAPI useful here

FastAPI is powerful because it combines:
- automatic data validation
- clean routing
- easy HTML view rendering
- API response generation
- error handling
- modern Python syntax

In this project, you see all of that working together in a simple app.

The biggest lessons are:
1. route decorators describe what URLs your app exposes
2. type hints are used for validation
3. HTML pages can be rendered with templates
4. static files are served separately from application code
5. custom exception handlers improve both API and browser UX

---

## 7. Important best practices visible in the project

- separate API routes from UI routes when possible
- use proper HTTP status codes
- provide user-friendly error pages for browser requests
- use JSON responses for API requests
- use template rendering for frontend pages
- keep HTML structure reusable through base templates

---

## 8. Real-world next steps from here

This project is a strong foundation, but the next real-world improvements would be:
- connect to a real database
- add CRUD operations (create, update, delete)
- add Pydantic models for data validation
- add authentication and authorization
- add database migrations
- add tests
- separate API and frontend more cleanly

A typical production version would likely use:
- FastAPI + Pydantic
- SQLAlchemy or another ORM
- SQLite/PostgreSQL
- templates or a frontend framework like React/Vue

---

## 9. Future commit extension pattern

This summary is intentionally structured so future commits can be added as new chapters without changing the earlier explanations too much.

When a new commit is introduced, add a new section in this format:

### [short commit title]

- What changed in the app?
- What new FastAPI concept was introduced?
- Which file or files were changed?
- What new route, pattern, feature, or best practice was learned?
- How does this change fit into the overall progression?

Then update the relevant earlier parts only if the new commit changes the way the app is structured or introduces a concept that should be listed in the core concepts section.

Examples of updates that may be needed later:
- add a new route type such as `POST`, `PUT`, or `DELETE`
- add database integration
- add Pydantic models
- add authentication
- add form handling
- add tests
- move the app into a more production-ready structure

---

## 10. Final takeaway

This repo teaches the core FastAPI workflow:

- define routes
- pass data into templates
- serve static assets
- validate inputs with Python types
- return JSON for APIs
- display HTML for browser pages
- handle errors gracefully

That is the foundation of most FastAPI applications.

If you understand everything in this repo, you already understand the basic architecture of a simple FastAPI web app.

As additional commits are added later, keep extending the same structure with a new chapter per commit and update the relevant concept sections when needed.
