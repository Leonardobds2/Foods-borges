borges-delivery/
├── client/
│   ├── index.html
│   ├── styles.css
│   └── script.js
├── server/
│   ├── app.js
│   ├── config.js
│   ├── routes/
│   │   ├── auth.js
│   │   ├── products.js
│   │   └── orders.js
│   └── models/
│       ├── User.js
│       ├── Product.js
│       └── Order.js
├── package.json
└── .env
npm init -y
npm install express mongoose bcryptjs jsonwebtoken dotenv cors
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
const express = require('express');
const mongoose = require('mongoose');
const dotenv = require('dotenv');
const cors = require('cors');
const authRoutes = require('./routes/auth');
const productRoutes = require('./routes/products');
const orderRoutes = require('./routes/orders');

dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
    .then(() => console.log('MongoDB connected'))
    .catch((error) => console.error(error));

app.use('/auth', authRoutes);
app.use('/products', productRoutes);
app.use('/orders', orderRoutes);

app.listen(3000, () => console.log('Server running on port 3000'));
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
    name: String,
    email: { type: String, unique: true },
    password: String,
});

UserSchema.pre('save', async function(next) {
    if (!this.isModified('password')) return next();
    this.password = await bcrypt.hash(this.password, 10);
    next();
});

module.exports = mongoose.model('User', UserSchema);
const mongoose = require('mongoose');

const ProductSchema = new mongoose.Schema({
    name: String,
    description: String,
    price: Number,
    category: String,
    available: Boolean,
});

module.exports = mongoose.model('Product', ProductSchema);
const mongoose = require('mongoose');

const OrderSchema = new mongoose.Schema({
    userId: String,
    products: [{ productId: String, quantity: Number }],
    total: Number,
    status: { type: String, default: 'Pending' },
});

module.exports = mongoose.model('Order', OrderSchema);
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

router.post('/register', async (req, res) => {
    const { name, email, password } = req.body;
    try {
        const user = new User({ name, email, password });
        await user.save();
        res.status(201).json({ message: 'User registered' });
    } catch (error) {
        res.status(400).json({ error: 'User registration failed' });
    }
});

router.post('/login', async (req, res) => {
    const { email, password } = req.body;
    try {
        const user = await User.findOne({ email });
        if (!user || !(await bcrypt.compare(password, user.password))) {
            return res.status(400).json({ error: 'Invalid credentials' });
        }
        const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
        res.json({ token });
    } catch (error) {
        res.status(400).json({ error: 'Login failed' });
    }
});

module.exports = router;
const express = require('express');
const Product = require('../models/Product');
const router = express.Router();

router.get('/', async (req, res) => {
    try {
        const products = await Product.find();
        res.json(products);
    } catch (error) {
        res.status(400).json({ error: 'Failed to fetch products' });
    }
});

router.post('/', async (req, res) => {
    const { name, description, price, category, available } = req.body;
    try {
        const product = new Product({ name, description, price, category, available });
        await product.save();
        res.status(201).json(product);
    } catch (error) {
        res.status(400).json({ error: 'Failed to create product' });
    }
});

module.exports = router;
const express = require('express');
const Order = require('../models/Order');
const router = express.Router();

router.post('/', async (req, res) => {
    const { userId, products, total } = req.body;
    try {
        const order = new Order({ userId, products, total });
        await order.save();
        res.status(201).json(order);
    } catch (error) {
        res.status(400).json({ error: 'Order creation failed' });
    }
});

router.get('/:userId', async (req, res) => {
    const { userId } = req.params;
    try {
        const orders = await Order.find({ userId });
        res.json(orders);
    } catch (error) {
        res.status(400).json({ error: 'Failed to fetch orders' });
    }
});

module.exports = router;
