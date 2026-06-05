---
tags:
  - Linux
---
~~*(old [MERN](https://github.com/hectorproko/MERN_STACK))*~~

> [!info]
> A technology stack is a set of frameworks and tools used to develop a software product. Here we will implement a **MERN** which consits of:
> 
>  **M**ongoDB - No-SQL database <br>
>  **E**xpressJS - A server side Web Application framework for Node.js <br>
>  **R**eactJS - A JavaScript based frontend framework used to build User Interface (UI) components <br>
>  **N**ode.js - A JavaScript runtime environment. It is used to run JavaScript on a machine rather than in a browser.
> 
> In this project the user interacts with the **ReactJS UI** components (application front-end ) from the browswer. This frontend is served by **ExpressJS** (running on top of **NodeJS**) which is the application backend. Any interaction that causes a data change request is sent to the **NodeJS** based **ExpressJS** server, which grabs data from the **MongoDB** database if required, and returns the data to the frontend of the application, which is then presented to the user.
> 
> Technologies/Tools used:
> * AWS (EC2)
> * Ubuntu Server 20.04 LTS
> * MonogDB
> * ExpressJS
> * ReactJS
> * Node.js
> * GitBash
> * Postman

Before we start we need to have an environment to work with. I will we using my AWS account to create an EC2 instance with an Ubuntu Server.

* First thing I'm going to do when I log in to AWS is look for the **EC2** services. There are various methods to navigate to it, here I'm using the **search bar** <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/ec2search.png) <br>

* Once you navigate to the **EC2** page look for a **Launch instance** button <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/launchInstance.png) <br>

* You will then be prompted to pick an OS Image. I will be using **Ubuntu Server 20.04 LTS (HVM), SSD Volume**. Once done click **Select**

* I will pick the **t2.micro** instace type <br /> 
 ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/t2micro.png) <br>

* I will leave default settings and click **Review and Launch** <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/reviewLaunch.png) <br>

* As you can see we have a Security Group applied to the instance by default which allows SSH connections <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/sshDefault.png) <br>

* After reviewing you can launch your instancing by clicking <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/launch.png) <br>

* We are prompted to create or use an existing Key Pair. I will be creating a new one. I will use this .pem key to SSH into the instance later on. <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/keyPair.png) <br>

* Once you have downloaded your key launch intance <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/LaunchInstances.png) <br>

* To go to the instances dashboard <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/ViewInstances.png) <br>

* If your instance is up and running you will see something like this <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/Running.png) <br>

* To find information on how to connect click on your **Instance ID**

* In the top-right corner you should see the button **Connect**, click on it <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/Connect.png) <br>

* Look for the **SSH client** tab <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/SSHclient.png) <br>

* Under **Example** you'll find an **ssh** command with eveything you need to connect to the instance from a terminal

**Examaple:**
```bash
ssh -i "daro.io.pem" ubuntu@ec2-3-216-90-84.compute-1.amazonaws.com
```
Make sure when you run the command that your current working directory in the terminal is where your KeyPair/.pem is located because in the above example I'm using a relative path to point to my key

* A successful log-in <br /> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/ubunutuLogIn.png) <br>


