import hashlib
from datetime import date
from http.server import BaseHTTPRequestHandler, HTTPServer
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

INDIAN_HUBS = {
    "Mumbai": [
        "Apollo Pharmacy - Shop 4, Bandra West, Linking Road, Mumbai, Maharashtra 400050 | Phone: +91 98200 12345",
        "MedPlus Pharmacy - 12 MG Road, Fort, Near CST Station, Mumbai, Maharashtra 400001 | Phone: +91 98201 67890",
        "SEWA NGO Medical Hub - Dadar East, Near Station Complex, Mumbai, Maharashtra 400014 | Phone: +91 98202 34567",
    ],
    "Delhi NCR": [
        "Wellness Forever - Block B, Connaught Place, New Delhi 110001 | Phone: +91 98100 11223",
        "Guardian Pharmacy - Sector 18, Near Metro Station, Noida, UP 201301 | Phone: +91 98101 44556",
        "Goonj Healthcare Distribution Hub - Sarita Vihar, New Delhi 110076 | Phone: +91 98102 77889",
    ],
    "Bengaluru": [
        "Apollo Pharmacy - 100 Feet Road, Indiranagar, Bengaluru, Karnataka 560038 | Phone: +91 98450 12345",
        "Frank Ross Pharmacy - 5th Block, Koramangala, Bengaluru, Karnataka 560095 | Phone: +91 98451 67890",
        "Smile Foundation Health Center - Jayanagar 4th Block, Bengaluru, Karnataka 560041 | Phone: +91 98452 34567",
    ],
    "Chennai": [
        "Apollo Pharmacy - Anna Salai, Thousand Lights, Chennai, Tamil Nadu 600006 | Phone: +91 98400 33445",
        "MedPlus Chemist - 2nd Avenue, Anna Nagar, Chennai, Tamil Nadu 600040 | Phone: +91 98401 66778",
        "Udavum Karangal NGO Hub - Extension Centre, T. Nagar, Chennai, Tamil Nadu 600017 | Phone: +91 98402 99001",
    ],
    "Hyderabad": [
        "Apollo Pharmacy - Road No. 36, Jubilee Hills, Hyderabad, Telangana 500033 | Phone: +91 98490 12345",
        "MedPlus Retail - Main Road, Banjara Hills, Hyderabad, Telangana 500028 | Phone: +91 98491 56789",
        "Being Social NGO Care Hub - Station Road, Secunderabad, Telangana 500003 | Phone: +91 98492 89012",
    ],
    "Kolkata": [
        "Frank Ross Pharmacy - Park Street, Near Metro Gate 2, Kolkata, West Bengal 700016 | Phone: +91 98300 22334",
        "Sanjeevani Chemist - Salt Lake Sector 1, Kolkata, West Bengal 700064 | Phone: +91 98301 55667",
        "Child In Need Institute (CINI) Hub - Jadavpur, Kolkata, West Bengal 700032 | Phone: +91 98302 88990",
    ],
}

