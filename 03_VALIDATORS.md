### Validators
* `mkdir validators`
* `cd validators`
*  Create index.js
*  Create auth.js
*  Copy and paste the following code in validators/auth.js
```javascript
const { check } = require('express-validator');

exports.userSignupValidator = [
    check('name')
        .not()
        .isEmpty()
        .withMessage('Name is required'),
    check('email')
        .isEmail()
        .withMessage('Must be a valid email address'),
    check('password')
        .isLength({ min: 6 })
        .withMessage('Password must be at least  6 characters long')
]
```
*  Copy and paste the following code in validators/index.js
```javascript
const { validationResult } = require('express-validator');

exports.runValidation = (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        return res.status(422).json({
            error: errors.array()[0].msg
        });
    }
    next();
};
```
* Update the models/auth.js with the following code
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
      required: true,
    },
    salt: String,
    role: {
      type: String,
      default: 'subscriber',
    },
    resetPasswordLink: {
      data: String,
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
* Update the routes/auth.js to incorporate validator middleware with the following code
```javascript
const express = require('express');
const router = express.Router();

// import controller
const { signup } = require('../controllers/auth');

// import validators
const { userSignupValidator } = require('../validators/auth');
const { runValidation } = require('../validators');

router.post('/signup', userSignupValidator, runValidation, signup);

module.exports = router;
```
* update the controllers/auth.js with the following code
```javascript
const User = require('../models/user')

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
```
* For Testing, execute or import following curl to postman
```
curl --location --request POST 'http://localhost:8000/api/signup' \ --header 'Content-Type: application/json' \ --data-raw '{
    "name": "Deepak",
    "email": "test@test.com",
    "password": "password"
}' 
```
