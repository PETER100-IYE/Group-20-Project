# Group-20-Project
200L Second Semester Project
import sqlite3
from flask import Flask, render_template_string, request, redirect, url_for, session
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
app.secret_key = "group20_secure_session_key_change_in_prod"
DB_NAME = "feedback.db"

# Hashed admin password (default: "admin123")
ADMIN_PASSWORD_HASH = generate_password_hash("admin123")

# Storefront Products with Attached Real Product Imagery
PRODUCTS = [
    {
        "name": "AeroPulse Wireless ANC Headphones",
        "price": "$129.99",
        "tag": "Audio",
        "icon": "fa-headphones",
        "image": "https://encrypted-tbn2.gstatic.com/licensed-image?q=tbn:ANd9GcRq0XQ4lZGwC_55lb-FClSytDStR_cxKA-dbTcYvFfqKQcGUQCMU8MDAT6mJ8aoM80JfwzEXjHjU3r6_gI",
        "desc": "40-hour battery life with adaptive active noise cancellation."
    },
    {
        "name": "NovaBook Ergonomic Laptop Stand",
        "price": "$49.50",
        "tag": "Workspace",
        "icon": "fa-laptop",
        "image": "https://encrypted-tbn1.gstatic.com/licensed-image?q=tbn:ANd9GcR0wEtHfSRVz67DX2k_aNV-dHpM6lmowVqqvln0-wx3FISY0EKQEJF_Bp2-wKAEOkODaouJZJaPVNZLU2c",
        "desc": "Precision-milled aluminum stand with adjustable cooling angles."
    },
    {
        "name": "VoltCharge 65W GaN Fast Charger",
        "price": "$39.00",
        "tag": "Power",
        "icon": "fa-bolt",
        "image": "https://encrypted-tbn2.gstatic.com/licensed-image?q=tbn:ANd9GcRUt3NHy0ih7oxPZX_iW15d9mKYCUaHp1egZZVu5fI3kubp3Ac8VO6w-yTfpnUw-q_VSNYXFFmeyIjJh_E",
        "desc": "Compact triple-port USB-C charger for laptops and phones."
    },
    {
        "name": "LuminaSmart Desk Lamp Pro",
        "price": "$64.00",
        "tag": "Lighting",
        "icon": "fa-lightbulb",
        "image": "https://encrypted-tbn0.gstatic.com/licensed-image?q=tbn:ANd9GcSrb7_lqq3Q7PkD3ZjJcpNezqlcUB1-27CaDxli7q5I2vxPBtF8thH9HAbSrjhrdmW--KInKT2bPjUrzG4",
        "desc": "Eye-care LED lighting with customizable color temperature."
    }
]

CATEGORIES = [
    {"name": "Product Quality", "icon": "fa-medal", "desc": "Build durability and performance"},
    {"name": "Customer Service", "icon": "fa-headset", "desc": "Support responsiveness and help"},
    {"name": "Delivery & Shipping", "icon": "fa-truck-fast", "desc": "Speed and packaging condition"},
    {"name": "Pricing", "icon": "fa-tags", "desc": "Value for money and affordability"}
]


def get_db_connection():
    conn = sqlite3.connect(DB_NAME)
    conn.row_factory = sqlite3.Row
    return conn


