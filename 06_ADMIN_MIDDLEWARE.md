### User authentication, authorization and admin role access
* Update controllers/user.js
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

exports.readAll = async (req, res) => {
  User.find()
    .select('-hashed_password')
    .select('-salt')
    .exec()
    .then((users) => {
      if (!users) {
        return res.status(400).json({
          error: 'Users not found',
        })
      }
      res.json(users)
    })
    .catch((err) => {
      return res.status(400).json({
        error: 'User not found',
      })
    })
}

exports.update = async (req, res) => {
  const userId = req.params.id
  User.findById(userId)
    .exec()
    .then((user) => {
      if (!user) {
        return res.status(400).json({
          error: 'User not found',
        })
      }
      user.set(req.body)
      user.password = req.body.password
      user
        .save()
        .then((savedDoc) => {
          console.log(JSON.stringify(savedDoc))
          savedDoc.hashed_password = undefined
          savedDoc.salt = undefined
          return res.json(savedDoc)
        })
        .catch((err) => {
          return res.status(400).json({
            error: err,
          })
        })
    })
    .catch((err) => {
      return res.status(401).json({
        error: 'User update failed',
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

exports.adminMiddleware = (req, res, next) => {
  User.findById({ _id: req.auth._id })
    .exec()
    .then((user) => {
      if (user.role !== 'admin') {
        return res.status(400).json({
          error: 'Admin resource. Access denied.',
        })
      }
      req.profile = user
      next()
    })
    .catch((err) => {
      return res.status(400).json({
        error: 'User not found',
      })
    })
}

exports.authorizeMiddleware = (req, res, next) => {
  User.findById({ _id: req.auth._id })
    .exec()
    .then((user) => {
      if (user.role !== 'admin' && user._id !== req.params.id) {
        return res.status(400).json({
          error: 'Access denied.',
        })
      }
      req.profile = user
      next()
    })
    .catch((err) => {
      return res.status(400).json({
        error: 'User not found',
      })
    })
}

```
* Update routes/user.js
```javascript
const express = require('express')
const router = express.Router()

// import controller
const {
  requireSignin,
  adminMiddleware,
  authorizeMiddleware,
} = require('../controllers/auth')
const { read, update, readAll } = require('../controllers/user')

// import validators
const {
  userPatchUpdateValidator,
  userUpdateValidator,
} = require('../validators/user')
const { runValidation } = require('../validators')

router.get('/users', requireSignin, adminMiddleware, readAll)
router.get('/users/:id', requireSignin, authorizeMiddleware, read)
router.put(
  '/users/:id',
  requireSignin,
  authorizeMiddleware,
  userUpdateValidator,
  runValidation,
  update
)

module.exports = router

```
* Update routes/auth.js
```javascript
const express = require('express')
const router = express.Router()

// import controller
const { signup, signin } = require('../controllers/auth')

// import validators
const {
  userSignupValidator,
  userSigninValidator,
} = require('../validators/auth')
const { runValidation } = require('../validators')

router.post('/signup', userSignupValidator, runValidation, signup)
router.post('/signin', userSigninValidator, runValidation, signin)

module.exports = router

```
* Update validators/user.js
```javascript
const { check } = require('express-validator')

exports.userUpdateValidator = [
  check('email')
    .optional()
    .isEmail()
    .withMessage('Must be a valid email address'),
  check('password')
    .optional()
    .isLength({ min: 6 })
    .withMessage('Password must be at least  6 characters long'),
  check('name').optional().not().isEmpty().withMessage('Name is required'),
]

```
* Update models/user.js
```javascript
const mongoose = require('mongoose')
const crypto = require('crypto')
// user schema
const userScheama = new mongoose.Schema(
  {
    name: {
      type: String,
      trim: true,
      required: true,
      max: 32,
    },
    email: {
      type: String,
      trim: true,
      required: true,
      unique: true,
      lowercase: true,
    },
    hashed_password: {
      type: String,
    },
    salt: String,
    role: {
      type: String,
      default: 'subscriber',
    },
    resetPasswordLink: {
      data: String,
    },
    verifyFlag: {
      type: Boolean,
      default: false,
    },
  },
  { timestamps: true }
)

// virtual
userScheama
  .virtual('password')
  .set(function (password) {
    this._password = password
    this.salt = this.makeSalt()
    this.hashed_password = this.encryptPassword(password)
  })
  .get(function () {
    return this._password
  })

// methods
userScheama.methods = {
  authenticate: function (plainText) {
    return this.encryptPassword(plainText) === this.hashed_password
  },

  encryptPassword: function (password) {
    if (!password) return ''
    try {
      return crypto.createHmac('sha1', this.salt).update(password).digest('hex')
    } catch (err) {
      return ''
    }
  },

  makeSalt: function () {
    return Math.round(new Date().valueOf() * Math.random()) + ''
  },
}

module.exports = mongoose.model('User', userScheama)

```  
