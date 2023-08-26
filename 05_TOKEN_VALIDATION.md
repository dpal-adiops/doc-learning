### Protect API Enpoint
* Create controller controllers/user.js with following code
```javascript
const User = require('../models/user')

exports.read = async (req, res) => {
  const userId = req.params.id
  User.findById(userId)
    .exec()
    .then((user) => {
      if (!user) {
        return res.status(400).json({
          error: 'User not found',
        })
      }
      user.hashed_password = undefined
      user.salt = undefined
      res.json(user)
    })
    .catch((err) => {
      return res.status(400).json({
        error: 'User not found',
      })
    })
}

```
* Update controllers/auth.js
```javascript
const User = require('../models/user')
const jwt = require('jsonwebtoken')
const { expressjwt: expressjwtRequest } = require('express-jwt')

exports.signup = async (req, res) => {
  // console.log('REQ BODY ON SIGNUP', req.body);
  const { name, email, password } = req.body

  const user = await User.findOne({ email }).exec()
  if (user) {
    return res.status(400).json({
      error: 'Email is taken',
    })
  }

  let newUser = new User({ name, email, password })

  newUser
    .save()
    .then((savedDoc) => {
      return res.json({
        message: 'Signup success! Please signin',
      })
    })
    .catch((err) => {
      return res.status(400).json({
        error: err,
      })
    })
}

exports.signin = async (req, res) => {
  const { email, password } = req.body
  // check if user exist
  User.findOne({ email })
    .exec()
    .then((user) => {
      if (!user) {
        return res.status(400).json({
          error: 'User with that email does not exist. Please signup',
        })
      }

      // authenticate
      if (!user.authenticate(password)) {
        return res.status(400).json({
          error: 'Email and password do not match',
        })
      }
      // generate a token and send to client
      const token = jwt.sign({ _id: user._id }, process.env.JWT_SECRET, {
        expiresIn: '7d',
      })
      const { _id, name, email, role } = user

      return res.json({
        token,
        user: { _id, name, email, role },
      })
    })
    .catch((err) => {
      return res.status(400).json({
        error: 'User with that email does not exist. Please signup',
      })
    })
}

exports.requireSignin = expressjwtRequest({
  secret: process.env.JWT_SECRET,
  algorithms: ['HS256'],
})

```
* Create routes/user.js with the following code
```javscript
const express = require('express')
const router = express.Router()

// import controller
const { requireSignin } = require('../controllers/auth')
const { read } = require('../controllers/user')

router.get('/user/:id', requireSignin, read)

module.exports = router

``` 
* Update server.js
```javascript
const express = require('express')
const morgan = require('morgan')
const cors = require('cors')
const bodyParser = require('body-parser')
const mongoose = require('mongoose')
require('dotenv').config()

const app = express()

// connect to db
mongoose
  .connect(process.env.DATABASE, {
    useNewUrlParser: true,
    //useFindAndModify: false,
    useUnifiedTopology: true,
    //useCreateIndex: true,
  })
  .then(() => console.log('DB connected'))
  .catch((err) => console.log('DB CONNECTION ERROR: ', err))

// import routes
const authRoutes = require('./routes/auth')
const userRoutes = require('./routes/user')

// app middlewares
app.use(morgan('dev'))
app.use(bodyParser.json())
// app.use(cors()); // allows all origins
if ((process.env.NODE_ENV = 'development')) {
  app.use(cors({ origin: `http://localhost:3000` }))
}

// middleware
app.use('/api', authRoutes)
app.use('/api', userRoutes)

const port = process.env.PORT || 8000
app.listen(port, () => {
  console.log(`API is running on port ${port}`)
})

```  
