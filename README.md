# Nymity
A graph DB based on PG and UName\*It

UName\*It was an object-oriented, "network" (meaning "graph") database that existed from c. 1996 through c. 2001.  I worked at the only customer that deployed it in production, and my then employer got source code when the vendor closed shop.  I became the maintainer of UName\*It, so I know it well, and I miss it.

This will be a proof of concept at least, and perhaps functional enough to actually make it usable in production, but at any rate it will be puer SQL and PlPgSQL.  No Python, no Java, no Go, no Rust, etc.  Perhaps a tiny bit of C to create a dynamic data type for PostgreSQL (this is easy to do, though I've not yet figured out how to index it correctly; the alternative is to use PostgreSQL's `JSONB` type).  In particular Nymity will not be a full-fledged system by itself -- others might add a RESTful server, or perhaps use [PostgREST](https://postgrest.org), a UI, etc.

All work on Nymity's implementation at this time is pending approval from `$WORK`'s Legal.

## What was UName\*It?

It was a database meant to hold things like: DNS zones, NIS maps (so '90s), etc., but expressed as objects in an object-oriente system, with "pointers" between them, a strict schema, and very flexible namespace functionality.  In fact, the _namespace_ functionality was critical to UName\*It, and thus its name.

Today we would term UName\*It a "graph database", but back then its creators called it a "network" database, where the network was one of relationships between objects.  A graph database is really just a database of vertices (objects) and edges (relationships, or in UName\*It's terminology, pointers, or in SQL terminology: foreign keys), and if you add typing to the vertices and relationships, as well as non-pointer attributes, then you have a pretty advanced graph database.

Of course, a graph database should come with a powerful query language, but UName\*It did not have one, sadly.

But again, the most important aspect of UName\*It was its namespacing functionality.

The primary developer of UName\*It was @vdukhovni.

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

## UName\*It's Type, O-O, and Data Model

The attributes of objects could be any of these types:

- string (text)
- integer
- real
- IP address
- boolean
- pointer (every object had a unique UUID, and the UUID was the real "value" of a pointer attribute)

Pointer types had to have an attribute _domain_, which was a class instances of which could be pointed to.

Attributes could have multiplicities of:

- scalar (zero or one value -- zero if the attribute were optional)
- set (zero, one, or more values)
- sequence (ordered, i.e., array of zero, one, or more values)

Sequence multiplicity was deprecated, since UName\*It mapped onto a SQL RDBMS, set semantics made more sense.

UName\*It had roughly this O-O model:

- multiple inheritance
  - and what was inherited?  attributes!
- attribute names were unique, but different constraints could be applied in different classes inheriting an attribute, but all had to have the same type

Every class had to have a `name` attribute of type `string` and a `domain` attribute of type pointer and attribute class `domain` (my memory is fuzzy as to class names, but if I remember correctly class names were not capitalized), where the domain pointed to some DNS-like domain, like `london.acme.example`, or `acme.example`, or even `.`, but not a TLD or ccTLD.  The domain attribute was used to derive the hierarchy levels for namespace rules.  Thus if `london.acme.example` had `acme.example` as its parent, and `acme.example` was owned by an `organization` called, say, `Exmplar ACME Inc.`, then the "level" values substituted into any namespace rules for objects in `london.acme.example` would be `london.acme.example`, `acme.example`, `Exmplar ACME Inc.`, and `.`, while for objects in `acme.example` the levels would be `acme.example`, `acme.example`, `Exmplar ACME Inc.`, and `.` (note that for the latter one level value appears twice, which is why collision strength `normal` had to exist).

## Implementation of the UName\*It data model

UName\*It used an object-oriented SQL RDBMS called UniSQL/X, a long since defunct RDBMS replaced by UniSQL/M (this happened after UName\*It stopped getting support, so a port to UniSQL/M never happened, at least to my knowledge).

UniSQL/X was the object-oriented database that PostgreSQL wanted to be.  Sadly PostgreSQL's object-oriented features are considered broken, and so are not used.  But in the 90s PostgreSQL wasn't the happening database that it is today anyways.

UniSQL/X's object-oriented model was well suited to UName\*It v1, but by v2 it no longer was, and UName\*It started relying less and less on UniSQL/X's object oriented features.  But not all aspects of UnixSQL/X's O-O model usage were removed from UName\*It v2.5.3, so when making schema changes in UName\*It the system had to "deconflict" the UniSQL/X schema -- a slow process.

At any rate, UName\*It's classes mapped onto SQL `TABLE`s, with inheritance used to make `SELECT`ing from a class with sub-classes also search the sub-classes.  Attributes mapped to columns.

However, for group-like objects such as user groups, mailing lists, and "netgroups" (which could group hosts as well as users) UName\*It had an entity-attribute-value (EAV) schema in the UName\*It shcema layer such that

a) all group-like classes derived from a common super-class of group-like things,
b) all member-like classes derived from a common super-class of member-like things,
c) all memberships were represented as instances of a membership class for the group that also inherited from a common super-class of group memberships,
d) all memberships were name-less entities with two pointer attributes: parent (pointing to the group object) and child (pointing to the member object).