# STEP1: BACKEND CONFIGURATION
---
First, make sure the system is up to date
```bash
sudo apt update && sudo apt upgrade -y
```
## INSTALL NODE.JS
We get the location of **Node.js** software from [Ubuntu repositories](https://github.com/nodesource/distributions#deb) .

```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
```
Install **Node.js** with the command below
```bash
sudo apt-get install -y nodejs
```
In addition, the command above installs **NPM** a package manager for Node used to install Node modules & packages and to manage dependency conflicts.

Verify the node installation with the command **node -v**
```bash
ubuntu@ip-172-31-59-251:~$ node -v
v12.22.5
```

We can also vertify the installation of Node by checking **NPM** version using command **npm -v**

```bash
ubuntu@ip-172-31-59-251:~$ npm -v
6.14.14
```
Create a new directory for your **To-Do** project and change to it
```bash
mkdir Todo
cd Todo
```

With **Todo** as your current working directory run **npm init** to initialize your project. The initialization process creates **package.json** a file that contains information about your application and dependencies that it needs to run. Follow the prompts after running command and press **Enter** to accepts default values, then accept to write out the package.json file by typing **yes**

```bash
#My output
ubuntu@ip-172-31-59-251:~/Todo$ npm init
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sensible defaults.

See `npm help init` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg>` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
package name: (todo)
version: (1.0.0)
description: Todo app description
entry point: (index.js)
test command:
git repository:
keywords: todo application
author: Hector Rodriguez
license: (ISC)
About to write to /home/ubuntu/Todo/package.json:

{
  "name": "todo",
  "version": "1.0.0",
  "description": "Todo app description",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [
    "todo",
    "application"
  ],
  "author": "Hector Rodriguez",
  "license": "ISC"
}


Is this OK? (yes)

ubuntu@ip-172-31-59-251:~/Todo$ ls #Checking newly created file
package.json
```

## INSTALL EXPRESSJS
**ExpressJS** (a framework for **Node.js**) simplifies development, and abstracts a lot of low level details. For example, Express helps to define routes of your application based on HTTP methods and URLs.

Install ExpressJS using **npm**:
```bash
npm install express
```
Install ExpressJS using **npm**:
```bash
npm install express
```
Create a file index.js with the command **touch index.js** and run **ls** to confirm file creation
```bash
ubuntu@ip-172-31-59-251:~/Todo$ touch index.js
ubuntu@ip-172-31-59-251:~/Todo$ ls
index.js  node_modules  package-lock.json  package.json
ubuntu@ip-172-31-59-251:~/Todo$
```
Notice there are additional things inside **Todo** after installing **ExpressJS**

Install the **dotenv** module
```bash
npm install dotenv
```
Open **index.js** with your favorite file editor such as nano or vim and copy/paste this code inside.
```javascript
const express = require('express');
require('dotenv').config();
const app = express();
const port = process.env.PORT || 5000;
app.use((req, res, next) => {
res.header("Access-Control-Allow-Origin", "\*");
res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
next();
});
app.use((req, res, next) => {
res.send('Welcome to Express');
});
app.listen(port, () => {
console.log(`Server running on port ${port}`)
});
```
Notice that we have specified to use port 5000 in the code. This will be required later when we go on the browser.

Now we start the server(**ExpressJS** + **Node.js**) to see if it works. In the same directory as **index.js** (dir **Todo**) run **node index.js**
```bash
ubuntu@ip-172-31-59-251:~/Todo$ node index.js

Server running on port 5000 #Output if everything is ok
```

* Now we are going to add a rule to our **Security Group** to open **TCP** port **5000** (while I'm here I will open port **3000** as well)
    * Navigate to your intances dashboard and select your instance by cliking the empty box <br /> 
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/checkMark.png) <br>
    * Look for the **Security** tab <br /> 
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/security.png) <br>
    * Click on top of your **Security Group**  <br />
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/securityGroup.png) <br> <br />
    Keep in mind yours might look different
    * Under the tab **Inbound Rules** click **Edit inbound rules** button <br />
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/editInboundRules.png) <br> <br />
    * Click **Add rule** <br />
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/addRule.png) <br> 
    * Pick type **Custom TCP** and source **0.0.0.0/0** meaning all IPs, you can also use drop-down menu **Anywhere-IPv4**
    * Save it <br />
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LAMP_STACK/main/images/SaveRules.png) <br> 

    * I will now test Express on the browser using the AWS EC2 Intance **Public IP** followed by the port **5000** <br>
    ![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/welcomeExpress.png) <br>

**Routes**<br>
Our **To-Do** application needs to be able to do the following 3 actions:
1. Create a new task
1. Display list of all tasks
1. Delete a completed task 

* **Routing** refers to how an application’s endpoints (URIs) respond to client requests.
* Each task will be associated with some particular endpoint and will use different standard HTTP request methods: POST, GET, DELETE.
* For each task, we need to create routes that will define various endpoints that the To-do app will depend on. So let us create a folder routes

Now lets create a directory **routes** and inside of it a file **api.js**
```bash
ubuntu@ip-172-31-59-251:~/Todo$ mkdir routes
ubuntu@ip-172-31-59-251:~/Todo$ cd routes
ubuntu@ip-172-31-59-251:~/Todo/routes$ touch api.js
ubuntu@ip-172-31-59-251:~/Todo/routes$ vim api.js
```

Using a file editor we'll open **api.js** and paste the following code inside
```javascript
const express = require ('express');
const router = express.Router();
router.get('/todos', (req, res, next) => {
});
router.post('/todos', (req, res, next) => {
});
router.delete('/todos/:id', (req, res, next) => {
})
module.exports = router;
```
## MODELS

We will need to create a model for our app to make user of **MongoDB**. In JavaScript a model is what makes it interactive. This is important for us to be able to define the fields stored in each Mongodb document. To create a Schema and a model, we need to install mongoose (a Node.js package that makes working with mongodb easier).

To install Mongoose change directory back **Todo** folder and run
```bash
npm install mongoose
```
We'll create a new directory **models** and inside of it a new file **todo.js**
```bash
ubuntu@ip-172-31-59-251:~/Todo$ mkdir models && cd models && touch todo.js
```

Open **todo.js** with a file editor to paste below code
```bash
ubuntu@ip-172-31-59-251:~/Todo/models$ vi todo.js
```
```Javascript
const mongoose = require('mongoose');
const Schema = mongoose.Schema;
//create schema for todo
const TodoSchema = new Schema({
action: {
type: String,
required: [true, 'The todo text field is required']
}
})o
//create model for todo
const Todo = mongoose.model('todo', TodoSchema);
module.exports = Todo;
```
Now we need to update our routes from the file **api.js** in **routes** directory to make use of the new model.
```bash
ubuntu@ip-172-31-59-251:~/Todo/routes$ vi api.js
```
We'll delete all the content inside **api.js** and paste code below
```Javascript
const express = require ('express');
const router = express.Router();
const Todo = require('../models/todo');
router.get('/todos', (req, res, next) => {
//this will return all the data, exposing only the id and action field to the client
Todo.find({}, 'action')
.then(data => res.json(data))
.catch(next)
});

router.post('/todos', (req, res, next) => {
if(req.body.action){
Todo.create(req.body)
.then(data => res.json(data))
.catch(next)
}else {
res.json({
error: "The input field is empty"
})
}
});

router.delete('/todos/:id', (req, res, next) => {
Todo.findOneAndDelete({"_id": req.params.id})
.then(data => res.json(data))
.catch(next)
})
module.exports = router;
```

## MONGODB DATABASE
We need a database where we will store our data. For this we will make use of mLab. mLab provides **MongoDB** database as a service solution (DBaaS).<br>
 

[Sign up here.](https://github.com/nodesource/distributions#deb) 
Follow the sign up process, select AWS as the cloud provider, and choose a region near you.

* Pick Shared and click Create<br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/shared.png) <br>

* Pick **aws** as your cloud provider and click **Create Cluster**
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/cloudProvider.png) <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/createCluster.png) <br>


* Allow access to the MongoDB database from anywhere (Not secure, but it is ideal for testing) <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/networkAccess.png) <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/addIP.png) <br> <br>
In the image below, make sure you change the time of deleting the entry from 6 Hours to 1 Week <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/addIP2.png) <br>



* Click on **Browse Collections**<br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/databases.png) <br>

* Click on **Add My Own Data**<br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/loadSampleDataset.png) <br>

* Created Database named **myFirstDatabase**<br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/createDatabase2.png) <br>

* Click **Connect** and on step 2 **Create a Database User** mine is  **user_sample**
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/connect.png) <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/databaseUser.png) <br>

* After user creation click  button **Choose a connection method**

* **Click** <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/connectApp.png) <br>

* Make sure Driver is set to **Node.js** and proper version (left mine 3.7) <br>
Copy the string from number 2. We'll need it to establish a connection<br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/connectToCluster.png) <br>

```bash
#My string
mongodb+srv://user_sample:<password>@cluster0.tzjfy.mongodb.net/myFirstDatabase?retryWrites=true&w=majority
```
Now we create a file **.env** where we will put the connection string above. This file is reference by **index.js** (process.env) in order to access the database 
```bash
ubuntu@ip-172-31-59-251:~/Todo$ touch .env
ubuntu@ip-172-31-59-251:~/Todo$ vi .env
ubuntu@ip-172-31-59-251:~/Todo$ cat .env
DB = 'mongodb+srv://user_sample:Passw0rd!@cluster0.tzjfy.mongodb.net/myFirstDatabase?retryWrites=true&w=majority'
```
Now we need to update the index.js to reflect the use of .env so that Node.js can connect to the database. Left Here
Simply delete existing content in the file, and update it with the entire code below.
```Javascript
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const routes = require('./routes/api');
const path = require('path');
require('dotenv').config();
const app = express();
const port = process.env.PORT || 5000;
//connect to the database
mongoose.connect(process.env.DB, { useNewUrlParser: true, useUnifiedTopology: true })
.then(() => console.log(`Database connected successfully`))
.catch(err => console.log(err));
//since mongoose promise is depreciated, we overide it with node's promise
mongoose.Promise = global.Promise;
app.use((req, res, next) => {
res.header("Access-Control-Allow-Origin", "\*");
res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
next();
});
app.use(bodyParser.json());
app.use('/api', routes);
app.use((err, req, res, next) => {
console.log(err);
next();
});
app.listen(port, () => {
console.log(`Server running on port ${port}`)
});
```
We could write the connection string directly inside the **index.js** application file but using environmental variables to store information is more secure and a best practice to separate configuration and secret data from the application

* Now we start the server (index.js) with commad **node index.js**
```bash
ubuntu@ip-172-31-59-251:~/Todo$ node index.js
Server running on port 5000
Database connected successfully
```
### **Testing Backend Code using RESTful API**
* Since we do not have a frontend UI yet we need a way to test our code using **RESTful API**. Therefore, we will need to make use of some API development client to test our code. We will use **Postman**

* We will test all the API endpoints and make sure they are working. For the endpoints that require body, we will send JSON back with the necessary fields since it’s what we have in our code.

* Now open your Postman, create a **POST** request to the API **http://PublicIP-or-PublicDNS:5000/api/todos**. This request sends a new task to our To-Do list so the application could store it in the database.

* Create a New Request <br> 
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/getStarted.png) <br>

* **Note**: make sure your set header key **Content-Type** as **application/json**
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/headers.png)<br>

* We make sure we have the right **endpoint** (next to POST), an **action** in the **Body** and Click **Send** <br>
Here we are storing/posting string **Finish Projects 1 - 30** in the Database using HTTP Request Method **POST**
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/postBody.png) <br>

* Now we will retrieve the contents of Database using HTTP Request Method **GET**<br>
The result is the string that we previously sent with an ID "**611b03d9998cb70389c86882**" <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/get.png) <br>

* Now we will delete the entry by using the ID "**611b03d9998cb70389c86882**" previously retrieved with HTTP Request Method **DELETE** <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/delete.png) <br>

* To double check the entry was deleted we use **GET** again to get the information and we see that nothing is there <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/getCheck.png) <br>

* With this successful test we know our applicatin backend is working

# STEP2: FRONTEND CREATION
---

It is time to create a user interface for a Web client (browser) to interact with the application via API. To start out with the frontend of the To-do app, we will use the **create-react-app** command to scaffold our app.
In the same root directory as your backend code (Todo) run:
 ```bash
 npx create-react-app client
```
This will create a new folder in **Todo** directory called **client**, where all the **react** code will be added.

**Running a React App**

* We'll install concurrently to run more than one command simultaneously from the same terminal window.
```bash
npm install concurrently --save-dev
```
* We'll install nodemon to run and monitor the server. If there is any change in the server code, nodemon will restart it automatically and load the new changes.
```bash
npm install nodemon --save-dev
```
* In directory **Todo** we need to edit file  **package.json** by replacing code block **"scripts":{...},** with
```bash
 "scripts": {
    "start": "node index.js",
    "start-watch": "nodemon index.js",
    "dev": "concurrently \"npm run start-watch\" \"cd client && npm start\""
  },
```

**Configure Proxy in package.json**

* Change directory to **client** and edit file **package.json** by adding the key value pair 
```JSON
"proxy": "http://localhost:5000"
```
```bash
#I added mine at the end
    ]
  },
  "proxy":"http://localhost:5000"
}
```

The purpose of adding the proxy configuration above is to make it possible to access the application directly from the browser by simply calling the server url like http://localhost:5000 rather than always including the entire path like http://localhost:5000/api/todos

Now, from inside the **Todo** directory run
```bash
npm run dev
```
The app should open and start running on localhost:3000 (keep in mind I already opened port **3000** along with **5000**)

**Creating React Components** <br>
* Components are reusable and also make code modular. For our Todo app, there will be two stateful components and one stateless component.

* Inside **src** folder I'll create another folder called **components**.
Inside **components** I'll create three files **Input.js**, **ListTodo.js** and **Todo.js**.
```bash
ubuntu@ip-172-31-59-251:~/Todo/client/src$ mkdir components
ubuntu@ip-172-31-59-251:~/Todo/client/src$ cd components/
ubuntu@ip-172-31-59-251:~/Todo/client/src/components$ touch Input.js ListTodo.js Todo.js
ubuntu@ip-172-31-59-251:~/Todo/client/src/components$ ls
Input.js  ListTodo.js  Todo.js
```

* I'll put the following code inside of **Input.js**
```Javascript
import React, { Component } from 'react';
import axios from 'axios';

