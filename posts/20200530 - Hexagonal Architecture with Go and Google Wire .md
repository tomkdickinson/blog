As someone who has recently started working with Go, I absolutely love it. It’s a language which is built for the web 
and has a lot of features and libraries built in.

However, when learning any new language it is really easy to generate a big ball of mud for your first few projects. 
For example, when I first started writing Angular.io applications with TypeScript, our code base was an absolute mess 
although in our defense there were few examples of good architecture at the time as the framework had only been around 
for 6 months.

To help avoid the inaugural mess that any new Go developer will make, in this post I want to share how I’ve been 
developing my Go projects recently using the Hexagonal Architecture and Google Wire as a dependency injection tool.


## Hexagonal Architecture

Hexagonal architecture is a popular pattern that’s normally associated with Domain Driven Design (at least that’s the 
first time I explicitly came across it). It goes by other names as well, such as the onion architecture (due to it’s 
layered nature), and Ports and Adapters (which is a more functional description).

A quick google with show many posts that will explain it far better than I can, and in more detail, but as a quick 
overview take a look at the following diagram.

![hexagonal-example](https://tomkdickinson-blog.s3.eu-west-2.amazonaws.com/hexagonal-architecture.png)

The general idea is to separate your interaction logic from your domain logic. For example, let’s say you have 
implemented this pattern and are currently using Mongo as your storage option but find your data has now become 
heavily relational. Without having to modify your domain logic, you can create a new Postgres implementation of your 
repository interface without worrying about touching the underlying business logic. The benefits of this mean fewer code
 changes, leading to a reduction in risk of bugs.

There is a also a benefit to testing. While we have unit tests to structure, document, and test our code, we also want 
to be able to black box test at the service boundary. Whether it’s a monalith, microservice, or serverless function,
 there is a cost and technical complexity to setting up dependent infrastructure like databases. However, with this 
 pattern, you can implement your own mock repository to run service level tests.

Let’s consider the scenario that when you make an API call to your service, the business logic executes and an event is
sourced on Kafka. One approach (which I’ve tried) is to use an actual implementation of Kafka, and have your service 
tests consume messages from it. It works, but it’s slow, brittle, and ties your tests directly into Kafka. Integration 
tests are important, but you don’t want to necessarily couple it to testing the underlying business logic. As a user I
normally don’t care about the infrastructure, just the behaviour.

If you follow the hexagonal architecture pattern however, you could implement a new version of your messaging interface 
and have it save all messages to a file. Then all your test has to do is read a json file from disk to see if a message 
has been sent.

You can still have tests that target Kafka, but these can be ran as a handful of small end-to-end tests across your 
deployed system. 


## Google Wire

Next I want to introduce Google Wire which is a compile time dependency injection framework for Go. By compile time, we 
mean that you run Wire against your source code to generate more native Go code derived from some configuration you set 
up.

The advantage to this approach is nothing is hidden, and better yet the code that is generated is Go code you would
 probably write yourself (no Dreamweaver shenanigans here!).

To give a very simple overview, Wire has the concept of Providers and Injectors. A Provider is essentially your factory 
method to create a new object. Sometimes it might be as simple as returning an empty struct, other times you might want
 to do something slightly more complex and have the option to return an error under certain conditions.

If you want to know a bit more, I’d highly recommend reading their docs: 
[https://github.com/google/wire/blob/master/docs/guide.md](https://github.com/google/wire/blob/master/docs/guide.md). 
They are not very long, and the explanations are pretty easy to grasp.

As of time when writing this post, Wire is still in beta (v0.4.0), but has been considered feature complete since 
v0.3.0. 

## Putting them together

Now we’ve covered the architecture pattern and a DI framework for Go, let’s take a look at how we might build an 
application.

The repository for this example can be found here: https://github.com/tomkdickinson/hexagonal-cart-service. The README 
file details how to start the service, and I’ve also included a docker compose file which I’ll refer to for the rest of
 this post. Feel free to run it however you want though.

To keep things nice and simple, this is a very simple ecommerce carts API service. It’s functionality comprises of:

* Adding items to the cart with a quantity
* View the items in the cart
* Persist the cart

Proper REST patterns have not been followed for this example, so when updating your cart you’ll use a GET request rather
 than a POST or PUT. The benefit of this is you can just use a browser to run the examples.

The service also has two options for storage; a Mongo repository, and a File repository which persists the data on disk 
as a JSON file.

By default it will use the Mongo repository, but you can switch to the file repository using the following environment 
variable:

```
CART_STORAGE=file
```

## The Folder Structure

This project uses the folder layout specified in https://github.com/golang-standards/project-layout. The only two 
folders that are used in this project though are cmd and internal. The former contains our main package, while the 
latter is our internal private packages specific to this project.

### The *internal* folder

Inside internal you will see that we have three packages: api, cart, and storage.

The api package contains our input into the application, or specifically our HTTP handlers. It’s responsibility is to 
handle incoming requests, call the cart domain logic, and return a suitable reply or error.

The cart package is the main business logic of the application. We have a `Service` struct that handles our business 
logic on a `Cart` model, and a `Repository` interface to persist our cart. There is no implementation of the repository 
interface  in this package, as our cart logic shouldn’t care (or even know) how the repository is implemented.

Instead, the storage package contains our different implementations for persisting to Mongo, and to disk with respective
mongo and file packages. 

### The *cmd* folder

If we now take a look in our **cmd** folder, we have our **app** application with three files: `main.go`, 
`inject_app.go`, and `wire_gen.go`.

`inject_main.go` is the wiring for our application. We provide two injectors:

```go
func handlersWithMongoStorage() (*api.Handlers, error) {
	panic(
		wire.Build(
			mongo.ProvideCollection,
			mongo.ProvideRepository,
			wire.Bind(new(cart.Repository), new(mongo.Repository)),
			cart.ProvideService,
			api.ProvideHandlers,
		))
}

func handlersWithFileStorage() (*api.Handlers, error) {
	panic(
		wire.Build(
			file.ProvideRepository,
			wire.Bind(new(cart.Repository), new(file.Repository)),
			cart.ProvideService,
			api.ProvideHandlers,
		))
}
```

Our first one builds our api handlers injected with a mongo implementation of the cart repository, while the second does
 it with the file implementation.

`wire_gen.go` is the code generated from running wire with the `inject_main.go` code. To generate the file you will need to 
first install Wire (just follow their repository instructions). Then, at the project root, you can either run wire 
[github.com/tomkdickinson/hexagonal-cart-service/cmd/app](github.com/tomkdickinson/hexagonal-cart-service/cmd/app) or 
`go generate ./...`

As you can see, the output from wire is pretty standard sensible Go code.

```go
func handlersWithMongoStorage() (*api.Handlers, error) {
	collection, err := mongo.ProvideCollection()
	if err != nil {
		return nil, err
	}
	repository := mongo.ProvideRepository(collection)
	service := cart.ProvideService(repository)
	handlers := api.ProvideHandlers(service)
	return handlers, nil
}

func handlersWithFileStorage() (*api.Handlers, error) {
	repository := file.ProvideRepository()
	service := cart.ProvideService(repository)
	handlers := api.ProvideHandlers(service)
	return handlers, nil
}
```

`main.go` is our application, which binds URL paths to their associative handler and starts the server. We also have the
following conditional logic on an environment variable:

```go
	switch os.Getenv("CART_STORAGE") {
	case "file":
		log.Println("Starting server with file repository")
		handlers, err = handlersWithFileStorage()
	default:
		log.Println("Starting server with mongo repository")
		handlers, err = handlersWithMongoStorage()
	}

	if err != nil {
		log.Fatal(fmt.Errorf("could not wire application: %w", err))
	}
```

This is where our app uses the injectors built from our `inject_main.go` file to switch between our two different 
implementations of the Cart repository.

### Seeing it in Action

Let’s start the service with:

```shell script
docker-compose up --build
```

This will build the services Dockerfile and start the server on port 8000 with a running Mongo instance. To interact 
with the cart you can use the following urls:

```
http://localhost:8000/cart/<cart_id>
http://localhost:8000/cart/<cart_id>/<item_id>/<quantity>
```

So to see an empty cart with id my-first-cart you can make the following call:

[http://localhost:8000/cart/my-first-cart](http://localhost:8000/cart/my-first-cart)

Next, try adding 20 pairs of jeans to your cart with the following:

[http://localhost:8000/cart/my-first-cart/a-pair-of-jeans/20](http://localhost:8000/cart/my-first-cart/a-pair-of-jeans/20)

To check if it’s being persisted in Mongo, feel free to to use a Mongo client, and check database carts and collection 
cart. The _id of the document is the cart id you use.

Ok, let’s see if we can see the file storage in action. Terminate the docker compose command you were running, or run
docker-compose down if in detached mode, and then lets remove Mongo from our docker-compose.yml file. Change it so it 
looks like this:

```yaml
version: "3.7"
services:
  cart-service:
    build: .
    container_name: hexagonal-cart-service
    ports:
      - "8000:8000"
    environment:
      - MONGO_DSN=mongodb://mongo:27017
```

Mongo has now been removed, but if you run `docker-compose up`, you’ll notice that the service is unresponsive as we are 
still using the Mongo repository interface without a running Mongo instance. If it’s not unresponsive, you may find you 
still have an accessible instance of Mongo running in the background.

Let’s set the service to use our file repository instead.

```yaml
version: "3.7"
services:
  cart-service:
    build: .
    container_name: hexagonal-cart-service
    ports:
      - "8000:8000"
    environment:
      - CART_STORAGE=file
```

Now you should see the service becomes responsive again, and you have the exact same behaviour as before. If you want to
see it persisting file changes to disk, you can connect to the container with:

```shell script
docker exec -it hexagonal-cart-service /bin/sh
```

Then when connected to the container you can run cat `carts_storage.json` to view the current state of the database. 
Each time you mutate your cart with an API call, `cart_storage.json` will update with the current state of all carts.

In a production environment, this would be terrible, but for running functional tests? The ability to see a JSON 
snapshot of the database in each scenario is far easier to test against than performing Mongo queries that might have 
to be re-written when you decide to move to Postgres.

## Conclusion

In this post, I’ve hopefully show how you can set up a Go project using a Hexagonal Architecture, and Google Wire as a dependency injection tool.

There are a few benefits of using this pattern:

* By separating your domain logic from your infrastructure logic, you can easily switch infrastructure with minimal 
impact on what the application does
* We can create repository mocks for functional tests that are easier to interact with than the underlying 
infrastructure
* As your project gets bigger and bigger, Google Wire makes it far easier to maintain and switch between different
 infrastructure implementations

In a follow up post, I’ll demonstrate the second point in more detail, showing how you can use something like Cucumber 
to easily test your service, but only focusing on the domain logic.