The type of the membership class was akin to the A in EAV, and the parent and child pointers were akin to the E and the V in EAV.

UName\*It also had a metaschema, which was hard-coded in.  The metaschema represented itself.  Readers should refer to The Art of the MetaObject Protocol, a wonderful book that deals extensively with metaschemas in object-oriented systems, though I believe UName\*It's authors did not know about it.

## UName\*It's Protocol and UI

UName\*It predated the popularity of RESTful APIs, so it lacked an HTTP API, sadly.  It's protocol was very simple: a TCP connection, a Kerberos authentication exchange (`AP-REQ` and `AP-REP`/`KRB-ERROR` PDUs), then `KRB-PRIV` messages encrypting the application protocol, with the application protocol consisting of Tcl objects(!).  The protocol was a strict request/response protocol.  There were no XID-like values.

The UI on the other hand was a Tcl/TK UI where the code for the UI was compiled by the server for each object class, which code the server sent to the client, and the client executed that code to render the UI.  This design was remarkably similar to todays web single-page apps (SPA)!, only without HTTP.

## SQLite3 Proof-of-Concept

In late 2012 I wrote a tiny SQLite3, all-SQL toy implementation of UName\*It using just an EAV table.  This was easy because SQLite3 is dynamically typed, so I could store pointers, strings, etc. in one column.  I had SQL that would compile the class definitions (including inheritance) into `VIEW`s, with a column per attribute for scalar attributes.  Thus one could write queries against the main EAV table as well as against the `VIEW`s.  The `VIEW`s obviously were meant to be more ergonomic for most users, but the EAV table had the benefit that writing partial (and full) transitive closure queries as SQL `RECURSIVE` common table expressions (CTEs) was possible such all such queries looked almost exactly the same.

I don't recall if I implemented referential integrity features like foreign keys, but that would be trivial regardless of the use of an EAV model.

The key insights here is this:

- an EAV model, though it is typically disparaged as an anti-pattern, enables simple transitive closure queries
- transitive closures are the heart of graph queries

## `FOREIGN KEY`s are Bi-Directional and Chiral

Also, in SQL pointers are `FOREIGN KEY`s (FKs), and they are inherently bi-directional even though they don't syntactically appear to so be.  This is because FKs require indices on both, the pointing side columns (to make `ON DELETE`/`ON UPDATE` referential integrity rules perform well) and the pointed-to side columns.  Therefore for any one `FOREIGN KEY` constraint one can always find rows that refer to a row as well as referenced rows.

