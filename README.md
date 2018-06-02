![](https://thepracticaldev.s3.amazonaws.com/i/q2skcobs31lu94rj80s3.png)

MongoDB has become one of the most popular noSQL databases. It is often used as a part of the MEAN/MERN stack because it is so easy to fit in the JavaScript ecosystem.
There are hundreds of tutorials on the Internet, tons of courses and some books about how to become a full-stack developer using MongoDB as the database system in the stack (The M in MERN/MEAN).
The problem is that the most of them don't focus on MongoDB schema design patterns. So that, operations/queries over designed schemas have so bad performance and/or don't scale.

One of the main problems you have to face on designing a MongoDB schema is how to model "One-to-N" (one to many) relationships.


Many beginners think that the only way to model “One-to-N” in MongoDB is to embed an array of sub-documents into the parent document, but that’s just not true. Just because you can embed a document, doesn’t mean you should embed a document. In fact, arrays that grows unbounded drops the performance. Moreover, the maximum document size is 16MB.

When designing a MongoDB schema, **you must start with the question : what is the cardinality of the relationship?** Is it **“one-to-few”**, **“one-to-many”**, or **“one-to-squillions”**? Depending on which one it is, you’d use a different format to model the relationship.

## One-to-Few

An example of “one-to-few” might be the addresses for a person. This is a good use case for embedding – you’d put the addresses in an array inside of your Person object:

```js
> db.person.findOne()
{
  name: 'Manuel Romero',
  ssn: '123-456-7890',
  addresses : [
     { street: '123 Sesame St', city: 'Anytown', cc: 'USA' },
     { street: '123 Avenue Q', city: 'New York', cc: 'USA' }
  ]
}
```
Pros:

- The main advantage is that you don’t have to perform a separate query to get the embedded details.

Cons:

- The main disadvantage is that you have no way of accessing the embedded details as stand-alone entities.



## One-to-Many

An example of “one-to-many” might be parts for a product in a replacement parts ordering system. Each product may have up to several hundred replacement parts, but never more than a couple thousand or so. (All of those different-sized bolts, washers, and gaskets add up.) This is a good use case for referencing – you’d put the ObjectIDs of the parts in an array in product document.

Part document:

```js
> db.parts.findOne()
{
    _id : ObjectID('AAAA'),
    partno : '123-aff-456',
    name : '#4 grommet',
    qty: 94,
    cost: 0.94,
    price: 3.99
}
```

Product document:

```js
> db.products.findOne()
{
    name : 'left-handed smoke shifter',
    manufacturer : 'Acme Corp',
    catalog_number: 1234,
    parts : [     // array of references to Part documents
        ObjectID('AAAA...'),    // reference to the #4 grommet above
        ObjectID('F17C...'),    // reference to a different Part
        ObjectID('D2AA...'),
        // etc
    ]
```
Pros:

- Each Part is a stand-alone document, so it’s easy to search them and update them independently.

- This schema lets you have individual Parts used by multiple Products, so your One-to-N schema just became an N-to-N schema without any need for a join table!

Cons:

- Having to perform a second query to get details about the Parts for a Product.

### One-to-Many with Denornmalization

Imagine that a frequent operation over our Products collection is: given the name of a part, to query if that part exists for that product. With the approach that we have implemented we would have two do a couple of queries. One to get the ObjectIDs for all the parts of a product and a second one to get the names of the parts. But, if this is a common data access pattern of our application, we can **denormalize** the field *name* of the part into the array of products parts:

```js
> db.products.findOne()
{
    name : 'left-handed smoke shifter',
    manufacturer : 'Acme Corp',
    catalog_number: 1234,
    parts : [
        {
         ObjectID('AAAA...'),
         name: '#4 grommet'
        },
        {
         ObjectID('F17C...'),    
         name: '#5 another part name'
        },
        {
         ObjectID('D2AA...'),
         name: '#3 another part name 2'
        }
        // etc
    ]
```

Pros:

- We can see all the parts that belong to a product (its name) with one single query.

Cons:

- Denornmalization makes sense when the denormalized field (*name* field in our case) is seldom updated. If we denormalize a field that is frequently updated, then the extra work of finding and updating all the instances is likely to overwhelm the savings that we get from denormalizing. A part's name will rarely change, so it's ok for us.

## One-to-Squillions

An example of “one-to-squillions” might be an event logging system that collects log messages for different machines. Any given host could generate enough messages to overflow the 16 MB document size, even if all you stored in the array was the ObjectID. This is the classic use case for “parent-referencing” – you’d have a document for the host, and then store the ObjectID of the host in the documents for the log messages.

Host document:

```js
> db.hosts.findOne()
{
    _id : ObjectID('AAA2...'),
    name : 'goofy.example.com',
    ipaddr : '127.66.66.66'
}
```
Message document:

```js
>db.logmsg.findOne()
{
    time : ISODate("2014-03-28T09:42:41.382Z"),
    message : 'cpu is on fire!',
    host: ObjectID('AAA2...')       // Reference to the Host document
}
```


## Conclusion

Based on the cardinality of our One-to-N relationship, we can pick one of the three basic One-to-N schema designs:

1. Embed the N side if the cardinality is one-to-few and there is no need to access the embedded object outside the context of the parent object.

2. Use an array of references to the N-side objects if the cardinality is one-to-many or if the N-side objects should stand alone for any reasons.

3. Use a reference to the One-side in the N-side objects if the cardinality is one-to-squillions.

And remember: **how we model our data depends – entirely – on our particular application’s data access patterns**. We want to structure our data to match the ways that our application queries and updates it.

[Reference](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1)