def init_db():
    with get_db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS feedback (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                customer_name TEXT NOT NULL,
                product_name TEXT NOT NULL DEFAULT 'General Store Experience',
                category TEXT NOT NULL,
                rating INTEGER NOT NULL,
                comments TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        cursor.execute("PRAGMA table_info(feedback)")
        columns = [col[1] for col in cursor.fetchall()]
        if "product_name" not in columns:
            cursor.execute(
                "ALTER TABLE feedback ADD COLUMN product_name TEXT NOT NULL DEFAULT 'General Store Experience'"
            )

        cursor.execute("SELECT COUNT(*) FROM feedback")
        if cursor.fetchone()[0] == 0:
            sample_data = [
                ("Amina Yusuf", "AeroPulse Wireless ANC Headphones", "Product Quality", 5, "Crystal clear sound and super comfortable for long study sessions."),
                ("David Okafor", "VoltCharge 65W GaN Fast Charger", "Delivery & Shipping", 4, "Arrived a day early and charges both my laptop and phone fast."),
                ("Chidinma Eze", "NovaBook Ergonomic Laptop Stand", "Pricing", 5, "Solid aluminum build quality for the price. Zero wobble."),
                ("Tunde Bakare", "LuminaSmart Desk Lamp Pro", "Customer Service", 4, "Support helped me pick the right model quickly. Great lighting.")
            ]
            cursor.executemany(
                "INSERT INTO feedback (customer_name, product_name, category, rating, comments) VALUES (?, ?, ?, ?, ?)",
                sample_data
            )
        conn.commit()


init_db()

# --- Shared Styles & High Contrast Dark/Light Themes ---
COMMON_HEAD = """
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Group 20 | Customer Experience & Feedback</title>
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root {
            --primary-gradient: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
            --card-border-radius: 20px;

            /* Light Theme Palette */
            --bg-color: #f8fafc;
            --card-bg: #ffffff;
            --text-color: #0f172a;
            --text-muted: #475569;
            --border-color: #cbd5e1;
            --input-bg: #ffffff;
            --input-text: #0f172a;
            --table-hover: #f1f5f9;
        }

        [data-theme="dark"] {
            /* High Contrast Dark Theme Palette */
            --bg-color: #0b0f19;
            --card-bg: #131b2e;
            --text-color: #f1f5f9;
            --text-muted: #94a3b8;
            --border-color: #2a364f;
            --input-bg: #0b0f19;
            --input-text: #f1f5f9;
            --table-hover: #1e293b;
        }

        body {
            font-family: 'Plus Jakarta Sans', sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            transition: background-color 0.25s ease, color 0.25s ease;
        }

        h1, h2, h3, h4, h5, h6, label, p, span, td, th {
            color: var(--text-color);
        }

        .dashboard-header {
            background: var(--primary-gradient);
            border-radius: var(--card-border-radius);
            color: #ffffff !important;
            padding: 2.5rem 2rem;
            margin-bottom: 2rem;
            box-shadow: 0 10px 25px -5px rgba(99, 102, 241, 0.3);
        }

        .group-badge {
            background: rgba(99, 102, 241, 0.15);
            border: 1px solid rgba(99, 102, 241, 0.35);
            font-weight: 700;
            letter-spacing: 0.5px;
            color: var(--text-color) !important;
        }

        .dashboard-header .group-badge {
            background: rgba(255, 255, 255, 0.2);
            border: 1px solid rgba(255, 255, 255, 0.3);
            color: #ffffff !important;
        }

        .glass-card {
            background: var(--card-bg);
            border: 1px solid var(--border-color);
            border-radius: var(--card-border-radius);
            box-shadow: 0 10px 30px -5px rgba(0, 0, 0, 0.08);
            transition: transform 0.25s ease, box-shadow 0.25s ease, border-color 0.25s ease;
        }

        .glass-card:hover {
            transform: translateY(-2px);
            box-shadow: 0 15px 30px -5px rgba(0, 0, 0, 0.12);
        }

        .product-card, .category-card {
            cursor: pointer;
            overflow: hidden;
        }

        .product-card:hover, .category-card:hover {
            border-color: #6366f1;
        }

        .product-img-wrapper {
            height: 140px;
            overflow: hidden;
            border-radius: 12px;
            background: var(--border-color);
            margin-bottom: 12px;
        }

        .product-img-wrapper img {
            width: 100%;
            height: 100%;
            object-fit: cover;
            transition: transform 0.3s ease;
        }

        .product-card:hover .product-img-wrapper img {
            transform: scale(1.05);
        }

        .metric-card-glow {
            background: var(--primary-gradient);
            color: #ffffff !important;
            border: none;
        }

        .metric-card-glow * {
            color: #ffffff !important;
        }

        .form-control, .form-select {
            background-color: var(--input-bg) !important;
            color: var(--input-text) !important;
            border-radius: 12px;
            padding: 0.75rem 1rem;
            border: 1.5px solid var(--border-color) !important;
        }

        .form-control::placeholder {
            color: var(--text-muted) !important;
            opacity: 0.75;
        }

        .form-control:focus, .form-select:focus {
            border-color: #6366f1 !important;
            box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.25);
        }

        .input-group-text {
            background-color: var(--input-bg) !important;
            color: var(--text-muted) !important;
            border-top-left-radius: 12px;
            border-bottom-left-radius: 12px;
            border: 1.5px solid var(--border-color) !important;
        }

        .star-rating-select {
            display: flex;
            flex-direction: row-reverse;
            justify-content: flex-end;
            gap: 12px;
        }

        .star-rating-select input { display: none; }

        .star-rating-select label {
            font-size: 2rem;
            color: var(--border-color);
            cursor: pointer;
            transition: color 0.15s ease, transform 0.15s ease;
        }

        .star-rating-select label:hover { transform: scale(1.15); }

        .star-rating-select input:checked ~ label,
        .star-rating-select label:hover,
        .star-rating-select label:hover ~ label {
            color: #f59e0b;
        }

        .table-custom {
            border-collapse: separate;
            border-spacing: 0 8px;
        }

        .table-custom tbody tr {
            background-color: var(--card-bg);
            box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
        }

        .table-custom tbody tr:hover {
            background-color: var(--table-hover);
        }

        .table-custom td, .table-custom th {
            padding: 1rem 1.25rem;
            border: none;
            color: var(--text-color);
        }

        .table-custom th {
            color: var(--text-muted);
            font-weight: 600;
            text-transform: uppercase;
            font-size: 0.75rem;
            letter-spacing: 0.05em;
        }

        .theme-toggle-btn {
            border-radius: 50rem;
            padding: 0.4rem 1rem;
            font-weight: 600;
            border: 1.5px solid var(--border-color);
            background: var(--card-bg);
            color: var(--text-color);
            cursor: pointer;
        }

        .text-custom-muted {
            color: var(--text-muted) !important;
        }
    </style>
</head>
"""

TOGGLE_SCRIPT = """
<script>
    function toggleTheme() {
        const currentTheme = document.documentElement.getAttribute('data-theme');
        const targetTheme = currentTheme === 'dark' ? 'light' : 'dark';
        document.documentElement.setAttribute('data-theme', targetTheme);
        localStorage.setItem('theme', targetTheme);
        updateToggleIcon(targetTheme);
        if (typeof updateChartColors === 'function') {
            updateChartColors(targetTheme);
        }
    }

    function updateToggleIcon(theme) {
        const btn = document.getElementById('themeToggleBtn');
        if (btn) {
            btn.innerHTML = theme === 'dark'
                ? '<i class="fa-solid fa-sun text-warning me-1"></i> Light Mode'
                : '<i class="fa-solid fa-moon text-primary me-1"></i> Dark Mode';
        }
    }

    (function() {
        const savedTheme = localStorage.getItem('theme') || 'dark';
        document.documentElement.setAttribute('data-theme', savedTheme);
        window.addEventListener('DOMContentLoaded', () => updateToggleIcon(savedTheme));
    })();
</script>
"""

# --- 1. Public Customer Storefront & Feedback Page ---
PUBLIC_TEMPLATE = """
<!DOCTYPE html>
<html lang="en" data-theme="dark">
""" + COMMON_HEAD + """
<body class="py-4">
    <div class="container" style="max-width: 960px;">

        <div class="d-flex justify-content-between align-items-center mb-4">
            <span class="badge group-badge text-uppercase px-3 py-2 rounded-pill shadow-sm">
                <i class="fa-solid fa-store me-1 text-primary"></i> Group 20 Tech Store
            </span>
            <div class="d-flex gap-2">
                <button id="themeToggleBtn" onclick="toggleTheme()" class="theme-toggle-btn shadow-sm">
                    <i class="fa-solid fa-moon me-1"></i> Dark Mode
                </button>
                <a href="/admin/login" class="btn btn-outline-primary btn-sm fw-semibold rounded-pill px-3 shadow-sm d-flex align-items-center">
                    <i class="fa-solid fa-shield-halved me-1"></i> Admin Portal
                </a>
            </div>
        </div>

        {% if success %}
        <div class="alert alert-success alert-dismissible fade show shadow-sm border-0 mb-4 rounded-4 p-3" role="alert" style="background: #ecfdf5; color: #065f46;">
            <div class="d-flex align-items-center">
                <i class="fa-solid fa-circle-check fs-4 me-3 text-success"></i>
                <div>
                    <strong>Thank you!</strong> Your feedback has been recorded successfully.
                </div>
            </div>
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
        {% endif %}

        {% if error %}
        <div class="alert alert-danger alert-dismissible fade show shadow-sm border-0 mb-4 rounded-4 p-3" role="alert">
            <i class="fa-solid fa-triangle-exclamation me-2"></i> {{ error }}
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
        {% endif %}

        <div class="text-center mb-4 py-2">
            <span class="badge bg-primary-subtle text-primary px-3 py-2 rounded-pill fw-bold text-uppercase mb-2">Customer Experience</span>
            <h1 class="fw-extrabold display-5 mb-2">We Value Your Feedback</h1>
            <p class="text-custom-muted fs-6 mx-auto" style="max-width: 560px;">
                Select a product and a feedback category below to share your experience with our team.
            </p>
        </div>

        <!-- Section 1: Featured Products Showcase -->
        <h5 class="fw-bold mb-3"><i class="fa-solid fa-box text-primary me-2"></i>1. Select a Product</h5>
        <div class="row g-3 mb-4">
            {% for product in products %}
            <div class="col-sm-6 col-lg-3">
                <div class="glass-card product-card p-3 h-100 d-flex flex-column justify-content-between" onclick="selectProduct('{{ product.name }}')">
                    <div>
                        <div class="product-img-wrapper">
                            <img src="{{ product.image }}" alt="{{ product.name }}">
                        </div>
                        <div class="d-flex justify-content-between align-items-center mb-2">
                            <span class="badge bg-secondary-subtle text-custom-muted rounded-pill small">{{ product.tag }}</span>
                            <span class="fw-bold text-primary small">{{ product.price }}</span>
                        </div>
                        <h6 class="fw-bold mb-1" style="font-size: 0.95rem;">{{ product.name }}</h6>
                    </div>
                    <div class="pt-2 border-top text-end" style="border-color: var(--border-color) !important;">
                        <span class="small fw-bold text-primary">Choose <i class="fa-solid fa-check ms-1"></i></span>
                    </div>
                </div>
            </div>
            {% endfor %}
        </div>

        <!-- Section 2: Feedback Categories Cards -->
        <h5 class="fw-bold mb-3"><i class="fa-solid fa-layer-group text-success me-2"></i>2. Or Focus by Category</h5>
        <div class="row g-3 mb-5">
            {% for cat in categories %}
            <div class="col-sm-6 col-lg-3">
                <div class="glass-card category-card p-3 h-100 text-center" onclick="selectCategory('{{ cat.name }}')">
                    <div class="bg-success-subtle text-success rounded-circle d-inline-flex align-items-center justify-content-center mb-2" style="width: 42px; height: 42px;">
                        <i class="fa-solid {{ cat.icon }} fs-6"></i>
                    </div>
                    <h6 class="fw-bold mb-1" style="font-size: 0.95rem;">{{ cat.name }}</h6>
                    <p class="text-custom-muted small mb-0" style="font-size: 0.75rem;">{{ cat.desc }}</p>
                </div>
            </div>
            {% endfor %}
        </div>

        <!-- Section 3: Feedback Submission Form -->
        <div id="feedbackFormCard" class="glass-card p-4 p-md-5 mb-5">
            <h5 class="fw-bold mb-4"><i class="fa-solid fa-pen-to-square text-primary me-2"></i>3. Submit Your Review</h5>
            <form action="/submit" method="POST">
                <div class="row g-3">
                    <div class="col-md-4 mb-2">
                        <label class="form-label fw-bold text-custom-muted small">YOUR NAME</label>
                        <div class="input-group">
                            <span class="input-group-text"><i class="fa-regular fa-user"></i></span>
                            <input type="text" name="customer_name" maxlength="80" class="form-control" placeholder="e.g. Alex Morgan" required>
                        </div>
                    </div>

                    <div class="col-md-4 mb-2">
                        <label class="form-label fw-bold text-custom-muted small">PRODUCT PURCHASED</label>
                        <select id="productSelect" name="product_name" class="form-select" required>
                            {% for product in products %}
                            <option value="{{ product.name }}">{{ product.name }}</option>
                            {% endfor %}
                            <option value="General Store Experience">General Store Experience</option>
                        </select>
                    </div>

                    <div class="col-md-4 mb-2">
                        <label class="form-label fw-bold text-custom-muted small">FEEDBACK CATEGORY</label>
                        <select id="categorySelect" name="category" class="form-select" required>
                            {% for cat in categories %}
                            <option value="{{ cat.name }}">{{ cat.name }}</option>
                            {% endfor %}
                            <option value="General Support">General Support</option>
                        </select>
                    </div>
                </div>

                <div class="my-4 p-3 rounded-4 text-center" style="background-color: var(--input-bg); border: 1px solid var(--border-color);">
                    <label class="form-label fw-bold text-custom-muted small d-block mb-1">HOW WOULD YOU RATE THIS EXPERIENCE?</label>
                    <div class="star-rating-select d-inline-flex my-1">
                        <input type="radio" id="star5" name="rating" value="5" required/><label for="star5" title="5 Stars"><i class="fa-solid fa-star"></i></label>
                        <input type="radio" id="star4" name="rating" value="4"/><label for="star4" title="4 Stars"><i class="fa-solid fa-star"></i></label>
                        <input type="radio" id="star3" name="rating" value="3"/><label for="star3" title="3 Stars"><i class="fa-solid fa-star"></i></label>
                        <input type="radio" id="star2" name="rating" value="2"/><label for="star2" title="2 Stars"><i class="fa-solid fa-star"></i></label>
                        <input type="radio" id="star1" name="rating" value="1"/><label for="star1" title="1 Star"><i class="fa-solid fa-star"></i></label>
                    </div>
                </div>

                <div class="mb-4">
                    <label class="form-label fw-bold text-custom-muted small">YOUR COMMENTS</label>
                    <textarea name="comments" maxlength="500" class="form-control" rows="4" placeholder="Share specific details about build quality, delivery speed, or support..." required></textarea>
                </div>

                <button type="submit" class="btn btn-primary w-100 py-3 fw-bold rounded-3 shadow" style="background: var(--primary-gradient); border: none; font-size: 1.05rem;">
                    <i class="fa-solid fa-paper-plane me-2"></i>Submit Feedback
                </button>
            </form>
        </div>

        <footer class="text-center text-custom-muted small">
            <p>Customer Feedback System • Powered by <strong>Group 20</strong></p>
        </footer>

    </div>

    <script>
        function selectProduct(productName) {
            const select = document.getElementById('productSelect');
            select.value = productName;
            const formCard = document.getElementById('feedbackFormCard');
            formCard.scrollIntoView({ behavior: 'smooth', block: 'center' });
            select.focus();
        }

        function selectCategory(categoryName) {
            const select = document.getElementById('categorySelect');
            select.value = categoryName;
            const formCard = document.getElementById('feedbackFormCard');
            formCard.scrollIntoView({ behavior: 'smooth', block: 'center' });
            select.focus();
        }
    </script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    """ + TOGGLE_SCRIPT + """
</body>
</html>
"""

# --- 2. Admin Login Page ---
LOGIN_TEMPLATE = """
<!DOCTYPE html>
<html lang="en" data-theme="dark">
""" + COMMON_HEAD + """
<body class="py-5 d-flex align-items-center justify-content-center min-vh-100">
    <div class="container" style="max-width: 420px;">
        <div class="d-flex justify-content-end mb-3">
            <button id="themeToggleBtn" onclick="toggleTheme()" class="theme-toggle-btn shadow-sm">
                <i class="fa-solid fa-moon me-1"></i> Dark Mode
            </button>
        </div>
        <div class="glass-card p-4 p-md-5">
            <div class="text-center mb-4">
                <div class="bg-primary-subtle text-primary rounded-circle d-inline-flex align-items-center justify-content-center mb-3" style="width: 60px; height: 60px;">
                    <i class="fa-solid fa-lock fs-4"></i>
                </div>
                <h3 class="fw-extrabold mb-1">Admin Portal</h3>
                <p class="text-custom-muted small">Enter password to view Group 20 dashboard analytics.</p>
            </div>

            {% if error %}
            <div class="alert alert-danger py-2 small text-center" role="alert">
                {{ error }}
            </div>
            {% endif %}

            <form action="/admin/login" method="POST">
                <div class="mb-3">
                    <label class="form-label fw-semibold text-custom-muted">Password</label>
                    <input type="password" name="password" class="form-control" placeholder="Enter admin password" required autofocus>
                </div>
                <button type="submit" class="btn btn-primary w-100 py-2.5 fw-bold shadow-sm mb-3" style="background: var(--primary-gradient); border: none;">
                    Login to Dashboard
                </button>
                <a href="/" class="btn btn-outline-secondary w-100 small fw-semibold">
                    <i class="fa-solid fa-arrow-left me-1"></i> Back to Storefront
                </a>
            </form>
        </div>
    </div>
    """ + TOGGLE_SCRIPT + """
</body>
</html>
"""

# --- 3. Protected Admin Dashboard Page ---
ADMIN_DASHBOARD_TEMPLATE = """
<!DOCTYPE html>
<html lang="en" data-theme="dark">
""" + COMMON_HEAD + """
<body class="py-4">
    <div class="container">

        <header class="dashboard-header text-center text-md-start d-md-flex align-items-center justify-content-between">
            <div>
                <span class="badge group-badge text-uppercase px-3 py-2 rounded-pill mb-2">
                    <i class="fa-solid fa-shield-halved me-1"></i> Admin Portal • Group 20
                </span>
                <h1 class="fw-extrabold mb-1">Analytics & Sectional Breakdown</h1>
                <p class="mb-0 opacity-75">Multi-dimensional performance analysis across products and categories.</p>
            </div>
            <div class="mt-3 mt-md-0 d-flex align-items-center gap-2 justify-content-center">
                <button id="themeToggleBtn" onclick="toggleTheme()" class="theme-toggle-btn shadow-sm me-2">
                    <i class="fa-solid fa-moon me-1"></i> Dark Mode
                </button>
                <a href="/" class="btn btn-outline-light btn-sm fw-bold px-3 py-2 rounded-pill">
                    <i class="fa-solid fa-store me-1"></i> Storefront
                </a>
                <a href="/admin/logout" class="btn btn-light btn-sm fw-bold px-3 py-2 rounded-pill shadow-sm">
                    <i class="fa-solid fa-right-from-bracket me-1"></i> Logout
                </a>
            </div>
        </header>

        <!-- Top KPI Cards -->
        <div class="row g-3 mb-4">
            <div class="col-md-4">
                <div class="glass-card p-4 metric-card-glow h-100 d-flex flex-column justify-content-between">
                    <div>
                        <span class="text-uppercase fw-bold small opacity-75">Overall Satisfaction</span>
                        <h1 class="fw-extrabold display-5 my-1">{{ "%.2f"|format(avg_rating) }} <span class="fs-6 opacity-75">/ 5.0</span></h1>
                    </div>
                    <div class="d-flex align-items-center justify-content-between opacity-75 small mt-2">
                        <span>Across {{ total_count }} reviews</span>
                        <i class="fa-solid fa-star text-warning fs-5"></i>
                    </div>
                </div>
            </div>

            <div class="col-md-4">
                <div class="glass-card p-4 h-100 d-flex flex-column justify-content-between">
                    <div>
                        <span class="text-custom-muted text-uppercase fw-bold small">Top Rated Product</span>
                        <h4 class="fw-extrabold my-2">{{ top_product }}</h4>
                    </div>
                    <div class="d-flex align-items-center justify-content-between text-custom-muted small mt-2">
                        <span>Highest average customer score</span>
                        <i class="fa-solid fa-trophy text-warning fs-5"></i>
                    </div>
                </div>
            </div>

            <div class="col-md-4">
                <div class="glass-card p-4 h-100 d-flex flex-column justify-content-between">
                    <div>
                        <span class="text-custom-muted text-uppercase fw-bold small">Total Submissions</span>
                        <h1 class="fw-extrabold display-5 my-1">{{ total_count }}</h1>
                    </div>
                    <div class="d-flex align-items-center justify-content-between text-custom-muted small mt-2">
                        <span>Stored securely in SQLite</span>
                        <i class="fa-solid fa-database text-primary fs-5"></i>
                    </div>
                </div>
            </div>
        </div>

        <!-- Charts Row -->
        <div class="row g-3 mb-4">
            <div class="col-md-6">
                <div class="glass-card p-4">
                    <h6 class="fw-bold mb-3"><i class="fa-solid fa-chart-column text-primary me-2"></i>Rating by Category Section</h6>
                    <div style="position: relative; height: 210px;">
                        <canvas id="feedbackChart"></canvas>
                    </div>
                </div>
            </div>

            <div class="col-md-6">
                <div class="glass-card p-4">
                    <h6 class="fw-bold mb-3"><i class="fa-solid fa-box-open text-success me-2"></i>Rating by Product Section</h6>
                    <div style="position: relative; height: 210px;">
                        <canvas id="productChart"></canvas>
                    </div>
                </div>
            </div>
        </div>

        <!-- Feedback Logs Table with Dual Filters -->
        <div class="glass-card p-4">
            <div class="d-flex flex-column flex-md-row justify-content-between align-items-md-center gap-3 mb-4">
                <div>
                    <h4 class="fw-bold mb-1"><i class="fa-solid fa-list-check text-primary me-2"></i>Feedback Logs</h4>
                    <p class="text-custom-muted small mb-0">Filter customer reviews by product or category section</p>
                </div>
                <div class="d-flex flex-wrap align-items-center gap-2">
                    <select id="productFilter" class="form-select form-select-sm" style="width: auto;" onchange="filterTable()">
                        <option value="All">All Products</option>
                        {% for product in products %}
                        <option value="{{ product.name }}">{{ product.name }}</option>
                        {% endfor %}
                        <option value="General Store Experience">General Store Experience</option>
                    </select>
                    <select id="categoryFilter" class="form-select form-select-sm" style="width: auto;" onchange="filterTable()">
                        <option value="All">All Categories</option>
                        {% for cat in categories %}
                        <option value="{{ cat.name }}">{{ cat.name }}</option>
                        {% endfor %}
                        <option value="General Support">General Support</option>
                    </select>
                </div>
            </div>

            <div class="table-responsive">
                <table class="table table-custom align-middle">
                    <thead>
                        <tr>
                            <th>#</th>
                            <th>Customer</th>
                            <th>Product</th>
                            <th>Category</th>
                            <th>Rating</th>
                            <th>Comments</th>
                            <th>Date</th>
                        </tr>
                    </thead>
                    <tbody id="feedbackTableBody">
                        {% for item in feedbacks %}
                        <tr data-product="{{ item['product_name'] }}" data-category="{{ item['category'] }}">
                            <td class="fw-bold text-custom-muted">#{{ item['id'] }}</td>
                            <td class="fw-semibold">{{ item['customer_name'] }}</td>
                            <td><span class="fw-semibold small">{{ item['product_name'] }}</span></td>
                            <td>
                                <span class="badge bg-primary-subtle text-primary border border-primary-subtle">{{ item['category'] }}</span>
                            </td>
                            <td class="text-nowrap">
                                <span class="text-warning me-1">
                                    {% for _ in range(item['rating']) %}<i class="fa-solid fa-star"></i>{% endfor %}
                                </span>
                                <small class="fw-bold text-custom-muted">({{ item['rating'] }})</small>
                            </td>
                            <td class="text-custom-muted">{{ item['comments'] }}</td>
                            <td class="text-nowrap"><small class="text-custom-muted">{{ item['created_at'] }}</small></td>
                        </tr>
                        {% else %}
                        <tr>
                            <td colspan="7" class="text-center py-4 text-custom-muted">No feedback records found yet.</td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>

        <footer class="text-center mt-5 text-custom-muted small">
            <p>Developed by <strong>Group 20</strong> • Small Business Customer Feedback Dashboard</p>
        </footer>

    </div>

    <script>
        function filterTable() {
            const selectedProduct = document.getElementById('productFilter').value;
            const selectedCategory = document.getElementById('categoryFilter').value;
            const rows = document.querySelectorAll('#feedbackTableBody tr');

            rows.forEach(row => {
                const rowProd = row.getAttribute('data-product');
                const rowCat = row.getAttribute('data-category');
                const matchesProduct = (selectedProduct === 'All' || rowProd === selectedProduct);
                const matchesCategory = (selectedCategory === 'All' || rowCat === selectedCategory);

                row.style.display = (matchesProduct && matchesCategory) ? '' : 'none';
            });
        }

        const catData = {{ category_averages | tojson }};
        const prodData = {{ product_averages | tojson }};

        function getThemeTextColor(theme) {
            return theme === 'dark' ? '#94a3b8' : '#475569';
        }

        const initialTheme = localStorage.getItem('theme') || 'dark';
        const initialTextColor = getThemeTextColor(initialTheme);

        const categoryChart = new Chart(document.getElementById('feedbackChart').getContext('2d'), {
            type: 'bar',
            data: {
                labels: Object.keys(catData),
                datasets: [{
                    label: 'Avg Rating',
                    data: Object.values(catData),
                    backgroundColor: 'rgba(99, 102, 241, 0.85)',
                    borderColor: 'rgba(129, 140, 248, 1)',
                    borderWidth: 2,
                    borderRadius: 8
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: { legend: { display: false } },
                scales: {
                    y: { beginAtZero: true, max: 5, ticks: { color: initialTextColor }, grid: { color: 'rgba(150, 150, 150, 0.15)' } },
                    x: { ticks: { color: initialTextColor }, grid: { display: false } }
                }
            }
        });

        const productChart = new Chart(document.getElementById('productChart').getContext('2d'), {
            type: 'bar',
            data: {
                labels: Object.keys(prodData).map(name => name.split(' ').slice(0, 2).join(' ')),
                datasets: [{
                    label: 'Avg Product Rating',
                    data: Object.values(prodData),
                    backgroundColor: 'rgba(16, 185, 129, 0.85)',
                    borderColor: '#10b981',
                    borderWidth: 2,
                    borderRadius: 8
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: { legend: { display: false } },
                scales: {
                    y: { beginAtZero: true, max: 5, ticks: { color: initialTextColor }, grid: { color: 'rgba(150, 150, 150, 0.15)' } },
                    x: { ticks: { color: initialTextColor }, grid: { display: false } }
                }
            }
        });

        function updateChartColors(theme) {
            const textColor = getThemeTextColor(theme);
            [categoryChart, productChart].forEach(chart => {
                chart.options.scales.x.ticks.color = textColor;
                chart.options.scales.y.ticks.color = textColor;
                chart.update();
            });
        }
    </script>
    """ + TOGGLE_SCRIPT + """
</body>
</html>
"""

# --- Flask Routes ---

@app.route("/")
def index():
    success = request.args.get("success", False)
    error = request.args.get("error")
    return render_template_string(
        PUBLIC_TEMPLATE,
        success=success,
        error=error,
        products=PRODUCTS,
        categories=CATEGORIES
    )


@app.route("/submit", methods=["POST"])
def submit():
    customer_name = request.form.get("customer_name", "").strip()[:80]
    product_name = request.form.get("product_name", "General Store Experience").strip()[:100]
    category = request.form.get("category", "General Support").strip()[:50]
    comments = request.form.get("comments", "").strip()[:500]

    try:
        rating = int(request.form.get("rating", 5))
        if not 1 <= rating <= 5:
            raise ValueError("Rating out of bounds")
    except ValueError:
        return redirect(url_for("index", error="Please select a valid star rating between 1 and 5."))

    if not customer_name or not comments:
        return redirect(url_for("index", error="Name and comments cannot be empty."))

    with get_db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO feedback (customer_name, product_name, category, rating, comments) VALUES (?, ?, ?, ?, ?)",
            (customer_name, product_name, category, rating, comments)
        )
        conn.commit()

    return redirect(url_for("index", success=True))


