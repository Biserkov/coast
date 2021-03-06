# Queries

## What is it

The main advantage of putting your schema in the framework's hands is that one, you don't need to declare things twice, like writing out a rails schema and then having to say "belongs_to" or "has_many" in a model somewhere. Two, you can get away with really short query building vs SQL, which, I love-hate SQL, but I love making web apps faster more than writing a lot of SQL.

## The goal

Remove the O and the M in ORM. Data in, data out. Everything transparent and declarative.

## CRUD in Coast

Not sure if it's any better than CRUD anywhere else, but the R in there is definitely unlike anything in any other web framework. Here's how it works.

## R in CRUD

```clojure
(ns r-in-crud
  (:require [coast.db :as db]))

(db/q '[:select author/name author/email post/title post/body
        :joins  author/posts
        :where  [author/name ?author/name]]
      {:author/name "Cody Coast"})
```

The following query looks pretty basic and it is, it uses this SQL to query the database

```sql
select author.name, author.email, post.title, post.body
from author
join post on post.author_id = author.id
where author.name = ?
-- queries are parameterized
```

Which assuming some data, would output this in your Clojure code

```clojure
[{:author/name "Cody Coast" :author/email "cody@coastonclojure.com" :post/title "First!" :post/body "Post!"}
 {:author/name "Cody Coast" :author/email "cody@coastonclojure.com" :post/title "Second!" :post/body "Post!"}
 {:author/name "Cody Coast" :author/email "cody@coastonclojure.com" :post/title "Third!" :post/body "Post!"}]
```

This isn't bad, but you can imagine a few more joins and a few more columns and things might get out of hand.
Even if they didn't get out of hand, you want something like this anyway

```clojure
[{:author/name "Cody Coast"
  :author/email "cody@coastonclojure.com"
  :author/posts [{:post/title "First"
                  :post/body "Post!"}
                 {:post/title "Second!"
                  :post/body "Post!"}
                 {:post/title "Third!"
                  :post/body "Post!"}]}]
```

Well you're in luck, thanks to letting Coast handle your schema, you can do just that. It's called `pull` and yes
it's shamelessly stolen from datomic. This is how it looks.

```clojure
(db/q '[:pull [author/id
               author/email
               author/name
               {:author/posts [post/title
                               post/body]}]
        :where [author/name ?author/name]]
      {:author/name "Cody Coast"})
```

Which will output what you saw earlier. It uses the relationship names and data from the schema earlier to build the select and join parts of the query.

## The limits of pull

But wait a minute, how do you control the order they get returned in that fancy pull query? Here's how

```clojure
(db/q '[:pull [author/email
               author/name
               {(:author/posts :order post/id desc) [post/title
                                                     post/body]}]
        :where [author/name ?author/name]
               [author/name != nil]
        :limit 10
        :order author/id desc]
      {:author/name "Cody Coast"})
```

One limitation of pull syntax is how do you limit how many nested rows get returned? You  can do that too with `:limit`

```clojure
(db/q '[:pull [author/email
               author/name
               {(:author/posts :order post/id desc
                               :limit 10) [post/title
                                           post/body]}]
        :where [author/name ?author/name]]
      {:author/name "Cody Coast"})
```

If you know you only want to pull nested rows from one entity, you can use the `pull` function

```clojure
(db/pull '[author/id
           author/email
           author/name
           {(:author/posts :order post/created-at desc
                           :limit 10) [post/title
                                       post/body]}]
         [:author/name "Cody Coast"])
```

Unfortunately, or fortunately, if you want to get *really* crazy with a pull, you can't. You'll have to drop down to SQL and manipulate things with clojure yourself. The point of pull is to handle the common case, it doesn't arbitrary SQL functions or crazy SQL syntax. You'll have to either call `q` for that or fall back to SQL.

## Transactions

Here's two examples of inserting

```clojure
(db/transact {:post/title "3 things you should know about Coast on Clojure"
              :post/body "1. It's great. 2. It's tremendous. 3. It's making web development fun again."
              :post/author [:author/name "Cody Coast"]})
```

In cases where you already know the primary key, you can just pass that in place of the ident

```clojure
(db/transact {:post/title "3 things you should know about Coast on Clojure"
              :post/body "1. It's great. 2. It's tremendous. 3. It's making web development fun again."
              :post/author 2})
```

Here's two examples using postgresql upserts to update records

```clojure
(db/transact {:post/title "5 things you should know about Coast on Clojure"
              :post/id 1})

;  or with an ident

(db/transact {:post/title "5 things you should know about Coast on Clojure"
              :post/slug "07-10-2018-3-things-you-should-know"})
```

## Delete

You can delete rows by any table/col pair, multiple column delete is still under construction...

```clojure
(db/delete {:author/name "Cody Coast"})
```

you can also delete multiple rows at a time with the same key

```clojure
(db/delete [{:author/name "Cody Coast"}
            {:author/name "Carol Coast"}])
```

and since Coast is managing your schema, you get `on delete cascade` without even thinking about it!
