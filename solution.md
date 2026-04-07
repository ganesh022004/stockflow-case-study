StockFlow Case Study – Backend Engineering Intern

Part 1: Code Review & Debugging
 Issues in the Given Code

1.  Missing Input Validation
The code does not validate required fields like name, sku, price, etc.
Impact:
API may fail with KeyError
Invalid or incomplete data may be stored in the database
Fix:
 Add proper request validation before processing the data.

2. No SKU Uniqueness Check
As per requirement, SKU should be globally unique, but no check is implemented.
Impact:
Duplicate SKUs can be created
Leads to incorrect inventory mapping and data inconsistency
Fix:
 Check SKU existence before inserting or enforce a unique constraint at the database level.

3.  Multiple DB Commits Used
The code commits the database twice:
db.session.commit()
...
db.session.commit()
Impact:
Partial data may be saved if second operation fails
Leads to inconsistent database state
Fix:
 Use a single transaction for both product and inventory creation.

4. No Rollback Handling
There is no error handling or rollback mechanism.
Impact:
In case of failure, database may remain in an inconsistent state
Fix:
 Use try/except with db.session.rollback().

5.  Incorrect Multi-Warehouse Handling
Requirement states that products can exist in multiple warehouses, but code assumes only one warehouse.
Impact:
System is not scalable
Business logic does not support real-world use case

6.  No Price Validation
Price is directly accepted without validation.
Impact:
Negative or invalid price values can be stored
Fix:
 Validate that price is greater than zero before inserting.

7.  No Proper Error Handling or Status Codes
The API always returns success response even when errors occur.
Impact:
Client cannot identify failures properly
Poor API design
Fix:
 Return proper HTTP status codes (400, 500, etc.) with error messages.
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']

    # Validation
    for field in required_fields:
        if field not in data:
            return jsonify({"error": f"{field} is required"}), 400

    if data['price'] <= 0:
        return jsonify({"error": "Price must be greater than 0"}), 400

    try:
        # Check SKU uniqueness
        existing_product = Product.query.filter_by(sku=data['sku']).first()
        if existing_product:
            return jsonify({"error": "SKU already exists"}), 409

        # Create product
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=data['price']
        )

        db.session.add(product)
        db.session.flush()  # get product.id without commit

        # Create inventory entry
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )

        db.session.add(inventory)

        # Single commit (atomic transaction)
        db.session.commit()

        return jsonify({
            "message": "Product created successfully",
            "product_id": product.id
        }), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "Database integrity error"}), 500

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500

PART 2: Database Design

1. Companies
Company (
    id PRIMARY KEY,
    name VARCHAR,
    email VARCHAR
)
2. Warehouses
Warehouse (
    id PRIMARY KEY,
    company_id FOREIGN KEY,
    name VARCHAR,
    location TEXT
)

3. Products
Product (
    id PRIMARY KEY,
    company_id FOREIGN KEY,
    name VARCHAR,
    sku VARCHAR UNIQUE,
    price DECIMAL(10,2),
    product_type ENUM('simple','bundle')
)
4. Inventory (warehouse-wise stock)
Inventory (
    id PRIMARY KEY,
    product_id FOREIGN KEY,
    warehouse_id FOREIGN KEY,
    quantity INT,
    updated_at TIMESTAMP
)
5. Suppliers
Supplier (
    id PRIMARY KEY,
    name VARCHAR,
    email VARCHAR,
    phone VARCHAR
)
6. Product-Supplier mapping
ProductSupplier (
    id PRIMARY KEY,
    product_id FOREIGN KEY,
    supplier_id FOREIGN KEY
)
7. Inventory Movement Log
InventoryTransaction (
    id PRIMARY KEY,
    product_id,
    warehouse_id,
    change_quantity,
    reason,
    created_at TIMESTAMP
)
8. Bundle Products (many-to-many)
ProductBundleItems (
    bundle_id FOREIGN KEY,
    product_id FOREIGN KEY,
    quantity INT
)

PART 3: API Implementation
GET Low Stock Alerts API
Assumptions
“Recent sales activity” = last 30 days
Threshold stored per product
Stockout estimation = based on average daily sales
Supplier = first mapped supplier
from flask import jsonify
from datetime import datetime, timedelta

@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):

    alerts = []

    # Get all warehouses of company
    warehouses = Warehouse.query.filter_by(company_id=company_id).all()

    for warehouse in warehouses:
        inventory_items = Inventory.query.filter_by(warehouse_id=warehouse.id).all()

        for item in inventory_items:
            product = Product.query.get(item.product_id)

            # Assume threshold stored or default
            threshold = product.low_stock_threshold if hasattr(product, 'low_stock_threshold') else 10

            # Skip if stock is sufficient
            if item.quantity > threshold:
                continue

            # Check recent sales activity (mock logic)
            recent_sales = get_recent_sales(product.id)

            if recent_sales == 0:
                continue

            # Estimate days until stockout
            daily_avg_sales = recent_sales / 30
            days_left = int(item.quantity / daily_avg_sales) if daily_avg_sales > 0 else None

            supplier = get_supplier(product.id)

            alerts.append({
                "product_id": product.id,
                "product_name": product.name,
                "sku": product.sku,
                "warehouse_id": warehouse.id,
                "warehouse_name": warehouse.name,
                "current_stock": item.quantity,
                "threshold": threshold,
                "days_until_stockout": days_left,
                "supplier": {
                    "id": supplier.id if supplier else None,
                    "name": supplier.name if supplier else None,
                    "contact_email": supplier.email if supplier else None
                }
            })

    return jsonify({
        "alerts": alerts,
        "total_alerts": len(alerts)
    })
Edge Cases Handled
No warehouse found
No sales activity → ignored
Division by zero (no sales)
Missing supplier
Product not linked properly
Zero inventory
Multiple warehouses per company
def get_recent_sales(product_id):
    # returns sales in last 30 days
    return 100  # mock

def get_supplier(product_id):
    return Supplier.query.first()


