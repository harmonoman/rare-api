# Rare — Architecture Overview

This document explains how the Rare application is built, why it is built
that way, and how to approach reading an unfamiliar codebase like this one.

It is written for someone who knows some Python and JavaScript but has not
worked with Django or React Router before.

---

## The Big Picture

Rare is two separate applications that work together.

```
┌─────────────────────────────────┐
│         Web Browser             │
│                                 │
│   React Application             │
│   localhost:3000                │
└──────────────┬──────────────────┘
               │
               │  HTTP requests
               │  (fetch API)
               │
┌──────────────▼──────────────────┐
│         Django API              │
│         localhost:8088          │
│                                 │
│   Django REST Framework         │
│   URL routing → Views           │
│   Serializers ↔ JSON            │
│   Models → ORM queries          │
└──────────────┬──────────────────┘
               │
               │  SQL queries
               │
┌──────────────▼──────────────────┐
│         PostgreSQL              │
│         localhost:5433          │
│         (Docker container)      │
└─────────────────────────────────┘
```

The React application runs in the browser. When a user does something
that requires data — loading a list of posts, submitting a comment,
logging in — React sends an HTTP request to Django. Django handles the
request, talks to the database, and sends back a JSON response. React
reads the JSON and updates what the user sees.

These two applications are in separate git repositories:

- `rare-api` — the Django backend
- `rare-client` — the React frontend

---

## Following a Single Request

The best way to understand a full-stack application is to trace one
complete request from the browser to the database and back.

**Example: A user loads the list of posts.**

```
1.  User navigates to /posts in the browser.

2.  React Router matches the /posts route to the <PostList> component.

3.  PostList calls getAllPosts() from PostManager.js.

4.  PostManager.js sends:
      GET http://localhost:8088/posts
      Authorization: Token abc123...

5.  Django URL router matches /posts to the post_list view function.

6.  The view checks the Authorization header.
    Valid token → continues. Invalid/missing → returns 401.

7.  The view queries the database:
      Post.objects.all()

8.  The view passes the queryset to PostSerializer.
    Serializer converts Python Post objects → list of JSON objects.

9.  Django sends:
      HTTP 200 OK
      [ { "id": 1, "title": "...", "content": "..." }, ... ]

10. PostManager.js receives the JSON and returns it to PostList.

11. PostList renders a <div> for each post.

12. The browser displays the list.
```

Every feature in the application follows this same path. The details
change — different URL, different model, different component — but the
shape is always the same.

---

## Backend Responsibilities

### Models — What data exists

Models define the shape of your data and live in `rareapi/models/`.

Each model is a Python class that maps to a database table. Django's ORM
(Object-Relational Mapper) translates between Python objects and SQL.

```python
class Post(models.Model):
    user = models.ForeignKey('RareUser', ...)   # who wrote it
    category = models.ForeignKey('Category', ...)
    title = models.CharField(max_length=300)
    content = models.TextField()
    approved = models.BooleanField(default=False)
```

You do not write SQL directly. You write Python:

```python
Post.objects.filter(approved=True)
# Django translates this to:
# SELECT * FROM rareapi_post WHERE approved = true;
```

**Read the models first.** Every view, serializer, and component depends
on these shapes. Understanding what data exists is the prerequisite for
understanding everything else.

### Views — How requests are handled

Views live in `rareapi/views/`. Each view is a function that:

1. Receives an HTTP request
2. Checks authentication and permissions
3. Reads or writes data through the ORM
4. Returns an HTTP response

```python
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def post_list(request):
    posts = Post.objects.all()
    serializer = PostSerializer(posts, many=True)
    return Response(serializer.data)
```

Each view file groups related views together:
`post_views.py`, `comment_views.py`, `user_views.py`, etc.

### Serializers — How Python objects become JSON

Serializers live in `rareapi/serializers/`. They solve a specific problem:
Python objects cannot be sent over HTTP. JSON can. Serializers translate
between the two.

They also handle validation. When a POST request arrives with a JSON body,
the serializer validates the data before the view touches it.

