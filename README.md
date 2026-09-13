# Week-3-Day-2-Cart-API-Server-Validation
move the cart logic from only the browser into the backend + MongoDB.


1. Create Cart.js

Create:

backend/models/Cart.js

Add:

const mongoose = require("mongoose");

const cartSchema = new mongoose.Schema(
  {
    customerId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    },

    storeId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Store",
      required: true
    },

    items: [
      {
        productId: {
          type: mongoose.Schema.Types.ObjectId,
          ref: "Product",
          required: true
        },

        name: {
          type: String,
          required: true
        },

        price: {
          type: Number,
          required: true
        },

        quantity: {
          type: Number,
          required: true,
          min: 1
        },

        image: {
          type: String,
          default: ""
        }
      }
    ],

    totalAmount: {
      type: Number,
      default: 0
    }
  },
  {
    timestamps: true
  }
);

cartSchema.index(
  { customerId: 1, storeId: 1 },
  { unique: true }
);

module.exports = mongoose.model("Cart", cartSchema);


2. Create Cart Controller

Create:

backend/controllers/cartController.js

Add:

const Cart = require("../models/Cart");
const Product = require("../models/Product");
const Store = require("../models/Store");


// Add product to cart
const addToCart = async (req, res) => {
  try {
    if (req.user.role !== "customer") {
      return res.status(403).json({
        message: "Only customers can use the cart"
      });
    }

    const { storeId } = req.params;
    const { productId, quantity } = req.body;

    const requestedQuantity = Number(quantity);

    if (
      !productId ||
      !requestedQuantity ||
      requestedQuantity < 1
    ) {
      return res.status(400).json({
        message: "Product and valid quantity are required"
      });
    }

    // Check store
    const store = await Store.findOne({
      _id: storeId,
      isActive: true
    });

    if (!store) {
      return res.status(404).json({
        message: "Store not found or inactive"
      });
    }

    // Get product from database
    const product = await Product.findOne({
      _id: productId,
      storeId: storeId
    });

    if (!product) {
      return res.status(404).json({
        message: "Product not found in this store"
      });
    }

    if (product.stock < requestedQuantity) {
      return res.status(400).json({
        message: `Only ${product.stock} items are available`
      });
    }

    // Find existing cart
    let cart = await Cart.findOne({
      customerId: req.user.userId,
      storeId: storeId
    });

    if (!cart) {
      cart = new Cart({
        customerId: req.user.userId,
        storeId: storeId,
        items: [],
        totalAmount: 0
      });
    }

    const existingItem = cart.items.find(
      (item) =>
        item.productId.toString() ===
        productId.toString()
    );

    if (existingItem) {

      const newQuantity =
        existingItem.quantity +
        requestedQuantity;

      if (newQuantity > product.stock) {
        return res.status(400).json({
          message: `Only ${product.stock} items are available`
        });
      }

      existingItem.quantity = newQuantity;

      // Always get latest price from database
      existingItem.price = product.price;
      existingItem.name = product.name;
      existingItem.image = product.image;

    } else {

      cart.items.push({
        productId: product._id,
        name: product.name,
        price: product.price,
        quantity: requestedQuantity,
        image: product.image
      });

    }

    cart.totalAmount = cart.items.reduce(
      (total, item) =>
        total +
        item.price * item.quantity,
      0
    );

    await cart.save();

    res.status(200).json({
      message: "Product added to cart",
      cart
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


// Get cart
const getCart = async (req, res) => {
  try {
    if (req.user.role !== "customer") {
      return res.status(403).json({
        message: "Only customers can access cart"
      });
    }

    const { storeId } = req.params;

    const cart = await Cart.findOne({
      customerId: req.user.userId,
      storeId: storeId
    });

    if (!cart) {
      return res.status(200).json({
        cart: {
          items: [],
          totalAmount: 0
        }
      });
    }

    res.status(200).json({
      cart
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


// Update quantity
const updateCartItem = async (req, res) => {
  try {
    if (req.user.role !== "customer") {
      return res.status(403).json({
        message: "Only customers can update cart"
      });
    }

    const { storeId, productId } = req.params;
    const quantity = Number(req.body.quantity);

    if (!quantity || quantity < 1) {
      return res.status(400).json({
        message: "Quantity must be at least 1"
      });
    }

    const product = await Product.findOne({
      _id: productId,
      storeId: storeId
    });

    if (!product) {
      return res.status(404).json({
        message: "Product not found"
      });
    }

    if (quantity > product.stock) {
      return res.status(400).json({
        message: `Only ${product.stock} items are available`
      });
    }

    const cart = await Cart.findOne({
      customerId: req.user.userId,
      storeId: storeId
    });

    if (!cart) {
      return res.status(404).json({
        message: "Cart not found"
      });
    }

    const item = cart.items.find(
      (item) =>
        item.productId.toString() ===
        productId.toString()
    );

    if (!item) {
      return res.status(404).json({
        message: "Product is not in the cart"
      });
    }

    item.quantity = quantity;

    // Refresh product information
    item.name = product.name;
    item.price = product.price;
    item.image = product.image;

    cart.totalAmount = cart.items.reduce(
      (total, item) =>
        total +
        item.price * item.quantity,
      0
    );

    await cart.save();

    res.status(200).json({
      message: "Cart updated",
      cart
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


// Remove item
const removeFromCart = async (req, res) => {
  try {
    if (req.user.role !== "customer") {
      return res.status(403).json({
        message: "Only customers can remove cart items"
      });
    }

    const { storeId, productId } = req.params;

    const cart = await Cart.findOne({
      customerId: req.user.userId,
      storeId: storeId
    });

    if (!cart) {
      return res.status(404).json({
        message: "Cart not found"
      });
    }

    cart.items = cart.items.filter(
      (item) =>
        item.productId.toString() !==
        productId.toString()
    );

    cart.totalAmount = cart.items.reduce(
      (total, item) =>
        total +
        item.price * item.quantity,
      0
    );

    await cart.save();

    res.status(200).json({
      message: "Product removed from cart",
      cart
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


// Clear cart
const clearCart = async (req, res) => {
  try {
    if (req.user.role !== "customer") {
      return res.status(403).json({
        message: "Only customers can clear cart"
      });
    }

    const { storeId } = req.params;

    const cart = await Cart.findOne({
      customerId: req.user.userId,
      storeId: storeId
    });

    if (!cart) {
      return res.status(404).json({
        message: "Cart not found"
      });
    }

    cart.items = [];
    cart.totalAmount = 0;

    await cart.save();

    res.status(200).json({
      message: "Cart cleared",
      cart
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};


module.exports = {
  addToCart,
  getCart,
  updateCartItem,
  removeFromCart,
  clearCart
};
3. Create Cart Routes

Create:

backend/routes/cartRoutes.js

Add:

const express = require("express");

const {
  addToCart,
  getCart,
  updateCartItem,
  removeFromCart,
  clearCart
} = require("../controllers/cartController");

const {
  protect,
  authorizeRoles
} = require("../middleware/authMiddleware");

const router = express.Router();


// Add product
router.post(
  "/:storeId/add",
  protect,
  authorizeRoles("customer"),
  addToCart
);


// Get cart
router.get(
  "/:storeId",
  protect,
  authorizeRoles("customer"),
  getCart
);


// Update quantity
router.put(
  "/:storeId/item/:productId",
  protect,
  authorizeRoles("customer"),
  updateCartItem
);


// Remove item
router.delete(
  "/:storeId/item/:productId",
  protect,
  authorizeRoles("customer"),
  removeFromCart
);


// Clear cart
router.delete(
  "/:storeId/clear",
  protect,
  authorizeRoles("customer"),
  clearCart
);


module.exports = router;

These are standard Express routes with authentication middleware before the controller.

4. Connect Cart Routes to Server

Open:

backend/server.js

Add:

const cartRoutes = require("./routes/cartRoutes");

Then add:

app.use("/api/cart", cartRoutes);

So your API section becomes:

app.use("/api/auth", authRoutes);
app.use("/api/users", userRoutes);
app.use("/api/stores", storeRoutes);
app.use("/api/products", productRoutes);
app.use("/api/upload", uploadRoutes);
app.use("/api/cart", cartRoutes);
5. Your Cart API

You now have:

Method	Endpoint	Purpose
POST	/api/cart/:storeId/add	Add product
GET	/api/cart/:storeId	Get cart
PUT	/api/cart/:storeId/item/:productId	Change quantity
DELETE	/api/cart/:storeId/item/:productId	Remove product
DELETE	/api/cart/:storeId/clear	Clear cart
6. Very Important Security Check

Suppose a customer sends:

{
  "productId": "123",
  "quantity": 2,
  "price": 1
}

The backend does not trust price: 1.

Instead it does:

const product = await Product.findOne({
  _id: productId,
  storeId: storeId
});

Then uses:

product.price

from MongoDB.

So if the actual price is:

₹999

the cart stores:

₹999

not:

₹1

This is a very important e-commerce security principle.

7. Test Backend

Start MongoDB and backend:

cd backend
node server.js

You should see:

MongoDB connected: ...
Server running on port 5000
8. Test Add to Cart

Login as a customer first.

Then use your Shop page.

When the customer clicks:

Add to Cart

the frontend should eventually send:

POST /api/cart/STORE_ID/add

with:

{
  "productId": "PRODUCT_ID",
  "quantity": 1
}

The backend checks:

Customer?
   ↓
Store exists?
   ↓
Store active?
   ↓
Product belongs to store?
   ↓
Enough stock?
   ↓
Add to MongoDB
9. Test Insufficient Stock

Suppose product stock is:

5

Try adding:

10

You should receive:

{
  "message": "Only 5 items are available"
}

This prevents customers from adding unavailable quantities.

10. One More Important Improvement

Your current Day 1 cart uses localStorage.

Now that we have the backend Cart API, the backend should become the source of truth.

The architecture should eventually become:

                 CUSTOMER
                    │
                    ▼
              React / Redux
                    │
                    ▼
                Axios API
                    │
                    ▼
             Express Backend
                    │
             ┌──────┴──────┐
             ▼             ▼
           JWT          Validation
             │             │
             └──────┬──────┘
                    ▼
                 MongoDB
                    │
                    ▼
                  Cart





























