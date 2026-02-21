# ⚔ Focus Battle

A gamified 25-minute focus timer. Lock in. Don't leave. Win or lose.

---

## Project Structure

```
focus-battle/
│
├── app.py                    ← Flask backend: all routes, DB, auth logic
├── requirements.txt          ← Python packages needed
├── focus_battle.db           ← SQLite database (auto-created on first run)
│
├── templates/                ← HTML files (Jinja2 templates)
│   ├── base.html             ← Shared layout (fonts, CSS link, body tag)
│   ├── login.html            ← Login page
│   ├── signup.html           ← Sign up page
│   ├── dashboard.html        ← Main app: timer + stats
│   └── leaderboard.html      ← Top users table
│
└── static/                   ← Files served directly to browser
    ├── css/
    │   └── main.css          ← Full design system + themes
    ├── js/
    │   ├── theme.js          ← Theme switcher (localStorage)
    │   └── timer.js          ← Timer logic + win/loss API calls
    └── images/
        ├── theme-lofi.jpg
        ├── theme-porsche.jpg
        ├── theme-f1.jpg
        ├── theme-nyc.jpg
        └── theme-liquid.jpg
```

---

## Setup & Run

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Run the app
python app.py

# 3. Open in browser
# http://localhost:5000
```

The database (`focus_battle.db`) is created automatically on first run.

---

## Data Flows Explained

### User Signs Up
1. Browser sends POST to `/signup` with username + password
2. Flask checks the username isn't already in the `users` table
3. `werkzeug.security.generate_password_hash()` hashes the password → a 60-char string that can't be reversed
4. New row inserted into `users` table
5. `session['user_id'] = new_user_id` — Flask stores this in an encrypted browser cookie
6. Browser redirects to `/dashboard`

### User Logs In
1. Browser sends POST to `/login` with credentials
2. Flask looks up the user by username with `SELECT * FROM users WHERE username = ?`
3. `check_password_hash(stored_hash, entered_password)` verifies it — returns True or False
4. If correct: `session['user_id'] = user['id']` and redirect to dashboard
5. If wrong: show error message

### User Completes Focus Session (WIN)
1. JavaScript `setInterval` counts down from 25:00 to 00:00
2. When `secondsLeft <= 0`: `clearInterval()` stops the ticker
3. `sessionLocked = true` prevents the beforeunload handler from firing
4. JS calls `fetch('/api/session/win', { method: 'POST' })`
5. Flask route increments `wins + 1` and `streak + 1` in the database
6. Server returns updated stats as JSON
7. JS updates the UI with the new numbers and shows the Victory overlay

### User Fails Focus Session (LOSS — leaves early)
1. User refreshes, closes the tab, or clicks "Abandon"
2. The `beforeunload` event fires (if they navigate away) or the abandon button's click handler fires
3. JS calls `fetch('/api/session/loss', { method: 'POST', keepalive: true })`
   - `keepalive: true` ensures the HTTP request completes even as the page unloads
4. Flask route increments `losses + 1` and sets `streak = 0`
5. If it was an explicit abandon (not a page close), show the Defeat overlay

---

## Adding More Themes

1. Add your image to `static/images/theme-yourtheme.jpg`
2. Add a CSS class to `static/css/main.css`:
   ```css
   .theme-yourtheme {
     background-image: url('/static/images/theme-yourtheme.jpg');
     --overlay: rgba(0,0,0,0.65);
     --accent: #your-color;
   }
   ```
3. Add a button in `templates/dashboard.html`:
   ```html
   <button class="theme-pill" data-theme="theme-yourtheme" title="Your Theme">🎨</button>
   ```
4. Add the class name to `VALID_THEMES` array in `static/js/theme.js`

---

## Version 2 Ideas

### Features
- **Custom session length** — let users choose 25/45/60 min
- **Session history** — log every session with timestamp, duration, theme used
- **Friends & challenges** — challenge another user to a simultaneous session
- **Badges** — achievements for streak milestones (5🔥, 10🔥, 25🔥)
- **Sound effects** — ambient audio per theme, completion chime

### Technical Improvements
- Move to **PostgreSQL** for production (SQLite is single-file, not great for multiple users writing simultaneously)
- Add **CSRF protection** to all POST routes (Flask-WTF)
- Use **environment variables** for `SECRET_KEY` (python-dotenv)
- Add **rate limiting** to login route to prevent brute force (Flask-Limiter)
- **WebSockets** for a real-time "others battling now" counter (Flask-SocketIO)
- Deploy to **Render** or **Railway** (both have free Flask tiers)