```
Incoming request body (JSON)
  ↓
Serializer validates and deserializes
  ↓
Python object (ready to save to database)

Python object (from database query)
  ↓
Serializer serializes
  ↓
JSON (ready to send in response)
```

### Services — Business logic that doesn't belong in views

`rareapi/services/admin_actions.py` is the only service in this project.

It implements the two-admin vote workflow for demoting or deactivating
admins. This logic could have been written directly in the views, but it
would be harder to test and harder to reuse.

By pulling it into a service:

- It can be tested without making HTTP requests
- The view stays simple: call the service, translate the result to HTTP
- The business rules are in one place

This is worth understanding as a pattern. As applications grow, business
logic extracted into services becomes increasingly valuable.

### URLs — The API's map

`rareapi/urls.py` is the complete list of every endpoint the API exposes.
Read this file to understand what the API can do before reading any view.

```python
path('posts', post_list, name='post_list'),
path('posts/<int:pk>', post_detail, name='post_detail'),
path('posts/<int:pk>/approve', approve_post, name='approve_post'),
```

The project has 35+ endpoints covering posts, categories, tags, comments,
users, subscriptions, reactions, and admin actions.

**Note:** `APPEND_SLASH = False` in settings. URLs must not have trailing
slashes or they will 404.

### Migrations — Version control for the database

Migrations live in `rareapi/migrations/`. They are a sequential history
of every change made to the database schema.

When you run `python manage.py migrate`, Django applies every migration
in order on a fresh database. The result is a database that exactly
matches the current state of the models.

This project has 5 application migrations:

| Migration | Change |
|---|---|
| 0001_initial | Creates all models (including an `active` field on RareUser) |
| 0002_add_subject_to_comment | Adds `subject` to Comment |
| 0003_add_ended_on_to_subscription | Adds `ended_on` to Subscription |
| 0004_add_created_on_to_comment | Adds `created_on` to Comment |
| 0005_remove_rareuser_active | Removes the `active` field from RareUser |

The `active` field was added and later removed during development. This is
normal. The migration history preserves this complete record.

---

## Frontend Responsibilities

### Rare.js — The root component

`src/Rare.js` manages the application's top-level state:

- `token` — the auth token from localStorage
- `currentUserId` — the logged-in user's ID
- `isAdmin` — whether the current user is staff

On page refresh, it calls `/me` to re-derive `isAdmin` from the server.
If the token is invalid, it clears auth state and the user lands on login.

### ApplicationViews.js — The route map

`src/views/ApplicationViews.js` defines every URL in the React application
and which component renders at each URL.

It uses two route guards:

```
<Authorized>       — any logged-in user can access these routes
  <AdminOnly>      — only staff users can access these routes
```

Read this file to understand the full user-facing surface of the application.

### components/ — The UI

Components are organized by feature:

```
components/
  auth/         Login, Register
  posts/        PostList, PostDetail, PostCreate, PostEdit, ...
  comments/     CommentCreate, CommentEdit
  categories/   CategoryList, CategoryCreate, CategoryEdit
  tags/         TagList, TagCreate, TagEdit
  users/        UserProfileList, UserProfileDetail, DemotionQueueList
  reactions/    ReactionCreate
  home/         Home
  nav/          NavBar
```

Each component is responsible for one piece of the UI. They call the
managers layer to get or send data.

### managers/ — The API communication layer

Each manager file handles all API communication for one resource:

```
managers/
  api.js              — exports API base URL and shared authHeader()
  PostManager.js      — getAllPosts(), getPost(), createPost(), ...
  CommentManager.js   — getComments(), createComment(), ...
  AuthManager.js      — loginUser(), registerUser(), getMe()
  CategoryManager.js
  TagManager.js
  UserManager.js
  ReactionManager.js
  SubscriptionManager.js
```

Every fetch call in the application goes through this layer. This is the
boundary between React and Django. If an API call is broken, look here first.

All authenticated calls use the shared `authHeader()`:

