### Routes
* `mkdir routes`
* Create file auth.js
* Copy and paste the following code to auth.js
```javascript

const express = require('express')
const router = express.Router()

router.get('/signup', (req, res) => {
  res.json({
    data: 'you hit signup endpoint,Deepak',
  })
})

module.exports = router

```
* Copy and paste the following code to server.js
```javascript
const express = require('express')
const app = express()

//import routes
const authRoutes = require('./routes/auth')

//middleware
app.use('/api', authRoutes)

const port = process.env.port || 8000

app.listen(port, () => {
  console.log(`API is running on port ${port}`)
})

```  
### Controllers
* `mkdir controllers`
* Create file auth.js
* Copy and paste the following code to auth.js
```javascript

exports.signup = (req, res) => {
  res.json({
    data: 'you hit signup endpoint',
  })
}


```
* Copy and paste the following code to routes/auth.js
```javascript
const express = require('express')
const router = express.Router()

//import controller

const { signup } = require('../controllers/auth')

router.get('/signup', signup)

module.exports = router


```
### Model
* `mkdir models`
* Create file user.js
* Copy and paste the following code to user.js
```javascript
const mongoose = require('mongoose')
const crypto = require('crypto')
// schema
const userSchema = new mongoose.Schema(
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
      lowercase: true,
      unique:true,
    },
    hashed_password: {
      type: String,
      trim: true,
      required: true,
    },
    salt: String,
    role: {
      type: String,
      default: 'guest',
    },
    resetPasswordLink: {
      data: String,
      default: '',
    },
  },
  { timeStamps: true }
)

//virtuals
userSchema.virtual('password'
.set(function(password){
    this._password=password
    this.salt=this.makeSalt()
    this.hashed_password=this.encryptPassword(password)
}).get(function(){
    return this._password
})

//methods
userSchema.methods={
    authenticate: function(plainText){
        return this.encryptPassword(plainText) == this.hashed_password
    },
    encryptPassword:function(password){
       if(!password) return ''
       try{
        return crypto.createHmac('sha1',this.salt).update(password).digest('hex')
       } catch(err){return ''}
    },
    makeSalt:function(){
        return  Math.round(new Date().valueOf() * Math.random()) 
    }

   
}

module.exports= mongoose.model('User',userSchema)

```
### Applying middlewares
* Create .env
* Copy and paste the following code to .env
```
NODE_ENV=development
PORT=8000
CLIENT_URL=http://localhost:3000
DATABASE='mongodb+srv://<username>:<password>@<cluster>.<id>.mongodb.net/<db>?retryWrites=true&w=majority'

```
* Copy and paste the following code to server.js
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

// app middlewares
app.use(morgan('dev'))
app.use(bodyParser.json())
// app.use(cors()); // allows all origins
if ((process.env.NODE_ENV = 'development')) {
  app.use(cors({ origin: `http://localhost:3000` }))
}

// middleware
app.use('/api', authRoutes)

const port = process.env.PORT || 8000
app.listen(port, () => {
  console.log(`API is running on port ${port}`)
})

```
