### Signin User
* Update validators/auth.js with the following code
```javascript
const { check } = require('express-validator')

exports.userSignupValidator = [
  check('name').not().isEmpty().withMessage('Name is required'),
  check('email').isEmail().withMessage('Must be a valid email address'),
  check('password')
    .isLength({ min: 6 })
    .withMessage('Password must be at least  6 characters long'),
]

exports.userSigninValidator = [
  check('email').isEmail().withMessage('Must be a valid email address'),
  check('password')
    .isLength({ min: 6 })
    .withMessage('Password must be at least  6 characters long'),
]

```
* Update controllers/auth.js with the following code
```javascript
const User = require('../models/user')
const jwt = require('jsonwebtoken')

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

```
* Update routes/auth.js with the following code
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