```js
export const authHeader = () => ({
  "Authorization": `Token ${localStorage.getItem("auth_token")}`,
  "Accept": "application/json"
})
```

---

## Authentication Flow

Understanding authentication is critical for debugging any feature that
requires a logged-in user.

```
1. User submits login form
      ↓
2. Login component calls loginUser() from AuthManager
      ↓
3. POST /login { username, password }
      ↓
4. Django finds user, checks password, returns:
   { valid: true, token: "abc123", user_id: 7, is_staff: false }
      ↓
5. React stores token in localStorage ("auth_token")
   React stores user_id in localStorage ("current_user_id")
   React sets isAdmin state based on is_staff
      ↓
6. All subsequent API calls include:
   Authorization: Token abc123
      ↓
7. Django validates the token on every request via TokenAuthentication
   Valid → request.user is set to the token's owner
   Invalid/missing → 401 Unauthorized
```

On page refresh, the token persists in localStorage. React's root
component reads it and calls `/me` to re-establish `isAdmin`.

---

## Data Model Relationships

```
RareUser
  ├── posts (one user → many posts)
  ├── comments (one user → many comments)
  ├── post_reactions (one user → many reactions on posts)
  ├── subscriptions (one user → following many authors)
  └── subscribers (one user → followed by many followers)

Post
  ├── belongs to RareUser (author)
  ├── belongs to Category
  ├── has many Comments
  ├── has many PostTags → Tags
  └── has many PostReactions → Reactions

Category
  └── has many Posts

Tag
  └── has many PostTags → Posts

Reaction
  └── has many PostReactions → Posts + Users

DemotionQueue
  ├── belongs to RareUser (admin who initiated the action)
  └── belongs to RareUser (admin who approved — approver_one)
```

---

## How to Approach an Unfamiliar Codebase

This is the reading order that builds understanding efficiently.
Each step depends on understanding the step before it.

**1. Models** — What data exists? What are the relationships?
Read every file in `rareapi/models/`. This is the foundation. Everything
else — views, serializers, components — operates on these shapes.

**2. Migrations** — How did the data evolve?
Read the migration files in order. Look for fields that were added or
removed. This tells you the history of decisions made during development.

**3. URLs** — What can the API do?
Read `rareapi/urls.py`. This is the complete API surface. Map every
endpoint before reading any view code. You want the full picture before
the details.

**4. Views** — How does the API handle each request?
Read view files alongside the URL that points to them. Pay attention to
what authentication is required and what the view does with the data.

**5. Serializers** — What does the JSON look like?
For each view you read, find the corresponding serializer. This tells you
what fields are exposed in the API response and what is required in a
request body.

**6. Services** — What business logic lives outside the views?
Read `rareapi/services/admin_actions.py`. Understand the two-admin vote
pattern. This is the most interesting architectural decision in the backend.

**7. React Routes** — What pages exist?
Read `src/views/ApplicationViews.js`. Map URL paths to component names.
This is the equivalent of `rareapi/urls.py` for the frontend.

**8. Components** — What does the UI look like for each page?
Read components in the same order as the API: auth → posts → comments →
categories → tags → users → reactions. Each component is self-contained.

**9. API Managers** — How does the frontend call the backend?
Read each manager file. Confirm the URLs match the Django URL patterns.
Confirm the HTTP methods are correct. This is where backend/frontend
mismatches surface.

---

## Common Debugging Starting Points

| Symptom | Where to look first |
|---|---|
| Blank white screen | Browser console for JS errors |
| Network error on API call | Browser network tab; is Django running on 8088? |
| 401 Unauthorized | Is the token in localStorage? Is the auth header being sent? |
| 403 Forbidden | Is the user an admin? Does the view check `is_staff`? |
| 404 Not Found | Does the URL in the manager match `rareapi/urls.py`? Note: no trailing slashes |
| 500 Server Error | Django terminal output; check the stack trace |
| Empty list when data should exist | Was the fixture loaded? `python manage.py loaddata rareapi/fixtures/initial_data.json` |
| Tests failing after a change | Run `pytest -v` and read the specific assertion that failed |