But `FOREIGN KEY`s are only defined on one side of the relation.  This means that FKs are _chiral_: they have a direction that can be thought of as more natural than another.  This is useful because we can use this to help define sub-graphs in graph databases.  Indeed, UName\*It's group membership classes were bi-directional _and_ chiral.  However, UName\*It's pointer attributes themselves were unidirectional -- I firmly believe this was a mistake, that pointer attributes needed to have come in _pairs_ of attributes, one for one class on one side of the relation, and the other for the other side of the relation.  If UName\*It's pointer attributes had been paired and thus bi-directional then the group memberships class would not have been needed!

## Graph Databases

Objects and pointers, or relationships if you wish, are the key to graph databases as the objects form vertices and the pointers form edges in a graph.

The other key ingredient to graph databases is a query language that can do useful things with a graph or sub-graph.  For example, traversing scalar pointers should be just a path expression like `someattr->someotherattr->...->lastattrinpath`.  But most importantly a graph database needs to be able to answer questions like:

- "what codebases depend on X?"
- "what codebases does X depend on?"
- "what codebases are affected by <CVE>"
- "who are all the members of this user group or mailing list?"
- "what are all this user's group memberships?"
- etc.

These are all just "transitive closure" queries over subsets of the full graph database -- sub-graphs.

But my SQLite3-based protocol did not have a query language of its own -- it was pure SQL.  A more serious project should have a SQL-like query language (QL) with extensions for graph queries, which is to say: path expressions for use in `SELECT` and other SQL clauses, and [partial] transitive closures as table sources (which can be `VIEW`s, which SQLite3 and PostgreSQL both have, or table-valued functions, which SQLite3 did not have but PostgreSQL does have).

# Implementation Plan

The implementation will target PostgreSQL and will be pure SQL and PlPgSQL.  All questions of how to access the data will be left to PostgreSQL itself (its protocol) and applications of PostgreSQL such as PostgREST.

I need to figure out if I can build a true `anyelement` type for PostgreSQL.  PostgreSQL has an `anyelement` type, but it cannot be used as the type of columns of `TABLE`s, sadly.  In fact, PostgreSQL has no dynamic typing except via `hstore` (an external contribution included in PG's `contrib/`), `json`, and `jsonb` (the latter two are built-in in PG).  But for an EAV-style schema I don't need arbitrarily complex JSON/hstore "documents" as attribute values -- I need only a single-valued container of values of arbitrary types.  Clearly it is possible to build such a thing using public C interfaces in PG, as that's exactly what `hstore` does.  In fact, I asked an LLM and it wrote me one.  But I need to understand how to handle indexing of values of different types in such a true `anyelement` type.  This should be the only non-SQL, non-PlPgSQL code, unless I just use `jsonb`.  Still, for a prototype using `jsonb` should be fine, so I might just do that.

As in my SQLite3 prototype from 2012 `VIEW`s will be automatically `CREATE OR REPLACE VIEW`ed using `TRIGGER`s on the EAV table for classes.  And as in that prototype this requires a recursive CTE for the class inheritance graph's transitive closure.

A foreign key constraint system just like UName\*It's (which itself resembled SQL's) should be trivial to build as `TRIGGER`s, much like how most SQL RDBMSes internally implement FKs as `TRIGGER`s.  (Indeed, SQLite3 used to include a command to generate such triggers, back when SQLite3 did not include full FK support.)

Namespaces will be implemented as a `TRIGGER`(s) for each namespace rule on the class(es)' `TABLE`s.

Inheritance is mainly a question of: a) the generation of `VIEW`s for each class, b) that have `UNION` queries when they have sub-classes.

Since I won't be building a query language nor extending PostgreSQL's, the only graph query language support will be:

a) a table-valued function that computes partial transitive closures for sub-graphs identified by its arguments (set of pointer types, and an object ID on the left or right side of the graph),

b) `VIEW`s that implement partial transitive closures for user-defined sub-graphs where the sub-graph definition consists of a set of pointer types).

That's roughly it.

In fact, anyone can do this based on this README.
