# 📝 Blog Posts

A full-featured, multi-user blog built with **Flask**. An admin writes and publishes rich-text posts, readers register and log in to leave comments, and every commenter gets an automatic Gravatar avatar. It uses SQLAlchemy relational models, secure password hashing, role-based access, and is ready to deploy with Gunicorn and PostgreSQL.

---

## 📚 Table of Contents

- [Features](#-features)
- [Tech Stack](#️-tech-stack)
- [Architecture](#️-architecture)
- [Detailed Workflow](#-detailed-workflow)
  - [1. App Startup](#1-app-startup)
  - [2. Registration](#2-registration)
  - [3. Login and Logout](#3-login-and-logout)
  - [4. Reading Posts](#4-reading-posts)
  - [5. Commenting](#5-commenting)
  - [6. Admin: Creating a Post](#6-admin-creating-a-post)
  - [7. Admin: Editing and Deleting](#7-admin-editing-and-deleting)
  - [8. Role-Based Access](#8-role-based-access)
- [Advantages](#-advantages)
- [Data Model](#️-data-model)
- [Routes](#️-routes)
- [Project Structure](#-project-structure)
- [Local Setup](#️-local-setup)
- [Environment Variables](#-environment-variables)
- [Deployment](#️-deployment)
- [Known Limitations and Future Improvements](#-known-limitations-and-future-improvements)
- [Author](#-author)

---

## 🚀 Features

- 📰 **Home feed** listing every post with title, subtitle, author and date.
- 📖 **Post pages** with a full-width header image and rich HTML content.
- ✍️ **Rich-text editor** (CKEditor) for writing posts and comments.
- 👤 **User accounts**: register, log in and log out with Flask-Login sessions.
- 🔒 **Secure passwords**: hashed and salted with PBKDF2-SHA256 (Werkzeug), never stored in plain text.
- 💬 **Comments** on every post, only for logged-in users.
- 🖼️ **Gravatar avatars** generated from each commenter's email.
- 🛡️ **Admin-only actions**: only the admin (user id 1) can create and delete posts; admin controls are hidden from everyone else.
- ✏️ **Edit posts** with the form pre-filled with the existing content.
- 🧾 **Form validation** with Flask-WTF / WTForms (required fields, valid image URL) and CSRF protection.
- 💬 **Flash messages** for errors like "email already registered" or "wrong password".
- 📱 **Responsive design** based on the Start Bootstrap *Clean Blog* theme with Bootstrap 5, Font Awesome and Google Fonts.
- 🧭 **Smart navbar** that hides when scrolling down and reappears when scrolling up.
- ℹ️ **About and Contact pages**.
- ☁️ **Deploy-ready** with Gunicorn (`Procfile`) and PostgreSQL via `DB_URL`.

---

## 🛠️ Tech Stack

| Layer | Technology | Role in the project |
|-------|-----------|---------------------|
| Language | Python 3.11 | Runtime |
| Web framework | Flask 2.3 | Routing, request handling, Jinja2 templates |
| ORM | Flask-SQLAlchemy 3.1 + SQLAlchemy 2.0 | Typed models (`Mapped`, `mapped_column`) and relationships |
| Database | PostgreSQL (`psycopg2-binary`) or SQLite | Users, posts and comments |
| Auth | Flask-Login | Session login, `current_user`, `UserMixin` |
| Password security | Werkzeug `generate_password_hash` / `check_password_hash` | PBKDF2-SHA256 with an 8-character salt |
| Forms | Flask-WTF + WTForms | Form classes, validation, CSRF tokens |
| Rich text | Flask-CKEditor | WYSIWYG editor for posts and comments |
| UI | Bootstrap-Flask (Bootstrap 5), Clean Blog theme | Layout and `render_form` macro |
| Avatars | Flask-Gravatar | Profile pictures from email hashes |
| Icons and fonts | Font Awesome 6, Google Fonts (Lora, Open Sans) | Typography and icons |
| App server | Gunicorn | Production WSGI server (`Procfile`) |

---

## 🏗️ Architecture

```
                    ┌───────────────────────────────┐
                    │            Browser            │
                    │ Clean Blog theme + Bootstrap 5│
                    │ CKEditor │ Gravatar images    │
                    └───────────────┬───────────────┘
                                    │ HTTP + session cookie + CSRF token
                    ┌───────────────▼───────────────┐
                    │          Flask (main.py)      │
                    │                               │
                    │  Flask-Login  (who is it?)    │
                    │  admin_only   (id == 1?)      │
                    │  Flask-WTF    (forms.py)      │
                    │  Jinja2 templates             │
                    └───────────────┬───────────────┘
                                    │ SQLAlchemy ORM
                    ┌───────────────▼───────────────┐
                    │  users ─┬─< blog_posts        │
                    │         └─< comments >─┘      │
                    │       PostgreSQL / SQLite     │
                    └───────────────────────────────┘
```

Everything lives in `main.py` (app, models, routes) and `forms.py` (WTForms classes), with Jinja2 templates sharing a common `header.html` and `footer.html`.

---

## 🔄 Detailed Workflow

### 1. App Startup

1. Flask reads `FLASK_KEY` (used to sign sessions and CSRF tokens) and `DB_URL` from the environment.
2. CKEditor, Bootstrap 5, Flask-Login and Gravatar are initialized on the app.
3. SQLAlchemy connects to the database and `db.create_all()` creates the `users`, `blog_posts` and `comments` tables if they do not exist, so no manual migration step is needed.

### 2. Registration

```
GET/POST /register
      │
      ▼
RegisterForm (email, password, name) validated
      │
      ▼
Email already in users table? ── yes ──► flash "already signed up" → /login
      │ no
      ▼
generate_password_hash(password, method="pbkdf2:sha256", salt_length=8)
      │
      ▼
New User saved → login_user(new_user) → redirect to home
```

The user is logged in right after signing up, with no extra step.

### 3. Login and Logout

```
POST /login (email, password)
      │
      ├── email not found      → flash "That email does not exist" → /login
      ├── password wrong       → flash "Password incorrect"        → /login
      └── check_password_hash ok → login_user(user) → home
```

`/logout` calls `logout_user()` and returns to the home page. The navbar switches between **Login / Register** and **Log Out** based on `current_user.is_authenticated`.

### 4. Reading Posts

1. `/` loads every `BlogPost` and renders a card for each with title, subtitle, author name and date.
2. Clicking a post opens `/post/<id>`, which shows the header image (`img_url`), the HTML body, and all comments.
3. A missing post id returns a clean `404` (`db.get_or_404`).

### 5. Commenting

```
POST /post/<id> (CommentForm, CKEditor field)
      │
      ▼
Logged in? ── no ──► flash "You need to login or register to comment." → /login
      │ yes
      ▼
Comment(text, comment_author=current_user, parent_post=post) saved
      │
      ▼
Post page re-renders; each comment shows:
  Gravatar image from the author's email  +  rich text  +  author name
```

### 6. Admin: Creating a Post

1. The admin (the first registered user, id 1) sees a **Create New Post** button on the home page.
2. `/new-post` is protected by the `admin_only` decorator; anyone else gets `403 Forbidden`.
3. `CreatePostForm` asks for title, subtitle, a valid image URL (WTForms `URL()` validator) and the body in CKEditor.
4. The post is saved with `author=current_user` and today's date formatted like `October 18, 2026`, then the admin is sent back to the home page.

### 7. Admin: Editing and Deleting

- **Edit:** an **Edit Post** button on the post page opens `/edit-post/<id>`. The same `CreatePostForm` is pre-filled with the current values and the page title changes to "Edit Post" (`is_edit=True`). Saving updates the row and returns to the post.
- **Delete:** a ✘ next to each post on the home page calls `/delete/<id>` (admin only), which removes the post.

### 8. Role-Based Access

| Action | Visitor | Logged-in user | Admin (id 1) |
|--------|:-------:|:--------------:|:------------:|
| Read posts and comments | ✅ | ✅ | ✅ |
| Register / log in | ✅ | | |
| Write comments | ❌ (redirected to login) | ✅ | ✅ |
| Create posts | ❌ | ❌ (403) | ✅ |
| Edit posts | ❌ | ❌ (button hidden) | ✅ |
| Delete posts | ❌ | ❌ (403) | ✅ |

---

## ✅ Advantages

- **Lightweight and easy to understand.** Flask plus a single `main.py` keeps the whole app readable in one sitting, which makes it easy to learn from and extend.
- **Real relational design.** One-to-many relationships (user → posts, user → comments, post → comments) with SQLAlchemy 2.0 typed models show proper database modelling.
- **Secure by default.** Passwords are salted and hashed, sessions are signed, and every form is CSRF-protected through Flask-WTF.
- **Clear roles.** A small `admin_only` decorator keeps authoring in the admin's hands while still letting the community comment.
- **Rich content.** CKEditor lets the author format posts with headings, links, lists and images without writing HTML.
- **Engaging comments.** Automatic Gravatar avatars make the comment section feel personal with zero setup for users.
- **Good UX.** Flash messages explain login problems, the user is logged in straight after registering, and the navbar adapts to login state and scroll direction.
- **Polished look for free.** The Clean Blog theme and Bootstrap 5 give a professional, mobile-friendly design.
- **Database-agnostic.** Switching between SQLite for development and PostgreSQL for production is only a change of `DB_URL`.
- **Zero-step schema setup.** `db.create_all()` builds the tables on first run.
- **Production-ready serving.** Gunicorn and a `Procfile` make deployment to Render or Heroku straightforward, with secrets kept in environment variables.

---

## 🗄️ Data Model

```
users                      blog_posts                    comments
─────────────────          ────────────────────────      ─────────────────────
id (PK)          ◄──┐      id (PK)            ◄──┐       id (PK)
email (unique)      ├───── author_id (FK users)  │       text
password (hash)     │      title (unique)        └────── post_id (FK blog_posts)
name                │      subtitle                      author_id (FK users) ──┐
                    │      date                                                 │
                    │      body (HTML)                                          │
                    │      img_url                                              │
                    └───────────────────────────────────────────────────────────┘

User.posts     ↔ BlogPost.author
User.comments  ↔ Comment.comment_author
BlogPost.comments ↔ Comment.parent_post
```

---

## 🗺️ Routes

| Route | Method | Access | Description |
|-------|--------|--------|-------------|
| `/` | GET | Everyone | All posts |
| `/post/<post_id>` | GET | Everyone | Read a post and its comments |
| `/post/<post_id>` | POST | Logged in | Add a comment |
| `/register` | GET, POST | Everyone | Create an account |
| `/login` | GET, POST | Everyone | Log in |
| `/logout` | GET | Logged in | Log out |
| `/new-post` | GET, POST | Admin | Create a post |
| `/edit-post/<post_id>` | GET, POST | Admin | Edit a post |
| `/delete/<post_id>` | GET | Admin | Delete a post |
| `/about` | GET | Everyone | About page |
| `/contact` | GET, POST | Everyone | Contact page |

---

## 📁 Project Structure

```
Blog_Posts/
├── main.py              # Flask app, models, auth, admin decorator, routes
├── forms.py             # CreatePostForm, RegisterForm, LoginForm, CommentForm
├── templates/
│   ├── header.html      # Shared head, navbar (login-aware)
│   ├── footer.html      # Shared footer and scripts
│   ├── index.html       # Home feed (+ admin create/delete controls)
│   ├── post.html        # Post page, comments, Gravatar, comment form
│   ├── make-post.html   # Create / edit post with CKEditor
│   ├── register.html
│   ├── login.html       # Shows flash messages
│   ├── about.html
│   └── contact.html
├── static/
│   ├── css/styles.css   # Clean Blog theme styles
│   ├── js/scripts.js    # Hide-on-scroll navbar
│   └── assets/          # Header images and favicon
├── requirements.txt
├── Procfile             # web: gunicorn main:app
└── render.yaml
```

---

## ⚙️ Local Setup

### 1. Clone the repository

```bash
git clone https://github.com/codinghub27/Blog_Posts.git
cd Blog_Posts
```

### 2. Create a virtual environment

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Set environment variables

macOS / Linux:

```bash
export FLASK_KEY="a-long-random-secret"
export DB_URL="sqlite:///posts.db"          # or postgresql://user:pass@localhost:5432/blog
```

Windows (PowerShell):

```powershell
$env:FLASK_KEY="a-long-random-secret"
$env:DB_URL="sqlite:///posts.db"
```

### 5. Run

```bash
python main.py
```

Open http://127.0.0.1:5001.

> **Tip:** register your own account first. The first user (id 1) becomes the admin who can create, edit and delete posts.

---

## 🔑 Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FLASK_KEY` | ✅ | Secret key that signs sessions and CSRF tokens |
| `DB_URL` | ✅ | SQLAlchemy database URL, e.g. `sqlite:///posts.db` or `postgresql://user:pass@host:5432/db` |

---

## ☁️ Deployment

The app runs on any platform that supports a `Procfile` (Render, Heroku, Railway):

1. Create a PostgreSQL database and copy its connection URL.
2. Create a web service from this repo.
3. **Build command:** `pip install -r requirements.txt`
4. **Start command:** `gunicorn main:app`
5. Set `FLASK_KEY` and `DB_URL` in the service's environment. If the host gives a URL starting with `postgres://`, change it to `postgresql://` for SQLAlchemy.
6. Deploy, then register your admin account first.

---

## 🚧 Known Limitations and Future Improvements

- **`/edit-post` is not protected** by `admin_only`; only the button is hidden, so the route itself should get the decorator.
- **Visitors hitting admin routes** (`/new-post`, `/delete`) while logged out cause a server error instead of a 403, because `admin_only` reads `current_user.id` on an anonymous user. Checking `current_user.is_authenticated` first fixes it.
- **Comments are rendered with `|safe`**, so HTML in a comment is trusted. Sanitizing it (for example with `bleach`) would prevent script injection.
- **Delete uses a GET link**; a POST with a confirmation would be safer.
- **The contact form is not wired up**; the email-sending code is still commented out in `main.py`.
- **`render.yaml` is not a valid Render blueprint** yet; use the build and start commands above instead.
- **Ideas:** database migrations with Flask-Migrate, post pagination, categories and tags, search, editing and deleting your own comments, and email verification.

---

## 🙌 Author

**Sravan** ([@codinghub27](https://github.com/codinghub27))

Built to practice full-stack Flask: authentication, relational databases, rich-text content and role-based access.
