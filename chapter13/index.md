---
tags: Introspection, Type Union Operator
---

# Chapter 13 - Meet the new Lucy

> This time it was too late to save Lucy, and she is dying. Suddenly she opens her eyes - they look very strange. She looks at Arthur and says “Arthur! Oh, my love, I am so glad you have come! Kiss me!” He tries to kiss her, but Van Helsing grabs him and says "Don't you dare!" It was not Lucy, but the vampire inside that was talking. She dies, and Van Helsing puts a golden crucifix on her lips to stop her from moving (crucifixes have that power over vampires). Unfortunately, the nurse steals it to sell when nobody is looking. A few days later there is news about a lady who is stealing and biting children - it's Vampire Lucy. The newspaper calls it the "Bloofer Lady", because the young children try to call her the "Beautiful Lady" but can't pronounce _beautiful_ right. Now Van Helsing tells the other people the truth about vampires, but Arthur doesn't believe him and becomes angry that he would say crazy things about his wife. Van Helsing says, "Fine, you don't believe me. Let's go to the graveyard together tonight and see what happens. Maybe then you will."

Looks like Lucy, an `NPC`, has become a `MinorVampire`. How should we show this in the database? Let's look at the types again first.

Right now `MinorVampire` is nothing special, just a type that extends `Person`:

```sdl
type MinorVampire extending Person {
}
```

Fortunately, according to the book she is a new "type" of person. The old Lucy is gone, and this new Lucy is now one of the `slaves` linked to the `Vampire` named Count Dracula.

So instead of trying to change the `NPC` type, we can just give `MinorVampire` an optional link to `Person`:

```sdl
type MinorVampire extending Person {
  link former_self -> Person;
}
```

It's optional because we don't always know anything people before they were made into vampires. For example, we don't know anything about the three vampire women before Dracula found them so we can't make an `NPC` type for them.

Another way to (informally) link them is to give the same date to `last_appearance` for an `NPC` and `first_appearance` for a `MinorVampire`. First we will update Lucy with her `last_appearance`:

```edgeql
UPDATE Person FILTER .name = 'Lucy Westenra'
SET {
  last_appearance := cal::to_local_date(1887, 9, 20)
};
```

Then we can add Lucy to the `INSERT` for Dracula. (If you are following along, just `DELETE Vampire;` and `DELETE MinorVampire;` first so we can practice doing this longer `INSERT`.)

Note the first line where we create a variable called `lucy`. We then use that to bring in all the data to make her a `MinorVampire`, which is much more efficient than manually inserting all the information. It also includes her strength: we add 5 to that, because vampires are stronger.

Here's the insert:

```edgeql
WITH lucy := assert_single(
    (SELECT Person FILTER .name = 'Lucy Westenra')
)
INSERT Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (INSERT MinorVampire {
      name := 'Woman 1',
    }),
    (INSERT MinorVampire {
      name := 'Woman 2',
    }),
    (INSERT MinorVampire {
      name := 'Woman 3',
    }),
    (INSERT MinorVampire {
      # We need to give a new name, so as not to clash with former_self.
      name := 'Lucy',
      former_self := lucy,
      first_appearance := lucy.last_appearance,
      strength := lucy.strength + 5,
    }),
  },
  places_visited := (SELECT Place FILTER .name IN {'Romania', 'Castle Dracula'})
};
```

And thanks to the `former_self` link, it's easy to find all the minor vampires that come from `Person` objects. Just filter by `EXISTS .former_self`:

```edgeql
SELECT MinorVampire {
  name,
  strength,
  first_appearance,
} FILTER EXISTS .former_self;
```

This gives us:

```
{
  default::MinorVampire {
    name: 'Lucy',
    strength: 5,
    first_appearance: <cal::local_date>'1887-09-20',
  },
}
```

Other filters such as `FILTER .name IN Person.name AND .first_appearance IN Person.last_appearance;` are possible too but checking if the link `EXISTS` is easiest. We could also switch to `cal::local_datetime` instead of `cal::local_date` to get the exact time down to the minute. But we won't need to get that precise just yet.

## The type union operator: |

Another operator related to types is `|`, which is used to combine them (similar to writing `OR`). This query for example pulling up all `Person` types will return true:

```
SELECT (SELECT Person FILTER .name LIKE 'Lucy%') IS NPC | MinorVampire | Vampire;
```

