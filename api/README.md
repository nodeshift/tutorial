# Building a CRUD REST API with Node.js

## Introduction

Building a microservice which provides a [REST API](https://www.redhat.com/en/topics/api/what-is-a-rest-api) is a common task as a Node.js developer. In this tutorial, we will take you through the basic steps of creating an API that implements the four base operations of persistent storage: create, read, update, and delete ([CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)). We will do this manually so that you learn the basics of what is going on behind the scenes.

When developing APIs for larger enterprise applications you will often want to use OpenAPI and an API-first approach. You can lean more about that in [Building a Node.js service using the API-first approach](https://developers.redhat.com/blog/2019/01/14/building-a-node-js-service-using-the-api-first-approach).

## Installing Express

The following steps cover creating a base Express.js application. Express.js is a popular web server framework for Node.js.

1. Create a directory to host your project:

   ```sh
     mkdir node-crud-api
     cd node-crud-api
   ```

2. Initialize your project with `npm` and install the Express.js module:

   ```sh
     npm init --yes
     npm install express
   ```

3. It is important to add effective logging to your Node.js applications to facilitate observability, that is to help you understand what is happening in your application. The [NodeShift Reference Architecture for Node.js](https://github.com/nodeshift/nodejs-reference-architecture/blob/main/docs/operations/logging.md) recommends using Pino, a JSON-based logger.

   Installing Pino:

   ```sh
     npm install pino
   ```

4. Now, let's start creating our server. Create a file named `server.js`:

   ```sh
   touch server.js
   ```

5. Add the following to `server.js` to produce an Express.js server that responds on the `/` route with 'Hello, World!'.

   ```js
   const express = require('express');
   const pino = require('pino')();
   const app = express();
   const PORT = process.env.PORT || 3000;

   app.get('/', (req, res) => {
     res.send('Hello, World!');
   });

   app.listen(PORT, () => {
     pino.info(`Server listening on port ${PORT}`);
   });
   ```

Start the application by running `node server.js` and then navigate to http://localhost:3000. You should see the server respond with 'Hello, World!'. You can stop your server by entering CTRL + C in your terminal window.

## Defining Routes

A REST API provides endpoints for the basic CRUD operations. In this section we are going to define these endpoints by adding a routes. CRUD operations are often mapped to HTTP operations as follows:

* create -> post
* read -> get
* put -> update
* delete -> delete

We will add an implementation for each of these.

Create the required CRUD API routes using the following steps:

1. Create a new file called `routes.js` and initialize a new express router:

   ```js
   const express = require('express');

   const router = express.Router();
   ```

2. Register the CRUD API routes to the router:

   ```js
   router.get('/todos', (req, res) => {
     res.json({
       message: 'Not yet implemented!'
     });
   });

   router.post('/todos', (req, res) => {
     res.json({
       message: 'Not yet implemented!'
     });
   });

   router.put('/todos/:id', (req, res) => {
     res.json({
       message: 'Not yet implemented!'
     });
   });

   router.delete('/todos/:id', (req, res) => {
     res.json({
       message: 'Not yet implemented!'
       
     });
   });

   router.delete('/todos', (req, res) => {
     res.json({
       message: 'Not yet implemented!'
     });
   });
   ```

3. Default export the created express router:

   ```js
   module.exports = router;
   ```

4. Install two additional npm packages needed for handling cors and parsing the request's body:

   ```sh
   npm install body-parser cors
   ```

   and then, register the packages as express middleware right before the `/` route declaration in the `server.js` file:

   ```js
   const cors = require('cors');
   const bodyParser = require('body-parser');

   ...

   app.use(cors());
   app.use(bodyParser.json());
   app.use(bodyParser.urlencoded({ extended: true }));

   ...
   ```

   Note: If you want to learn more about CORS (Cross Origin Resource Sharing) take a look at <a href="https://en.wikipedia.org/wiki/Cross-origin_resource_sharing">here</a>.

5. Register the exported router as an express route right after the middleware registration in the `server.js` file:

   ```js
   const apiRoutes = require('./routes');

   ...

   app.use('/api', apiRoutes);

   ...
   ```

The `server.js` should look like this:

```js
const express = require('express');
const pino = require('pino')();
const app = express();
const cors = require('cors');
const bodyParser = require('body-parser');

const apiRoutes = require('./routes');

const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

app.use('/api', apiRoutes);

app.get('/', (req, res) => {
  res.send('Hello, World!');
});

app.listen(PORT, () => {
  pino.info(`Server listening on port ${PORT}`);
});
```

Restart the server, navigate to http://localhost:3000/api/todos and you should see the server response `{ message: 'Not yet implemented!' }` presented in JSON format.

## Data persistence logic

The persistence layer is most often provided by a database. For our simple example we are going to avoid adding the complexity of configuring a database and simply store items in memory. We will update each the functions mapped into each of the CRUD operations through routes so that they update the in-memory array of items.

Create a persistent layer for the todos and update the express routes following these steps:

1. Create a global `todos` array inside `routes.js`:

   ```js
   let todos = []; // this is our in-memory persistence layer
   ```

2. Import `pino` as the logger:

   ```js
   const pino = require('pino')();
   ```

3. Implement the `GET /todos` route:

   ```js
   router.get('/todos', (req, res) => {
     res.json(todos);
   });
   ```

4. Implement the `POST /todos` route:

   ```js
   router.post('/todos', (req, res) => {
     const author = req.body.author;
     const task = req.body.task;

     if (!task || !author) {
       res.status(422).send("Unprocessable Entity");
       return;
     }

     const id = todos.length;

     const todo = {
       id,
       author: author,
       task: task
     };

     todos.push(todo);

     res.json({
       success: true,
       message: 'Todo successfully added!',
     });
   });
   ```

5. Implement the `PUT /todos/:id` route:

   ```js
   router.put('/todos/:id', (req, res) => {
     const { id } = req.params;
     const task = req.body.task;

     pino.info(`Attempting to update a todo by ID (${id})`);
     todos = todos.map((t) => t.id === id ? task : t);

     res.json({
       success: true,
       message: 'Updated todo ${id}'
     });
   });
   ```

6. Implement the `DELETE /todos/:id` route:

   ```js
   router.delete('/todos/:id', (req, res) => {
     const { id } = req.params;
     todos = todos.filter((t) => t.id !== id);

     res.json({
       success: true,
       message: 'Todo has been deleted'
     });
   });
   ```

7. Implement the `DELETE /todos` route:

   ```js
   router.delete('/todos', (req, res) => {
     todos = [];

     res.json({
       success: true,
       message: 'Deleted todos data'
     });
   });
   ```

Starting the server using `node server.js` and using cURL we can test our API:

1. Create a new todo:

   ```sh
   curl -X POST http://localhost:3000/api/todos -H "Content-Type: application/json" -d '{"author": "nodeshift", "task": "Learn Node.js"}'
   ```

2. Get all the todos from the server:

   ```sh
   curl http://localhost:3000/api/todos
   ```

   ```
   [
     {
       "id": 0,
       "author": "nodeshift",
       "task": "Learn Node.js"
     }
   ]
   ```

## Validating inputs

Any input that is passed to the API should be validated before it is used. This is important
because using invalid inputs within your application could lead to failures and even worse
may be crafted by an attacker to cause your application to do something unintended or return
more information than you planned. A good example of this is
[SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection).

Try running:

```shell
curl -v -X POST http://localhost:3000/api/todos -H "Content-Type: application/json" -d '{"author": "nodeshift", "tasaxk": "Learn Node.js"}'

```

You'll notice that `task` is mispelled as `tasaxk` and from the output you can see that the request was rejected:

```shell
< HTTP/1.1 422 Unprocessable Entity
```

That is because we had added validation in the post method:

```
  const author = req.body.author;
  const task = req.body.task;

  if (!task || !author) {
    res.status(422).send("Unprocessable Entity");
    return;
  }
```

This was straight forward because the input was simple. For more complex APIs you may want to use
a JSON validator such as [ajv](https://www.npmjs.com/package/ajv). It is also something that using OpenAPI and API-first tools help with.

## Hardening headers with Helmet

It's important to make any APIs that you expose as safe/secure as possible. In larger enterprise deployments this security may be provided by an API gateway like [Red Hat 3scale API Management](https://www.redhat.com/en/technologies/jboss-middleware/3scale). When that is not the case it's wise to use a package like Helmet to secure your Node.js application from some obvious threats. You can use Helmet to safeguard your application from usual security risks like XSS, Content Security Policy, and others.

In this section we'll show you how to add helmet to your application.

Enable the `helmet` express middleware by following the steps:

1. Install helmet plugin with NPM:

   ```sh
   npm install helmet
   ```

2. Register helmet as the **first** middleware in `server.js`:

   ```js
   const helmet = require('helmet');

   ...

   app.use(helmet());
   ```

By using the `helmet()` middleware like this we are enabling the following helmet <a href="https://github.com/helmetjs/helmet#reference">options</a>:

- contentSecurityPolicy
- dnsPrefetchControl
- expectCt
- frameguard
- hidePoweredBy
- hsts
- ieNoOpen
- noSniff
- permittedCrossDomainPolicies
- referrerPolicy
- xssFilter

At this point you have a REST API which supports the basic CRUD operations and is protected from some of typical threats.

## Wrap up and next steps

This tutorial provided you with a basic introduction to creating a REST API that implements the four CRUD operations and you should understand the underlying concepts.

If you have not already read some of the references we provided along the way now would be a great time to either go back in order to get a deeper level of understanding of some of the concepts or note them down to check out later.
