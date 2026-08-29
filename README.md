from http.server import BaseHTTPRequestHandler, HTTPServer
import hashlib
import json
import urllib.parse
import mysql.connector

# Database Configuration
DB_CONFIG = {
    "host": "localhost",
    "user": "root",
    "password": "password1",
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
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>MedPulse | Surplus Medicine Redistribution</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800;900&family=Sora:wght@600;700;800&display=swap" rel="stylesheet">
  <style>
    :root {{
      --primary: #0284c7;
      --primary-hover: #0369a1;
      --primary-light: #e0f2fe;
      --accent: #0f766e;
      --accent-light: #ccfbf1;
      --bg: #f4f7fb;
      --surface: #ffffff;
      --border: #e2e8f0;
      --text-main: #0f172a;
      --text-muted: #64748b;
      --radius-sm: 8px;
      --radius: 14px;
      --radius-lg: 20px;
      --shadow-sm: 0 1px 3px rgba(15, 23, 42, 0.06);
      --shadow-md: 0 10px 20px -6px rgba(2, 132, 199, 0.12), 0 4px 8px -4px rgba(15, 23, 42, 0.06);
      --shadow-lg: 0 25px 40px -10px rgba(2, 132, 199, 0.18), 0 8px 16px -8px rgba(15, 23, 42, 0.08);
      --ring: 0 0 0 4px var(--primary-light);
    }}

    * {{ margin: 0; padding: 0; box-sizing: border-box; font-family: 'Plus Jakarta Sans', sans-serif; }}

    html {{ scroll-behavior: smooth; }}

    body {{
      background:
        radial-gradient(1100px 480px at 12% -10%, #e0f2fe 0%, rgba(224,242,254,0) 60%),
        radial-gradient(900px 420px at 110% 0%, #ccfbf1 0%, rgba(204,251,241,0) 55%),
        var(--bg);
      color: var(--text-main);
      line-height: 1.55;
      -webkit-font-smoothing: antialiased;
      min-height: 100vh;
    }}

    ::selection {{ background: var(--primary-light); color: var(--primary-hover); }}

    h1, h2, h3 {{ font-family: 'Sora', 'Plus Jakarta Sans', sans-serif; }}

    @keyframes fadeUp {{
      from {{ opacity: 0; transform: translateY(14px); }}
      to {{ opacity: 1; transform: translateY(0); }}
    }}
    @keyframes fadeIn {{
      from {{ opacity: 0; }}
      to {{ opacity: 1; }}
    }}
    @keyframes floatSlow {{
      0%, 100% {{ transform: translateY(0); }}
      50% {{ transform: translateY(-8px); }}
    }}

    /* Layout Structure */
    .app-shell {{ display: flex; flex-direction: column; min-height: 100vh; }}

    /* Navigation Bar */
    .navbar {{
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 1rem 3rem;
      background: rgba(255, 255, 255, 0.78);
      backdrop-filter: blur(14px) saturate(160%);
      -webkit-backdrop-filter: blur(14px) saturate(160%);
      border-bottom: 1px solid rgba(226, 232, 240, 0.8);
      position: sticky;
      top: 0;
      z-index: 100;
    }}
    .brand {{
      font-family: 'Sora', sans-serif;
      font-size: 1.5rem;
      font-weight: 800;
      color: var(--primary);
      text-decoration: none;
      display: flex;
      align-items: center;
      gap: 0.55rem;
      letter-spacing: -0.03em;
    }}
    .brand::before {{
      content: "";
      width: 30px;
      height: 30px;
      border-radius: 9px;
      background: linear-gradient(135deg, var(--primary) 0%, var(--accent) 100%);
      display: inline-block;
      box-shadow: 0 4px 10px rgba(2, 132, 199, 0.35);
      flex-shrink: 0;
    }}
    .brand span {{ color: var(--accent); }}
    .nav-actions {{ display: flex; gap: 0.85rem; align-items: center; }}

    /* Buttons */
    .btn {{
      padding: 0.7rem 1.5rem;
      border-radius: 999px;
      font-weight: 600;
      font-size: 0.9rem;
      text-decoration: none;
      border: 1px solid transparent;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      gap: 0.5rem;
      cursor: pointer;
      transition: all 0.22s cubic-bezier(0.4, 0, 0.2, 1);
      white-space: nowrap;
    }}
    .btn-login {{ color: var(--text-main); background: transparent; border-color: var(--border); }}
    .btn-login:hover {{ background: #f1f5f9; border-color: #cbd5e1; }}
    .btn-primary {{
      color: #fff;
      background: linear-gradient(135deg, var(--primary) 0%, #0369a1 100%);
      box-shadow: 0 6px 16px rgba(2, 132, 199, 0.3);
    }}
    .btn-primary:hover {{ filter: brightness(1.06); transform: translateY(-2px); box-shadow: 0 10px 22px rgba(2, 132, 199, 0.38); }}
    .btn-primary:active {{ transform: translateY(0); }}
    .btn-block {{ width: 100%; justify-content: center; margin-top: 0.9rem; padding: 0.85rem 1.4rem; font-size: 0.95rem; border-radius: var(--radius-sm); }}

    /* Main Container */
    .main-wrapper {{ flex: 1; padding: 3.5rem 1.5rem; max-width: 1100px; margin: 0 auto; width: 100%; animation: fadeIn 0.4s ease; }}

    .card {{
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: var(--radius-lg);
      box-shadow: var(--shadow-md);
      padding: 2.75rem;
      margin-bottom: 2rem;
      animation: fadeUp 0.5s cubic-bezier(0.16, 1, 0.3, 1);
      position: relative;
      overflow: hidden;
    }}
    .card::before {{
      content: "";
      position: absolute;
      top: 0; left: 0; right: 0;
      height: 4px;
      background: linear-gradient(90deg, var(--primary), var(--accent));
    }}

    /* Typography Setup */
    h2 {{ font-size: 1.85rem; font-weight: 700; color: var(--text-main); letter-spacing: -0.02em; margin-bottom: 0.5rem; }}
    h3 {{ font-size: 1.2rem; font-weight: 700; color: var(--text-main); margin-bottom: 1rem; display: flex; align-items: center; gap: 0.5rem; }}
    p.subtitle {{ color: var(--text-muted); font-size: 0.98rem; margin-bottom: 2rem; }}

    /* Inputs & Forms */
    .form-group {{ margin-bottom: 1.35rem; text-align: left; }}
    label {{ display: block; font-size: 0.85rem; font-weight: 600; margin-bottom: 0.45rem; color: #334155; letter-spacing: 0.01em; }}
    input, select, textarea {{
      width: 100%;
      padding: 0.8rem 1rem;
      border: 1.5px solid var(--border);
      border-radius: var(--radius-sm);
      font-size: 0.95rem;
      background-color: #f8fafc;
      color: var(--text-main);
      transition: all 0.2s;
      font-family: inherit;
    }}
    select {{ cursor: pointer; }}
    input::placeholder {{ color: #94a3b8; }}
    input:hover, select:hover, textarea:hover {{ border-color: #cbd5e1; }}
    input:focus, select:focus, textarea:focus {{
      outline: none;
      border-color: var(--primary);
      background-color: #fff;
      box-shadow: var(--ring);
    }}
    input:disabled, select:disabled {{ opacity: 0.55; cursor: not-allowed; background: #f1f5f9; }}

    /* System Alerts */
    .alert {{
      padding: 1.1rem 1.3rem;
      margin-bottom: 1.5rem;
      border-radius: var(--radius);
      font-weight: 500;
      font-size: 0.92rem;
      display: flex;
      gap: 0.8rem;
      align-items: flex-start;
      line-height: 1.55;
      animation: fadeUp 0.4s ease;
    }}
    .alert span {{ font-size: 1.15rem; line-height: 1; flex-shrink: 0; }}
    .alert-error {{ background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }}
    .alert-success {{ background: #f0fdf4; color: #166534; border: 1px solid #bbf7d0; }}
    .alert-info {{ background: var(--primary-light); color: var(--primary-hover); border: 1px solid #bae6fd; }}
    .alert a {{ color: inherit; font-weight: 700; text-decoration: underline; text-underline-offset: 2px; }}

    /* Cards & Stats Grid */
    .stats-card {{
      background: linear-gradient(135deg, #0284c7 0%, #0f766e 100%);
      color: white;
      border-radius: var(--radius-lg);
      padding: 2rem 2.25rem;
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 2.25rem;
      box-shadow: var(--shadow-lg);
      position: relative;
      overflow: hidden;
      flex-wrap: wrap;
      gap: 1rem;
    }}
    .stats-card::after {{
      content: "";
      position: absolute;
      width: 220px; height: 220px;
      background: rgba(255,255,255,0.08);
      border-radius: 50%;
      top: -90px; right: -60px;
      animation: floatSlow 6s ease-in-out infinite;
    }}
    .points-badge {{
      background: rgba(255, 255, 255, 0.22);
      backdrop-filter: blur(8px);
      padding: 0.6rem 1.2rem;
      border-radius: 999px;
      font-weight: 700;
      font-size: 1.05rem;
      border: 1px solid rgba(255,255,255,0.25);
      position: relative;
      z-index: 1;
    }}
    .action-grid {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1.5rem; margin-top: 1.5rem; }}
    .action-card {{
      padding: 2.1rem;
      border: 1px solid var(--border);
      border-radius: var(--radius-lg);
      text-decoration: none;
      color: var(--text-main);
      background: var(--surface);
      transition: all 0.28s cubic-bezier(0.4, 0, 0.2, 1);
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      position: relative;
    }}
    .action-card:hover {{
      border-color: var(--primary);
      transform: translateY(-6px);
      box-shadow: var(--shadow-lg);
    }}
    .action-card:hover .action-icon {{ transform: scale(1.08) rotate(-4deg); }}
    .action-icon {{
      width: 52px;
      height: 52px;
      border-radius: 14px;
      background: linear-gradient(135deg, var(--primary-light), var(--accent-light));
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 1.6rem;
      margin-bottom: 1.1rem;
      transition: transform 0.28s ease;
    }}
    .action-title {{ font-weight: 700; font-size: 1.25rem; color: var(--text-main); margin-bottom: 0.45rem; letter-spacing: -0.01em; }}
    .action-desc {{ font-size: 0.9rem; color: var(--text-muted); line-height: 1.55; }}

    /* Modern Tables */
    .table-container {{ overflow-x: auto; border-radius: var(--radius); border: 1px solid var(--border); margin-top: 1rem; box-shadow: var(--shadow-sm); }}
    table {{ width: 100%; border-collapse: collapse; font-size: 0.9rem; background: var(--surface); text-align: left; min-width: 560px; }}
    th {{ background: #f8fafc; color: var(--text-muted); font-weight: 700; padding: 1rem 1.1rem; border-bottom: 1px solid var(--border); text-transform: uppercase; font-size: 0.72rem; letter-spacing: 0.05em; }}
    td {{ padding: 1rem 1.1rem; border-bottom: 1px solid var(--border); color: var(--text-main); vertical-align: middle; }}
    tr:last-child td {{ border-bottom: none; }}
    tr {{ transition: background 0.15s ease; }}
    tr:hover td {{ background: #f8fafc; }}

    /* Status Pills */
    .pill {{ padding: 0.3rem 0.85rem; border-radius: 999px; font-weight: 700; font-size: 0.72rem; display: inline-flex; align-items: center; gap: 0.35rem; letter-spacing: 0.02em; }}
    .pill::before {{ content: ""; width: 6px; height: 6px; border-radius: 50%; background: currentColor; }}
    .pill-pending {{ background: #fef3c7; color: #b45309; }}
    .pill-verified {{ background: #dcfce7; color: #15803d; }}

    /* Hero Landing */
    .hero {{ text-align: center; padding: 5rem 1rem 4rem; animation: fadeUp 0.6s cubic-bezier(0.16, 1, 0.3, 1); }}
    .hero .eyebrow {{
      display: inline-flex;
      align-items: center;
      gap: 0.4rem;
      background: var(--surface);
      border: 1px solid var(--border);
      padding: 0.45rem 1rem;
      border-radius: 999px;
      font-size: 0.82rem;
      font-weight: 600;
      color: var(--primary-hover);
      box-shadow: var(--shadow-sm);
      margin-bottom: 1.75rem;
    }}
    .hero h1 {{ font-size: 3.25rem; font-weight: 800; letter-spacing: -0.03em; color: var(--text-main); margin-bottom: 1.1rem; line-height: 1.12; }}
    .hero h1 .grad {{
      background: linear-gradient(135deg, var(--primary), var(--accent));
      -webkit-background-clip: text;
      background-clip: text;
      color: transparent;
    }}
    .hero p {{ font-size: 1.2rem; color: var(--text-muted); max-width: 600px; margin: 0 auto 2.25rem auto; }}
    .hero .btn {{ padding: 0.85rem 1.9rem; font-size: 0.95rem; }}

    @media (max-width: 640px) {{
      .navbar {{ padding: 1rem 1.25rem; }}
      .main-wrapper {{ padding: 2rem 1rem; }}
      .card {{ padding: 1.75rem 1.5rem; }}
      .hero {{ padding: 3rem 0.5rem 2.5rem; }}
      .hero h1 {{ font-size: 2.15rem; }}
      .hero p {{ font-size: 1.05rem; }}
      .stats-card {{ flex-direction: column; align-items: flex-start; }}
    }}
  </style>
</head>
<body>

  <div class="app-shell">
    <nav class="navbar">
      <a href="/" class="brand">MedPulse<span>.</span></a>
      <div class="nav-actions">
        <a href="/login" class="btn btn-login">Log In</a>
        <a href="/signup" class="btn btn-primary">Sign Up</a>
      </div>
    </nav>

    <main class="main-wrapper">
      {content}
    </main>
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
            <div class="card" style="max-width: 480px; margin: 0 auto;">
                <h2>Create Account</h2>
                <p class="subtitle">Join MedPulse to request or safely donate medications.</p>
                <form action="/signup" method="POST">
                    <div class="form-group"><label>Full Name / Organization</label><input type="text" name="full_name" placeholder="John Doe" required></div>
                    <div class="form-group"><label>Email Address</label><input type="email" name="email" placeholder="name@domain.com" required></div>
                    <div class="form-group"><label>Password</label><input type="password" name="password" placeholder="••••••••" required></div>
                    <div class="form-group">
                        <label>Account Type</label>
                        <select name="role" required>
                            <option value="user">Patient / Individual Donor</option>
                            <option value="pharmacist">Pharmacist / NGO Hub Manager</option>
                        </select>
                    </div>
                    <button type="submit" class="btn btn-primary btn-block">Get Started</button>
                </form>
            </div>
            """
            self.send_html(form_html)

        elif self.path == "/login":
            form_html = """
            <div class="card" style="max-width: 440px; margin: 0 auto;">
                <h2>Welcome Back</h2>
                <p class="subtitle">Access your account or partner portal</p>
                <form action="/login" method="POST">
                    <div class="form-group"><label>Email Address</label><input type="email" name="email" placeholder="name@domain.com" required></div>
                    <div class="form-group"><label>Password</label><input type="password" name="password" placeholder="••••••••" required></div>
                    <button type="submit" class="btn btn-primary btn-block">Sign In</button>
                </form>
            </div>
            """
            self.send_html(form_html)

        elif self.path == "/give":
            give_html = """
            <div class="card" style="max-width: 650px; margin: 0 auto;">
                <h2>Donate Surplus Medication</h2>
                <p class="subtitle">Help reduce medical waste and support community members in need.</p>

                <div class="alert alert-info">
                    <span>💡</span>
                    <div>
                        <strong>Safety Guidelines:</strong><br>
                        We accepts only sealed, unexpired, and unused medications in original packaging. Verification happens at the drop-off location.
                    </div>
                </div>

                <form action="/give" method="POST">
                    <div class="form-group">
                        <label>Medicine Name & Quantity</label>
                        <input type="text" name="medicine_name" placeholder="e.g., Amoxicillin 500mg (2 Unopened Strips)" required>
                    </div>
                    <div class="form-group">
                        <label>Expiration Date</label>
                        <input type="date" name="expiry_date" required>
                    </div>
                    <div class="form-group">
                        <label>Drop-off Partner Hub</label>
                        <select name="location" required>
                            <option value="">-- Select Drop-off Location --</option>
                            <option value="Partner Pharmacy A - Main Street">Partner Pharmacy A - Main Street Hub</option>
                            <option value="Partner Pharmacy B - Health Plaza">Partner Pharmacy B - Health Plaza Hub</option>
                            <option value="NGO Health Center - Central District">NGO Health Center - Central District Pickup Point</option>
                        </select>
                    </div>
                    <button type="submit" class="btn btn-primary btn-block">Confirm Donation Offer</button>
                </form>
            </div>
            """
            self.send_html(give_html)

        elif self.path == "/receive":
            try:
                conn = get_db()
                cursor = conn.cursor(dictionary=True)
                cursor.execute(
                    "SELECT DISTINCT medicine_name, location FROM donations WHERE status = 'RECEIVED_AND_VERIFIED'"
                )
                available_items = cursor.fetchall()
                cursor.close()
                conn.close()
            except mysql.connector.Error:
                available_items = []

            med_location_map = {}
            for item in available_items:
                med = item["medicine_name"]
                loc = item["location"]
                med_location_map.setdefault(med, [])
                if loc not in med_location_map[med]:
                    med_location_map[med].append(loc)

            med_options = "".join([
                f'<option value="{med}">{med}</option>' for med in med_location_map.keys()
            ]) or '<option value="">-- No Verified Medicines Available Currently --</option>'

            js_object_str = json.dumps(med_location_map)

            receive_html = f"""
            <div class="card" style="max-width: 650px; margin: 0 auto;">
                <h2>Request Free Prescription Supply</h2>
                <p class="subtitle">Select verified surplus items from partner distribution points.</p>

                <form action="/receive" method="POST">
                    <div class="form-group">
                        <label>Available Verified Medicine</label>
                        <select name="needed_medicine" id="medicineSelect" onchange="updateLocations()" required>
                            <option value="">-- Choose Medicine --</option>
                            {med_options}
                        </select>
                    </div>

                    <div class="form-group">
                        <label>Pickup Location</label>
                        <select name="location" id="locationSelect" required disabled>
                            <option value="">-- Select a Medicine First --</option>
                        </select>
                    </div>

                    <div class="form-group">
                        <label>Patient ID / NGO Reference Number</label>
                        <input type="text" name="patient_id" placeholder="e.g. National Health Card ID" required>
                    </div>

                    <button type="submit" class="btn btn-primary btn-block">Submit Request</button>
                </form>
            </div>

            <script>
                const locationData = {js_object_str};

                function updateLocations() {{
                    const medSelect = document.getElementById('medicineSelect');
                    const locSelect = document.getElementById('locationSelect');
                    const selectedMed = medSelect.value;

                    locSelect.innerHTML = '';

                    if (selectedMed && locationData[selectedMed]) {{
                        locSelect.disabled = false;
                        const defaultOpt = document.createElement('option');
                        defaultOpt.value = '';
                        defaultOpt.textContent = '-- Select Pick-Up Location --';
                        locSelect.appendChild(defaultOpt);

                        locationData[selectedMed].forEach(loc => {{
                            const opt = document.createElement('option');
                            opt.value = loc;
                            opt.textContent = loc;
                            locSelect.appendChild(opt);
                        }});
                    }} else {{
                        locSelect.disabled = true;
                        const opt = document.createElement('option');
                        opt.value = '';
                        opt.textContent = '-- Select a Medicine First --';
                        locSelect.appendChild(opt);
                    }}
                }}
            </script>
            """
            self.send_html(receive_html)

        elif self.path == "/pharmacist-portal":
            self.render_pharmacist_portal()

        else:
            home_html = """
            <div class="hero">
                <div class="eyebrow">🩺 Community-powered medicine sharing</div>
                <h1>Bridging Surplus Medicine<br>To <span class="grad">Those Who Need It.</span></h1>
                <p>MedPulse connects patients, verified donor hubs, and pharmacies on a single transparent platform.</p>
                <div style="display: flex; gap: 1rem; justify-content: center; flex-wrap: wrap;">
                    <a href="/signup" class="btn btn-primary">Create an Account</a>
                    <a href="/login" class="btn btn-login">Sign In to Dashboard</a>
                </div>
            </div>
            """
            self.send_html(home_html)

    def do_POST(self):
        length = int(self.headers["Content-Length"])
        body = self.rfile.read(length).decode("utf-8")
        form_data = urllib.parse.parse_qs(body)

        if self.path == "/signup":
            self.process_signup(form_data)
        elif self.path == "/login":
            self.process_login(form_data)
        elif self.path == "/give":
            self.process_give(form_data)
        elif self.path == "/receive":
            self.process_receive(form_data)
        elif self.path == "/pharmacist/update-donation":
            self.process_donation_update(form_data)
        elif self.path == "/pharmacist/update-request":
            self.process_request_update(form_data)

    def process_signup(self, form_data):
        full_name = form_data.get("full_name", [""])[0]
        email = form_data.get("email", [""])[0]
        password = form_data.get("password", [""])[0]
        role = form_data.get("role", ["user"])[0]

        password_hash = hash_password(password)

        try:
            conn = get_db()
            cursor = conn.cursor()
            query = "INSERT INTO users (full_name, email, password_hash, role) VALUES (%s, %s, %s, %s)"
            cursor.execute(query, (full_name, email, password_hash, role))
            conn.commit()
            cursor.close()
            conn.close()

            message = '<div class="alert alert-success"><span>✓</span> Account successfully created! <a href="/login">Click here to Sign In</a></div>'
        except mysql.connector.Error as err:
            message = f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>'

        self.send_html(message)

    def process_login(self, form_data):
        email = form_data.get("email", [""])[0]
        password = form_data.get("password", [""])[0]

        password_hash = hash_password(password)

        try:
            conn = get_db()
            cursor = conn.cursor(dictionary=True)
            query = "SELECT * FROM users WHERE email = %s AND password_hash = %s"
            cursor.execute(query, (email, password_hash))
            user = cursor.fetchone()
            cursor.close()
            conn.close()

            if user:
                user_role = user.get("role") or "user"

                if user_role in ["pharmacist", "ngo"]:
                    self.render_pharmacist_portal()
                else:
                    dashboard_html = f"""
                    <div class="stats-card">
                        <div>
                            <h2 style="color: white; margin: 0;">Welcome, {user["full_name"]}</h2>
                            <p style="opacity: 0.9; margin: 0;">Manage your donations and recipient requests</p>
                        </div>
                        <div class="points-badge">
                            🪙 {user.get('medcoins', 0)} MedCoins
                        </div>
                    </div>

                    <h3>Actions Overview</h3>
                    <div class="action-grid">
                      <a href="/receive" class="action-card">
                        <div class="action-icon">📥</div>
                        <div class="action-title">Receive Medicine</div>
                        <div class="action-desc">Request verified unexpired medications free of charge at a hub near you.</div>
                      </a>
                      <a href="/give" class="action-card">
                        <div class="action-icon">🎁</div>
                        <div class="action-title">Give / Donate</div>
                        <div class="action-desc">Offer your unused sealed medicine to partner distribution points.</div>
                      </a>
                    </div>
                    """
                    self.send_html(dashboard_html)
            else:
                message = '<div class="alert alert-error"><span>✕</span> Invalid credentials. <a href="/login">Please try again</a>.</div>'
                self.send_html(message)

        except mysql.connector.Error as err:
            message = f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>'
            self.send_html(message)

    def process_give(self, form_data):
        med_name = form_data.get("medicine_name", [""])[0]
        expiry = form_data.get("expiry_date", [""])[0]
        location = form_data.get("location", [""])[0]

        try:
            conn = get_db()
            cursor = conn.cursor()
            query = "INSERT INTO donations (medicine_name, expiry_date, location) VALUES (%s, %s, %s)"
            cursor.execute(query, (med_name, expiry, location))
            conn.commit()
            cursor.close()
            conn.close()

            confirmation = f"""
            <div class="card" style="max-width: 550px; margin: 0 auto; text-align: center;">
                <div class="alert alert-success">
                    <span>✓</span>
                    <div>
                        <strong>Donation Registered!</strong><br>
                        Please bring <b>{med_name}</b> to <b>{location}</b> for physical validation.
                    </div>
                </div>
                <a href="/" class="btn btn-primary" style="margin-top: 1rem;">Back to Dashboard</a>
            </div>
            """
        except mysql.connector.Error as err:
            confirmation = f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>'

        self.send_html(confirmation)

    def process_receive(self, form_data):
        med_name = form_data.get("needed_medicine", [""])[0]
        location = form_data.get("location", [""])[0]
        patient_id = form_data.get("patient_id", [""])[0]

        try:
            conn = get_db()
            cursor = conn.cursor()
            query = "INSERT INTO requests (needed_medicine, location, patient_id) VALUES (%s, %s, %s)"
            cursor.execute(query, (med_name, location, patient_id))
            conn.commit()
            cursor.close()
            conn.close()

            confirmation = f"""
            <div class="card" style="max-width: 550px; margin: 0 auto; text-align: center;">
                <div class="alert alert-success">
                    <span>✓</span>
                    <div>
                        <strong>Request Submitted!</strong><br>
                        Your request for <b>{med_name}</b> is registered for pickup at <b>{location}</b>.
                    </div>
                </div>
                <a href="/" class="btn btn-primary" style="margin-top: 1rem;">Back to Dashboard</a>
            </div>
            """
        except mysql.connector.Error as err:
            confirmation = f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>'

        self.send_html(confirmation)

    def render_pharmacist_portal(self):
        try:
            conn = get_db()
            cursor = conn.cursor(dictionary=True)

            cursor.execute("SELECT * FROM donations ORDER BY id DESC")
            donations = cursor.fetchall()

            cursor.execute("SELECT * FROM requests ORDER BY id DESC")
            requests = cursor.fetchall()

            cursor.close()
            conn.close()

            pending_donations = [d for d in donations if d.get("status") == "PENDING"]
            completed_donations = [d for d in donations if d.get("status") != "PENDING"]

            pending_requests = [r for r in requests if r.get("status") == "PENDING"]
            completed_requests = [r for r in requests if r.get("status") != "PENDING"]

            pending_donations_rows = "".join([
                f"""<tr>
                    <td>#{d['id']}</td>
                    <td><b>{d['medicine_name']}</b></td>
                    <td>{d['expiry_date']}</td>
                    <td>{d['location']}</td>
                    <td><span class="pill pill-pending">Pending Inspection</span></td>
                    <td>
                        <form action="/pharmacist/update-donation" method="POST" style="display:flex; gap:0.4rem;">
                            <input type="hidden" name="id" value="{d['id']}">
                            <input type="email" name="email" placeholder="Donor Email" required style="padding: 0.5rem 0.7rem;">
                            <input type="number" name="points" value="50" style="width:64px; padding: 0.5rem 0.6rem;" required>
                            <button type="submit" class="btn btn-primary" style="padding: 0.5rem 0.9rem; font-size: 0.8rem;">Verify</button>
                        </form>
                    </td>
                </tr>""" for d in pending_donations
            ]) or "<tr><td colspan='6' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No pending donations for verification.</td></tr>"

            completed_donations_rows = "".join([
                f"""<tr>
                    <td>#{d['id']}</td>
                    <td>{d['medicine_name']}</td>
                    <td>{d['expiry_date']}</td>
                    <td>{d['location']}</td>
                    <td><span class="pill pill-verified">Verified</span></td>
                </tr>""" for d in completed_donations
            ]) or "<tr><td colspan='5' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No historical donations found.</td></tr>"

            pending_requests_rows = "".join([
                f"""<tr>
                    <td>#{r['id']}</td>
                    <td><b>{r['needed_medicine']}</b></td>
                    <td>{r['patient_id']}</td>
                    <td>{r['location']}</td>
                    <td><span class="pill pill-pending">Pending Pickup</span></td>
                    <td>
                        <form action="/pharmacist/update-request" method="POST">
                            <input type="hidden" name="id" value="{r['id']}">
                            <button type="submit" class="btn btn-primary" style="padding:0.5rem 0.9rem; font-size:0.8rem; background: var(--accent); box-shadow: 0 6px 16px rgba(15, 118, 110, 0.3);">Mark Distributed</button>
                        </form>
                    </td>
                </tr>""" for r in pending_requests
            ]) or "<tr><td colspan='6' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No pending pickup requests.</td></tr>"

            completed_requests_rows = "".join([
                f"""<tr>
                    <td>#{r['id']}</td>
                    <td>{r['needed_medicine']}</td>
                    <td>{r['patient_id']}</td>
                    <td>{r['location']}</td>
                    <td><span class="pill pill-verified">Fulfilled</span></td>
                </tr>""" for r in completed_requests
            ]) or "<tr><td colspan='5' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No completed request logs.</td></tr>"

            portal_html = f"""
            <div class="card">
                <h2>Pharmacist & NGO Hub Management</h2>
                <p class="subtitle">Inspect incoming stock, grant points, and record patient distributions.</p>

                <h3>📋 1. Pending Physical Inspection (Donations)</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Medicine</th><th>Expiry</th><th>Location</th><th>Status</th><th>Action</th></tr></thead>
                        <tbody>{pending_donations_rows}</tbody>
                    </table>
                </div>

                <h3 style="margin-top: 2.5rem;">📦 2. Recipient Pickup Queue</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Medicine Requested</th><th>Patient Ref</th><th>Hub Location</th><th>Status</th><th>Action</th></tr></thead>
                        <tbody>{pending_requests_rows}</tbody>
                    </table>
                </div>

                <h3 style="margin-top: 2.5rem;">✅ 3. History: Verified Stock Received</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Medicine Name</th><th>Expiry Date</th><th>Location Hub</th><th>Status</th></tr></thead>
                        <tbody>{completed_donations_rows}</tbody>
                    </table>
                </div>

                <h3 style="margin-top: 2.5rem;">🚚 4. History: Dispensed Medications</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Medicine</th><th>Patient Ref</th><th>Location Hub</th><th>Status</th></tr></thead>
                        <tbody>{completed_requests_rows}</tbody>
                    </table>
                </div>
            </div>
            """
            self.send_html(portal_html)

        except mysql.connector.Error as err:
            self.send_html(f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>')

    def process_donation_update(self, form_data):
        donation_id = form_data.get("id", [""])[0]
        donor_email = form_data.get("email", [""])[0]
        points = int(form_data.get("points", [50])[0])

        try:
            conn = get_db()
            cursor = conn.cursor()
            cursor.execute("UPDATE donations SET status = 'RECEIVED_AND_VERIFIED' WHERE id = %s", (donation_id,))
            cursor.execute("UPDATE users SET medcoins = medcoins + %s WHERE email = %s", (points, donor_email))
            conn.commit()
            cursor.close()
            conn.close()

            self.render_pharmacist_portal()
        except mysql.connector.Error as err:
            self.send_html(f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>')

    def process_request_update(self, form_data):
        request_id = form_data.get("id", [""])[0]

        try:
            conn = get_db()
            cursor = conn.cursor()
            cursor.execute("UPDATE requests SET status = 'DELIVERED' WHERE id = %s", (request_id,))
            conn.commit()
            cursor.close()
            conn.close()

            self.render_pharmacist_portal()
        except mysql.connector.Error as err:
            self.send_html(f'<div class="alert alert-error"><span>✕</span> Database Error: {err.msg}</div>')


if __name__ == "__main__":
    server = HTTPServer(("", 8000), RequestHandler)
    print("Server running at http://localhost:8000")
    server.serve_forever()