It returns true if the `Person` type selected is of type `NPC`, `MinorVampire`, or `Vampire`. Since both Lucy the `NPC` and Lucy the `MinorVampire` match any of the three types, the return value is `{true, true}`.

One cool thing about the type union operator is that you can also add it to links in your schema. Let's say for example there are other `Vampire` objects in the game, and one `Vampire` that is extremely powerful can control the other. Right now though `Vampire`s can only control `MinorVampire`s:

```
type Vampire extending Person {
  multi link slaves -> MinorVampire;
}
```

So to represent this change, you could just use `|` and add another type:

```
type Vampire extending Person {
  multi link slaves -> MinorVampire | Vampire;
}
```

We only have Count Dracula in our database as the main `Vampire` type so we won't change our schema in this way, but keep this `|` operator in mind in case you need it.

## On target delete

We've decided to keep the old `NPC` type for Lucy, because that Lucy will be in the game until September 1887. Maybe later `PC` types will interact with her, for example. But this might make you wonder about deleting links. What if we had chosen to delete the old type when she became a `MinorVampire`? Or more realistically, what if all `MinorVampire` types connected to a `Vampire` should be deleted when the vampire dies? We won't do that for our game, but you can do it with `on target delete`. `on target delete` means "when the target is deleted", and it goes inside `{}` after the link declaration. For this we have {ref}`four options <docs:ref_datamodel_link_deletion>`:

- `restrict`: forbids you from deleting the target object.

So if you declared `MinorVampire` like this:

```sdl
type MinorVampire extending Person {
  link former_self -> Person {
    on target delete restrict;
  }
}
```

then you wouldn't be able to delete Lucy the `NPC` once she was connected to Lucy the `MinorVampire`.

- `delete source`: in this case, deleting Lucy the `Person` (the target of the link) will automatically delete Lucy the `MinorVampire` (the source of the link).

- `allow`: this one simply lets you delete the target (this is the default setting).

- `deferred restrict`: forbids you from deleting the target object, unless it is no longer a target object by the end of a transaction. So this option is like `restrict` but with a bit more flexibility.

So if you wanted to have all the `MinorVampire` types automatically deleted when their `Vampire` dies, you would add a link from `MinorVampire` to the `Vampire` type. Then you would add `on target delete delete source` to this: `Vampire` is the target of the link, and `MinorVampire` is the source that gets deleted.

Now let's look at some tips for making queries.

## Using DISTINCT

`DISTINCT` is easy: just change `SELECT` to `SELECT DISTINCT` to get only unique results. We can see that right now there are quite a few duplicates in our `Person` objects if we `SELECT Person.strength;`. It looks something like this:

```
{5, 4, 4, 4, 4, 4, 10, 2, 2, 2, 2, 2, 2, 2, 3, 3}
```

Change it to `SELECT DISTINCT Person.strength;` and the output will now be `{2, 3, 4, 5, 10}`.

`DISTINCT` works by item and doesn't unpack, so `SELECT DISTINCT {[7, 8], [7, 8], [9]};` will return `{[7, 8], [9]}` and not `{7, 8, 9}`.

## Getting `__type__` all the time

We saw that we can use `__type__` to get object types in a query, and that `__type__` always has `.name` that shows us the type's name (otherwise we will only get the `uuid`). In the same way that we can get all the names with `SELECT Person.name`, we can get all the type names like this:

```edgeql
SELECT Person.__type__ {
  name
};
```

It shows us all the types attached to `Person` so far:

```
{
  schema::ObjectType {name: 'default::NPC'},
  schema::ObjectType {name: 'default::Crewman'},
  schema::ObjectType {name: 'default::MinorVampire'},
  schema::ObjectType {name: 'default::Vampire'},
  schema::ObjectType {name: 'default::PC'},
  schema::ObjectType {name: 'default::Sailor'},
}
```

Or we can use it in a regular query to return the types as well. Let's see what types there are that have the name `Lucy`:

```edgeql
SELECT Person {
  __type__: {
    name
  },
  name
} FILTER .name LIKE 'Lucy%';
```

This shows us the objects that match, and of course they are `NPC` and `MinorVampire`.

```
{
  default::NPC {__type__: schema::ObjectType {name: 'default::NPC'}, name: 'Lucy Westenra'},
  default::MinorVampire {__type__: schema::ObjectType {name: 'default::MinorVampire'}, name: 'Lucy'},
}
```

