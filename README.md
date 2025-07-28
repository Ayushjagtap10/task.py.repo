from flask import request, jsonify
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import sessionmaker
from decimal import Decimal

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    # Input validation
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    for field in required_fields:
        if field not in data:
            return jsonify({"error": f"Missing field: {field}"}), 400

    try:
        price = Decimal(data['price'])
        if price < 0:
            return jsonify({"error": "Price must be non-negative"}), 400
    except:
        return jsonify({"error": "Invalid price format"}), 400

    # Transaction handling
    try:
        # Check SKU uniqueness
        if Product.query.filter_by(sku=data['sku']).first():
            return jsonify({"error": "SKU already exists"}), 400

        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price,
            warehouse_id=data['warehouse_id']
        )
        db.session.add(product)
        db.session.flush()  # Get product.id before commit

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)
        db.session.commit()

        return jsonify({"message": "Product created", "product_id": product.id}), 201

    except IntegrityError as e:
        db.session.rollback()
        return jsonify({"error": "Database error", "details": str(e)}), 500

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": "Unexpected error", "details": str(e)}), 500