@app.route("/admin/login", methods=["GET", "POST"])
def admin_login():
    error = None
    if request.method == "POST":
        password = request.form.get("password", "")
        if check_password_hash(ADMIN_PASSWORD_HASH, password):
            session["admin_logged_in"] = True
            return redirect(url_for("admin_dashboard"))
        error = "Incorrect password. Please try again."
    return render_template_string(LOGIN_TEMPLATE, error=error)


@app.route("/admin/dashboard")
def admin_dashboard():
    if not session.get("admin_logged_in"):
        return redirect(url_for("admin_login"))

    with get_db_connection() as conn:
        cursor = conn.cursor()

        cursor.execute("SELECT * FROM feedback ORDER BY id DESC")
        feedbacks = cursor.fetchall()

        cursor.execute("SELECT AVG(rating), COUNT(*) FROM feedback")
        avg_result = cursor.fetchone()
        avg_rating = avg_result[0] if avg_result[0] else 0.0
        total_count = avg_result[1] if avg_result[1] else 0

        cursor.execute("SELECT category, AVG(rating) FROM feedback GROUP BY category")
        category_averages = {row[0]: round(row[1], 2) for row in cursor.fetchall()}

        cursor.execute("SELECT product_name, AVG(rating) FROM feedback GROUP BY product_name ORDER BY AVG(rating) DESC")
        prod_rows = cursor.fetchall()
        product_averages = {row[0]: round(row[1], 2) for row in prod_rows}
        top_product = prod_rows[0][0] if prod_rows else "No reviews yet"

    return render_template_string(
        ADMIN_DASHBOARD_TEMPLATE,
        feedbacks=feedbacks,
        avg_rating=avg_rating,
        total_count=total_count,
        top_product=top_product,
        category_averages=category_averages,
        product_averages=product_averages,
        products=PRODUCTS,
        categories=CATEGORIES
    )


@app.route("/admin/logout")
def admin_logout():
    session.pop("admin_logged_in", None)
    return redirect(url_for("admin_login"))


if __name__ == "__main__":
    app.run(debug=True)
