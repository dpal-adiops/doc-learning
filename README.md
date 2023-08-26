# Mern Project
This project builds a boilerplate code that can be used for any MERN stack project.
## Getting Start
### Setup Client
* `npx create-react-app client`
* `cd client`
* `npm start`
* Browse http://localhost:3000
### Setup server
* `cd ..`
* `mkdir server`
* `cd server`
* `npm init -y`
* `npm i express nodemon`
* Create server.js
* Add copy and paste the following code 

```javascript
const express = require('express')
const app = express()

app.get('/api/signup', (req, res) => {
  res.json({
    data: 'you hit signup endpoint',
  })
})

const port = process.env.port || 8000

app.listen(port, () => {
  console.log(`API is running on port ${port}`)
})
```
* `node server.js`
* Browse http://localhost:8000/api/signup
* Go to package.json
* Update scripts with the following code
```json
"scripts": {
  "start": "nodemon server.js"
}
```
* Execute `npm start`
### Installing NPM packages
* `npm i body-parser cors dotenv express-jwt express-validator google-auth-library jsonwebtoken mongoose morgan @sendgrid/mail`

