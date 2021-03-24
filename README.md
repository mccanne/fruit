
#  23 Mar 2021

This is all designed to work at petabyte, cloud scale, but we'll walk
through some simple examples to give you a feel for how Z works at
small scale.

Here is some example ZSON:
```
{name:"apple", color:"red", flavor: "tart"} (=fruit)
{name:"banana", color:"yellow", flavor: "sweet"} (fruit)
{name:"bob", likes: "tart", age:61} (=person)
{name:"strawberry", color:"red", flavor: "sweet"} (fruit)
{name:"dates", color:"brown", flavor: "sweet" } (fruit)
{name:"joe", likes: "sweet", age:14} (person)
{name:"figs", color:"brown", flavor: "plain" } (fruit)
{name:"pomegranate", color:"red", flavor: "tart" } (fruit)
{name:"jim", likes: "plain", age:30} (person)
```

NOTE: we can put this in an efficient row-based binary format called ZNG
but we won't bother here because the data file is small.
e.g.,
```
zq -o data.zng data.zson
zq -Z data.zng
```
We can try csv...
```
zq -f csv data.zng
```
Hmm that didn't work.  Oh, because there are different kinds of rows.
We need to fuse them...
```
zq -f csv fuse data.zng
```
Here is a clearer picture of what fuse does...
```
zq -f table fuse data.zng
```

Z is a simple search language:
```
zq -z "red AND tart" data.zson

{name:"apple",color:"red",flavor:"tart"} (=fruit)
{name:"pomegranate",color:"red",flavor:"tart"} (fruit)


zq -z "red tart | cut name" data.zson

{name:"apple"}
{name:"pomegranate"}
```

These queries don't care which "table" you are searching.
In this way, it's like a log search sysytem:
```
zq -z "joe" data.zson

{name:"joe",likes:"sweet",age:14} (=person)
```

## Whither SQL

No one wants to learn a new language like Z.

Ok, so SQL expressions are Z operators:

* You can put SQL in Z.
* You can Z in SQL.

```
zq -z "SELECT *" data.zson

{name:"apple",color:"red",flavor:"tart"} (=fruit)
{name:"banana",color:"yellow",flavor:"sweet"} (fruit)
{name:"bob",likes:"tart",age:61} (=person)
{name:"strawberry",color:"red",flavor:"sweet"} (fruit)
{name:"dates",color:"brown",flavor:"sweet"} (fruit)
{name:"joe",likes:"sweet",age:14} (person)
{name:"figs",color:"brown",flavor:"plain"} (fruit)
{name:"pomegranate",color:"red",flavor:"tart"} (fruit)
{name:"jim",likes:"plain",age:30} (person)
```

> NOTE: SELECT * just matched everything so you get data that looks like 'fruit'
> and data that looks like 'person'

```
zq -z "red tart | cut name" data.zson
vs
zq -z 'SELECT name WHERE color="red" AND flavor="tart"' data.zson

{name:"apple"}
{name:"pomegranate"}
```

```
zq -z "SELECT * ORDER BY name LIMIT 3" data.zson

{name:"apple",color:"red",flavor:"tart"} (=fruit)
{name:"banana",color:"yellow",flavor:"sweet"} (fruit)
{name:"dates",color:"brown",flavor:"sweet"} (fruit)
```

## Tables as Z types

But wait there's more.

Notice the types... `fruit` and `person`.
No need to create tables.  Just throw the data into a ZNG file
or a Z lake.

Suppose we didn't know what data was in there...
do an aggregation on the "type value" of the outer record stream:
```
zq -Z "SELECT typeof(.) FROM * GROUP BY typeof" data.zson

{
    typeof: (fruit=({name:string,color:string,flavor:string}))
}
{
    typeof: (person=({name:string,likes:string,age:int64}))
}
```

> REMEMBER: everything here can be run from a beautiful UI with
> lots of UX shortcuts and intuitive views.

Or better, sample a piece of data from each type...

```
zq -Z "SELECT any(.) as sample,typeof(.) as shape FROM * GROUP BY sample,shape" data.zson

{
    shape: (person=({name:string,likes:string,age:int64})),
    sample: {
        name: "bob",
        likes: "tart",
        age: 61
    } (=person)
} (=0)
{
    shape: (fruit=({name:string,color:string,flavor:string})),
    sample: {
        name: "apple",
        color: "red",
        flavor: "tart"
    } (=fruit)
} (=1)
```
That's noisy.  Lets SELECT just the data sample.  Rather than figure out
a complex nested SQL we can just exploit the fact that this is also Z and
put the output of the first SQL expression to a second one:
```
zq -Z "SELECT any(.) as sample,typeof(.) FROM * GROUP BY sample,typeof | SELECT sample" data.zson

{
    any: {
        name: "apple",
        color: "red",
        flavor: "tart"
    } (=fruit)
} (=0)
{
    any: {
        name: "bob",
        likes: "tart",
        age: 61
    } (=person)
} (=1)
```

## Taste

We call this query a "taste" because it lets you explore the shape of
your data.  Z has a operator called "taste" that is shorthand for the
above query...

