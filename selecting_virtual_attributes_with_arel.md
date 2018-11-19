# Selecting Virtual Attributes with Arel

## Arel: A Quick Intro
[Arel](https://github.com/rails/arel) is a SQL AST (Abstract Syntax Tree-like) manager for Ruby. It allows us to write complex SQL queries in a semantic, reusable fashion. Arel is "framework framework"; it's designed to optimize object and collection modeling over database compatibility. For example, Active Record is built on top of Arel.

Arel maps our data domain into a tree-like structure. For example, we can grab a tree-like representation of the portion of our data model related to users by calling:

```prettyprint lang-ruby
User.arel_tabel

=> #<Arel::Table:0x007fd42da94350
 @aliases=[],
 @columns=nil,
 @engine=
  User(id: integer, name: string),
 @name="user",
 @primary_key=nil,
 @table_alias=nil>
```

On our `Arel::Table` instance we can execute queries using [predicates](https://www.rubydoc.info/github/rails/arel/master/Arel/Predications) like:

```prettyprint lang-ruby
users = User.arel_table  
User.select(users[Arel.star]).where(users[:id].eq(1))  
```

## Selecting Virtual Attributes

### The Problem

Yesterday I found myself needing to query for a record _and add some attribute with a fixed value to that record_ that was not a column on the record's table. I was working on a feature to update various records in [Learn.co](https://learn.co/) with the awareness that a given lesson had been "deployed" to GitHub (i.e. the repo containing the lesson had been created). In order to do so, I needed to query for the lesson with a given UUID, attach the GitHub repository ID to the lesson, and pass the lesson down the chain of several other method calls in order for all of the necessary records to get updated.

There was just one problem though. The lessons table *doesn't have a `github_repository_id` column*. If only there was some way for me to query for a lesson and temporarily attach the GitHub repo ID to the resulting record, so that this nice little package of lesson information could contain everything my code needs to do its work.

Good news! Arel can totally do this for us!

### The Solution

We can execute a simple `SELECT` statement using Arel by first grabbing the `Lesson` model's arel table and using `where` with the appropriate predicate:

```prettyprint lang-ruby
lessons = Lesson.arel_table

Lesson
  .select(lessons[Arel.star])
  .where(lessons[:uuid].eq("a0cea74c-302f-4a47-a096-6d04690468a9"))
```

Next up we want to add to our select statement. We'll tell our select statement to select all of the columns from the lessons table, _and_ a virtual attribute with a fixed value:

```prettyprint lang-ruby
# where the GitHub repo ID for the lesson is 1234

lesson = Lesson
          .select(lessons[Arel.star], "'1234' as 'github_repository_id'")
          .where(lessons[:uuid].eq("a0cea74c-302f-4a47-a096-6d04690468a9"))
          .first
```

Now we have a lesson that we can work with like this:

```prettyprint lang-ruby
lesson[:github_repository_id]
# => '1234'
```

That's it!

## Further Reading
For a more advanced look at Arel querying, check out my post [Composable Query Builders in Rails with Arel](https://www.thegreatcodeadventure.com/composable-query-builders-with-arel-in-rails/)
