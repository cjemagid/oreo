[![Oreo](https://raw.github.com/will123195/oreo/master/oreo.png)](https://github.com/will123195/oreo)



[![Build Status](https://travis-ci.org/will123195/oreo.svg?branch=master)](https://travis-ci.org/will123195/oreo)



## Features

- No configuration necessary
- Automatically discovers tables, primary keys, foreign keys and master/standby servers
- Ability to "hydrate" foreign keys
- Object caching & memoization

## Installation

```bash
npm install oreo
npm install pg
```

## Example

```js
// Initialize Oreo
var oreo = require('oreo')
var db = oreo({
  driver: 'pg',
  hosts: ['localhost'],
  port: 5432,
  name: 'database',
  user: 'username',
  pass: 'password'
})

// Discover tables, primary keys and foreign keys
db.discover().on('ready', function() {

  // Insert a new book and it's author
  db.books.insert({
    title: 'On the Road',
    author: {
      name: 'Jack Kerouac'
    }
  }, function(err, book) {
    console.log(book)
    // { id: 1, title: On the Road, author_id: 1 }

    // Get the author by primary key
    db.authors.get(book.author_id, function(err, author) {
      console.log(author)
      // { id: 1, name: Jack Kerouac, books: [ 1 ] }
    })
  })
})
```

The example above will work with the following database schema:
```sql
CREATE TABLE authors (
  id SERIAL,
  name VARCHAR,
  books INTEGER[],
  CONSTRAINT author_pkey PRIMARY KEY(id)
);

CREATE TABLE books (
  id SERIAL,
  title VARCHAR,
  author_id INTEGER,
  CONSTRAINT book_pkey PRIMARY KEY(id),
  CONSTRAINT book_fk1 FOREIGN KEY (author_id) REFERENCES authors(id)
);
```

## Documentation

### Database

* [`discover`](#discover)
* [`execute`](#execute)

### Table

* [`find`](#find)
* [`findOne`](#findOne)
* [`get`](#get)
* [`insert`](#insert)
* [`mget`](#mget)

### Row

* [`hydrate`](#hydrate)
* [`save`](#save)
* [`set`](#set)
* [`update`](#update)


## Database

<a name="discover" />
### db.discover( [cb] )

Adds a property to the `db` object for every table in the database.

```js
db.discover()
```

<a name="execute" />
### db.execute( query, [data], [cb] )

Executes an SQL query.
```js
db.execute([
  'select now()',
  'as now'
], function(err, rs) {
  console.log(rs[0]) // 2014-06-24 21:03:08.652861-04
})
```

SQL injection safe:
```js
db.execute([
  'select id',
  'from authors',
  'where name = :name'
], {
  name: 'Jack Kerouac',
}, function(err, rs) {
  console.log(rs[0].id) // 1
})
```

## Table

<a name="find" />
### db.table.find( opts, [cb] )

Finds one or more rows:
```js
db.authors.find({
  where: ["name ilike 'Jack%'"],
  order: 'name asc',
  offset: 5,
  limit: 5
}, function(err, authors) {
  console.log(authors[0].id) // 1
})
```

<a name="findOne" />
### db.table.findOne( [cb] )

Finds exactly one row:
```js
db.authors.findOne({
  where: ["name ilike 'Jack%'"],
  order: 'name asc',
  offset: 5
}, function(err, author) {
  console.log(author.id) // 1
})
```

<a name="get" />
### db.table.get( primaryKey, [cb] )

Finds a row by primary key:
```js
db.authors.get(1, function(err, author) {
  console.log(author) // { id: 1, name: Jack Kerouak, books: [1] }
})
```

Multi-column (composite) primary key:
```js
db.parts.get({
  company: 'Cogswell Cogs',
  part_no: 'A-12345'
}, function(err, part) {

})
```

<a name="insert" />
### db.table.insert( data, [cb] )

Inserts a new row.
```js
db.books.insert({
  title: 'On the Road',
  author_id: 1
}, function(err, book) {
  console.log(book)
  // { id: 1, title: On the Road, author_id: 1 }
})
```

Insert multiple rows into related tables in a single transaction:
```js
db.books.insert({
  title: 'On the Road',
  author: {
    name: 'Jack Kerouac'
  }
}, function(err, book) {
  console.log(book)
  // { id: 1, title: On the Road, author_id: 1 }
  book.hydrate(function(err, book) {
    console.log(book)
    // { id: 1, title: On the Road, author_id: 1, author: { id: 1, name: Jack Kerouac, books: [1] } }
  })
})
```

<a name="mget" />
### db.table.mget( primaryKeys, [cb] )

Gets many rows from the database by primary key:
```js
var bookIds = [1]
db.books.mget(bookIds, function(err, books) {
  console.log(books)
  // [ { id: 1, title: On the Road, author_id: 1 } ]
})
```

## Row

<a name="hydrate" />
### row.hydrate( [cb] )

Populates the related data rows (1-to-1 foreign keys):
```js
db.books.get(1, function(err, book) {
  console.log(book)
  // { id: 1, title: On the Road, author_id: 1 }
  book.hydrate(function(err, book) {
    console.log(book)
    // { id: 1, title: On the Road, author_id: 1, author: { id: 1, name: Jack Kerouac, books: [1] } }
  })
})
```

<a name="save" />
### row.save( [cb] )

Saves the modified property values to the database (recursively):
```js
db.books.get(1, function(err, book) {
  console.log(book)
  // { id: 1, title: On the Road, author_id: 1 }
  book.author_id = 2
  book.save(function(err, book) {
    console.log(book)
    // { id: 1, title: On the Road, author_id: 2 }
  })
})
```

<a name="set" />
### row.set( data )

Sets multiple property values but does not save yet:
```js
db.books.get(1, function(err, book) {
  console.log(book)
  // { id: 1, title: On the Road, author_id: 1 }
  book.set({
    title: 'New Title',
    author_id: 2
  })
  book.save()
})
```

<a name="update" />
### row.update( data, [cb] )

Update an existing row:
```js
book.update({
  title: 'New Title'
}, function(err, book) {
  console.log(book)
  // { id: 1, title: New Title, author_id: 1 }
})
```

## Advanced Usage

### Create triggers that automatically populate arrays of 1-to-many foreign keys

```sql
CREATE OR REPLACE FUNCTION authors_books() RETURNS trigger AS
$body$
BEGIN
  IF TG_OP != 'INSERT' THEN
    UPDATE authors
    SET books = (
      SELECT ARRAY(
        SELECT id
        FROM books
        WHERE author_id = OLD.author_id
        ORDER BY title ASC
      )
    )
    WHERE id = OLD.author_id;
  END IF;
  UPDATE authors
  SET books = (
    SELECT ARRAY(
      SELECT id
      FROM books
      WHERE author_id = NEW.author_id
      ORDER BY title ASC
    )
  )
  WHERE id = NEW.author_id;
  RETURN null;
END;
$body$
LANGUAGE 'plpgsql';

CREATE TRIGGER book_tr1
  AFTER INSERT OR UPDATE OF author_id OR DELETE
  ON books FOR EACH ROW
  EXECUTE PROCEDURE authors_books();
```

### Create triggers that "replicate" to Redis Foreign Data Wrapper (for high-speed reads)
