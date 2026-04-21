CREATE DATABASE ecommerce;
USE ecommerce;

CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    password VARCHAR(50)
);

CREATE TABLE products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    price INT,
    stock INT
);

CREATE TABLE cart (
    cart_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    product_id INT,
    quantity INT
);

CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    total INT
);

INSERT INTO products (name, price, stock) VALUES
('Laptop', 55000, 10),
('Smartphone', 20000, 25),
('Headphones', 1500, 50),
('Keyboard', 800, 40),
('Mouse', 500, 60),
('Monitor', 12000, 15),
('Printer', 7000, 10),
('Tablet', 18000, 20),
('Smart Watch', 5000, 30),
('Speaker', 2500, 35),

('USB Cable', 200, 100),
('Power Bank', 1200, 45),
('External Hard Disk', 4000, 20),
('Pendrive 32GB', 600, 70),
('Pendrive 64GB', 900, 60),
('Router', 2500, 25),
('Webcam', 1500, 30),
('Microphone', 1800, 20),
('Gaming Chair', 15000, 5),
('Desk Lamp', 700, 40),

('Office Chair', 6000, 10),
('Study Table', 8000, 8),
('Bookshelf', 5000, 12),
('Fan', 2000, 25),
('Air Cooler', 9000, 10),
('Air Conditioner', 35000, 5),
('Refrigerator', 25000, 6),
('Washing Machine', 22000, 7),
('Mixer Grinder', 3000, 15),
('Electric Kettle', 1200, 30),

('Toaster', 1500, 20),
('Induction Stove', 2500, 18),
('Gas Stove', 4000, 12),
('Water Purifier', 10000, 8),
('Vacuum Cleaner', 8000, 10),
('Iron Box', 1200, 25),
('Hair Dryer', 1000, 35),
('Trimmer', 1500, 30),
('Backpack', 2000, 40),
('Wallet', 800, 50),

('Shoes', 3000, 30),
('T-Shirt', 700, 60),
('Jeans', 1500, 45),
('Jacket', 2500, 20),
('Cap', 400, 50),
('Sunglasses', 1200, 35),
('Watch', 2500, 25),
('Perfume', 1800, 20),
('Notebook', 100, 100),
('Pen Pack', 200, 80);

#Python-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
import mysql.connector

# DB CONNECTION
db = mysql.connector.connect(
    host="localhost",
    user="root",
    password="S@njay2011",
    database="ecommerce"
)

cursor = db.cursor()

# ---------------- USERS ---------------- #

def register():
    username = input("Enter username: ")
    password = input("Enter password: ")

    cursor.execute("INSERT INTO users (username, password) VALUES (%s, %s)",
                   (username, password))
    db.commit()
    print("Registered successfully!\n")


def login():
    username = input("Enter username: ")
    password = input("Enter password: ")

    cursor.execute("SELECT * FROM users WHERE username=%s AND password=%s",
                   (username, password))
    user = cursor.fetchone()

    if user:
        print("Login successful!\n")
        return user[0]  # user_id
    else:
        print("Invalid credentials\n")
        return None


# ---------------- PRODUCTS ---------------- #

def view_products():
    cursor.execute("SELECT * FROM products")
    products = cursor.fetchall()

    print("\n--- Products ---")
    print(f"{'ID':<5} {'Name':<25} {'Price':<10} {'Stock':<10}")
    print("-" * 55)

    for p in products:
        print(f"{p[0]:<5} {p[1]:<25} ₹{p[2]:<9} {p[3]:<10}")
    print()

# ---------------- CART ---------------- #

def add_to_cart(user_id):
    product_id = int(input("Enter product ID: "))
    quantity = int(input("Enter quantity: "))

    # ❌ Prevent zero or negative quantity
    if quantity <= 0:
        print("❌ Quantity must be greater than 0\n")
        return

    # Check stock
    cursor.execute("SELECT stock, name FROM products WHERE product_id = %s", (product_id,))
    result = cursor.fetchone()

    if result:
        stock, name = result

        if quantity > stock:
            print(f"❌ Only {stock} items available for '{name}'\n")
        else:
            cursor.execute("""
                INSERT INTO cart (user_id, product_id, quantity)
                VALUES (%s, %s, %s)
            """, (user_id, product_id, quantity))
            db.commit()
            print("✅ Added to cart!\n")
    else:
        print("❌ Product not found\n")

def view_cart(user_id):
    cursor.execute("""
        SELECT p.name, p.price, c.quantity
        FROM cart c
        JOIN products p ON c.product_id = p.product_id
        WHERE c.user_id = %s
    """, (user_id,))

    items = cursor.fetchall()

    print("\n--- Your Cart ---")
    print(f"{'Product':<25} {'Price':<10} {'Qty':<10} {'Total':<10}")
    print("-" * 60)

    for item in items:
        total = item[1] * item[2]
        print(f"{item[0]:<25} ₹{item[1]:<9} {item[2]:<10} ₹{total:<10}")

    print()

# ---------------- CHECKOUT ---------------- #

def checkout(user_id):
    cursor.execute("""
        SELECT p.product_id, p.name, p.price, p.stock, c.quantity
        FROM cart c
        JOIN products p ON c.product_id = p.product_id
        WHERE c.user_id = %s AND c.quantity > 0
    """, (user_id,))

    items = cursor.fetchall()

    if not items:
        print("❌ No valid items in cart\n")
        return

    total = 0

    # ✅ FIRST CHECK ALL STOCK
    for item in items:
        product_id, name, price, stock, quantity = item

        if quantity > stock:
            print(f"❌ Not enough stock for '{name}'. Available: {stock}")
            return

    # ✅ IF ALL GOOD → PROCEED
    for item in items:
        product_id, name, price, stock, quantity = item

        total += price * quantity

        # Reduce stock safely

        # OLD CODE (you must replace this)
        cursor.execute("""
            UPDATE products 
            SET stock = stock - %s 
            WHERE product_id = %s
        """, (quantity, product_id))
        

    cursor.execute("""
        INSERT INTO orders (user_id, total) VALUES (%s, %s)
    """, (user_id, total))

    cursor.execute("""
        DELETE FROM cart WHERE user_id = %s
    """, (user_id,))

    db.commit()

    print(f"✅ Order placed! Total = ₹{total}\n")

# ---------------- MAIN MENU ---------------- #

def main():
    while True:
        print("1. Register")
        print("2. Login")
        print("3. Exit")

        choice = input("Enter choice: ")

        if choice == "1":
            register()

        elif choice == "2":
            user_id = login()

            if user_id:
                while True:
                    print("\n1. View Products")
                    print("2. Add to Cart")
                    print("3. View Cart")
                    print("4. Checkout")
                    print("5. Logout")

                    ch = input("Enter choice: ")

                    if ch == "1":
                        view_products()
                    elif ch == "2":
                        add_to_cart(user_id)
                    elif ch == "3":
                        view_cart(user_id)
                    elif ch == "4":
                        checkout(user_id)
                    elif ch == "5":
                        break

        elif choice == "3":
            break


main()
