# Walkable Schema

As you can see in **Usage** guide, we need to provide
`sqb/compile-schema` a map describing the schema. You've got some idea
about how such a schema looks like as you glanced the example in
**Overview** section. Now let's walk through each key of the schema
map in details.

> Notes:
> - SQL snippets have been simplified for explanation purpose
> - Backticks are used as quote marks
> - For clarity, part of the schema may not be included

## 1 :idents

Idents are the root of all queries. From an SQL dbms perspective, you
must start your graph query from somewhere, and it's a table.

### 1.1 Keyword idents

Keyword idents can be defined as simple as:

```clj
;; schema
{:idents {:people/all "person"}}
```

so queries like:

```clj
[:people/all [:person/id :person/name]]
```

will result in an SQL query like:

```sql
SELECT `id`, `name` FROM `person`
```

### 1.2 Vector idents

These are idents whose key implies some condition. Instead of
providing just the table, you provide the column (as a namespaced
keyword) whose value match the ident argument found in the ident key.

For example, the following vector ident:

```clj
;; dispatch-key: :person/by-id
;; ident arguments: 1
[:person/by-id 1]
```

will require a schema like:

```clj
;; schema
{:idents {:person/by-id :person/id}}
```

so queries like:

```clj
;; query
[[:person/by-id 1] [:person/id :person/name]]
```

will result in an SQL query like:

```sql
SELECT `id`, `name` FROM `person` WHERE `person`.`id` = 1
```

## 2 :joins

Each join schema describes the "path" from the source table (of the
source entity) to the target table (and optionally the join table).

Let's see some examples.

### Example 1: Join column living in source table

Assume table `cow` contains:

```
|----+-------|
| id | color |
|----+-------|
| 10 | white |
| 20 | brown |
|----+-------|
```

and table `farmer` has:

```
|----+------+--------|
| id | name | cow_id |
|----+------+--------|
|  1 | jon  |     10 |
|  2 | mary |     20 |
|----+------+--------|
```

and you want to get a farmer along with their cow using the query:

```clj
;; query
[{[:farmer/by-id 1]} [:farmer/name {:farmer/cow [:cow/id :cow/color]}]]
```

> For the join `:farmer/cow`, table `farmer` is the source and table
  `cow` is the target.

then you must define the join "path" like this:

```clj
;; schema
{:joins {:farmer/cow [:farmer/cow-id :cow/id]}}
```

the above join path says: start with the value of column
`farmer.cow_id` (the join column) then find the correspondent in the
column `cow.id`.

Internally, Walkable will generate this query to fetch the entity
whose ident is `[:farmer/by-id 1]`:

```sql
SELECT `farmer`.`name`, `farmer`.`cow_id` FROM `farmer` WHERE `farmer`.`id` = 1
```

the value of column `farmer`.`cow_id` will be collected (for this
example it's `10`). Walkable will then build the query for the join `:farmer/cow`:

```sql
SELECT `cow`.`id`, `cow`.`color` FROM `cow` WHERE `cow`.`id` = 10
```

Finally, Walkable will combine the results of the above SQL queries
and return the final result:

```clj
{[:farmer/by-id 1] #:farmer{:number 1,
                            :name "jon",
                            :cow #:cow{:index 10,
                                       :color "white"}}}
```

### Example 2: A join involving a join table

Assume the following tables:

source table  `person`:

```
|----+------|
| id | name |
|----+------|
|  1 | jon  |
|  2 | mary |
|----+------|
```

target table `pet`:

```
|----+--------|
| id | name   |
|----+--------|
| 10 | kitty  |
| 11 | maggie |
| 20 | ginger |
|----+--------|
```

join table `person_pet`:

```
|-----------+--------+---------------|
| person_id | pet_id | adoption_year |
|-----------+--------+---------------|
|         1 |     10 |          2010 |
|         1 |     11 |          2011 |
|         2 |     20 |          2010 |
|-----------+--------+---------------|
```

you may query for a person and their pets along with their adoption year

```clj
[{[:person/by-id 1] [:person/name {:person/pets [:pet/name :person-pet/adoption-year]}]}]
```

then the schema for the join is as simple as:

```clj
;; schema
{:joins {:person/pets [:person/id :person-pet/person-id
                       :person-pet/pet-id :pet/id]}}
```

Walkable will issue an SQL query for `[:person/by-id 1]`:

```sql
SELECT `person`.`name` FROM `person` WHERE `person`.`id` = 1
```

and another query for the join `:person/pets`:

```sql
SELECT `pet`.`name`, `person_pet`.`adoption_year`
FROM `person_pet` JOIN `pet` ON `person_pet`.`pet_id` = `pet`.`id` WHERE `person_pet`.`person_id` = 1
```

and our not-so-atonishing result:

```clj
;; result
{[:person/by-id 1] #:person{:id 1,
                            :name "jon",
                            :pets [{:pet/id 10,
                                    :pet/name "kitty"
                                    :person-pet/adoption-year 2010}
                                   {:pet/id 11,
                                    :pet/name "maggie"
                                    :person-pet/adoption-year 2011}]}}
```

### Example 3: Join column living in target table

No big deal. This is no more difficult than example 1.

Assume table `farmer` contains:

```
|----+-------|
| id | name  |
|----+-------|
| 1  | jon   |
| 2  | mary  |
|----+-------|
```

and table `cow` has:

```
|----+-------+----------|
| id | name  | owner_id |
|----+-------+----------|
| 10 | white |     1    |
| 20 | brown |     2    |
|----+-------+----------|
```

The schema for this example can be a good exercise for the reader of
this documentation. (Sorry, actually I'm just too lazy to type it
here :D )

## 3 :reversed-joins

join specs are lengthy, to avoid typing the reversed path again
also it helps with semantics.

todo: example 1 and example 3 from `:join` section

## 4 :columns

list of available columns must be provided beforehand.

- pre-compute strings
- keywords not in the column set will be ignored

## 5 :cardinality

Idents and joins can have cardinality of either "many" (which is
default) or "one".

## 6 :extra-conditions

## 7 :quote-marks

Different SQL databases use different strings to denote quotation. For
instance, MySQL use a pair of backticks:

```sql
SELECT * FROM `table`
```

while PostgreSQL use quotation marks:

```sql
SELECT * FROM "table"
```

You need to provide the quote-marks as a vector of two strings

```clj
;; schema for mysql
{:quote-marks ["`", "`"]}
```

or
```clj
;; schema for postgres
{:quote-marks ["\"", "\""]}
```

For convenience, you can use the predefined vars instead:

```clj
;; schema for mysql
{:quote-marks sqb/backticks}
;; or postgresql
{:quote-marks sqb/quotation-marks}
```

The default quote-marks are the backticks, which work for mysql and
sqlite.

## 8 :sqlite-union

Set it to `true` if you're using SQLite, otherwise ignore
it. Basically SQLite don't allow to union queries directly.
With other SQL DBMS, the union query can be something like:

```sql
SELECT a, b FROM foo WHERE ...
UNION ALL
SELECT a, b FROM foo WHERE ...
```

SQLite will raise a syntax error exception for such queries. Instead,
a valid query for SQLite must be something like:

```sql
SELECT * FROM (SELECT a, b FROM foo WHERE ...)
UNION ALL
SELECT * FROM (SELECT a, b FROM foo WHERE ...)
```

`{:sqlite-union true}` is for enforcing just that.

## 9 :required-columns
## 10 :pseudo-columns (Experimental)

## Syntactic sugars

### Multiple dispatch keys for the same configuration