Using `__type__` to display type information is useful when the results don't include this information, for example, when the results are in JSON format. There is a setting you can use to switch format to JSON: just type `\set output-format json-pretty`. If you do that and repeat the previous query, you'll get:

```
{"__type__": {"name": "default::NPC"}, "name": "Lucy Westenra"}
{"__type__": {"name": "default::MinorVampire"}, "name": "Lucy"}
```

To restore the default format type: `\set output-format default`. Because it's so convenient, this book will show results as given with the default format options.

## Being introspective

The keyword `INTROSPECT` allows us to see more details about types. Every type has the following fields that we can access: `name`, `properties` and `links`, and `INTROSPECT` lets us see them. Let's give that a try and see what we get. We'll start with this on our `Ship` type, which is fairly small but has all four. Here are the properties and links of `Ship` again so we don't forget:

```sdl
type Ship {
  property name -> str;
  multi link sailors -> Sailor;
  multi link crew -> Crewman;
}
```

First, here is the simplest `INTROSPECT` query:

```edgeql
SELECT (INTROSPECT Ship);
```

This query isn't very useful to us but it does show how it works: it returns `{schema::ObjectType {id: 28e74d09-0209-11ec-99f6-f587a1696697}}`. Note that `INTROSPECT` and the type go inside brackets; it's sort of a `SELECT` expression for types that you then select again to capture.

Now let's put `name`, `properties` and `links` inside the introspection:

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties,
  links,
};
```

This gives us:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {id: 28e76c59-0209-11ec-adaa-85e1ecb99e47},
      schema::Property {id: 28e94e33-0209-11ec-9818-fb533a2c495f},
    },
    links: {
      schema::Link {id: 28e87ca8-0209-11ec-9ba8-71ef0b23db38},
      schema::Link {id: 28e8ee51-0209-11ec-b47e-8fd9b07debd3},
      schema::Link {id: 29176353-0209-11ec-a6c5-797987ef08b5},
    },
  },
}
```

Just like using `SELECT` on a type, if the output contains another type, property etc. we will just get an id. We will have to specify what we want there as well.

So let's add some more to the query to get the information we want:

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {
    name,
    target: {
      name
    }
  },
  links: {
    name,
    target: {
      name
    },
  },
};
```

So what this will give is:

1. The type name for `Ship`,
2. The properties, and their names. But we also use `target`, which is what a property points to (the part after the `->`). For example, the target of `property name -> str` is `std::str`. And we want the target name too; without it we'll get an output like `target: schema::ScalarType {id: 00000000-0000-0000-0000-000000000100}`.
3. The links and their names, and the targets to the links...and the names of _their_ targets too.

With all that together, we get something readable and useful. The output looks like this:

```
{
  schema::ObjectType {
    name: 'default::Ship',
    properties: {
      schema::Property {name: 'id', target: schema::ScalarType {name: 'std::uuid'}},
      schema::Property {name: 'name', target: schema::ScalarType {name: 'std::str'}},
    },
    links: {
      schema::Link {name: '__type__', target: schema::ObjectType {name: 'schema::Type'}},
      schema::Link {name: 'crew', target: schema::ObjectType {name: 'default::Crewman'}},
      schema::Link {name: 'sailors', target: schema::ObjectType {name: 'default::Sailor'}},
    },
  },
}
```

This type of query seems complex but it is just built on top of adding things like {name} every time you get output that only a machine can understand.

Plus, if the query isn't too complex (like ours), you might find it easier to read without so many new lines and indentation. Here's the same query written that way, which looks much simpler now:

```edgeql
SELECT (INTROSPECT Ship) {
  name,
  properties: {name, target: {name}},
  links: {name, target: {name}},
};
```

[Here is all our code so far up to Chapter 13.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you insert an `NPC` named 'Mr. Swales' who has visited the `City` called 'York', the `Country` called 'England', and the `OtherPlace` called 'Whitby Abbey'? Try it in a single insert.

2. How readable is this introspect query?

   ```edgeql
   SELECT (INTROSPECT Ship) {
     name,
     properties,
     links
   };
   ```

3. What would be the shortest way to see what links from the `Vampire` type?

4. What do you think the output of `SELECT DISTINCT {1, 2} + {1, 2};` will be?

   Hint: don't forget the Cartesian multiplication.

5. What do you think the output of `SELECT DISTINCT {2, 2} + {2, 2};` will be?

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _An old friend returns._
