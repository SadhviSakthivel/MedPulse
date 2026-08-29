from http.server import BaseHTTPRequestHandler, HTTPServer
import hashlib
import urllib.parse
import mysql.connector

# Database Configuration
DB_CONFIG = {
    "host": "localhost",
    "user": "root",  # Replace with your MySQL username
    "password": "password1",  # Replace with your MySQL password
    "database": "medpulse_db",
}


def get_db():
    return mysql.connector.connect(**DB_CONFIG)


def hash_password(password):
    return hashlib.sha256(password.encode("utf-8")).hexdigest()


HTML_PAGE = """<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>MedPulse</title>
  <style>
    * {{ margin: 0; padding: 0; box-sizing: border-box; font-family: 'Segoe UI', sans-serif; }}
    body {{ background-color: #f8fafc; }}
    
    .navbar {{ display: flex; justify-content: space-between; align-items: center; padding: 0.8rem 2rem; background: #fff; border-bottom: 1px solid #e2e8f0; }}
    .brand {{ font-size: 1.35rem; font-weight: 700; color: #0284c7; text-decoration: none; }}
    .nav-actions {{ display: flex; gap: 0.75rem; }}
    .btn {{ padding: 0.5rem 1.1rem; border-radius: 6px; font-weight: 600; text-decoration: none; border: 1px solid #0284c7; }}
    .btn-login {{ color: #0284c7; background: transparent; }}
    .btn-signup {{ color: #fff; background: #0284c7; }}

    .container {{ max-width: 400px; margin: 3rem auto; padding: 2rem; background: #fff; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }}
    h2 {{ color: #0284c7; margin-bottom: 1rem; }}
    .form-group {{ margin-bottom: 1rem; }}
    label {{ display: block; font-size: 0.85rem; font-weight: 600; margin-bottom: 0.3rem; }}
    input {{ width: 100%; padding: 0.5rem; border: 1px solid #cbd5e1; border-radius: 4px; }}
    button {{ width: 100%; padding: 0.6rem; background: #0284c7; color: white; border: none; border-radius: 4px; font-weight: 600; cursor: pointer; }}
    
    .alert {{ padding: 0.75rem; margin-bottom: 1rem; border-radius: 4px; text-align: center; font-weight: 500; }}
    .alert-error {{ background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }}
    .alert-success {{ background: #f0fdf4; color: #166534; border: 1px solid #bbf7d0; }}
  </style>
</head>
<body>

  <nav class="navbar">
    <a href="/" class="brand">MedPulse</a>
    <div class="nav-actions">
      <a href="/login" class="btn btn-login">Log In</a>
      <a href="/signup" class="btn btn-signup">Sign Up</a>
    </div>
  </nav>

  <div class="container">
    {content}
  </div>

</body>
</html>
"""


class RequestHandler(BaseHTTPRequestHandler):
    def send_html(self, content):
        self.send_response(200)
        self.send_header("Content-Type", "text/html")
        self.end_headers()
        response = HTML_PAGE.format(content=content)
        self.wfile.write(response.encode("utf-8"))

    def do_GET(self):
        if self.path == "/signup":
            form_html = """
                <h2>Sign Up</h2>
                <form action="/signup" method="POST">
                    <div class="form-group"><label>Full Name</label><input type="text" name="full_name" required></div>
                    <div class="form-group"><label>Email</label><input type="email" name="email" required></div>
                    <div class="form-group"><label>Password</label><input type="password" name="password" required></div>
                    <button type="submit">Create Account</button>
                </form>
            """
            self.send_html(form_html)

        elif self.path == "/login":
            form_html = """
                <h2>Log In</h2>
                <form action="/login" method="POST">
                    <div class="form-group"><label>Email</label><input type="email" name="email" required></div>
                    <div class="form-group"><label>Password</label><input type="password" name="password" required></div>
                    <button type="submit">Sign In</button>
                </form>
            """
            self.send_html(form_html)

        else:
            home_html = "<h2>Welcome to MedPulse</h2><p style='margin-top:0.5rem;'>Click <b>Log In</b> or <b>Sign Up</b> in the taskbar above to get started.</p>"
            self.send_html(home_html)

    def do_POST(self):
        length = int(self.headers["Content-Length"])
        body = self.rfile.read(length).decode("utf-8")
        form_data = urllib.parse.parse_qs(body)

        if self.path == "/signup":
            self.process_signup(form_data)
        elif self.path == "/login":
            self.process_login(form_data)

    def process_signup(self, form_data):
        full_name = form_data.get("full_name", [""])[0]
        email = form_data.get("email", [""])[0]
        password = form_data.get("password", [""])[0]

        password_hash = hash_password(password)

        try:
            conn = get_db()
            cursor = conn.cursor()
            query = "INSERT INTO users (full_name, email, password_hash) VALUES (%s, %s, %s)"
            cursor.execute(query, (full_name, email, password_hash))
            conn.commit()
            cursor.close()
            conn.close()

            message = '<div class="alert alert-success">Account created successfully! <a href="/login">Click here to Log In</a>.</div>'
        except mysql.connector.Error as err:
            message = f'<div class="alert alert-error">Database Error: {err.msg}</div>'

        self.send_html(message)

    def process_login(self, form_data):
        email = form_data.get("email", [""])[0]
        password = form_data.get("password", [""])[0]

        password_hash = hash_password(password)

        try:
            conn = get_db()
            cursor = conn.cursor(dictionary=True)
            query = (
                "SELECT * FROM users WHERE email = %s AND password_hash = %s"
            )
            cursor.execute(query, (email, password_hash))
            user = cursor.fetchone()
            cursor.close()
            conn.close()

            if user:
                message = f'<div class="alert alert-success">Welcome back, {user["full_name"]}!</div>'
            else:
                message = '<div class="alert alert-error">Invalid email or password. <a href="/login">Try again</a>.</div>'
        except mysql.connector.Error as err:
            message = f'<div class="alert alert-error">Database Error: {err.msg}</div>'

        self.send_html(message)


if __name__ == "__main__":
    server = HTTPServer(("", 8000), RequestHandler)
    print("Server running at http://localhost:8000")
    print("Refresh your browser window at http://localhost:8000")
    server.serve_forever()
