---
date: '2025-01-22T13:00:00+01:00'
draft: false
title: 'Generic Adapters for MongoDB: A Go Solution Inspired by Rust'
author: "Jimmy"
tags: ["Go", "Rust", "Generics"]
params:
  ShowCodeCopyButtons: true
cover:
    image: "images/mongodb-go.png" # image path/url
    alt: "Go MongoDB Cover" # alt text
    #caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page

---
I'm working several years with Go and MongoDB and decided once, that I write some Rust code with the [official mongo-rust-driver](https://github.com/mongodb/mongo-rust-driver) to test new things. 
I wouldn't consider myself as a `Rustacean`, but I do understand the basic principle of the borrowing, ownership and lifetimes in Rust. After reviewing the mongo-rust-driver docs, I started with the basic setup.

```rust
let db = Client::with_uri_str(mongodb_uri).await?.database("my_database")
```


After connecting it, I want to try out how to perform CRUD (Create, Read, Update and Delete) operations. To perform that, I have to create a collection. So I checked the docs again and saw the following:

```rust
mongodb::db::Database
pub fn collection<T>(&self, name: &str) -> Collection<T>
where
    T: Send + Sync,
```

&nbsp; 

![WHOA!](/images/whoa-spit-out-drink.gif "WHOA!")

&nbsp;

#### What does this mean?
Let us breakdown this Rust function:

1. This function is part of the `Database` struct in the `mongodb::db` module.
2. `pub fn collection<T>` is declaring a public function with a generic type parameter `T`
3. `(&self, name: &str)` are function parameters,
    - `&self` is a reference to the `Database` instance on which the method is called.
    - `name: &str` is a string slice parameter with the name of the collection.
4. `-> Collection<T>:` specifies the return type `Collection<T>` where `T`is the generic type parameter.
5. `where T: Send + Sync,:` is a trait bound to the generic type, which requires it that `T` has to implement the [`Send` and `Sync`](https://doc.rust-lang.org/nomicon/send-and-sync.html) trait.

In the end it is possible to infer a created type and create a collection with the following code:

```rust
let my_collection = db.collection::<model::MyStruct>("my_collection");
```

#### So why I'm so excited?
Let us have a look at the Go implementation for creating a collection. 

```go
// Collection gets a handle for a collection with the given name configured with the given CollectionOptions.
func (db *Database) Collection(name string, opts ...*options.CollectionOptions) *Collection {
	return newCollection(db, name, opts...)
}
```

Let us breakdown this Go function:
1. This function is part of the `Database` struct in the `mongo` package.
2. `func (db *Database) Collection` is declaring a public function due to the capitalized first character.
3. `(name string, opts ...*options.CollectionOptions)` are function parameters.
    - `name string` is a string parameter with the name of the collection.
    - `opts ...*options.CollectionOptions` is a parameter that represents options that can be used to configure a Collection.
4. The function returns a reference of the `*Collection` instance.

So, the main difference, from the perspective of my usecase, is that when you create an `Collection` instance you can't infer a type in Go. This leads for a single read operation to the following code in Go:

<br><br/>  

```Go
    db := client.Database("my_database").Collection("my_collection")
	var data MyStruct
	err := db.FindOne(ctx, filter).Decode(&data)
	if err != nil {
        return errors.New("custom error")
	}
```

<br><br/>  

The same operation does look like the following in Rust:

```Rust
let result = my_collection.find_one(filter).await?.ok_or(Error::CustomError)?;
```
 
  
&nbsp; 

See the difference? No, not the one-liner and the error handling. You don't have to decode your operation result manually like in Go, instead, you're using the power of type inference in Rust which will decode it for you under the hood. 
I decided to write a small wrapper around the Go MongoDB driver, which allows me to use it almost the same way as the [official mongo-rust-driver](https://github.com/mongodb/mongo-rust-driver). For that I created a struct type named `GenericMongoAdapter` with a generic type `T`. `[T any]` indicates that this is a generic type with a type parameter `T`. The `any` constraint means that `T` can be any type.

&nbsp; 

```Go
type GenericMongoAdapter[T any] struct {
	collection *mongo.Collection
}

func NewGenericMongoAdapter[T any](collection *mongo.Collection) GenericMongoAdapter[T] {
	return GenericMongoAdapter[T]{collection: collection}
}

```

&nbsp; 


Also I created a function which returns an instance of `GenericMongoAdapter`. After that it was easy to write down the CRUD operations for it:


```Go
func (g GenericMongoAdapter[T]) Create(ctx context.Context, data T) (primitive.ObjectID, error) {
	result, err := g.collection.InsertOne(ctx, data)
	if err != nil {
		return nil, err
	}

	return result.InsertedID.(primitive.ObjectID), nil
}

func (g GenericMongoAdapter[T]) Read(ctx context.Context, query bson.M) (*T, error) {
	var data *T
	err := g.collection.FindOne(ctx, query).Decode(&data)
	if err != nil {
		return nil, err
	}

	return data, nil
}

func (g GenericMongoAdapter[T]) Update(ctx context.Context, query bson.M, updateObject bson.M, opts ...*options.UpdateOptions) error {
	result, err := g.collection.UpdateOne(ctx,
		query,
		updateObject,
		opts...,
	)
	if err != nil {
		return nil, err
	}

	return nil
}

func (g GenericMongoAdapter[T]) Delete(ctx context.Context, query bson.M, opts ...*options.DeleteOptions) error {
	result, err := g.collection.DeleteOne(ctx, query, opts...)
	if err != nil {
		return nil, err
	}

	return nil
}
```

After writing down these generic functions, let's have a look on how to use them.

1. First of all we are creating a MongoDB client instance, for this we have to connect to the database.

```Go
	client, err := mongo.Connect(ctx, options.Client().ApplyURI(url))
	if err != nil {
		return nil, err
	}
	err = client.Ping(ctx, readpref.Primary())
	if err != nil {
		return nil, err
	}
```

2. Afterwards we are creating an instance of the collection we want to use.

```Go
myCollection := client.Database("my_database").Collection("my_collection")
```

3. Thirdly, we are creating an instance of our `GenericMongoAdapter`. For this I'll also define a struct.

```Go
type MyStruct struct {
	ID        primitive.ObjectID `bson:"_id"`
	MyItems   []string           `bson:"my_items"`
}

myStructAdapter := genericAdapter.NewGenericMongoAdapter[MyStruct](myCollection)

```

4. Lastly, we perform a read operation.

```Go
result, err := myStructAdapter.Read(ctx, bson.M{"_id": id})
if err != nil {
    return nil, err
}
```

Now if we compare the read operation with the Rust equivalent.

```Rust
let result = my_collection.find_one(filter).await?.ok_or(Error::CustomError)?;
```
 
From my point of view it is almost the same. And it saves you the boilerplate code with custom decoding in Go.

### Summary
First of all both languages are having the right to exist and having their ups and downs. My intention was to show how to port the developer experience, which I faced in Rust, with the [mongo-rust-driver](https://github.com/mongodb/mongo-rust-driver)
in Go with the [mongo-go-driver](https://github.com/mongodb/mongo-go-driver). My pain point was, the missing type inference for decoding and in terms of scaling it could lead to much boilerplate code. 

