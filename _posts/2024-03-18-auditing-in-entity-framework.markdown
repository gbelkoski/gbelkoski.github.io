---
layout: post
title:  "Auditing and reverting changes in entity framework 6"
date:   2024-03-18 18:46:37 +0100
categories: dotnet entity-framework
---

In this post I will show you how to audit changes in entity framework 6. This is a very common requirement in enterprise applications.
The auditing can serve a variety of purposes such as:
- Security
- Compliance
- Debugging
- Reverting changes

In the enterprise application world it is very important to know who changed what and when. This is where auditing comes in.

Now before digging into the implementation details lets think about the design for a moment. In a typical enterprise application you will likely have a lot of entities that you will want to audit. Also there will be many endpoints that end up mutating the state of these entities and persisting the changes. This means that you will have to write a lot of boilerplate and lot of repetitive code to audit changes for each entity. You can see this quicly becomming very cumbersome and tedious for maintenance. We want to write the code once and reuse it for all entities. Ideally we do not want to have to change the implementation each time a new entity or a new endpoint is added.

This leads us in the direction of thinking about a generic solution. One that will rely on some kind of interception mechanism or a middleware that will be able to capture the changes and persist them to the database each time an entity state is mutated. Furtnermore the implemenation of the audit log should not be dealing with any particular instance of an entity, rather it should be able to deal with an interface that auditable entities implement.

Our auditing solution will be taking snapshots of each auditable entity in json format, saving that snapshot in the database and associating it with the entity that was audited. This way we can easily revert the changes by loading the snapshot and applying it to the entity. The revert strategy will not as advanced as the one that git provides but it will be sufficient for our needs. User would be able to revert the changes to the last snapshot or to any snapshot in the history.

Lets look at how our db schema will look like:
  
  ```sql diagram


  +-----------------+    +-----------------+    +-----------------+
  |     Entity      |    |     Snapshot    |    |     AuditLog    |
  +-----------------+    +-----------------+    +-----------------+
  | Id              |    | Id              |    | Id              |
  | CreatedAt       |    | CreatedAt       |    | CreatedAt       |
  | UpdatedAt       |    | UpdatedAt       |    | UpdatedAt       |
  | DeletedAt       |    | EntityId        |    | EntityId        |
  |                 |    | EntityName      |    | EntityName      |
  |                 |    | Snapshot        |    | Snapshot        |
  |                 |    |                 |    |                 |
  +-----------------+    +-----------------+    +-----------------+

  ```

  The Snapshot table will hold the snapshots of the entities and their many-to-many relationships in json format. The AuditLog table group together multiple snapshots of entities that were audited during a single transaction. The term transaction here can refer to either a database transaction or a logical/business transaction. In our example we will use database transaction as a unit of work, mostly for simplicity, but still without sacrifising the flexibility of the solution.

  Okay enough talking, lets start coding.

  As a sample solution we will use a simple blog application with two entities: Post and Comment. The Post entity will have a one-to-many relationship with the Comment entity. We will audit changes to both entities. The solution will use entity framework 6, but similar approach can be used with entity framework core.

  {% highlight c# %}
    public class Post
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
        public DateTime? DeletedAt { get; set; }
        public virtual ICollection<Comment> Comments { get; set; }
    }
  {% endhighlight %}

  {% highlight c# %}
    public class Comment
    {
        public int Id { get; set; }
        public string Content { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
        public DateTime? DeletedAt { get; set; }
        public int PostId { get; set; }
        public virtual Post Post { get; set; }
    }
  {% endhighlight %}

<!-- `YEAR-MONTH-DAY-title.MARKUP`

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/ -->