HTML_PAGE = """<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>MedPulse | Surplus Medicine Redistribution India</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
  <style>
    :root {{
      --primary: #00f5c9;
      --primary-hover: #1dffd6;
      --primary-dim: rgba(0, 245, 201, 0.15);
      --accent: #ff4d6d;
      --accent-dim: rgba(255, 77, 109, 0.15);
      --violet: #8b7cff;
      --violet-deep: #6c5ce7;
      --bg: #05070d;
      --surface: #0d1420;
      --surface-2: #121a2c;
      --border: rgba(255, 255, 255, 0.09);
      --text-main: #f4f7fb;
      --text-muted: #8f9bb3;
      --radius-sm: 10px;
      --radius: 16px;
      --radius-lg: 22px;
      --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.4);
      --shadow-md: 0 12px 28px -8px rgba(0, 245, 201, 0.16), 0 6px 14px -6px rgba(0, 0, 0, 0.5);
      --shadow-lg: 0 30px 50px -12px rgba(0, 245, 201, 0.24), 0 10px 20px -8px rgba(0, 0, 0, 0.55);
      --ring: 0 0 0 4px var(--primary-dim);
    }}

    * {{ margin: 0; padding: 0; box-sizing: border-box; font-family: 'Inter', sans-serif; }}
    html {{ scroll-behavior: smooth; }}

    body {{
      background:
        radial-gradient(900px 460px at 8% -8%, rgba(0, 245, 201, 0.16) 0%, rgba(0, 245, 201, 0) 60%),
        radial-gradient(820px 420px at 108% 6%, rgba(255, 77, 109, 0.14) 0%, rgba(255, 77, 109, 0) 55%),
        radial-gradient(1000px 620px at 50% 112%, rgba(139, 124, 255, 0.14) 0%, rgba(139, 124, 255, 0) 60%),
        var(--bg);
      color: var(--text-main);
      line-height: 1.55;
      -webkit-font-smoothing: antialiased;
      min-height: 100vh;
    }}

    ::selection {{ background: var(--primary-dim); color: var(--primary); }}
    h1, h2, h3 {{ font-family: 'Space Grotesk', 'Inter', sans-serif; }}

    @keyframes fadeUp {{
      from {{ opacity: 0; transform: translateY(14px); }}
      to {{ opacity: 1; transform: translateY(0); }}
    }}
    @keyframes fadeIn {{
      from {{ opacity: 0; }}
      to {{ opacity: 1; }}
    }}
    @keyframes pulseDraw {{
      0% {{ stroke-dashoffset: 340; opacity: 0.3; }}
      55% {{ opacity: 1; }}
      100% {{ stroke-dashoffset: -340; opacity: 0.3; }}
    }}
    @keyframes pulseDot {{
      0%, 100% {{ opacity: 0.55; box-shadow: 0 0 0 4px var(--primary-dim), 0 0 10px var(--primary); }}
      50% {{ opacity: 1; box-shadow: 0 0 0 6px var(--primary-dim), 0 0 18px var(--primary); }}
    }}

    .app-shell {{ display: flex; flex-direction: column; min-height: 100vh; }}

    .navbar {{
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 1rem 3rem;
      background: rgba(8, 12, 20, 0.72);
      backdrop-filter: blur(16px) saturate(160%);
      -webkit-backdrop-filter: blur(16px) saturate(160%);
      border-bottom: 1px solid var(--border);
      position: sticky;
      top: 0;
      z-index: 100;
    }}
    .navbar::after {{
      content: "";
      position: absolute;
      left: 0; right: 0; bottom: -1px;
      height: 1px;
      background: linear-gradient(90deg, transparent, var(--primary), var(--violet), var(--accent), transparent);
      opacity: 0.65;
    }}
    .brand {{
      font-family: 'Space Grotesk', sans-serif;
      font-size: 1.5rem;
      font-weight: 700;
      color: var(--text-main);
      text-decoration: none;
      display: flex;
      align-items: center;
      gap: 0.6rem;
      letter-spacing: -0.02em;
    }}
    .brand::before {{
      content: "";
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background: var(--primary);
      display: inline-block;
      flex-shrink: 0;
      animation: pulseDot 1.8s ease-in-out infinite;
    }}
    .brand span {{ color: var(--accent); }}
    .nav-actions {{ display: flex; gap: 0.85rem; align-items: center; }}

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
    .btn-login:hover {{ background: var(--surface-2); border-color: rgba(255, 255, 255, 0.2); }}
    .btn-primary {{
      color: #04140f;
      background: linear-gradient(135deg, var(--primary) 0%, #00c2a0 100%);
      box-shadow: 0 6px 20px rgba(0, 245, 201, 0.32);
      font-weight: 700;
    }}
    .btn-primary:hover {{ filter: brightness(1.08); transform: translateY(-2px); box-shadow: 0 10px 28px rgba(0, 245, 201, 0.42); }}
    .btn-block {{ width: 100%; justify-content: center; margin-top: 0.9rem; padding: 0.85rem 1.4rem; font-size: 0.95rem; border-radius: var(--radius-sm); }}

    .main-wrapper {{ flex: 1; padding: 3.5rem 1.5rem; max-width: 1200px; margin: 0 auto; width: 100%; animation: fadeIn 0.4s ease; }}

    .card {{
      background: linear-gradient(180deg, var(--surface) 0%, var(--surface-2) 100%);
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
      height: 2px;
      background: linear-gradient(90deg, var(--primary), var(--violet), var(--accent));
    }}

    h2 {{ font-size: 1.85rem; font-weight: 700; color: var(--text-main); letter-spacing: -0.02em; margin-bottom: 0.5rem; }}
    h3 {{ font-size: 1.2rem; font-weight: 700; color: var(--text-main); margin-bottom: 1rem; display: flex; align-items: center; gap: 0.5rem; }}
    p.subtitle {{ color: var(--text-muted); font-size: 0.98rem; margin-bottom: 2rem; }}

    .form-group {{ margin-bottom: 1.35rem; text-align: left; }}
    label {{ display: block; font-size: 0.85rem; font-weight: 600; margin-bottom: 0.45rem; color: #b6c0d4; letter-spacing: 0.01em; }}
    input, select, textarea {{
      width: 100%;
      padding: 0.8rem 1rem;
      border: 1.5px solid var(--border);
      border-radius: var(--radius-sm);
      font-size: 0.95rem;
      background-color: rgba(255, 255, 255, 0.03);
      color: var(--text-main);
      transition: all 0.2s;
      font-family: inherit;
      color-scheme: dark;
    }}
    select {{ cursor: pointer; }}
    select option {{ background-color: var(--surface); color: var(--text-main); }}
    select option:checked, select option:hover {{ background-color: var(--surface-2); color: var(--text-main); }}
    input::placeholder {{ color: #5c6b85; }}
    input:focus, select:focus, textarea:focus {{
      outline: none;
      border-color: var(--primary);
      background-color: rgba(0, 245, 201, 0.05);
      box-shadow: var(--ring);
    }}
    input:disabled, select:disabled {{ opacity: 0.5; cursor: not-allowed; background: rgba(255, 255, 255, 0.02); }}

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
    .alert-error {{ background: rgba(255, 77, 109, 0.1); color: #ff8fa3; border: 1px solid rgba(255, 77, 109, 0.35); }}
    .alert-success {{ background: rgba(0, 245, 201, 0.08); color: var(--primary); border: 1px solid rgba(0, 245, 201, 0.35); }}
    .alert-info {{ background: rgba(139, 124, 255, 0.1); color: #c3baff; border: 1px solid rgba(139, 124, 255, 0.35); }}
    .alert a {{ color: inherit; font-weight: 700; text-decoration: underline; text-underline-offset: 2px; }}

    .table-container {{ overflow-x: auto; border-radius: var(--radius); border: 1px solid var(--border); margin-top: 1rem; box-shadow: var(--shadow-sm); }}
    table {{ width: 100%; border-collapse: collapse; font-size: 0.9rem; background: var(--surface); text-align: left; min-width: 650px; }}
    th {{ background: var(--surface-2); color: var(--text-muted); font-weight: 700; padding: 1rem 1.1rem; border-bottom: 1px solid var(--border); text-transform: uppercase; font-size: 0.72rem; letter-spacing: 0.05em; }}
    td {{ padding: 1rem 1.1rem; border-bottom: 1px solid var(--border); color: var(--text-main); vertical-align: middle; }}
    tr:last-child td {{ border-bottom: none; }}
    tr:hover td {{ background: rgba(255, 255, 255, 0.025); }}

    .pill {{ padding: 0.3rem 0.85rem; border-radius: 999px; font-weight: 700; font-size: 0.72rem; display: inline-flex; align-items: center; gap: 0.35rem; letter-spacing: 0.02em; }}
    .pill::before {{ content: ""; width: 6px; height: 6px; border-radius: 50%; background: currentColor; box-shadow: 0 0 6px currentColor; }}
    .pill-pending {{ background: rgba(255, 190, 0, 0.12); color: #ffc247; }}
    .pill-verified {{ background: rgba(0, 245, 201, 0.12); color: var(--primary); }}
    .action-card {{ transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1); }}
    .action-card:hover {{ transform: translateY(-4px); box-shadow: var(--shadow-md); border-color: var(--primary) !important; }}

    .hero {{ text-align: center; padding: 4rem 1rem 4rem; animation: fadeUp 0.6s cubic-bezier(0.16, 1, 0.3, 1); }}
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
      color: var(--primary);
      box-shadow: var(--shadow-sm);
      margin-bottom: 1.75rem;
    }}
    .hero h1 {{ font-size: 3.4rem; font-weight: 700; letter-spacing: -0.03em; color: var(--text-main); margin-bottom: 1.1rem; line-height: 1.12; }}
    .hero h1 .grad {{
      background: linear-gradient(135deg, var(--primary), var(--violet) 55%, var(--accent));
      -webkit-background-clip: text;
      background-clip: text;
      color: transparent;
    }}
    .hero p {{ font-size: 1.2rem; color: var(--text-muted); max-width: 600px; margin: 0 auto 2.25rem auto; }}

    .pulse-line {{ display: block; width: 100%; max-width: 620px; height: 64px; margin: 0 auto 1.75rem auto; overflow: visible; }}
    .pulse-line path {{
      fill: none;
      stroke: url(#pulseGradient);
      stroke-width: 2.5;
      stroke-linecap: round;
      stroke-linejoin: round;
      stroke-dasharray: 620;
      animation: pulseDraw 3.2s linear infinite;
      filter: drop-shadow(0 0 6px rgba(0, 245, 201, 0.55));
    }}

    /* Ambient drifting background mesh */
    body::before {{
      content: "";
      position: fixed;
      inset: 0;
      z-index: -1;
      background:
        radial-gradient(720px 380px at 22% 18%, rgba(0, 245, 201, 0.10) 0%, rgba(0, 245, 201, 0) 60%),
        radial-gradient(680px 360px at 78% 82%, rgba(255, 77, 109, 0.09) 0%, rgba(255, 77, 109, 0) 60%);
      animation: meshDrift 18s ease-in-out infinite;
      pointer-events: none;
    }}
    @keyframes meshDrift {{
      0%, 100% {{ transform: translate(0, 0) scale(1); }}
      50% {{ transform: translate(2.5%, -3%) scale(1.07); }}
    }}

    /* Scroll-triggered reveal */
    .reveal {{ opacity: 0; transform: translateY(26px); transition: opacity 0.7s cubic-bezier(0.16, 1, 0.3, 1), transform 0.7s cubic-bezier(0.16, 1, 0.3, 1); }}
    .reveal.is-visible {{ opacity: 1; transform: translateY(0); }}

    /* 3D tilt on hover (JS-driven) */
    .tilt-card {{ transition: transform 0.35s cubic-bezier(0.16, 1, 0.3, 1), box-shadow 0.35s; will-change: transform; }}
    .tilt-card:hover {{ box-shadow: var(--shadow-lg); }}

    /* Button ripple */
    .btn {{ position: relative; overflow: hidden; }}
    .ripple {{
      position: absolute;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.45);
      transform: scale(0);
      animation: rippleEffect 0.6s ease-out;
      pointer-events: none;
    }}
    @keyframes rippleEffect {{ to {{ transform: scale(3); opacity: 0; }} }}

    /* Floating hero illustrations */
    .hero {{ position: relative; }}
    .hero-deco {{
      position: absolute;
      opacity: 0.6;
      animation: floatY 6s ease-in-out infinite;
      pointer-events: none;
    }}
    @keyframes floatY {{
      0%, 100% {{ transform: translateY(0) rotate(var(--rot, 0deg)); }}
      50% {{ transform: translateY(-16px) rotate(var(--rot, 0deg)); }}
    }}

    /* How it works steps */
    .steps-grid {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 1.5rem; margin-top: 1rem; }}
    .step-card {{
      background: linear-gradient(180deg, var(--surface) 0%, var(--surface-2) 100%);
      border: 1px solid var(--border);
      border-radius: var(--radius-lg);
      padding: 2rem 1.75rem;
      text-align: left;
      transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    }}
    .step-card:hover {{ transform: translateY(-6px); border-color: var(--primary); box-shadow: var(--shadow-md); }}
    .step-icon {{
      width: 52px; height: 52px; border-radius: 14px;
      display: flex; align-items: center; justify-content: center;
      background: var(--surface-2); border: 1px solid var(--border);
      margin-bottom: 1.25rem;
    }}
    .step-num {{ font-family: 'Space Grotesk', sans-serif; font-size: 0.78rem; font-weight: 700; color: var(--primary); letter-spacing: 0.08em; margin-bottom: 0.5rem; display: block; }}
    .step-card h3 {{ margin-bottom: 0.5rem; }}
    .step-card p {{ color: var(--text-muted); font-size: 0.92rem; margin: 0; }}
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

  <script>
    (function() {{
      var revealEls = document.querySelectorAll('.reveal');
      if ('IntersectionObserver' in window && revealEls.length) {{
        var io = new IntersectionObserver(function(entries) {{
          entries.forEach(function(entry) {{
            if (entry.isIntersecting) {{
              entry.target.classList.add('is-visible');
              io.unobserve(entry.target);
            }}
          }});
        }}, {{ threshold: 0.12 }});
        revealEls.forEach(function(el) {{ io.observe(el); }});
      }} else {{
        revealEls.forEach(function(el) {{ el.classList.add('is-visible'); }});
      }}

      document.querySelectorAll('.btn').forEach(function(btn) {{
        btn.addEventListener('click', function(e) {{
          var rect = btn.getBoundingClientRect();
          var ripple = document.createElement('span');
          var size = Math.max(rect.width, rect.height);
          ripple.className = 'ripple';
          ripple.style.width = ripple.style.height = size + 'px';
          ripple.style.left = (e.clientX - rect.left - size / 2) + 'px';
          ripple.style.top = (e.clientY - rect.top - size / 2) + 'px';
          btn.appendChild(ripple);
          setTimeout(function() {{ ripple.remove(); }}, 650);
        }});
      }});

      document.querySelectorAll('.tilt-card').forEach(function(card) {{
        card.addEventListener('mousemove', function(e) {{
          var rect = card.getBoundingClientRect();
          var x = (e.clientX - rect.left) / rect.width - 0.5;
          var y = (e.clientY - rect.top) / rect.height - 0.5;
          card.style.transform = 'perspective(900px) rotateY(' + (x * 8) + 'deg) rotateX(' + (-y * 8) + 'deg) translateY(-4px)';
        }});
        card.addEventListener('mouseleave', function() {{
          card.style.transform = '';
        }});
      }});
    }})();
  </script>
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

    def render_auth_page(self, mode, selected_role=None):
        """
        Show role selection first. The user only sees the corresponding
        login/signup form after choosing Donor/Recipient or Pharmacist/NGO Hub.
        """
        is_signup = mode == "signup"
        page_title = "Join MedPulse India" if is_signup else "Welcome Back to MedPulse"
        action_text = "register" if is_signup else "log in"
        page_intro = (
            "Choose how you want to use MedPulse before creating your account."
            if is_signup
            else "Choose your portal before entering your login details."
        )
        button_text = "Sign Up" if is_signup else "Log In"

        base_path = "/signup" if is_signup else "/login"

        if selected_role not in ("user", "pharmacist"):
            form_html = f"""
            <div style="max-width: 900px; margin: 0 auto;">
                <div style="text-align: center; margin-bottom: 2.5rem;">
                    <h2>{page_title}</h2>
                    <p class="subtitle" style="margin-bottom: 0;">{page_intro}</p>
                </div>

                <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(340px, 1fr)); gap: 2rem;">
                    <a href="{base_path}?role=user" class="card tilt-card"
                       style="margin-bottom: 0; display: block; text-decoration: none; color: var(--text-main);">
                        <div style="display: flex; align-items: center; gap: 0.75rem; margin-bottom: 0.7rem;">
                            <span style="font-size: 1.9rem;">👤</span>
                            <h3 style="margin-bottom: 0;">Donor / Recipient</h3>
                        </div>
                        <p class="subtitle" style="margin-bottom: 1.5rem; font-size: 0.9rem;">
                            {("Create an account to donate unused medicine or request verified medicine."
                              if is_signup else
                              "Access your account to donate surplus medicine or request verified medicine.")}
                        </p>
                        <span class="btn btn-primary btn-block">{button_text} as Donor / Recipient</span>
                    </a>

                    <a href="{base_path}?role=pharmacist" class="card tilt-card"
                       style="margin-bottom: 0; display: block; text-decoration: none; color: var(--text-main); border-color: rgba(139, 124, 255, 0.4);">
                        <div style="display: flex; align-items: center; gap: 0.75rem; margin-bottom: 0.7rem;">
                            <span style="font-size: 1.9rem;">🏥</span>
                            <h3 style="margin-bottom: 0;">Pharmacist / NGO Hub</h3>
                        </div>
                        <p class="subtitle" style="margin-bottom: 1.5rem; font-size: 0.9rem;">
                            {("Register your certified pharmacy or NGO hub to verify and distribute donations."
                              if is_signup else
                              "Access your pharmacy or NGO hub portal to verify medicines and track distributions.")}
                        </p>
                        <span class="btn btn-primary btn-block"
                              style="background: linear-gradient(135deg, var(--violet) 0%, var(--violet-deep) 100%);">
                            {button_text} as Pharmacist / NGO
                        </span>
                    </a>
                </div>
            </div>
            """
            self.send_html(form_html)
            return

        if selected_role == "user":
            role_label = "Donor / Recipient"
            icon = "👤"
            if is_signup:
                form_html = f"""
                <div class="card" style="max-width: 620px; margin: 0 auto;">
                    <div style="text-align: center; margin-bottom: 1.8rem;">
                        <div style="font-size: 2rem; margin-bottom: 0.4rem;">{icon}</div>
                        <h2>Create Donor / Recipient Account</h2>
                        <p class="subtitle" style="margin-bottom: 0;">
                            Register to donate unused medicine or request verified medicine.
                        </p>
                    </div>

                    <form action="/signup" method="POST">
                        <input type="hidden" name="role" value="user">
                        <div class="form-group">
                            <label>Full Name</label>
                            <input type="text" name="full_name" placeholder="Aarav Sharma" required>
                        </div>
                        <div class="form-group">
                            <label>Email Address</label>
                            <input type="email" name="email" placeholder="aarav@domain.in" required>
                        </div>
                        <div class="form-group">
                            <label>Password</label>
                            <input type="password" name="password" placeholder="••••••••" required>
                        </div>
                        <button type="submit" class="btn btn-primary btn-block">
                            Sign Up as Donor / Recipient
                        </button>
                    </form>

                    <a href="/signup" class="btn btn-login btn-block" style="margin-top: 0.7rem;">
                        ← Choose Different Role
                    </a>
                </div>
                """
            else:
                form_html = f"""
                <div class="card" style="max-width: 620px; margin: 0 auto;">
                    <div style="text-align: center; margin-bottom: 1.8rem;">
                        <div style="font-size: 2rem; margin-bottom: 0.4rem;">{icon}</div>
                        <h2>Donor / Recipient Login</h2>
                        <p class="subtitle" style="margin-bottom: 0;">
                            Sign in to donate surplus medicine or request verified medicine.
                        </p>
                    </div>

                    <form action="/login" method="POST">
                        <input type="hidden" name="login_type" value="user">
                        <div class="form-group">
                            <label>Email Address</label>
                            <input type="email" name="email" placeholder="aarav@domain.in" required>
                        </div>
                        <div class="form-group">
                            <label>Password</label>
                            <input type="password" name="password" placeholder="••••••••" required>
                        </div>
                        <button type="submit" class="btn btn-primary btn-block">
                            Log In as Donor / Recipient
                        </button>
                    </form>

                    <a href="/login" class="btn btn-login btn-block" style="margin-top: 0.7rem;">
                        ← Choose Different Role
                    </a>
                </div>
                """
        else:
            role_label = "Pharmacist / NGO Hub"
            icon = "🏥"
            if is_signup:
                form_html = f"""
                <div class="card" style="max-width: 620px; margin: 0 auto; border-color: rgba(139, 124, 255, 0.4);">
                    <div style="text-align: center; margin-bottom: 1.8rem;">
                        <div style="font-size: 2rem; margin-bottom: 0.4rem;">{icon}</div>
                        <h2>Create Pharmacist / NGO Account</h2>
                        <p class="subtitle" style="margin-bottom: 0;">
                            Register your certified pharmacy or distribution hub.
                        </p>
                    </div>

                    <form action="/signup" method="POST">
                        <input type="hidden" name="role" value="pharmacist">
                        <div class="form-group">
                            <label>Hub Representative Name</label>
                            <input type="text" name="full_name" placeholder="Dr. Rajesh Kumar" required>
                        </div>
                        <div class="form-group">
                            <label>Work / Organization Email</label>
                            <input type="email" name="email" placeholder="rajesh@pharmacy.in" required>
                        </div>
                        <div class="form-group">
                            <label>Password</label>
                            <input type="password" name="password" placeholder="••••••••" required>
                        </div>
                        <button type="submit" class="btn btn-primary btn-block"
                                style="background: linear-gradient(135deg, var(--violet) 0%, var(--violet-deep) 100%);">
                            Sign Up as Pharmacist / NGO
                        </button>
                    </form>

                    <a href="/signup" class="btn btn-login btn-block" style="margin-top: 0.7rem;">
                        ← Choose Different Role
                    </a>
                </div>
                """
            else:
                form_html = f"""
                <div class="card" style="max-width: 620px; margin: 0 auto; border-color: rgba(139, 124, 255, 0.4);">
                    <div style="text-align: center; margin-bottom: 1.8rem;">
                        <div style="font-size: 2rem; margin-bottom: 0.4rem;">{icon}</div>
                        <h2>Pharmacist / NGO Login</h2>
                        <p class="subtitle" style="margin-bottom: 0;">
                            Sign in to verify incoming medications and track distributions.
                        </p>
                    </div>

                    <form action="/login" method="POST">
                        <input type="hidden" name="login_type" value="pharmacist">
                        <div class="form-group">
                            <label>Hub / Work Email</label>
                            <input type="email" name="email" placeholder="pharmacist@hub.in" required>
                        </div>
                        <div class="form-group">
                            <label>Password</label>
                            <input type="password" name="password" placeholder="••••••••" required>
                        </div>
                        <button type="submit" class="btn btn-primary btn-block"
                                style="background: linear-gradient(135deg, var(--violet) 0%, var(--violet-deep) 100%);">
                            Log In as Pharmacist / NGO
                        </button>
                    </form>

                    <a href="/login" class="btn btn-login btn-block" style="margin-top: 0.7rem;">
                        ← Choose Different Role
                    </a>
                </div>
                """

        self.send_html(form_html)

    def do_GET(self):
        parsed_url = urllib.parse.urlparse(self.path)
        request_path = parsed_url.path
        query_params = urllib.parse.parse_qs(parsed_url.query)

        if request_path == "/signup":
            selected_role = query_params.get("role", [None])[0]
            self.render_auth_page("signup", selected_role)

        elif request_path == "/login":
            selected_role = query_params.get("role", [None])[0]
            self.render_auth_page("login", selected_role)

        elif self.path.startswith("/give"):
            query_components = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            logged_in_user = query_components.get("user_name", ["Logged-in User"])[0]

            city_options = "".join([f'<option value="{city}">{city}</option>' for city in INDIAN_HUBS.keys()])
            hubs_json = json.dumps(INDIAN_HUBS)

            give_html = f"""
            <div class="card" style="max-width: 680px; margin: 0 auto;">
                <h2>Donate Surplus Medication</h2>
                <p class="subtitle">Help reduce medical waste and support patients across India.</p>

                <div class="alert alert-info">
                    <span>💡</span>
                    <div>
                        <strong>Safety Guidelines:</strong><br>
                        We accept sealed, unexpired, and unused medications in original blister strips/bottles. Verification is conducted at the selected Indian partner hub.
                    </div>
                </div>

                <form action="/give" method="POST">
                    <div class="form-group">
                        <label>Donor Name (Logged In)</label>
                        <input type="text" name="donor_name" value="{logged_in_user}" readonly style="background-color: rgba(255, 255, 255, 0.03); color: var(--text-muted); cursor: not-allowed;">
                    </div>
                    <div class="form-group">
                        <label>Donor Phone Number</label>
                        <input type="tel" name="donor_phone" placeholder="Enter phone number" autocomplete="off" required>
                    </div>
                    <div class="form-group">
                        <label>Medicine Name</label>
                        <input type="text" name="medicine_name" placeholder="e.g., Paracetamol 650mg" required>
                    </div>
                    <div class="form-group">
                        <label>Quantity / Pack Count</label>
                        <input type="number" name="quantity" min="1" value="1" placeholder="e.g., 3" required>
                    </div>

                    <div class="form-group">
                        <label>Expiration Date</label>
                        <input type="date" name="expiry_date" required>
                    </div>

                    <div class="form-group">
                        <label>Select City in India</label>
                        <select id="citySelect" onchange="updateHubs()" required>
                            <option value="">-- Choose City --</option>
                            {city_options}
                        </select>
                    </div>

                    <div class="form-group">
                        <label>Select Local NGO / Pharmacy Drop-off Point</label>
                        <select name="location" id="hubSelect" required disabled>
                            <option value="">-- Select City First --</option>
                        </select>
                    </div>

                    <button type="submit" class="btn btn-primary btn-block">Confirm Donation Offer</button>
                </form>
            </div>

            <script>
                const hubsData = {hubs_json};

                function updateHubs() {{
                    const citySelect = document.getElementById('citySelect');
                    const hubSelect = document.getElementById('hubSelect');
                    const selectedCity = citySelect.value;

                    hubSelect.innerHTML = '';

                    if (selectedCity && hubsData[selectedCity]) {{
                        hubSelect.disabled = false;
                        const defaultOpt = document.createElement('option');
                        defaultOpt.value = '';
                        defaultOpt.textContent = '-- Select Local Pharmacy / NGO Hub --';
                        hubSelect.appendChild(defaultOpt);

                        hubsData[selectedCity].forEach(hub => {{
                            const opt = document.createElement('option');
                            opt.value = hub;
                            opt.textContent = hub;
                            hubSelect.appendChild(opt);
                        }});
                    }} else {{
                        hubSelect.disabled = true;
                        const opt = document.createElement('option');
                        opt.value = '';
                        opt.textContent = '-- Select City First --';
                        hubSelect.appendChild(opt);
                    }}
                }}
            </script>
            """
            self.send_html(give_html)

        elif self.path.startswith("/receive"):
            query_components = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
            logged_in_user = query_components.get("user_name", ["Logged-in User"])[0]

            try:
                conn = get_db()
                cursor = conn.cursor(dictionary=True)
                # Pull expiry_date too so Python can enforce freshness itself,
                # rather than trusting the DB engine's date handling/timezone.
                cursor.execute(
                    "SELECT DISTINCT medicine_name, location, expiry_date FROM donations WHERE status = 'RECEIVED_AND_VERIFIED'"
                )
                available_items = cursor.fetchall()
                cursor.close()
                conn.close()
            except mysql.connector.Error:
                available_items = []

            today = date.today()
            med_location_map = {}
            med_display_names = {}  # normalized key -> first-seen display casing
            for item in available_items:
                expiry = item.get("expiry_date")
                # Skip anything already expired (or missing an expiry date) so
                # it never appears in the "available medicines" dropdown.
                if not expiry or expiry < today:
                    continue

                med_raw = item["medicine_name"].strip()
                loc = item["location"]
                med_key = med_raw.lower()  # de-dupe case/whitespace variants

                med_display_names.setdefault(med_key, med_raw)
                med_location_map.setdefault(med_key, [])
                if loc not in med_location_map[med_key]:
                    med_location_map[med_key].append(loc)

            # Re-key by the original display casing so the submitted form value
            # still matches the medicine_name string stored in the DB.
            final_location_map = {
                med_display_names[med_key]: locations
                for med_key, locations in med_location_map.items()
            }

            med_options = "".join([
                f'<option value="{med}">{med}</option>' for med in final_location_map.keys()
            ]) or '<option value="">-- No Verified Medicines Available Currently --</option>'

            js_object_str = json.dumps(final_location_map)

            receive_html = f"""
            <div class="card" style="max-width: 680px; margin: 0 auto;">
                <h2>Request Free Prescription Supply</h2>
                <p class="subtitle">Select verified surplus items from partner distribution points in India.</p>

                <form action="/receive" method="POST">
                    <div class="form-group">
                        <label>Recipient Name (Logged In)</label>
                        <input type="text" name="recipient_name" value="{logged_in_user}" readonly style="background-color: rgba(255, 255, 255, 0.03); color: var(--text-muted); cursor: not-allowed;">
                    </div>
                    <div class="form-group">
                        <label>Recipient Phone Number</label>
                        <input type="tel" name="recipient_phone" placeholder="Enter phone number" autocomplete="off" required>
                    </div>
                    <div class="form-group">
                        <label>Available Verified Medicine</label>
                        <select name="needed_medicine" id="medicineSelect" onchange="updateLocations()" required>
                            <option value="">-- Choose Medicine --</option>
                            {med_options}
                        </select>
                    </div>

                    <div class="form-group">
                        <label>Pharmacy Pickup Hub & Contact</label>
                        <select name="location" id="locationSelect" required disabled>
                            <option value="">-- Select a Medicine First --</option>
                        </select>
                    </div>

                    <div class="form-group">
                        <label>Patient ID / NGO Reference Number</label>
                        <input type="text" name="patient_id" placeholder="e.g. [Reference ID] or Ayushman Bharat ID" required>
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
                        defaultOpt.textContent = '-- Select Pick-Up Hub & Indian Contact Number --';
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
                <svg class="hero-deco" style="top: 6%; left: 6%; width: 46px; height: 46px; --rot: -18deg; animation-duration: 6.5s;" viewBox="0 0 24 24" fill="none" stroke="#00f5c9" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <rect x="2.5" y="9" width="19" height="6" rx="3" transform="rotate(-35 12 12)"/>
                    <line x1="9.5" y1="9.5" x2="14.5" y2="14.5" transform="rotate(-35 12 12)"/>
                </svg>
                <svg class="hero-deco" style="top: 12%; right: 8%; width: 40px; height: 40px; --rot: 12deg; animation-duration: 7.5s; animation-delay: 0.6s;" viewBox="0 0 24 24" fill="none" stroke="#8b7cff" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <line x1="12" y1="4" x2="12" y2="20"/>
                    <line x1="4" y1="12" x2="20" y2="12"/>
                </svg>
                <svg class="hero-deco" style="bottom: 8%; left: 10%; width: 42px; height: 42px; --rot: -8deg; animation-duration: 8s; animation-delay: 1.1s;" viewBox="0 0 24 24" fill="none" stroke="#ff4d6d" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M12 20s-6.5-4-8.5-8.2C2 8.4 3.3 5 6.8 5c1.9 0 3.3 1.1 5.2 3.4C13.9 6.1 15.3 5 17.2 5c3.5 0 4.8 3.4 3.3 6.8C18.5 16 12 20 12 20z"/>
                </svg>
                <svg class="hero-deco" style="bottom: 14%; right: 12%; width: 40px; height: 40px; --rot: 16deg; animation-duration: 6.8s; animation-delay: 0.3s;" viewBox="0 0 24 24" fill="none" stroke="#00f5c9" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M12 3l7 3.2v5.3c0 4.6-3 7.6-7 9-4-1.4-7-4.4-7-9V6.2L12 3z"/>
                    <path d="M9 12l2 2 4-4"/>
                </svg>

                <svg class="pulse-line" viewBox="0 0 620 64" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
                    <defs>
                        <linearGradient id="pulseGradient" x1="0%" y1="0%" x2="100%" y2="0%">
                            <stop offset="0%" stop-color="#00f5c9"/>
                            <stop offset="50%" stop-color="#8b7cff"/>
                            <stop offset="100%" stop-color="#ff4d6d"/>
                        </linearGradient>
                    </defs>
                    <path d="M0,32 L140,32 L162,32 L176,10 L192,54 L208,4 L224,32 L246,32 L440,32 L458,32 L472,14 L488,50 L504,10 L520,32 L542,32 L620,32"/>
                </svg>
                <div class="eyebrow">🩺 Community-powered medicine sharing in India</div>
                <h1>Bridging Surplus Medicine<br>To <span class="grad">Those Who Need It.</span></h1>
                <p>MedPulse connects patients, verified NGO hubs, and retail pharmacies across major Indian cities.</p>
                <div style="display: flex; gap: 1rem; justify-content: center; flex-wrap: wrap;">
                    <a href="/signup" class="btn btn-primary">Create an Account</a>
                    <a href="/login" class="btn btn-login">Sign In to Dashboard</a>
                </div>
            </div>

            <div style="max-width: 1000px; margin: 4rem auto 0 auto; text-align: center;">
                <h2 class="reveal" style="font-size: 1.6rem;">How MedPulse Works</h2>
                <p class="subtitle reveal" style="transition-delay: 0.05s;">Three simple steps from surplus strip to a patient who needs it.</p>
                <div class="steps-grid">
                    <div class="step-card reveal">
                        <div class="step-icon">
                            <svg width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="#00f5c9" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="10" width="16" height="7" rx="3.5"/><path d="M8 10V7a4 4 0 0 1 8 0v3"/></svg>
                        </div>
                        <span class="step-num">STEP 01</span>
                        <h3>Donate Surplus Medicine</h3>
                        <p>List sealed, unexpired medication you no longer need and pick a nearby drop-off hub.</p>
                    </div>
                    <div class="step-card reveal" style="transition-delay: 0.12s;">
                        <div class="step-icon">
                            <svg width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="#8b7cff" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3l7 3v6c0 4.5-3 7.5-7 9-4-1.5-7-4.5-7-9V6l7-3z"/><path d="M9 12l2 2 4-4"/></svg>
                        </div>
                        <span class="step-num">STEP 02</span>
                        <h3>Pharmacist Verifies It</h3>
                        <p>A certified pharmacy or NGO hub inspects the medicine and confirms it's safe to redistribute.</p>
                    </div>
                    <div class="step-card reveal" style="transition-delay: 0.24s;">
                        <div class="step-icon">
                            <svg width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="#ff4d6d" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20.5s-6.5-4-8.5-8.2C2 8.7 3.3 5.3 6.8 5.3c1.9 0 3.3 1.1 5.2 3.4 1.9-2.3 3.3-3.4 5.2-3.4 3.5 0 4.8 3.4 3.3 6.8-2 4.2-8.5 8.2-8.5 8.2z"/></svg>
                        </div>
                        <span class="step-num">STEP 03</span>
                        <h3>Patients Receive It Free</h3>
                        <p>Recipients pick up verified medicine at the hub — no cost, no wait for a prescription refill.</p>
                    </div>
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
        login_type = form_data.get("login_type", ["user"])[0]

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
                user_name_encoded = urllib.parse.quote(user["full_name"])

                if login_type == "pharmacist" and user_role not in ["pharmacist", "ngo"]:
                    message = '<div class="alert alert-error"><span>✕</span> Access Denied: Account does not have Pharmacist / NGO privileges. <a href="/login">Please use the Donor / Patient login card</a>.</div>'
                    self.send_html(message)
                    return

                if user_role in ["pharmacist", "ngo"]:
                    self.render_pharmacist_portal()
                else:
                    dashboard_html = f"""
                    <div class="stats-card reveal" style="background: linear-gradient(135deg, var(--primary) 0%, var(--violet) 100%); color: white; border-radius: var(--radius-lg); padding: 2rem 2.25rem; display: flex; justify-content: space-between; align-items: center; margin-bottom: 2.25rem; box-shadow: var(--shadow-lg);">
                        <div>
                            <h2 style="color: white; margin: 0;">Namaste, {user["full_name"]}</h2>
                            <p style="opacity: 0.9; margin: 0;">Manage your donations and medicine requests in India</p>
                        </div>
                        <div class="points-badge" style="background: rgba(255, 255, 255, 0.22); padding: 0.6rem 1.2rem; border-radius: 999px; font-weight: 700;">
                            🪙 {user.get('medcoins', 0)} MedCoins
                        </div>
                    </div>

                    <h3>Actions Overview</h3>
                    <div class="action-grid" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1.5rem;">
                      <a href="/receive?user_name={user_name_encoded}" class="action-card reveal" style="padding: 2.1rem; border: 1px solid var(--border); border-radius: var(--radius-lg); text-decoration: none; color: var(--text-main); background: var(--surface);">
                        <div class="action-title" style="font-weight: 700; font-size: 1.25rem;">📥 Receive Medicine</div>
                        <div class="action-desc" style="font-size: 0.9rem; color: var(--text-muted);">Request verified unexpired medications free of charge at an Indian partner hub near you.</div>
                      </a>
                      <a href="/give?user_name={user_name_encoded}" class="action-card reveal" style="padding: 2.1rem; border: 1px solid var(--border); border-radius: var(--radius-lg); text-decoration: none; color: var(--text-main); background: var(--surface); transition-delay: 0.1s;">
                        <div class="action-title" style="font-weight: 700; font-size: 1.25rem;">🎁 Give / Donate</div>
                        <div class="action-desc" style="font-size: 0.9rem; color: var(--text-muted);">Offer unused sealed medicine to local NGO hubs or pharmacies.</div>
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
        quantity = form_data.get("quantity", ["1"])[0]
        expiry = form_data.get("expiry_date", [""])[0]
        location = form_data.get("location", [""])[0]
        donor_name = form_data.get("donor_name", [""])[0]
        donor_phone = form_data.get("donor_phone", [""])[0]

        try:
            conn = get_db()
            cursor = conn.cursor()
            query = "INSERT INTO donations (medicine_name, quantity, expiry_date, location, donor_name, donor_phone) VALUES (%s, %s, %s, %s, %s, %s)"
            cursor.execute(query, (med_name, quantity, expiry, location, donor_name, donor_phone))
            conn.commit()
            cursor.close()
            conn.close()

            confirmation = f"""
            <div class="card" style="max-width: 680px; margin: 0 auto; text-align: left;">
                <div class="alert alert-success">
                    <span>✓</span>
                    <div>
                        <strong>Donation Registered Successfully!</strong><br><br>
                        <b>Donor Name:</b> {donor_name}<br>
                        <b>Item Details:</b> {med_name} (Qty: {quantity})<br>
                        <b>Expiry Date:</b> {expiry}<br><br>
                        <b>Selected NGO / Pharmacy Hub & Contact:</b><br>
                        {location}<br><br>
                        <i>Please hand over your medicine strip/pack at the address above for quality verification.</i>
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
        recipient_name = form_data.get("recipient_name", [""])[0]
        recipient_phone = form_data.get("recipient_phone", [""])[0]

        try:
            conn = get_db()
            cursor = conn.cursor()
            query = "INSERT INTO requests (needed_medicine, location, patient_id, recipient_name, recipient_phone) VALUES (%s, %s, %s, %s, %s)"
            cursor.execute(query, (med_name, location, patient_id, recipient_name, recipient_phone))
            conn.commit()
            cursor.close()
            conn.close()

            confirmation = f"""
            <div class="card" style="max-width: 680px; margin: 0 auto; text-align: left;">
                <div class="alert alert-success">
                    <span>✓</span>
                    <div>
                        <strong>Medicine Request Submitted!</strong><br><br>
                        <b>Recipient Name:</b> {recipient_name}<br>
                        <b>Requested Item:</b> {med_name}<br>
                        <b>Reference ID:</b> {patient_id}<br><br>
                        <b>Pickup Location & Contact Number:</b><br>
                        {location}<br><br>
                        <i>Please show your Reference ID at the store counter upon arrival.</i>
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
                    <td><b>{d.get('donor_name', 'N/A')}</b></td>
                    <td><small>{d.get('donor_phone', 'N/A')}</small></td>
                    <td><b>{d['medicine_name']}</b></td>
                    <td>{d.get('quantity', 1)}</td>
                    <td>{d['expiry_date']}</td>
                    <td>{d['location']}</td>
                    <td><span class="pill pill-pending">Pending Inspection</span></td>
                    <td>
                        <form action="/pharmacist/update-donation" method="POST" style="display:flex; gap:0.4rem; align-items:center;">
                            <input type="hidden" name="id" value="{d['id']}">
                            <input type="number" name="points" value="50" style="width:70px; padding: 0.5rem 0.6rem;" required>
                            <button type="submit" class="btn btn-primary" style="padding: 0.5rem 0.9rem; font-size: 0.8rem;">Verify</button>
                        </form>
                    </td>
                </tr>""" for d in pending_donations
            ]) or "<tr><td colspan='9' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No pending donations for verification.</td></tr>"

            completed_donations_rows = "".join([
                f"""<tr>
                    <td>#{d['id']}</td>
                    <td><b>{d.get('donor_name', 'N/A')}</b></td>
                    <td><small>{d.get('donor_phone', 'N/A')}</small></td>
                    <td>{d['medicine_name']}</td>
                    <td>{d.get('quantity', 1)}</td>
                    <td>{d['expiry_date']}</td>
                    <td>{d['location']}</td>
                    <td><span class="pill pill-verified">Verified</span></td>
                </tr>""" for d in completed_donations
            ]) or "<tr><td colspan='8' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No historical donations found.</td></tr>"

            pending_requests_rows = "".join([
                f"""<tr>
                    <td>#{r['id']}</td>
                    <td><b>{r.get('recipient_name', 'N/A')}</b><br><small>{r.get('recipient_phone', 'N/A')}</small></td>
                    <td><b>{r['needed_medicine']}</b></td>
                    <td>{r['patient_id']}</td>
                    <td>{r['location']}</td>
                    <td><span class="pill pill-pending">Pending Pickup</span></td>
                    <td>
                        <form action="/pharmacist/update-request" method="POST">
                            <input type="hidden" name="id" value="{r['id']}">
                            <button type="submit" class="btn btn-primary" style="padding:0.5rem 0.9rem; font-size:0.8rem; background: var(--accent);">Mark Distributed</button>
                        </form>
                    </td>
                </tr>""" for r in pending_requests
            ]) or "<tr><td colspan='7' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No pending pickup requests.</td></tr>"

            completed_requests_rows = "".join([
                f"""<tr>
                    <td>#{r['id']}</td>
                    <td><b>{r.get('recipient_name', 'N/A')}</b><br><small>{r.get('recipient_phone', 'N/A')}</small></td>
                    <td>{r['needed_medicine']}</td>
                    <td>{r['patient_id']}</td>
                    <td>{r['location']}</td>
                    <td><span class="pill pill-verified">Fulfilled</span></td>
                </tr>""" for r in completed_requests
            ]) or "<tr><td colspan='6' style='text-align:center; color: var(--text-muted); padding: 1.75rem;'>No completed request logs.</td></tr>"

            portal_html = f"""
            <div class="card">
                <h2>Pharmacist & NGO Hub Management (India)</h2>
                <p class="subtitle">Inspect incoming stock, grant MedCoins, and record patient distributions.</p>

                <h3>📋 1. Pending Physical Inspection (Donations)</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Donor Name</th><th>Donor Phone</th><th>Medicine</th><th>Qty</th><th>Expiry</th><th>Hub Details</th><th>Status</th><th>MedCoins</th></tr></thead>
                        <tbody>{pending_donations_rows}</tbody>
                    </table>
                </div>

                <h3 style="margin-top: 2.5rem;">📦 2. Recipient Pickup Queue</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Recipient Info</th><th>Medicine Requested</th><th>Ref ID</th><th>Hub Details</th><th>Status</th><th>Action</th></tr></thead>
                        <tbody>{pending_requests_rows}</tbody>
                    </table>
                </div>

                <h3 style="margin-top: 2.5rem;">✅ 3. History: Verified Stock Received</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Donor Name</th><th>Donor Phone</th><th>Medicine Name</th><th>Qty</th><th>Expiry Date</th><th>Hub Details</th><th>Status</th></tr></thead>
                        <tbody>{completed_donations_rows}</tbody>
                    </table>
                </div>

                <h3 style="margin-top: 2.5rem;">🚚 4. History: Dispensed Medications</h3>
                <div class="table-container">
                    <table>
                        <thead><tr><th>ID</th><th>Recipient Info</th><th>Medicine</th><th>Ref ID</th><th>Hub Details</th><th>Status</th></tr></thead>
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
        points = int(form_data.get("points", [50])[0])

        try:
            conn = get_db()
            cursor = conn.cursor(dictionary=True)
            
            cursor.execute("SELECT donor_name FROM donations WHERE id = %s", (donation_id,))
            donation = cursor.fetchone()

            if donation:
                donor_name = donation["donor_name"]
                cursor.execute("UPDATE donations SET status = 'RECEIVED_AND_VERIFIED' WHERE id = %s", (donation_id,))
                cursor.execute("UPDATE users SET medcoins = medcoins + %s WHERE full_name = %s", (points, donor_name))
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