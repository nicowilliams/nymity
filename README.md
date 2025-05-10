# nymity
A graph DB based on PG and UName\*It

UName\*It was an object-oriented, "network" (meaning "graph") database that existed from c. 1996 through c. 2001.  I worked at the only customer that deployed it in production, and my then employer got source code when the vendor closed shop.  I became the maintainer of UName\*It, so I know it well, and I miss it.

This will be a proof of concept at least, and perhaps functional enough to actually make it usable in production.

All work on Nymity's implementation at this time is pending approval from `$WORK`'s Legal.

## What was UName\*It?

It was a database meant to hold things like: DNS zones, NIS maps (so '90s), etc., but expressed as objects in an object-oriente system, with "pointers" between them, a strict schema, and very flexible namespace functionality.  In fact, the _namespace_ functionality was critical to UName\*It, and thus its name.

Today we would term UName\*It a "graph database", but back then its creators called it a "network" database, where the network was one of relationships between objects.  A graph database is really just a database of vertices (objects) and edges (relationships, or in UName\*It's terminology, pointers, or in SQL terminology: foreign keys), and if you add typing to the vertices and relationships, as well as non-pointer attributes, then you have a pretty advanced graph database.

Of course, a graph database should come with a powerful query language, but UName\*It did not have one, sadly.

But again, the most important aspect of UName\*It was its namespacing functionality.

## Uname\*It's Namespaces

In UName\*It a "namespace" was akin to a SQL table where triggers on one or more other tables cause corresponding `INSERT`/`UPDATE`/`DELETE` statements on the namespace table such that there can be cross-table uniqueness constraints!

Besides "namespaces" there were also "namespace rules".  A namespace rule had these elements:

- a reference to a namespace
- a reference to a class (a type of object -- a table in SQL)
- references to some number of attributes (fields, columns) of that class
- a hierarchy level
- a collision strength

and for any given namespace all rules' attribute references have to agree on their types, but they could have different attribute names.

Thus if you had different classes of objects with different naming attributes but which all nonetheless impliedly had to have corresponding names in the DNS, then you could have them all contribute to a DNS namespace such that they must all be unique!

This is a generalization of SQL `PRIMARY KEY`s.

The hierarchy level referred to the contributing object's different places in a hierarchy.  Thus an object named `foo` in a domain named `tokyo.acme.example` would have these possible levels:

- `tokyo.acme.example`
- `acme.example`
- the organization that owns `acme.example` and possibly other domains
- the global scope (think of IANA registries -- these are "global")

Finally the collision strength had to be one of:

- `strong` (there can be only one entry in the namespace with a given name, that is, the values of the contributing object's attributes referenced by the rule for the object's class)
- `normal` (same as `strong`, but multiple entries from the _same_ object are allowed)
- `weak` (multiple entries with the same name are allowed, as long as they are all `weak`)
- `none` (no entry is created in the namespace)

The idea was that one could have rules such as:

- there can be different mailboxes with the same name in every domain and sub-domain of the org's
- but there can be only one user through a domain with a give username
- usernames _are_ mailbox names
- usernames therefore preclude mailing lists with the same names!

