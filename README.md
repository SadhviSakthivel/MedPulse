from http.server import BaseHTTPRequestHandler, HTTPServer
import hashlib
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
  <title>MedPulse</title>
  <style>
    * {{ margin: 0; padding: 0; box-sizing: border-box; font-family: 'Segoe UI', sans-serif; }}
    body {{ background-color: #f8fafc; }}
    
    .navbar {{ display: flex; justify-content: space-between; align-items: center; padding: 0.8rem 2rem; background: #fff; border-bottom: 1px solid #e2e8f0; }}
    .brand {{ font-size: 1.35rem; font-weight: 700; color: #0284c7; text-decoration: none; }}
    .nav-actions {{ display: flex; gap: 0.75rem; align-items: center; }}
    .btn {{ padding: 0.5rem 1.1rem; border-radius: 6px; font-weight: 600; text-decoration: none; border: 1px solid #0284c7; display: inline-block; cursor: pointer; }}
    .btn-login {{ color: #0284c7; background: transparent; }}
    .btn-signup {{ color: #fff; background: #0284c7; }}

    .container {{ max-width: 750px; margin: 2rem auto; padding: 2rem; background: #fff; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }}
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

    /* Tables for Pharmacist Dashboard */
    table {{ width: 100%; border-collapse: collapse; margin-top: 1rem; font-size: 0.85rem; }}
    th, td {{ padding: 0.6rem; text-align: left; border-bottom: 1px solid #e2e8f0; }}
    th {{ background: #f1f5f9; color: #334155; }}
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
                <p class="subtitle">Join MedPulse as a Patient, Donor, or Pharmacist/NGO</p>
                <form action="/signup" method="POST">
                    <div class="form-group"><label>Full Name / Hub Name</label><input type="text" name="full_name" required></div>
                    <div class="form-group"><label>Email</label><input type="email" name="email" required></div>
                    <div class="form-group"><label>Password</label><input type="password" name="password" required></div>
                    <div class="form-group">
                        <label>Account Role</label>
                        <select name="role" required>
                            <option value="user">Patient / Donor</option>
                            <option value="pharmacist">Pharmacist / NGO Hub Manager</option>
                        </select>
                    </div>
                    <button type="submit">Create Account</button>
                </form>
            """
            self.send_html(form_html)

        elif self.path == "/login":
            form_html = """
                <h2>Log In</h2>
                <p class="subtitle">Access your account or Pharmacist Hub</p>
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
                    • MedCoins are awarded upon physical inspection at partner hubs.
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
                            <option value="Partner Pharmacy A - Main Street">Partner Pharmacy A - Main Street Hub</option>
                            <option value="Partner Pharmacy B - Health Plaza">Partner Pharmacy B - Health Plaza Hub</option>
                            <option value="NGO Health Center - Central District">NGO Health Center - Central District Pickup Point</option>
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

        elif self.path == "/pharmacist-portal":
            self.render_pharmacist_portal()

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
                        <div class="alert alert-success">Welcome back, <b>{user["full_name"]}</b>!</div>
                        <div style="margin-bottom: 1.5rem;">
                            <span class="points-badge">🪙 Health Points (MedCoins): {user.get('medcoins', 0)} pts</span>
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
                <div class="alert alert-success">
                    <strong>Donation Offer Registered!</strong><br><br>
                    Please bring <b>{med_name}</b> to <b>{location}</b>.<br>
                    Once physically inspected by a pharmacist, MedCoins will be issued.
                </div>
                <a href="/" class="btn btn-login" style="margin-top:1rem;">Return to Dashboard</a>
            """
        except mysql.connector.Error as err:
            confirmation = f'<div class="alert alert-error">Database Error: {err.msg}</div>'

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
                <div class="alert alert-success">
                    <strong>Request Registered Successfully!</strong><br><br>
                    Your request for <b>{med_name}</b> is pending fulfillment at <b>{location}</b>.
                </div>
                <a href="/" class="btn btn-login" style="margin-top:1rem;">Return to Dashboard</a>
            """
        except mysql.connector.Error as err:
            confirmation = f'<div class="alert alert-error">Database Error: {err.msg}</div>'

        self.send_html(confirmation)

    
    def render_pharmacist_portal(self):
        try:
            conn = get_db()
            cursor = conn.cursor(dictionary=True)

            # Fetch ALL donations (Pending + Verified)
            cursor.execute(
                "SELECT * FROM donations ORDER BY id DESC"
            )
            donations = cursor.fetchall()

            # Fetch ALL requests (Pending + Delivered)
            cursor.execute(
                "SELECT * FROM requests ORDER BY id DESC"
            )
            requests = cursor.fetchall()

            cursor.close()
            conn.close()

            # Separate Pending vs Completed Donations
            pending_donations = [d for d in donations if d.get("status") == "PENDING"]
            completed_donations = [d for d in donations if d.get("status") != "PENDING"]

            # Separate Pending vs Completed Requests
            pending_requests = [r for r in requests if r.get("status") == "PENDING"]
            completed_requests = [r for r in requests if r.get("status") != "PENDING"]

            # Build HTML rows for Pending Donations
            pending_donations_rows = "".join([
                f"""<tr>
                    <td>{d['id']}</td>
                    <td>{d['medicine_name']}</td>
                    <td>{d['expiry_date']}</td>
                    <td>{d['location']}</td>
                    <td><span style="color:#b45309; font-weight:bold;">{d.get('status', 'PENDING')}</span></td>
                    <td>
                        <form action="/pharmacist/update-donation" method="POST" style="display:flex; gap:0.2rem;">
                            <input type="hidden" name="id" value="{d['id']}">
                            <input type="email" name="email" placeholder="Donor Email" style="width:130px;" required>
                            <input type="number" name="points" value="50" style="width:50px;" required>
                            <button type="submit" style="padding:0.2rem 0.5rem; font-size:0.75rem; width:auto;">Verify & Add Points</button>
                        </form>
                    </td>
                </tr>""" for d in pending_donations
            ]) or "<tr><td colspan='6'>No pending donations.</td></tr>"

            # Build HTML rows for Completed Donations History
            completed_donations_rows = "".join([
                f"""<tr>
                    <td>{d['id']}</td>
                    <td>{d['medicine_name']}</td>
                    <td>{d['expiry_date']}</td>
                    <td>{d['location']}</td>
                    <td><span style="color:#166534; font-weight:bold;">{d.get('status')}</span></td>
                </tr>""" for d in completed_donations
            ]) or "<tr><td colspan='5'>No completed donation history.</td></tr>"

            # Build HTML rows for Pending Requests
            pending_requests_rows = "".join([
                f"""<tr>
                    <td>{r['id']}</td>
                    <td>{r['needed_medicine']}</td>
                    <td>{r['patient_id']}</td>
                    <td>{r['location']}</td>
                    <td><span style="color:#b45309; font-weight:bold;">{r.get('status', 'PENDING')}</span></td>
                    <td>
                        <form action="/pharmacist/update-request" method="POST">
                            <input type="hidden" name="id" value="{r['id']}">
                            <button type="submit" style="padding:0.2rem 0.5rem; font-size:0.75rem; background:#166534; width:auto;">Mark Delivered</button>
                        </form>
                    </td>
                </tr>""" for r in pending_requests
            ]) or "<tr><td colspan='6'>No pending requests.</td></tr>"

            # Build HTML rows for Completed Requests History
            completed_requests_rows = "".join([
                f"""<tr>
                    <td>{r['id']}</td>
                    <td>{r['needed_medicine']}</td>
                    <td>{r['patient_id']}</td>
                    <td>{r['location']}</td>
                    <td><span style="color:#166534; font-weight:bold;">{r.get('status')}</span></td>
                </tr>""" for r in completed_requests
            ]) or "<tr><td colspan='5'>No completed request history.</td></tr>"

            portal_html = f"""
                <h2>Pharmacist / NGO Hub Portal</h2>
                <p class="subtitle">Inspect incoming donations, award MedCoins, track recipient deliveries, and view complete history.</p>
                
                <h3>1. Action Needed: Verify Incoming Donations</h3>
                <table>
                    <thead><tr><th>ID</th><th>Medicine</th><th>Expiry</th><th>Location</th><th>Status</th><th>Action</th></tr></thead>
                    <tbody>{pending_donations_rows}</tbody>
                </table>

                <h3 style="margin-top:2rem;">2. Action Needed: Pending Recipient Pickups</h3>
                <table>
                    <thead><tr><th>ID</th><th>Medicine Needed</th><th>Patient ID / Ref</th><th>Location</th><th>Status</th><th>Action</th></tr></thead>
                    <tbody>{pending_requests_rows}</tbody>
                </table>

                <hr style="margin: 2.5rem 0; border: none; border-top: 2px dashed #cbd5e1;">

                <h3 style="color: #334155;">3. History: Verified Donations (Senders Log)</h3>
                <table>
                    <thead><tr><th>ID</th><th>Medicine Name</th><th>Expiry Date</th><th>Location Hub</th><th>Status</th></tr></thead>
                    <tbody>{completed_donations_rows}</tbody>
                </table>

                <h3 style="margin-top:2rem; color: #334155;">4. History: Fulfilled Requests (Receivers Log)</h3>
                <table>
                    <thead><tr><th>ID</th><th>Medicine Requested</th><th>Patient ID / Ref</th><th>Location Hub</th><th>Status</th></tr></thead>
                    <tbody>{completed_requests_rows}</tbody>
                </table>
            """
            self.send_html(portal_html)

        except mysql.connector.Error as err:
            self.send_html(f'<div class="alert alert-error">Database Error: {err.msg}</div>')
    def process_donation_update(self, form_data):
        donation_id = form_data.get("id", [""])[0]
        donor_email = form_data.get("email", [""])[0]
        points = int(form_data.get("points", [50])[0])

        try:
            conn = get_db()
            cursor = conn.cursor()
            
            # Update donation status
            cursor.execute("UPDATE donations SET status = 'RECEIVED_AND_VERIFIED' WHERE id = %s", (donation_id,))
            
            # Award points to donor email
            cursor.execute("UPDATE users SET medcoins = medcoins + %s WHERE email = %s", (points, donor_email))
            
            conn.commit()
            cursor.close()
            conn.close()

            self.render_pharmacist_portal()
        except mysql.connector.Error as err:
            self.send_html(f'<div class="alert alert-error">Database Error: {err.msg}</div>')

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
            self.send_html(f'<div class="alert alert-error">Database Error: {err.msg}</div>')


if __name__ == "__main__":
    server = HTTPServer(("", 8000), RequestHandler)
    print("Server running at http://localhost:8000")
    server.serve_forever()