```
zq -Z taste data.zson

{
    any: {
        name: "apple",
        color: "red",
        flavor: "tart"
    } (=fruit)
} (=0)
{
    any: {
        name: "bob",
        likes: "tart",
        age: 61
    } (=person)
} (=1)
```

You can give taste an argument to taste just certain fields:

```
zq -z "taste name" data.zson
{taste:"apple"}

zq -z "taste likes" data.zson
{taste:"tart"}
```

> NOTE: This works remarkably well for very large data sets.  You can
> ingest a bunch of random data from various sources into one ZNG file
> or Z lake and start to explore with taste...

> TBD: We should taste so the key isn't "taste" but the original field name?
> If it's not ".".

## Types are like tables

Top-level types are like "tables" in SQL...
`fruit` is a table and `person` is a table.
The above FROM clause using type equality is the same as just giving a
type value to FROM...
```
zq -z "SELECT * FROM fruit" data.zson

{name:"apple",color:"red",flavor:"tart"} (=fruit)
{name:"banana",color:"yellow",flavor:"sweet"} (fruit)
{name:"strawberry",color:"red",flavor:"sweet"} (fruit)
{name:"dates",color:"brown",flavor:"sweet"} (fruit)
{name:"figs",color:"brown",flavor:"plain"} (fruit)
{name:"pomegranate",color:"red",flavor:"tart"} (fruit)
```
Here is the person table:
```
zq -z "SELECT * FROM person" data.zson

{name:"bob",likes:"tart",age:61} (=person)
{name:"joe",likes:"sweet",age:14} (person)
{name:"jim",likes:"plain",age:30} (person)
```

You can run _ad hoc_ SQL queries:
```
zq -z "SELECT * FROM fruit ORDER BY name LIMIT 3" data.zson

{name:"apple",color:"red",flavor:"tart"} (=fruit)
{name:"banana",color:"yellow",flavor:"sweet"} (fruit)
{name:"dates",color:"brown",flavor:"sweet"} (fruit)
```


### Joins

You can uae `-I` to "include" multi-line Z from a file.
Let's say we have such a file called `join.zed`:
```
  SELECT p.name, union(f.name) AS likes
  FROM fruit f
  LEFT JOIN person p ON f.flavor = p.likes
  GROUP BY p.name
```  
so running this gives...
```
zq -f table -I join.zed data.zson

NAME LIKES
jim  figs
joe  dates,banana,strawberry
bob  apple,pomegranate
```
or with Z, you can see the "set" data type...
```
zq -z -I join.zed data.zson

{name:"bob",likes:|["apple","pomegranate"]|}
{name:"jim",likes:|["figs"]|}
{name:"joe",likes:|["dates","banana","strawberry"]|}
```

### ETL and Finding Messy Data

Through data discovery, we figure out the types in the table.
These types happen to already have typedef names...
```
zq -z "by typeof(.)" data.zson

{typeof:(fruit=({name:string,color:string,flavor:string}))}
{typeof:(person=({name:string,likes:string,age:int64}))}
```

```
cat messy.zson

{name:"steve", likes: "spicy", age:"52"}
```

```
cat data.zson messy.zson > new.zson ; cat new.zson
```
Hmm some bad data got into our lake...
```
zq -z "by typeof(.)" new.zson

{typeof:(fruit=({name:string,color:string,flavor:string}))}
{typeof:(person=({name:string,likes:string,age:int64}))}
{typeof:({name:string,likes:string,age:string})}
```
Let's clean it up.  If you're lake had a ton of data in it, you might say this:
```
zq -z "missing(nameof(.)) | by typeof(.)" new.zson
```
It looks like it should be classified as a 'person',
but age is a string.  That's okay, if we cast this record to a 'person' the
right stuff will happen.  Let's create "types.zed"...
```
type fruit = {name:string,color:string,flavor:string}
type person = {name:string,likes:string,age:int64}
```
Now we can say...
```
zq -z -I types.zed "missing(nameof(.)) | put .=cast(.,type(person))" new.zson
```

### SQL maps on to the kernel operators of the Z dataflow

```
ast -s -C "SELECT * FROM person ORDER BY name LIMIT 3"

filter is(type(person))
| sort name
| head 3
```

And this more complicated join and aggregation maps onto
a merge sort over parallel paths...

```
ast -s -C -I join.zed

split (
  =>
    filter is(type(fruit))
    | cut f=.
    | sort f.flavor
  =>
    filter is(type(person))
    | cut p=.
    | sort p.likes
)
| join on f.flavor=p.likes
  join-cut p
| summarize
    union(f.name) by p.name
```

## Long-term vision

Open source project -> bottom-up

Z is in the middle of JSON and the rigid relational world.

_Listen your data._

It has something to say.  The messiness _means something_.

Let it in the door.  Let it coexist.  Your queries can decide
when to include it and not.  You can query to messy data then
do something about it.

DON'T IMPOSE THE POLICY OF WHAT'S CLEAN UPON THE FORMAT OF THE DATA.