class Input extends Component {

state = {
action: ""
}

addTodo = () => {
const task = {action: this.state.action}

    if(task.action && task.action.length > 0){
      axios.post('/api/todos', task)
        .then(res => {
          if(res.data){
            this.props.getTodos();
            this.setState({action: ""})
          }
        })
        .catch(err => console.log(err))
    }else {
      console.log('input field required')
    }

}

handleChange = (e) => {
this.setState({
action: e.target.value
})
}

render() {
let { action } = this.state;
return (
<div>
<input type="text" onChange={this.handleChange} value={action} />
<button onClick={this.addTodo}>add todo</button>
</div>
)
}
}


export default Input
```
* Now I will install **Axios** (a Promise based HTTP client for the browser and node.js). From the **src** directory I will run
```bash
npm install axios
```

* I'll put the following code inside of **ListTodo.js**
```Javascript
import React from 'react';
const ListTodo = ({ todos, deleteTodo }) => {
return (
<ul>
{
todos &&
todos.length > 0 ?
(
todos.map(todo => {
return (
<li key={todo._id} onClick={() => deleteTodo(todo._id)}>{todo.action}</li>
)
})
)
:
(
<li>No todo(s) left</li>
)
}
</ul>
)
}

export default ListTodo
```

* I'll put the following code inside of **Todo.js**
```Javascript
import React, {Component} from 'react';
import axios from 'axios';
import Input from './Input';
import ListTodo from './ListTodo';
class Todo extends Component {
state = {
todos: []
}
componentDidMount(){
this.getTodos();
}
getTodos = () => {
axios.get('/api/todos')
.then(res => {
if(res.data){
this.setState({
todos: res.data
})
}
})
.catch(err => console.log(err))
}
deleteTodo = (id) => {
axios.delete(`/api/todos/${id}`)
      .then(res => {
        if(res.data){
          this.getTodos()
        }
      })
      .catch(err => console.log(err))
}
render() {
let { todos } = this.state;
return(
      <div>
        <h1>My Todo(s)</h1>
        <Input getTodos={this.getTodos}/>
        <ListTodo todos={todos} deleteTodo={this.deleteTodo}/>
      </div>
    )
}
}
export default Todo;
```

* I'll put the following code inside of **App.js** located in **src**
```Javascript
import React from 'react';
import Todo from './components/Todo';
import './App.css';
const App = () => {
return (
<div className="App">
<Todo />
</div>
);
}
export default App;
```
* I'll put the following code inside of **App.css** located in **src**
```css
.App {
text-align: center;
font-size: calc(10px + 2vmin);
width: 60%;
margin-left: auto;
margin-right: auto;
}
input {
height: 40px;
width: 50%;
border: none;
border-bottom: 2px #101113 solid;
background: none;
font-size: 1.5rem;
color: #787a80;
}
input:focus {
outline: none;
}
button {
width: 25%;
height: 45px;
border: none;
margin-left: 10px;
font-size: 25px;
background: #101113;
border-radius: 5px;
color: #787a80;
cursor: pointer;
}
button:focus {
outline: none;
}
ul {
list-style: none;
text-align: left;
padding: 15px;
background: #171a1f;
border-radius: 5px;
}
li {
padding: 15px;
font-size: 1.5rem;
margin-bottom: 15px;
background: #282c34;
border-radius: 5px;
overflow-wrap: break-word;
cursor: pointer;
}
@media only screen and (min-width: 300px) {
.App {
width: 80%;
}
input {
width: 100%
}
button {
width: 100%;
margin-top: 15px;
margin-left: 0;
}
}
@media only screen and (min-width: 640px) {
.App {
width: 60%;
}
input {
width: 50%;
}
button {
width: 30%;
margin-left: 10px;
margin-top: 0;
}
}
```

* I'll put the following code inside of **index.css** located in **src**
```css
body {
margin: 0;
padding: 0;
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto", "Oxygen",
"Ubuntu", "Cantarell", "Fira Sans", "Droid Sans", "Helvetica Neue",
sans-serif;
-webkit-font-smoothing: antialiased;
-moz-osx-font-smoothing: grayscale;
box-sizing: border-box;
background-color: #282c34;
color: #787a80;
}
code {
font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New",
monospace;
}
```

* From directory **Todo** run
```bash
npm run dev
```
* Assuming there are no errors you can now access the application from your browser at PublicIP:3000
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/myTodo1.png) <br>
After playing around a little <br>
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/MERN_STACK/main/images/myTodo.png) <br>
