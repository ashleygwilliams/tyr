#tyr: test your REST
a [`node`](http://nodejs.org/) test generator for API endpoints using [`mocha`](http://mochajs.org/) and [`superagent`](https://github.com/visionmedia/superagent)

tyr is a test generator developed out of the frustration of writing redundant endpoint tests for APIs. if we build using conventions we should be able to use those conventions to mitigate redundancy in our applications, as well as share resources, like tests! assuming your routes are resource based and follow a convention like [json-api](http://www.json-api.com), tyr will generate a series of mocha/superagent endpoint tests for your API.

examples IRL:
- [piep/piep-api](https://github.com/piep/piep-api/tree/master/test)
- [endpoints/example](https://github.com/endpoints/example/tree/master/test)

# what is it
tyr is comprised of 4 elements: 

- `test_template.js`: the test suite template

  this contains [`superagent`](https://github.com/visionmedia/superagent) tests for (in order):
    - `POST /collection`: create an instance of a resource
    - `GET /element/:id`: retrieve an instance of a resource
    - `GET /collection`: retrieve a collection of resources
    - `PUT /element/:id`: update an instance of a resource
    - `DELETE /element/:id`: delete an instance of a resource
  
- `tyr.js`: the test builder
  
  a node script that builds a test file for each resource in the `resources` array in `config.js` using the `test_template.js`. each test file is deposited in the same location as `tyr.js` and is named `<resource_name>_test.js`.

- `config.js`: array of resources to generate tests for, api-server host, port, namespace
- `/mocks`: a folder containing a mocks object for every resource to be tested
- `db_utils.js`: a utility for reseting your db, requires a database `config` file and a `knexfile`

# let's do this

## step 1: build your mocks

in order to test a `POST` and a `PUT`, you will need 2 objects, respectively. when you create a resource for your application, create a file in `/mocks` with the name of the resource. inside this file, export an object with two attributes: `mock_resource` and `mock_update`. these will be required in `tyr.js` and each assigned, respectively, to variables of the same name.

for example:

```js
// /mocks/books.js

var mock_resource = {
  author_id: 1,
  title: "Spacecats! An astronomical tutorial on building single page web applications with AngularJS",
  date_published: "jan 1 2015"
};
var mock_update = {
  author_id: 1,
  title: "Spacecats! An astronomical tutorial on building single page web applications with AngularJS",
  date_published: "jan 30 2015"
};


exports.mock_resource = mock_resource;
exports.mock_update = mock_update;
```

## step 2: configure your test suite

the `config.js` file in tyr is where you will specify the `host`, `port`, and `namespace` for your API server. 

additionally, and most importantly, this is where you list your resources. you should list the path to your resources as strings in the `resources` array. tyr assumes that the path points to a folder named after the resource that contains a `model.js` file.

be sure to add your resources in the correct order according to their relations. this is  critical, as most APIs built on relational databases define the related `resource_id` attribute on a resource as non-nullable. if you put them in the wrong order, and you have related resources, the tests should fail immediately, as the `POST` in the `beforeEach` of the tests will fail.

an example:

```js
// /test/config.js

module.exports = {
  host: 'http://0.0.0.0',
  port: ':8080',
  namespace: '/api/1/',

  resources: ['authors', 'books', 'chapters', 'series']
};
```

because a `book` resource requires an `author_id`, if books were listed first, the `POST` would fail because no `author` resources would exist.

## step 3: hook up your DB

in order for the tests to be accurate, it's best we reset our DB before each test. `db_utils.js` has a utility function for doing just this. 

this utility, as it is written in the included file, assumes that you are using [`knex`](http://knexjs.org/) and [`bookshelf`](http://bookshelfjs.org/). at the top of the file, point the constant `DB` to a `require` of your bookshelf config file, and point the constant `config` to a `require` of your `knexfile`.

if you are using something different for SQL queries and/or an ORM, simply rewrite the file to export a `reset()` function, as this is all that tyr anticipates. for a closer look: the util is required on [line 15 of `test_template.js`](https://github.com/ashleygwilliams/tyr/blob/master/test_template.js#L5) and the `reset` function is used on [line 18 of `test_template.js`](https://github.com/ashleygwilliams/tyr/blob/master/test_template.js#L18).

## step 4: build and run those tests

you're all done! step 1, 2, and 3 are all that's required to get a base set of endpoints tests for each resource in your API.

- build the tests: `node tyr.js`
- run the tests: `mocha <resource_name>_test.js`

we recommend that you automate the running of all the tests so that you don't have to manually run each. however, we *do* believe that having generated each suite individually is important. you may want to debug a single resource and having to run a whole set of tests each time can be a pain. additionally, now you can reuse your models/endpoints and have a test suite that's portable :)
