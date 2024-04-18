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
    public class Post : IAuditable
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
        public DateTime? DeletedAt { get; set; }
        public virtual ICollection<Comment> Comments { get; set; }

        public string GetEntityIdentifier()
        {
            return Id.ToString();
        }
    }
  {% endhighlight %}

  {% highlight c# %}
    public class Comment : IAuditable
    {
        public int Id { get; set; }
        public string Content { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
        public DateTime? DeletedAt { get; set; }
        public int PostId { get; set; }
        public virtual Post Post { get; set; }

        public string GetEntityIdentifier()
        {
            return Id.ToString();
        }
    }
  {% endhighlight %}

  As you can notice both classes implement the IAuditable interface. This interface will be used to identify the entities that should be audited. The interface will have a single method that will return an entity identifier. This will be used to identify the entity in the audit log.

  {% highlight c# %}
    public interface IAuditable
    {
        string GetEntityIdentifier();
    }
  {% endhighlight %}

  Now the next step is to override the SaveChanges method in the DbContext. When using entity framework 6 you can override the SaveChanges method in the DbContext to intercept the changes before they are persisted to the database. This is not the only way to intercept changes in entity framework, but it is the most convenient way for our purpose. In the SaveChanges method we will iterate over the entities that implement the IAuditable interface and are in the Added, Modified or Deleted state. We will then create a snapshot of the entity and persist it to the database.

  {% highlight c# %}
    public class BlogDbContext : DbContext
    {
        public DbSet<Post> Posts { get; set; }
        public DbSet<Comment> Comments { get; set; }
        public DbSet<Snapshot> Snapshots { get; set; }
        public DbSet<AuditLog> AuditLogs { get; set; }

        public override int SaveChanges()
        {
            var entries = ChangeTracker.Entries().Where(e => e.Entity is IAuditable && (e.State == EntityState.Added || e.State == EntityState.Modified || e.State == EntityState.Deleted)).ToList();

            foreach (var entry in entries)
            {
                var entity = entry.Entity as IAuditable;
                var entityId = entity.GetEntityIdentifier();
                var entityName = entity.GetType().Name;

                var snapshot = new Snapshot
                {
                    CreatedAt = DateTime.Now,
                    UpdatedAt = DateTime.Now,
                    EntityId = entityId,
                    EntityName = entityName,
                    Snapshot = JsonConvert.SerializeObject(entity)
                };

                Snapshots.Add(snapshot);
            }

            return base.SaveChanges();
        }
    }
  {% endhighlight %}

  Taking the whole snapshot of the entity a simple way to enable rollback mechanism. Now because this is not the most efficient way to do this, storage wise, you might want to consider storing only the changes that were made to the entity. This will require a more complex implementation but it will save you a possibly lots of memory in the database. Other option is to purge the snapshots after a certain period of time, if the business case allows that. If it doesn't then you might need to migrate the snapshots to a different type of storage, like cold storage, to save on costs.

  Let's now implement a rollback service. The rollback service will be used to revert the changes to the entity to a desired state. The rollback service will have a single method that will take the entity identifier and the snapshot identifier as arguments. The method will load the snapshot from the database and apply it to the entity.

  {% highlight c# %}
    public class RollbackService
    {
        private readonly BlogDbContext _dbContext;

        public RollbackService(BlogDbContext dbContext)
        {
            _dbContext = dbContext;
        }

        public void Rollback(string entityIdentifier, int snapshotId)
        {
            var snapshot = _dbContext.Snapshots.Find(snapshotId);

            if (snapshot == null)
            {
                throw new Exception("Snapshot not found");
            }

            var entity = _dbContext.Set(snapshot.EntityName).Find(entityIdentifier);

            if (entity == null)
            {
                throw new Exception("Entity not found");
            }

            JsonConvert.PopulateObject(snapshot.Snapshot, entity);

            _dbContext.SaveChanges();
        }
    }
  {% endhighlight %}

Check the code in the repo for full implementation.

[ef-audit-sample]: https://github.com/gbelkoski/gbelkoski.github.io