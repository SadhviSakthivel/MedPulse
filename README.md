from http.server import BaseHTTPRequestHandler, HTTPServer
import hashlib
import urllib.parse
import mysql.connector

# Database Configuration
DB_CONFIG = {
    "host": "localhost",
    "user": "root",  
    "password": "yourpassword",  
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
    .nav-actions {{ display: flex; gap: 0.75rem; align-items: center; }}
    .btn {{ padding: 0.5rem 1.1rem; border-radius: 6px; font-weight: 600; text-decoration: none; border: 1px solid #0284c7; display: inline-block; }}
    .btn-login {{ color: #0284c7; background: transparent; }}
    .btn-signup {{ color: #fff; background: #0284c7; }}

    .container {{ max-width: 550px; margin: 2rem auto; padding: 2rem; background: #fff; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }}
    h2 {{ color: #0284c7; margin-bottom: 0.5rem; }}
    p.subtitle {{ color: #64748b; font-size: 0.9rem; margin-bottom: 1.5rem; }}

    .form-group {{ margin-bottom: 1rem; text-align: left; }}
    label {{ display: block; font-size: 0.85rem; font-weight: 600; margin-bottom: 0.3rem; color: #334155; }}
    input, select, textarea {{ width: 100%; padding: 0.55rem; border: 1px solid #cbd5e1; border-radius: 4px; font-size: 0.9rem; }}
    button {{ width: 100%; padding: 0.65rem; background: #0284c7; color: white; border: none; border-radius: 4px; font-weight: 600; cursor: pointer; font-size: 1rem; margin-top: 0.5rem; }}
    button:hover {{ background: #0369a1; }}
    
    .alert {{ padding: 0.75rem; margin-bottom: 1.5rem; border-radius: 4px; text-align: center; font-weight: 500; font-size: 0.9rem; }}
    .alert-error {{ background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }}
    .alert-success {{ background: #f0fdf4; color: #166534; border: 1px solid #bbf7d0; }}
    .alert-info {{ background: #e0f2fe; color: #0369a1; border: 1px solid #bae6fd; text-align: left; line-height: 1.4; }}

    /* Action Grid for Give/Receive */
    .action-grid {{ display: flex; gap: 1rem; margin-top: 1.5rem; }}
    .action-card {{ flex: 1; padding: 1.5rem 1rem; border: 2px solid #e2e8f0; border-radius: 8px; text-decoration: none; color: #334155; text-align: center; transition: all 0.2s; }}
    .action-card:hover {{ border-color: #0284c7; background: #f0f9ff; transform: translateY(-2px); }}
    .action-icon {{ font-size: 2rem; margin-bottom: 0.5rem; display: block; }}
    .action-title {{ font-weight: 700; font-size: 1.1rem; color: #0284c7; margin-bottom: 0.25rem; }}
    .action-desc {{ font-size: 0.8rem; color: #64748b; }}

    /* Reward Points Badge */
    .points-badge {{ background: #fef3c7; color: #b45309; border: 1px solid #fde68a; padding: 0.4rem 0.8rem; border-radius: 20px; font-weight: 700; font-size: 0.85rem; display: inline-flex; align-items: center; gap: 0.3rem; }}
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
                <p class="subtitle">Join MedPulse to donate or request verified medicines</p>
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
                <p class="subtitle">Access your account and MedCoin balance</p>
                <form action="/login" method="POST">
                    <div class="form-group"><label>Email</label><input type="email" name="email" required></div>
                    <div class="form-group"><label>Password</label><input type="password" name="password" required></div>
                    <button type="submit">Sign In</button>
                </form>
            """
            self.send_html(form_html)

        elif self.path == "/give":
            give_html = """
                <h2>Give / Donate Medicines</h2>
                <p class="subtitle">Earn Health Points (MedCoins) for verified sealed surplus donations</p>
                
                <div class="alert alert-info">
                    <strong>Legal & Operational Notice:</strong><br>
                    • Only <b>sealed, unexpired, unused</b> medicines are accepted.<br>
                    • MedCoins are awarded upon physical inspection at partner hubs.<br>
                    • Redeemed MedCoins unlock 10%–15% discount coupons for retail items at partner pharmacies.
                </div>

                <form action="/give" method="POST">
                    <div class="form-group">
                        <label>Medicine Name & Quantity</label>
                        <input type="text" name="medicine_name" placeholder="e.g. Paracetamol 500mg (2 Strips)" required>
                    </div>
                    <div class="form-group">
                        <label>Expiry Date</label>
                        <input type="date" name="expiry_date" required>
                    </div>
                    <div class="form-group">
                        <label>Drop-off Location / Delivery Hub</label>
                        <select name="location" required>
                            <option value="">-- Select Partner Pharmacy or NGO Hub --</option>
                            <option value="Partner Pharmacy A - Main Street">Partner Pharmacy A - Main Street Hub (10%-15% Coupon Sponsor)</option>
                            <option value="Partner Pharmacy B - Health Plaza">Partner Pharmacy B - Health Plaza Hub</option>
                            <option value="NGO Health Center - Central District">NGO Health Center - Central District Pickup Point</option>
                            <option value="Home Pickup - NGO Volunteer">Home Pickup via NGO Courier Service</option>
                        </select>
                    </div>
                    <button type="submit">Submit Donation Offer</button>
                </form>
            """
            self.send_html(give_html)

        elif self.path == "/receive":
            receive_html = """
                <h2>Receive Free Medicines</h2>
                <p class="subtitle">100% Free of charge for verified underprivileged patients</p>

                <div class="alert alert-info">
                    <strong>Zero-Cash Policy Notice:</strong><br>
                    • Recipient patients <b>never pay cash</b> for donated medicines.<br>
                    • Selling donated drugs without a full commercial distributor license violates laws like the <i>Drugs and Cosmetics Act</i>.<br>
                    • All medicine requests are covered via CSR / Health Grants.
                </div>

                <form action="/receive" method="POST">
                    <div class="form-group">
                        <label>Required Medicine / Supply Name</label>
                        <input type="text" name="needed_medicine" placeholder="e.g. Insulin / Blood Pressure Medication" required>
                    </div>
                    <div class="form-group">
                        <label>Preferred Pickup Location / Clinic</label>
                        <select name="location" required>
                            <option value="">-- Select Pickup Location --</option>
                            <option value="Non-Profit Clinic A - North Wing">Non-Profit Free Clinic - North Wing</option>
                            <option value="Community Health Center - South Hub">Community Health Center - South Hub</option>
                            <option value="Partner Pharmacy Drop Point">Partner Pharmacy Verification Center</option>
                        </select>
                    </div>
                    <div class="form-group">
                        <label>Patient ID / NGO Verification Reference</label>
                        <input type="text" name="patient_id" placeholder="e.g. Health Card / Gov ID Reference" required>
                    </div>
                    <button type="submit">Request Free Medicines</button>
                </form>
            """
            self.send_html(receive_html)

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
        elif self.path == "/give":
            self.process_give(form_data)
        elif self.path == "/receive":
            self.process_receive(form_data)

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
                dashboard_html = f"""
                    <div class="alert alert-success">Welcome back, <b>{user["full_name"]}</b>!</div>
                    
                    <div style="margin-bottom: 1.5rem;">
                        <span class="points-badge">🪙 Health Points (MedCoins): 50 pts</span>
                    </div>

                    <h3>Select an Action</h3>
                    <div class="action-grid">
                      <a href="/receive" class="action-card">
                        <span class="action-icon">📥</span>
                        <div class="action-title">Receive</div>
                        <div class="action-desc">100% Free medicines for verified recipient patients</div>
                      </a>
                      <a href="/give" class="action-card">
                        <span class="action-icon">🎁</span>
                        <div class="action-title">Give</div>
                        <div class="action-desc">Donate unexpired sealed items & earn 10%-15% discount coupons</div>
                      </a>
                    </div>
                """
                self.send_html(dashboard_html)
            else:
                message = '<div class="alert alert-error">Invalid email or password. <a href="/login">Try again</a>.</div>'
                self.send_html(message)

        except mysql.connector.Error as err:
            message = f'<div class="alert alert-error">Database Error: {err.msg}</div>'
            self.send_html(message)

    def process_give(self, form_data):
        med_name = form_data.get("medicine_name", [""])[0]
        location = form_data.get("location", [""])[0]

        confirmation = f"""
            <div class="alert alert-success">
                <strong>Donation Request Submitted!</strong><br><br>
                Please bring <b>{med_name}</b> to:<br>
                📍 <b>{location}</b><br><br>
                Once inspected by the pharmacy/NGO hub, <b>+50 MedCoins</b> will be credited to your account for 10%–15% retail discounts!
            </div>
            <a href="/" class="btn btn-login" style="margin-top:1rem;">Return to Dashboard</a>
        """
        self.send_html(confirmation)

    def process_receive(self, form_data):
        med_name = form_data.get("needed_medicine", [""])[0]
        location = form_data.get("location", [""])[0]

        confirmation = f"""
            <div class="alert alert-success">
                <strong>Request Registered Successfully!</strong><br><br>
                Your request for <b>{med_name}</b> is routed to:<br>
                📍 <b>{location}</b><br><br>
                <i>Reminder: This service is 100% free of charge sponsored by CSR / Health Grants.</i>
            </div>
            <a href="/" class="btn btn-login" style="margin-top:1rem;">Return to Dashboard</a>
        """
        self.send_html(confirmation)


if __name__ == "__main__":
    server = HTTPServer(("", 8000), RequestHandler)
    print("Server running at http://localhost:8000")
    server.serve_forever()
