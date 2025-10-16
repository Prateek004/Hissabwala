import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from datetime import datetime, timedelta
import json
import time
import random
from streamlit_option_menu import option_menu
import sqlite3
import hashlib
import os

# App configuration
st.set_page_config(
    page_title="Smart Business POS",
    page_icon="🏪",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Constants
CURRENCY = "₹"
DATABASE_FILE = "pos_database.db"

# Database initialization - Using single line SQL to avoid indentation issues
def init_database():
    conn = sqlite3.connect(DATABASE_FILE)
    cursor = conn.cursor()
    
    # Users table
    cursor.execute('CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE NOT NULL, email TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL, full_name TEXT NOT NULL, phone TEXT, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)')
    
    # Businesses table
    cursor.execute('CREATE TABLE IF NOT EXISTS businesses (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER NOT NULL, name TEXT NOT NULL, industry TEXT NOT NULL, owner_name TEXT NOT NULL, phone TEXT, gst_number TEXT, address TEXT, currency TEXT DEFAULT "₹", created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, FOREIGN KEY (user_id) REFERENCES users (id))')
    
    # Products table
    cursor.execute('CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY AUTOINCREMENT, business_id INTEGER NOT NULL, sku TEXT NOT NULL, name TEXT NOT NULL, price REAL NOT NULL, stock INTEGER DEFAULT 0, unit TEXT DEFAULT "piece", category TEXT, brand TEXT, description TEXT, cost_price REAL, min_stock INTEGER DEFAULT 5, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, FOREIGN KEY (business_id) REFERENCES businesses (id), UNIQUE(business_id, sku))')
    
    # Sales table
    cursor.execute('CREATE TABLE IF NOT EXISTS sales (id INTEGER PRIMARY KEY AUTOINCREMENT, business_id INTEGER NOT NULL, sale_data TEXT NOT NULL, total_amount REAL NOT NULL, payment_method TEXT, customer_name TEXT, customer_phone TEXT, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, FOREIGN KEY (business_id) REFERENCES businesses (id))')
    
    conn.commit()
    conn.close()

init_database()

# Password hashing
def hash_password(password):
    return hashlib.sha256(password.encode()).hexdigest()

def verify_password(password, hashed):
    return hash_password(password) == hashed

# User authentication
def create_user(username, email, password, full_name, phone=""):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        password_hash = hash_password(password)
        cursor.execute(
            'INSERT INTO users (username, email, password_hash, full_name, phone) VALUES (?, ?, ?, ?, ?)',
            (username, email, password_hash, full_name, phone)
        )
        conn.commit()
        user_id = cursor.lastrowid
        conn.close()
        return user_id
    except sqlite3.IntegrityError:
        return None
    except Exception as e:
        print(f"Error creating user: {e}")
        return None

def authenticate_user(username, password):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute('SELECT id, username, password_hash, full_name FROM users WHERE username = ?', (username,))
        user = cursor.fetchone()
        conn.close()
        if user and verify_password(password, user[2]):
            return {
                "id": user[0],
                "username": user[1],
                "full_name": user[3]
            }
        return None
    except Exception as e:
        print(f"Error authenticating user: {e}")
        return None

def get_user_businesses(user_id):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM businesses WHERE user_id = ?', (user_id,))
        businesses = cursor.fetchall()
        conn.close()
        return businesses
    except Exception as e:
        print(f"Error getting user businesses: {e}")
        return []

# Data storage functions
def save_business_data(business_id, data_type, data):
    if "businesses" not in st.session_state:
        st.session_state.businesses = {}
    if business_id not in st.session_state.businesses:
        st.session_state.businesses[business_id] = {}
    st.session_state.businesses[business_id][data_type] = data
    
    if data_type == "products":
        save_products_to_db(business_id, data)
    elif data_type == "sales":
        save_sales_to_db(business_id, data)

def load_business_data(business_id, data_type):
    if (st.session_state.get("businesses") and
        business_id in st.session_state.businesses and
        data_type in st.session_state.businesses[business_id]):
        return st.session_state.businesses[business_id][data_type]
    
    if data_type == "products":
        return load_products_from_db(business_id)
    elif data_type == "sales":
        return load_sales_from_db(business_id)
    elif data_type == "info":
        return load_business_info_from_db(business_id)
    
    return {}

def save_products_to_db(business_id, products):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute('DELETE FROM products WHERE business_id = ?', (business_id,))
        
        for sku, product in products.items():
            cursor.execute('INSERT INTO products (business_id, sku, name, price, stock, unit, category, brand, description, cost_price, min_stock) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)',
                (business_id, sku, product['name'], product['price'],
                 product.get('stock', 0), product.get('unit', 'piece'),
                 product.get('category', ''), product.get('brand', ''),
                 product.get('description', ''), product.get('cost_price', 0),
                 product.get('min_stock', 5))
            )
        
        conn.commit()
        conn.close()
    except Exception as e:
        print(f"Error saving products to DB: {e}")

def load_products_from_db(business_id):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute('SELECT sku, name, price, stock, unit, category, brand, description, cost_price, min_stock FROM products WHERE business_id = ?', (business_id,))
        
        products = {}
        for row in cursor.fetchall():
            products[row[0]] = {
                'sku': row[0], 'name': row[1], 'price': row[2], 'stock': row[3],
                'unit': row[4], 'category': row[5], 'brand': row[6],
                'description': row[7], 'cost_price': row[8], 'min_stock': row[9]
            }
        
        conn.close()
        return products
    except Exception as e:
        print(f"Error loading products from DB: {e}")
        return {}

def save_sales_to_db(business_id, sales):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        
        for sale_id, sale_data in sales.items():
            cursor.execute('SELECT id FROM sales WHERE id = ?', (sale_id,))
            if not cursor.fetchone():
                cursor.execute('INSERT INTO sales (business_id, sale_data, total_amount, payment_method, customer_name, customer_phone) VALUES (?, ?, ?, ?, ?, ?)',
                    (business_id, json.dumps(sale_data), sale_data['grand_total'],
                     sale_data.get('payment_method', ''), sale_data.get('customer_name', ''),
                     sale_data.get('customer_phone', ''))
                )
        
        conn.commit()
        conn.close()
    except Exception as e:
        print(f"Error saving sales to DB: {e}")

def load_sales_from_db(business_id):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute('SELECT id, sale_data, total_amount, payment_method, customer_name, customer_phone, created_at FROM sales WHERE business_id = ?', (business_id,))
        
        sales = {}
        for row in cursor.fetchall():
            sale_data = json.loads(row[1])
            sale_data['db_id'] = row[0]
            sale_data['created_at'] = row[6]
            sales[f"sale_{row[0]}"] = sale_data
        
        conn.close()
        return sales
    except Exception as e:
        print(f"Error loading sales from DB: {e}")
        return {}

def load_business_info_from_db(business_id):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM businesses WHERE id = ?', (business_id,))
        row = cursor.fetchone()
        conn.close()
        
        if row:
            return {
                'id': row[0], 'user_id': row[1], 'name': row[2], 'industry': row[3],
                'owner': row[4], 'phone': row[5], 'gst_number': row[6],
                'address': row[7], 'currency': row[8], 'created': row[9]
            }
        return {}
    except Exception as e:
        print(f"Error loading business info from DB: {e}")
        return {}

# Industry configurations - FIXED UNITS
INDUSTRY_CONFIGS = {
    "Kirana Store": {
        "measurement_units": ["piece", "kg", "L", "packet", "dozen", "gram", "ml"],
        "default_tax": 5.0,
        "quick_categories": ["Grains", "Pulses", "Oils", "Spices", "Dairy", "Snacks"],
        "color": "#FF6B35",
        "icon": "🛒"
    },
    "Clothing Store": {
        "measurement_units": ["piece"],
        "default_tax": 12.0,
        "quick_categories": ["Men", "Women", "Kids", "Accessories"],
        "color": "#4ECDC4",
        "icon": "👕"
    },
    "Hardware Store": {
        "measurement_units": ["piece", "kg", "L", "packet", "bag", "meter", "set", "box"],
        "default_tax": 18.0,
        "quick_categories": ["Cement", "Steel", "Tools", "Paints", "Electrical"],
        "color": "#45B7D1",
        "icon": "🔧"
    },
    "Electronics Store": {
        "measurement_units": ["piece"],
        "default_tax": 18.0,
        "quick_categories": ["Mobile", "Laptop", "Audio", "Accessories"],
        "color": "#96CEB4",
        "icon": "📱"
    },
    "Restaurant": {
        "measurement_units": ["plate", "piece", "glass", "bowl", "cup"],
        "default_tax": 5.0,
        "quick_categories": ["Starters", "Main Course", "Desserts", "Beverages"],
        "color": "#FECA57",
        "icon": "🍽️"
    },
    "Medical Store": {
        "measurement_units": ["strip", "bottle", "tube", "piece", "packet", "box"],
        "default_tax": 12.0,
        "quick_categories": ["Medicines", "Supplements", "First Aid", "Personal Care"],
        "color": "#FF9FF3",
        "icon": "💊"
    }
}

# Demo products
DEMO_PRODUCTS = {
    "Kirana Store": [
        {"sku": "RICE001", "name": "Basmati Rice", "price": 80.0, "cost_price": 65.0, "category": "Grains", "stock": 50, "unit": "kg"},
        {"sku": "ATTA001", "name": "Wheat Flour", "price": 45.0, "cost_price": 35.0, "category": "Grains", "stock": 30, "unit": "kg"},
        {"sku": "OIL001", "name": "Sunflower Oil", "price": 180.0, "cost_price": 150.0, "category": "Oils", "stock": 15, "unit": "L"}
    ],
    "Hardware Store": [
        {"sku": "CEM001", "name": "Cement Bag", "price": 380.0, "cost_price": 320.0, "category": "Cement", "stock": 100, "unit": "bag"},
        {"sku": "STEEL001", "name": "Steel Rod 12mm", "price": 85.0, "cost_price": 70.0, "category": "Steel", "stock": 200, "unit": "kg"},
        {"sku": "TOOL001", "name": "Tool Kit", "price": 1200.0, "cost_price": 900.0, "category": "Tools", "stock": 15, "unit": "set"}
    ],
    "Clothing Store": [
        {"sku": "SHIRT001", "name": "Formal Shirt", "price": 899.0, "cost_price": 600.0, "category": "Men", "stock": 15, "unit": "piece"},
        {"sku": "JEAN001", "name": "Denim Jeans", "price": 1299.0, "cost_price": 900.0, "category": "Men", "stock": 20, "unit": "piece"}
    ],
    "Restaurant": [
        {"sku": "BIR001", "name": "Chicken Biryani", "price": 220.0, "cost_price": 120.0, "category": "Main Course", "stock": 50, "unit": "plate"},
        {"sku": "PIZ001", "name": "Margherita Pizza", "price": 299.0, "cost_price": 150.0, "category": "Main Course", "stock": 30, "unit": "piece"}
    ]
}

# Initialize session state
def init_session_state():
    defaults = {
        "logged_in": False,
        "current_user": None,
        "current_business": None,
        "current_industry": None,
        "cart": [],
        "businesses": {},
        "selected_category": "All",
        "user_businesses": [],
        "analytics_period": "7d",
        "customer_phone": "",
        "customer_name": ""
    }
    
    for key, value in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = value

init_session_state()

# Utility functions
def get_available_stock(business_id, sku):
    products = load_business_data(business_id, "products")
    product = products.get(sku, {})
    current_stock = product.get("stock", 0)
    
    if st.session_state.cart:
        in_cart = sum(item["quantity"] for item in st.session_state.cart if item["sku"] == sku)
        available = current_stock - in_cart
        return max(0, available)
    
    return current_stock

def create_business(user_id, business_name, industry, owner_name, phone, address="", gst_number=""):
    try:
        conn = sqlite3.connect(DATABASE_FILE)
        cursor = conn.cursor()
        cursor.execute(
            'INSERT INTO businesses (user_id, name, industry, owner_name, phone, gst_number, address) VALUES (?, ?, ?, ?, ?, ?, ?)',
            (user_id, business_name, industry, owner_name, phone, gst_number, address)
        )
        conn.commit()
        business_id = cursor.lastrowid
        conn.close()
        return business_id
    except Exception as e:
        print(f"Error creating business: {e}")
        return None

def setup_demo_business(user_id, business_name, industry, owner_name, phone):
    business_id = create_business(user_id, business_name, industry, owner_name, phone)
    
    if not business_id:
        return None
    
    business_info = {
        "id": business_id,
        "name": business_name,
        "industry": industry,
        "owner": owner_name,
        "phone": phone,
        "created": datetime.now().isoformat(),
        "currency": CURRENCY
    }
    
    products = {p["sku"]: p for p in DEMO_PRODUCTS.get(industry, [])}
    save_business_data(business_id, "products", products)
    save_business_data(business_id, "sales", {})
    
    return business_id

def add_to_cart(product, quantity, unit=None, variant=None):
    business_id = st.session_state.current_business
    sku = product["sku"]
    available_stock = get_available_stock(business_id, sku)
    
    if quantity <= 0:
        st.error("❌ Quantity must be greater than 0")
        return False
    
    if quantity > available_stock:
        st.error(f"❌ Only {available_stock} {product.get('unit', 'pcs')} available!")
        return False
    
    cart_item = {
        "sku": product["sku"],
        "name": product["name"],
        "price": product["price"],
        "quantity": quantity,
        "unit": unit or product.get("unit", "piece"),
        "timestamp": datetime.now().isoformat()
    }
    
    if variant:
        cart_item["variant"] = variant
        cart_item["name"] = f"{product['name']} - {variant}"
    
    st.session_state.cart.append(cart_item)
    return True

# POS Interfaces
def kirana_pos_ui(business_id):
    st.header("🛒 Kirana Store POS")
    products = load_business_data(business_id, "products")
    config = INDUSTRY_CONFIGS["Kirana Store"]
    
    # Categories
    st.subheader("📦 Categories")
    cols = st.columns(6)
    categories = ["All"] + config["quick_categories"]
    
    for idx, category in enumerate(categories):
        if cols[idx % 6].button(category, use_container_width=True,
                               type="primary" if st.session_state.selected_category == category else "secondary"):
            st.session_state.selected_category = category
            st.rerun()
    
    # Products grid
    st.subheader("🛍️ Products")
    filtered_products = [
        p for p in products.values()
        if st.session_state.selected_category == "All" or p.get("category") == st.session_state.selected_category
    ]
    
    if not filtered_products:
        st.info("No products found.")
        return
    
    cols_per_row = 3
    for i in range(0, len(filtered_products), cols_per_row):
        cols = st.columns(cols_per_row)
        for j in range(cols_per_row):
            if i + j < len(filtered_products):
                product = filtered_products[i + j]
                available_stock = get_available_stock(business_id, product["sku"])
                
                with cols[j]:
                    st.markdown(f"**{product['name']}**")
                    st.markdown(f"**Price:** {CURRENCY}{product['price']}/{product.get('unit', 'piece')}")
                    
                    stock_color = "🟢" if available_stock > 10 else "🟡" if available_stock > 0 else "🔴"
                    st.markdown(f"**Stock:** {stock_color} {available_stock}")
                    
                    col_q1, col_q2 = st.columns(2)
                    with col_q1:
                        if available_stock > 0:
                            quantity = st.number_input("Qty", 0.0, float(available_stock), 0.0, 0.5,
                                                     key=f"qty_{product['sku']}", label_visibility="collapsed")
                        else:
                            st.write("Out of Stock")
                            quantity = 0.0
                    
                    with col_q2:
                        unit_options = config["measurement_units"]
                        unit = st.selectbox("Unit", unit_options,
                                          key=f"unit_{product['sku']}", label_visibility="collapsed")
                    
                    if st.button("➕ Add to Cart", key=f"add_{product['sku']}", use_container_width=True,
                               disabled=available_stock == 0 or quantity == 0):
                        if add_to_cart(product, quantity, unit):
                            st.success(f"✅ Added {quantity}{unit} {product['name']}")
                            time.sleep(0.5)
                            st.rerun()

def hardware_pos_ui(business_id):
    st.header("🔧 Hardware Store POS")
    products = load_business_data(business_id, "products")
    
    st.subheader("🏗️ Project Details")
    col1, col2 = st.columns(2)
    with col1:
        contractor = st.selectbox("Contractor", ["Sharma Builders", "Gupta Constructions", "Walk-in Customer"])
    with col2:
        project = st.text_input("Project Name", placeholder="e.g., Galaxy Apartments")
    
    st.subheader("🛠️ Construction Materials")
    
    for product in products.values():
        available_stock = get_available_stock(business_id, product["sku"])
        col1, col2, col3, col4 = st.columns([3, 2, 2, 1])
        
        with col1:
            st.markdown(f"**{product['name']}**")
            st.markdown(f"**{CURRENCY}{product['price']}/{product.get('unit', 'piece')}**")
            stock_color = "🟢" if available_stock > 20 else "🟡" if available_stock > 0 else "🔴"
            st.markdown(f"Stock: {stock_color} {available_stock}")
        
        with col2:
            if available_stock > 0:
                quick_options = ["Custom"] + [str(i) for i in [1, 5, 10, 25] if i <= available_stock]
                bulk_opt = st.selectbox("Quick Qty", quick_options,
                                      key=f"bulk_{product['sku']}", label_visibility="collapsed")
            else:
                bulk_opt = "Custom"
        
        with col3:
            if available_stock > 0:
                if bulk_opt == "Custom":
                    quantity = st.number_input("Quantity", 0, available_stock, 0, key=f"qty_{product['sku']}",
                                            label_visibility="collapsed")
                else:
                    quantity = int(bulk_opt)
                    st.markdown(f"**Qty: {quantity}**")
            else:
                quantity = 0
        
        with col4:
            if st.button("➕ Add", key=f"add_{product['sku']}", use_container_width=True,
                       disabled=available_stock == 0 or quantity == 0):
                if add_to_cart(product, quantity):
                    st.success(f"✅ Added {quantity} {product['name']}")
                    time.sleep(0.5)
                    st.rerun()

def clothing_pos_ui(business_id):
    st.header("👕 Clothing Store POS")
    products = load_business_data(business_id, "products")
    
    st.subheader("👗 Fashion Products")
    
    for product in products.values():
        available_stock = get_available_stock(business_id, product["sku"])
        
        with st.expander(f"{product['name']} - {CURRENCY}{product['price']} | Stock: {available_stock}"):
            col1, col2, col3 = st.columns(3)
            
            with col1:
                size = st.selectbox("Size", ["S", "M", "L", "XL", "XXL"], key=f"size_{product['sku']}")
            with col2:
                color = st.selectbox("Color", ["Red", "Blue", "Black", "White", "Green"], key=f"color_{product['sku']}")
            with col3:
                if available_stock > 0:
                    quantity = st.number_input("Qty", 0, available_stock, 0, key=f"qty_{product['sku']}")
                else:
                    st.write("Out of Stock")
                    quantity = 0
            
            if st.button("➕ Add to Cart", key=f"add_{product['sku']}", use_container_width=True,
                       disabled=available_stock == 0 or quantity == 0):
                variant = f"{size} - {color}"
                if add_to_cart(product, quantity, variant=variant):
                    st.success(f"✅ Added {size} {color} {product['name']}")
                    time.sleep(0.5)
                    st.rerun()

def checkout_ui(business_id):
    st.header("💰 Checkout")
    
    if not st.session_state.cart:
        st.info("🛒 Cart is empty")
        return
    
    # Display cart
    cart_df = pd.DataFrame([{
        "Item": item["name"],
        "Qty": f"{item['quantity']} {item.get('unit', '')}",
        "Price": f"{CURRENCY}{item['price']}",
        "Total": f"{CURRENCY}{item['price'] * item['quantity']:.2f}"
    } for item in st.session_state.cart])
    
    st.dataframe(cart_df, use_container_width=True)
    
    # Customer info
    st.subheader("👤 Customer Information")
    col1, col2 = st.columns(2)
    with col1:
        customer_name = st.text_input("Customer Name", value=st.session_state.get("customer_name", ""))
    with col2:
        customer_phone = st.text_input("Customer Phone", value=st.session_state.get("customer_phone", ""))
    
    # Calculate totals
    subtotal = sum(item["price"] * item["quantity"] for item in st.session_state.cart)
    industry = st.session_state.current_industry
    tax_rate = INDUSTRY_CONFIGS[industry]["default_tax"]
    tax_amount = subtotal * (tax_rate / 100)
    
    # Discounts
    col1, col2 = st.columns(2)
    with col1:
        discount_type = st.selectbox("Discount Type", ["None", "Percentage", "Fixed Amount"])
    with col2:
        if discount_type == "Percentage":
            discount_value = st.number_input("Discount %", 0, 100, 0)
            discount_amount = subtotal * (discount_value / 100)
        elif discount_type == "Fixed Amount":
            discount_amount = st.number_input("Discount Amount", 0.0, float(subtotal), 0.0)
        else:
            discount_amount = 0
    
    grand_total = subtotal + tax_amount - discount_amount
    
    # Display totals
    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.metric("Subtotal", f"{CURRENCY}{subtotal:.2f}")
    with col2:
        st.metric("Tax", f"{CURRENCY}{tax_amount:.2f}")
    with col3:
        st.metric("Discount", f"-{CURRENCY}{discount_amount:.2f}")
    with col4:
        st.metric("Grand Total", f"{CURRENCY}{grand_total:.2f}", delta=f"Tax: {tax_rate}%")
    
    # Payment
    st.subheader("💳 Payment")
    payment_method = st.selectbox("Payment Method", ["Cash", "Card", "UPI", "Online Transfer"])
    
    if st.button("✅ Complete Sale", type="primary", use_container_width=True):
        # Validate stock
        products = load_business_data(business_id, "products")
        stock_violations = []
        
        for item in st.session_state.cart:
            product = products.get(item["sku"])
            if product and item["quantity"] > product.get("stock", 0):
                stock_violations.append(f"{item['name']} - Only {product.get('stock', 0)} available")
        
        if stock_violations:
            st.error("❌ Stock violation detected!")
            for violation in stock_violations:
                st.write(f"• {violation}")
            return
        
        # Save sale
        sale_data = {
            "business_id": business_id,
            "cart": st.session_state.cart.copy(),
            "subtotal": subtotal,
            "tax": tax_amount,
            "discount": discount_amount,
            "grand_total": grand_total,
            "payment_method": payment_method,
            "customer_name": customer_name,
            "customer_phone": customer_phone,
            "datetime": datetime.now().isoformat(),
            "industry": industry
        }
        
        sales = load_business_data(business_id, "sales")
        sale_id = f"sale_{int(time.time())}"
        sales[sale_id] = sale_data
        save_business_data(business_id, "sales", sales)
        
        # Update stock
        for item in st.session_state.cart:
            if item["sku"] in products:
                products[item["sku"]]["stock"] = max(0, products[item["sku"]]["stock"] - item["quantity"])
        save_business_data(business_id, "products", products)
        
        st.success("🎉 Sale completed successfully!")
        st.session_state.cart = []
        st.session_state.customer_name = ""
        st.session_state.customer_phone = ""
        time.sleep(2)
        st.rerun()
    
    if st.button("🗑️ Clear Cart", use_container_width=True):
        st.session_state.cart = []
        st.success("Cart cleared!")
        st.rerun()

# Enhanced Inventory Management
def inventory_ui(business_id):
    st.header("📦 Inventory Management")
    
    products = load_business_data(business_id, "products") or {}
    industry = st.session_state.current_industry
    config = INDUSTRY_CONFIGS.get(industry, {})
    measurement_units = config.get("measurement_units", ["piece", "kg", "L", "packet", "bag", "set"])
    
    if not products:
        st.info("No products in inventory.")
        st.subheader("➕ Add Your First Product")
        with st.form("add_first_product"):
            col1, col2 = st.columns(2)
            with col1:
                new_sku = st.text_input("SKU*", key="first_sku")
                new_name = st.text_input("Product Name*", key="first_name")
                new_price = st.number_input("Price*", min_value=0.0, value=0.0, key="first_price")
            with col2:
                new_stock = st.number_input("Stock*", min_value=0, value=0, key="first_stock")
                new_unit = st.selectbox("Unit", measurement_units, key="first_unit")
                new_category = st.text_input("Category", key="first_category")
            
            if st.form_submit_button("➕ Add Product"):
                if not new_sku or not new_name:
                    st.error("SKU and Product Name are required")
                else:
                    products[new_sku] = {
                        "sku": new_sku, "name": new_name, "price": new_price,
                        "stock": new_stock, "unit": new_unit, "category": new_category,
                        "cost_price": 0, "min_stock": 5
                    }
                    save_business_data(business_id, "products", products)
                    st.success(f"✅ {new_name} added!")
                    st.rerun()
        return
    
    # Low stock alerts
    low_stock_items = [p for p in products.values() if p.get("stock", 0) < p.get("min_stock", 5)]
    if low_stock_items:
        st.warning("🚨 Low Stock Alert!")
        for product in low_stock_items:
            st.write(f"• {product['name']}: Only {product.get('stock', 0)} left")
    
    # Edit products
    st.subheader("✏️ Manage Products")
    
    for sku, product in products.items():
        with st.expander(f"{product['name']} (SKU: {sku}) - Stock: {product.get('stock', 0)}"):
            col1, col2 = st.columns(2)
            
            with col1:
                name = st.text_input("Product Name", value=product["name"], key=f"name_{sku}")
                price = st.number_input("Price", value=float(product["price"]), min_value=0.0, key=f"price_{sku}")
                cost_price = st.number_input("Cost Price", value=float(product.get("cost_price", 0)), min_value=0.0, key=f"cost_{sku}")
                category = st.text_input("Category", value=product.get("category", ""), key=f"cat_{sku}")
            
            with col2:
                stock = st.number_input("Stock", value=int(product.get("stock", 0)), min_value=0, key=f"stock_{sku}")
                min_stock = st.number_input("Min Stock", value=int(product.get("min_stock", 5)), min_value=0, key=f"min_{sku}")
                current_unit = product.get("unit", "piece")
                try:
                    unit_index = measurement_units.index(current_unit)
                except ValueError:
                    unit_index = 0
                unit = st.selectbox("Unit", measurement_units, index=unit_index, key=f"unit_{sku}")
            
            col_save, col_del = st.columns(2)
            with col_save:
                if st.button("💾 Save", key=f"save_{sku}"):
                    products[sku].update({
                        "name": name, "price": price, "cost_price": cost_price,
                        "stock": stock, "min_stock": min_stock, "unit": unit, "category": category
                    })
                    save_business_data(business_id, "products", products)
                    st.success("✅ Product updated!")
                    st.rerun()
            
            with col_del:
                if st.button("🗑️ Delete", key=f"del_{sku}"):
                    del products[sku]
                    save_business_data(business_id, "products", products)
                    st.success("✅ Product deleted!")
                    st.rerun()
    
    # Add new product
    st.divider()
    st.subheader("➕ Add New Product")
    
    with st.form("add_new_product"):
        col1, col2 = st.columns(2)
        
        with col1:
            new_sku = st.text_input("SKU*", key="new_sku")
            new_name = st.text_input("Product Name*", key="new_name")
            new_price = st.number_input("Price*", min_value=0.0, value=0.0, key="new_price")
            new_cost = st.number_input("Cost Price", min_value=0.0, value=0.0, key="new_cost")
        
        with col2:
            new_stock = st.number_input("Stock*", min_value=0, value=0, key="new_stock")
            new_min_stock = st.number_input("Min Stock", min_value=0, value=5, key="new_min_stock")
            new_unit = st.selectbox("Unit", measurement_units, key="new_unit")
            new_category = st.text_input("Category", key="new_category")
        
        if st.form_submit_button("➕ Add Product"):
            if not new_sku or not new_name:
                st.error("SKU and Product Name are required")
            elif new_sku in products:
                st.error("❌ SKU already exists!")
            else:
                products[new_sku] = {
                    "sku": new_sku, "name": new_name, "price": new_price, "cost_price": new_cost,
                    "stock": new_stock, "min_stock": new_min_stock, "unit": new_unit, "category": new_category
                }
                save_business_data(business_id, "products", products)
                st.success(f"✅ {new_name} added!")
                st.rerun()
    
    # Inventory summary
    st.divider()
    st.subheader("📊 Inventory Summary")
    
    total_products = len(products)
    total_stock_value = sum(p["price"] * p.get("stock", 0) for p in products.values())
    total_cost_value = sum(p.get("cost_price", 0) * p.get("stock", 0) for p in products.values())
    potential_profit = total_stock_value - total_cost_value
    out_of_stock = sum(1 for p in products.values() if p.get("stock", 0) == 0)
    low_stock = sum(1 for p in products.values() if 0 < p.get("stock", 0) < p.get("min_stock", 5))
    
    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.metric("Total Products", total_products)
    with col2:
        st.metric("Stock Value", f"{CURRENCY}{total_stock_value:,.2f}")
    with col3:
        st.metric("Potential Profit", f"{CURRENCY}{potential_profit:,.2f}")
    with col4:
        st.metric("Low/No Stock", f"{low_stock + out_of_stock}")

# Enhanced Analytics
def industry_analytics_ui(business_id):
    st.header("📊 Business Analytics")
    
    sales_data = load_business_data(business_id, "sales")
    products = load_business_data(business_id, "products")
    
    if not sales_data:
        st.info("No sales data available yet. Start making sales!")
        return
    
    # Convert to DataFrame
    sales_list = []
    for sale_id, sale in sales_data.items():
        sale_date = datetime.fromisoformat(sale.get("created_at", sale["datetime"]))
        for item in sale.get("cart", []):
            sales_list.append({
                "sale_id": sale_id,
                "datetime": sale_date,
                "product_name": item["name"],
                "quantity": item["quantity"],
                "price": item["price"],
                "revenue": item["price"] * item["quantity"],
                "payment_method": sale["payment_method"]
            })
    
    if not sales_list:
        st.info("No detailed sales data available.")
        return
    
    sales_df = pd.DataFrame(sales_list)
    sales_df["date"] = sales_df["datetime"].dt.date
    sales_df["hour"] = sales_df["datetime"].dt.hour
    
    # Key Metrics
    st.subheader("📈 Key Performance Indicators")
    col1, col2, col3, col4 = st.columns(4)
    
    total_revenue = sales_df["revenue"].sum()
    total_sales = len(sales_df["sale_id"].unique())
    avg_ticket = total_revenue / total_sales if total_sales > 0 else 0
    total_items = sales_df["quantity"].sum()
    
    with col1:
        st.metric("Total Revenue", f"{CURRENCY}{total_revenue:,.2f}")
    with col2:
        st.metric("Total Sales", total_sales)
    with col3:
        st.metric("Avg Ticket", f"{CURRENCY}{avg_ticket:.2f}")
    with col4:
        st.metric("Items Sold", total_items)
    
    # Charts
    col1, col2 = st.columns(2)
    
    with col1:
        # Daily Sales Trend
        daily_sales = sales_df.groupby("date")["revenue"].sum().reset_index()
        if len(daily_sales) > 1:
            fig_daily = px.line(daily_sales, x="date", y="revenue", title="📈 Daily Sales Trend")
            st.plotly_chart(fig_daily, use_container_width=True)
    
    with col2:
        # Top Products
        top_products = sales_df.groupby("product_name")["quantity"].sum().nlargest(10).reset_index()
        if not top_products.empty:
            fig_products = px.bar(top_products, x="quantity", y="product_name", orientation='h',
                                 title="📦 Top Selling Products")
            st.plotly_chart(fig_products, use_container_width=True)
    
    # Payment Methods
    st.subheader("💳 Payment Method Analysis")
    payment_stats = sales_df.groupby("payment_method")["revenue"].sum().reset_index()
    if not payment_stats.empty:
        fig_payment = px.pie(payment_stats, values="revenue", names="payment_method",
                            title="💰 Payment Method Distribution")
        st.plotly_chart(fig_payment, use_container_width=True)

# Authentication Screens
def show_login_screen():
    st.title("🔐 Smart Business POS")
    st.subheader("Complete POS Solution for Every Indian Business")
    
    tab1, tab2 = st.tabs(["🚀 Login", "📝 Sign Up"])
    
    with tab1:
        st.subheader("Login to Your Account")
        with st.form("login_form"):
            username = st.text_input("Username")
            password = st.text_input("Password", type="password")
            login_btn = st.form_submit_button("🔓 Login")
            
            if login_btn:
                if not username or not password:
                    st.error("Please enter both username and password")
                else:
                    user = authenticate_user(username, password)
                    if user:
                        st.session_state.logged_in = True
                        st.session_state.current_user = user
                        st.session_state.user_businesses = get_user_businesses(user["id"])
                        st.success(f"Welcome back, {user['full_name']}!")
                        time.sleep(1)
                        st.rerun()
                    else:
                        st.error("Invalid username or password")
    
    with tab2:
        st.subheader("Create New Account")
        with st.form("signup_form"):
            col1, col2 = st.columns(2)
            with col1:
                full_name = st.text_input("Full Name*")
                username = st.text_input("Username*")
                password = st.text_input("Password*", type="password")
            with col2:
                email = st.text_input("Email*")
                phone = st.text_input("Phone")
                confirm_password = st.text_input("Confirm Password*", type="password")
            
            signup_btn = st.form_submit_button("📝 Create Account")
            
            if signup_btn:
                if not all([full_name, username, email, password, confirm_password]):
                    st.error("Please fill all required fields (*)")
                elif password != confirm_password:
                    st.error("Passwords do not match")
                elif len(password) < 6:
                    st.error("Password must be at least 6 characters")
                else:
                    user_id = create_user(username, email, password, full_name, phone)
                    if user_id:
                        st.success("Account created successfully! Please login.")
                        time.sleep(2)
                        st.rerun()
                    else:
                        st.error("Username or email already exists")

def show_business_selection():
    st.title("🏪 Select Your Business")
    
    if not st.session_state.user_businesses:
        st.info("You don't have any businesses yet. Let's create one!")
        show_business_creation()
        return
    
    st.subheader("Your Businesses")
    cols = st.columns(2)
    
    for idx, business in enumerate(st.session_state.user_businesses):
        with cols[idx % 2]:
            industry_config = INDUSTRY_CONFIGS.get(business[3], {})
            icon = industry_config.get("icon", "🏪")
            color = industry_config.get("color", "#666666")
            
            st.markdown(f"""
            <div style="border: 2px solid {color}; border-radius: 10px; padding: 20px; margin: 10px 0;">
                <h3>{icon} {business[2]}</h3>
                <p><strong>Industry:</strong> {business[3]}</p>
                <p><strong>Owner:</strong> {business[4]}</p>
            </div>
            """, unsafe_allow_html=True)
            
            if st.button(f"Select {business[2]}", key=f"select_{business[0]}", use_container_width=True):
                st.session_state.current_business = business[0]
                st.session_state.current_industry = business[3]
                st.success(f"Selected {business[2]}!")
                time.sleep(1)
                st.rerun()
    
    st.divider()
    if st.button("➕ Create New Business", use_container_width=True):
        st.session_state.show_business_creation = True
        st.rerun()

def show_business_creation():
    st.title("🏪 Create New Business")
    
    with st.form("business_creation"):
        col1, col2 = st.columns(2)
        
        with col1:
            business_name = st.text_input("Business Name*", placeholder="My Kirana Store")
            owner_name = st.text_input("Owner Name*", value=st.session_state.current_user["full_name"])
            phone = st.text_input("Phone*", placeholder="9876543210")
        
        with col2:
            industry = st.selectbox("Industry*", list(INDUSTRY_CONFIGS.keys()))
            gst_number = st.text_input("GST Number", placeholder="Optional")
            address = st.text_area("Address", placeholder="Business address")
        
        create_btn = st.form_submit_button("🎪 Create Business")
        
        if create_btn:
            if not all([business_name, owner_name, phone, industry]):
                st.error("Please fill all required fields (*)")
            else:
                business_id = setup_demo_business(
                    st.session_state.current_user["id"],
                    business_name,
                    industry,
                    owner_name,
                    phone
                )
                
                if business_id:
                    st.session_state.user_businesses = get_user_businesses(st.session_state.current_user["id"])
                    st.session_state.current_business = business_id
                    st.session_state.current_industry = industry
                    st.session_state.show_business_creation = False
                    st.success(f"🎉 {business_name} created successfully!")
                    time.sleep(2)
                    st.rerun()
                else:
                    st.error("Failed to create business")
    
    if st.button("← Back to Business Selection"):
        st.session_state.show_business_creation = False
        st.rerun()

# Main Application
def main():
    st.sidebar.title("🏪 Smart Business POS")
    
    if not st.session_state.logged_in:
        show_login_screen()
    elif not st.session_state.current_business:
        if st.session_state.get("show_business_creation"):
            show_business_creation()
        else:
            show_business_selection()
    else:
        show_main_application()

def show_main_application():
    business_id = st.session_state.current_business
    industry = st.session_state.current_industry
    biz_info = load_business_data(business_id, "info")
    
    with st.sidebar:
        st.write(f"**{biz_info['name']}**")
        st.write(f"*{industry}*")
        st.divider()
        
        menu_options = ["POS", "Analytics", "Inventory", "Settings"]
        menu_icons = ["cash-coin", "graph-up", "box-seam", "gear"]
        
        selected = option_menu(
            menu_title="Navigation",
            options=menu_options,
            icons=menu_icons,
            menu_icon="cast",
            default_index=0
        )
        
        if st.session_state.cart:
            st.divider()
            cart_total = sum(item["price"] * item["quantity"] for item in st.session_state.cart)
            st.write(f"🛒 Cart: {len(st.session_state.cart)} items")
            st.write(f"Total: {CURRENCY}{cart_total:.2f}")
        
        st.divider()
        st.write(f"👤 {st.session_state.current_user['full_name']}")
        
        col1, col2 = st.columns(2)
        with col1:
            if st.button("🏪 Switch Business"):
                st.session_state.current_business = None
                st.session_state.current_industry = None
                st.session_state.cart = []
                st.rerun()
        with col2:
            if st.button("🚪 Logout"):
                st.session_state.logged_in = False
                st.session_state.current_user = None
                st.session_state.current_business = None
                st.session_state.current_industry = None
                st.session_state.cart = []
                st.rerun()
    
    # Main content
    if selected == "POS":
        if industry == "Kirana Store":
            kirana_pos_ui(business_id)
        elif industry == "Hardware Store":
            hardware_pos_ui(business_id)
        elif industry == "Clothing Store":
            clothing_pos_ui(business_id)
        else:
            kirana_pos_ui(business_id)
        
        checkout_ui(business_id)
    
    elif selected == "Analytics":
        industry_analytics_ui(business_id)
    
    elif selected == "Inventory":
        inventory_ui(business_id)
    
    elif selected == "Settings":
        st.header("⚙️ Settings")
        st.write(f"**Business:** {biz_info.get('name')}")
        st.write(f"**Industry:** {biz_info.get('industry')}")
        st.write(f"**Owner:** {biz_info.get('owner')}")

if __name__ == "__main__":
    main()
