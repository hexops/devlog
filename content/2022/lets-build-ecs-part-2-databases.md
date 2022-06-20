---
author: "Stephen Gutekanst"
title: "Let's build an Entity Component System (part 2): databases"
date: "2022-05-28"
draft: false
categories:
- mach
- zig
- gamedev
- ecs
- build-an-ecs
description: "In this series we build Mach Engine's Entity Component System from first principles in the Zig programming language. In part one we looked at how ECS relates to data oriented design, in part two we look at database design and how it relates to ECS as we begin writing our implementation."
images: ["https://user-images.githubusercontent.com/3173176/149644281-df5a7846-eefb-4482-929b-2ac7243de7a2.png"]
---

<p>
  <img alt="ECS connected to databases and data oriented design" class="color-auto-light" style="height: 20rem; float: left; padding-right: 1rem;" src="https://user-images.githubusercontent.com/3173176/166091237-6e9455df-ede9-4e34-a606-451b0c0c3f2a.png">
  <br><br>
  In this series we build the <a href="https://machengine.org">Mach engine</a> Entity Component System from scratch in <a href="https://ziglang.org">the Zig programming language</a>.
  <br><br>
  In part one, we looked at how ECS intersects with <em>data oriented design</em>, starting without any foundational understanding of how ECS typically works and instead working from first-principles to arrive at what would probably be the most computationally efficient implementation.
  <br><br> 
  In this ~24 page part two, we examine functionality gaps our first approach had, explore how databases relate to ECS, and begin writing our actual implementation in Zig! By the end, you'll have an archetypal ECS with the ability to add/remove entities and components. In part 3, we'll cover queries. 
  <br><br>
  Check out the <a href="https://devlog.hexops.com/categories/build-an-ecs/">prior parts of this series</a> if you haven't already!
</p>

# The case for a general-purpose runtime ECS

In part 1 we proposed an architecture which would have had you end up with [something like this](https://github.com/hexops/mach/blob/dcb5c3aed2ba705d8d0ec854148a90628369f410/ecs/src/main.zig) where you declare an entity archetype _at comptime_, as a struct type:

```zig
const Player = struct {
    name: []const u8, // a string / byte slice
    location: Vec3,
    velocity: Vec3,
    health: u8,
    team: Team,
};
```

In this code, our struct fields `name`, `location`, etc. are said to be our entity _components_ for the `Player` archetype. To create a `Player` entity, we would simply create a value of this type. We proposed using `std.MultiArrayList` to store lists of `Player` entities for efficient CPU cache utilization:

```zig
var players: MultiArrayList(Player) = .{}; // all players
```

This approach is minimal, simplistic, and has an extremely efficient memory layout. Anyone who has used a production-worthy ECS, though, will tell you: _that lacks flexibility._

Here are a few reasons why a general-purpose runtime ECS is more flexible.

## Operating on components, not entities

Imagine you'd like to have your physics system operate on every entity with `velocity` and `location` components. We know that as long as we have those two values, we can do some maths and update the location of an entity to where it should be. But wait, which entities? 

```zig
var players: MultiArrayList(Player) = .{};   // all players
var monsters: MultiArrayList(Monster) = .{}; // all monsters
var cameras: MultiArrayList(Camera) = .{};   // all monsters
var lights: MultiArrayList(Light) = .{};     // all lights
```

We could look at these types as a programmer and find which ones have `velocity` and `location` fields, probably just `players` and `monsters` do. But now, how do we write our physics code to operate on both lists of `Player` and `Monster` entities? We just need the `velocity` and `location` fields - we don't care if it's a player or monster!

## Rapid, iterative, game design

<p>
  <img alt="server giving the stovetop entity a sword, scripting language giving the stovetop entity physics" style="height: 30rem; float:right;" src="https://user-images.githubusercontent.com/3173176/166111509-123f36c1-3421-49bb-8d9f-cb9da7e98514.png">
  If we wish to add a <code>weapon: Weapon</code> component to our <code>Player</code> entity, all good: we update the <code>Player</code> struct to have that field. If we want to add a weapon at runtime, we make it an optional <code>weapon: ?Weapon</code> so it can be <code>null</code>.
  <br><br>
  Let's say we're working on a whacky new cooking simulator game: you've got a kitchen stove, ingredients, utensils, etc. as entities. Some code checks if an entity touches the stovetop and, if it has a <code>cookable: void</code> component, then it gets cooked. If we're trying to build a 100% science-based cooking simulator, well, then we could probably plan ahead and "know" that <code>Ingredient</code> entities should have the <code>cookable</code> component while <code>Utensil</code> entities should not. But often, there's <em>immense joy in strange mechanics:</em> What if utensils were <code>cookable</code>?!
  <br><br>
  Maybe even a game server has made this decision, or a scripting language. We didn't anticipate this at compile time! It'd be great if we could quickly try it out at the flip of a switch, though, while the game is running. And especially without having to track down every codepath handling <code>cookable</code> <code>ingredients</code> to now handle <code>cookable</code> <code>utensils</code>!
</p>

# Runtime components? Just as fast

**We want runtime components in Mach engine for the reasons above, all of which boil down to _rapid, iterative game design_.** Integration of our ECS with a GUI level editor, etc. all require deep levels of runtime introspection of the data in our ECS.

One may assume that runtime ECS will just naturally be slower than a comptime ECS.

It's important to note that just because we're defining components at runtime, it doesn't mean we cannot take special care to follow data oriented design and structure our memory in a way that is very efficient for CPU cache.

# Thinking in terms of databases

With more complex aspects of an ECS, there are just _tradeoffs_, _tradeoffs_, _tradeoffs_ everywhere!

* Querying
  * "find all entities that have Physics and Location components"
  * "find all entities within 5 units distance from (x, y, z)"
  * "find me all player entities whose Name component starts with 'ziggy'"
  * ...
* Indexing queries (how to make complex queries fast?)
* Dense vs. sparse storage
  * "almost every player has a Weapon component"
  * "only a few players have a Weapon component, most don't"

Could there ever be a perfect way to represent ECS data in memory to handle all possible ways someone might want to use it? You might see this as a drawback - there cannot be a perfect ECS! "Maybe that means you shouldn't use one at all" you might think

But, if we begin to think about an ECS as nothing more than an in-memory database for game entities, it's incredibly tempting to draw analogies with traditional databases:

<img style="max-width: 100%; max-height: unset;" src="https://user-images.githubusercontent.com/3173176/166112215-59d1e70d-bddd-4abb-a1dc-688155227b9e.png">

# Pushing the database analogy further

## Multi-threaded queries / writes

A physics system which wishes to calculate physics for any entity with `location` and `velocity` components (columns on any table) ideally can run in parallel with other systems which wish to query and mutate entities.

Such a physics system could interact with the ECS through a "database connection" or "database handle" which synchronizes access (say through table locks, column locks, row locks, etc.) to ensure conflict-free parallel execution with other systems.

Additionally, finding entities with `location` and `velocity` components is as simple as asking: which tables have those columns? Every entity in such a table is guaranteed to have those components, we don't need to check each entity to see if it has those components.

## Indexing queries

<img src="https://user-images.githubusercontent.com/3173176/166115369-6976bfa4-5103-4d24-a833-d79ed1c71110.png">

The natural row-by-row order of database tables is great, but we could have _indexes_ to optimize specific query usage patterns without fundamentally changing our architecture:

* **Spatial index:** Maybe there are 1 million entities spread across a huge area, we only want to find those within 10 meters from our player. A spatial index could utilize an octree to optimize such queries.
* **Graph relation index:** If you anticipate walking up/down a graph of entities (think a GUI / scene graph) a lot, then it's important to have a fast way to lookup a given entity's parent/children - a graph relation index could efficiently keep track of such relations.
* **Generic probability index:** Sometimes you'll need to "find all entities where component X has a value Y", a probability index could maintain [fastfilters](https://github.com/hexops/fastfilter) to statistically answer "these entities likely have value Y (though a few might not)" extremely quickly.
* **Generic function index:** An escape hatch - maybe you want to find all entities where `arbitraryFunction(entity)` returns `true`, a generic index could keep track of when entities (rows) are changed and only invoke `arbitraryFunction` when changes occur.

## Other ECS implementations

After thinking about this anology quite far, writing our implementation around it, etc. I was quite happy to find that I wasn't the only one who thought of this: The Rust Bevy authors [also describe ECS as a data structure in this way](https://bevy-cheatbook.github.io/programming/ecs-intro.html#ecs-as-a-data-structure) and after writing my implementation I got in touch with them to discuss tradeoffs, get advice, etc. (many thanks!)

While this is a helpful analogy to have in the back of your head, we won't take it _too far_ - it's not completely perfect. For example, the database equivalent of 'sparse storage' might be "every row of our "players" table has a foreign key (the row ID of another table with less rows)", but in reality we wouldn't want our ECS sparse storage to pay the cost of storing that ID for every table row: only rows of entities where we want such a component value. Instead, sparse storage in an ECS is more like a mapping of `row ID -> component_value`.

# Writing our ECS implementation in Zig

At this point we've made a large amount of the architecture decisions for our ECS: we understand how ECS relates to data oriented design, databases, and the tradeoffs we'll make with our implementation. From this point on, this series will be much more code-heavy!

## Representing a "world" of entities

The first thing we need is a way to represent our "database of tables", the tables that will contain our entities, usage may look something like:

```zig
var world = Entities.init(allocator); // create a world
defer world.deinit(); // free the world
```

We can define this as a struct:

```zig
pub const Entities = struct {
    allocator: Allocator,

    pub fn init(allocator: Allocator) Entities {
        return .{
            .allocator = allocator,
        };
    }

    pub fn deinit(entities: *Entities) void {
      _ = entities;
      // TODO: release anything we allocate
    }
};
```

So if `Entities` is our "database", we need a way to represent our "tables" (or "archetypes") that will store our actual entities component data:


```zig
 pub const Entities = struct {
     allocator: Allocator,

+    /// A mapping of archetype hash to their storage.
+    ///
+    /// Database equivalent: table name -> tables representing entities.
+    archetypes: std.AutoArrayHashMapUnmanaged(u64, ArchetypeStorage) = .{},

     pub fn init(allocator: Allocator) Entities {
         return .{
             .allocator = allocator,
         };
     }

     pub fn deinit(entities: *Entities) void {
-        _ = entities;
-        // TODO: release anything we allocate
+        var iter = entities.archetypes.iterator();
+        while (iter.next()) |entry| {
+            entry.value_ptr.deinit();
+        }
+        entities.archetypes.deinit(entities.allocator);
     }
 };
```

The `Entities.archetypes` field is a hashmap of _archetype hashes_ (more on these later) to the _archetype storage_, where our actual entities component values will be stored.

`std.AutoArrayHashMapUnmanaged(u64, ArchetypeStorage)` is a bit of a mouth full! If you're keen to understand more about Zig hashmaps, I wrote [a quick article explaining Zig hashmaps](https://devlog.hexops.com/2022/zig-hashmaps-explained) you should check out, but all you need to know is this: it's a hashmap of `u64` keys to `ArchetypeStorage` struct values.

The `deinit` function we've added just iterates over each value in the hashmap and calls `deinit` on it so it has a chance to free it's allocated memory before we free the entire hashmap.

## `ArrayHashMap` as an alternative to sparse sets

Importantly, we use an `ArrayHashMap` here not a regular hash map: an `ArrayHashMap` is actually just backed by an ordered array behind the scenes, and because of this it's optimized for _iteration over the hashmap values_ rather than _hashmap lookups_, since consecutive values are very likely to be in CPU cache.

Critically, we can directly index into the ordered backing array: if we know the index of a table we'd like to lookup, that's a simple O(1) index operation and not a hashmap lookup - we'll take great advantage of this later as an alternative to 'sparse sets' you may read about in other ECS implementations.

## Why our archetype table names are hashes: entities move between tables

You may have noticed we use `u64` values to name our archetype storage tables: why not strings? In a traditional database, these would be strings:

<img src="https://user-images.githubusercontent.com/3173176/166116966-9ad16198-fdcc-4284-bd36-338afc295697.png">

In our ECS, though, there's a trick: tables will not be user-defined, they'll be automatically created and destroyed as needed for you. We'll just put entities into the table that has all the needed columns, and so our `players` and `monsters` tables above would actually just be one big table (since they have identical components) with a name like `has_sword__and__health__and__location`:

<img src="https://user-images.githubusercontent.com/3173176/166117179-31d0449f-6114-4720-932e-02810b203bdd.png">

Let's say we want to give a `player` entity a new component, like a `rotation`, then we'll just create a new table with `has_sword, health, location, rotation` columns and move _just that one entity_ over to the new table.

And so the _names_ for our archetype tables are actually just a hash of all the component names/types! This means that when we add that `rotation` component to another `player` entity, we can merely hash all the component names the entity will _now_ have to quickly check: does a table for storing this archetype of entity already exist, or do we need to create a new one?

## Creating our first archetype table

When we first create an entity, it's not going to have any components. We need a way to represent entities that do not have any components - for this, we'll create a special "void archetype", an empty table where entities will start out:

```zig
+pub const void_archetype_hash = std.math.maxInt(u64);

 pub const Entities = struct {
     ...

-    pub fn init(allocator: Allocator) !Entities {
-        return .{
-            .allocator = allocator,
-        };
+    pub fn init(allocator: Allocator) !Entities {
+        var entities = Entities{ .allocator = allocator };
+
+        try entities.archetypes.put(allocator, void_archetype_hash, ArchetypeStorage{
+            .allocator = allocator,
+            .components = .{},
+            .hash = void_archetype_hash,
+        });
+
+        return entities;
     }
 };
 ```

This puts a single item into our `entities.archetypes` hashmap: `void_archetype_hash` as the key, which will be a special key for entities without any components, and the value is a new `ArchetypeStorage{...}` table, which looks like this:

```zig
pub const ArchetypeStorage = struct {
    allocator: Allocator,

    /// The hash of every component name in this archetype, i.e. the name of this archetype.
    hash: u64,

    /// A string hashmap of component_name -> type-erased *ComponentStorage(Component)
    components: std.StringArrayHashMapUnmanaged(ErasedComponentStorage),

    pub fn deinit(storage: *ArchetypeStorage) void {
        for (storage.components.values()) |erased| {
            erased.deinit(erased.ptr, storage.allocator);
        }
        storage.components.deinit(storage.allocator);
    }
};
```

`ArchetypeStorage` is representing _all entities_ that have the exact same set of component types (think back to our "monsters and players go in one table" diagram above.) The database equivalent of `ArchetypeStorage` is a table, where rows are entities and columns are components (dense storage.)

It's aware of it's own `hash` (table name), and maintains it's own hashmap `components` which maps _component names_ (strings) to the actual place in memory where we store the components' values. Here, this is `ErasedComponentStorage`, a type-erased pointer to `*ComponentStorage(Component)`, which brings us to..

## Storing components in memory

Within our `ArchetypeStorage` database tables, we need to actually store the values for components _somewhere_. And how we represent these in memory is critical. You may recall from Andrew Kelley's [“A Practical Guide to Applying Data-Oriented Design”](https://media.handmade-seattle.com/practical-data-oriented-design/) talk that if we use `std.MultiArrayList` it would store our data in a way that is more efficient for CPU cache, leading to much greater performance. We get to take advantage of that here by storing component values as a struct-of-arrays instead of array-of-structs which, as the talk describes, helps to reduce the in-memory size of our data and ensure more of them are in CPU cache.

A component will be a relatively simple, small value - such as:

```zig
const Location = struct {
    x: f32 = 0,
    y: f32 = 0,
    z: f32 = 0,
};
```

On an entity, there may be _many of these_. Thanks to the database model we have and tables being laid out, we get to store all `Location` component values in contiguous memory which is great for CPU caches:

```zig
/// Represents the storage for a single type of component within a single type of entity.
///
/// Database equivalent: a column within a table.
pub fn ComponentStorage(comptime Component: type) type {
    return struct {
        /// A reference to the total number of entities with the same type as is being stored here.
        total_rows: *usize,

        /// The actual densely stored component data.
        data: std.ArrayListUnmanaged(Component) = .{},

        const Self = @This();

        pub fn deinit(storage: *Self, allocator: Allocator) void {
            storage.data.deinit(allocator);
        }
    };
}
```

So when we have a table of entities that have a `Location` component, our `ArchetypeStorage.components` hashmap will have a `"location"` entry for example that points to a `*ComponentStorage(Location)` densely storing all location values for every entity in the entire table.

## Type-erased component storage

You might recall our `components` hashmap in our table is `ErasedComponentStorage`, not `*ComponentStorage(Component)`:

```zig
pub const ArchetypeStorage = struct {
    ...

    components: std.StringArrayHashMapUnmanaged(ErasedComponentStorage),
};
```

What gives? Well, the problem is that we need to store multiple component types in this hashmap. For example, if this table represents player entities we may need two entries:

* `"weapon"` -> `*ComponentStorage(Weapon)`
* `"location"` -> `*ComponentStorage(Location)`

Here `Weapon` and `Location` are generic type parameters. Our `ArchetypeStorage.components` hashmap can only point to one type of value, though! So we must first turn our `*ComponentStorage(Weapon)` into a type-erased pointer `*anyopaque` (equal to C's `void*`). Of course, this can make working with the values quite difficult because then we don't know what data type they were supposed to have! To aid with this, we introduce a `ErasedComponentStorage` type:

```zig
/// A type-erased representation of ComponentStorage(T) (where T is unknown).
pub const ErasedComponentStorage = struct {
    ptr: *anyopaque,

    // Casts this `ErasedComponentStorage` into `*ComponentStorage(Component)` with the given type
    // (unsafe).
    pub fn cast(ptr: *anyopaque, comptime Component: type) *ComponentStorage(Component) {
        var aligned = @alignCast(@alignOf(*ComponentStorage(Component)), ptr);
        return @ptrCast(*ComponentStorage(Component), aligned);
    }
};
```

This is useful as it allows us to store all of the typed `ComponentStorage(T)` as values in a hashmap despite having different `T` types, and allows us to still interact with them in consistent ways even though we don't remember what the underlying type is. For example, add the requirement that `ErasedComponentStorage` knows how to deinitialize itself:

```zig
 pub const ErasedComponentStorage = struct {
     ptr: *anyopaque,
+    deinit: fn (erased: *anyopaque, allocator: Allocator) void,

     ...
 };
```

When we go to actually create an `ErasedComponentStorage` value, we know the type, and so we can set up a function that does the deinitialization for us:

```zig
pub const Entities = struct {
    ...

    pub fn initErasedStorage(entities: *const Entities, total_rows: *usize, comptime Component: type) !ErasedComponentStorage {
        var new_ptr = try entities.allocator.create(ComponentStorage(Component));
        new_ptr.* = ComponentStorage(Component){ .total_rows = total_rows };

        return ErasedComponentStorage{
            .ptr = new_ptr,
            .deinit = (struct {
                pub fn deinit(erased: *anyopaque, allocator: Allocator) void {
                    var ptr = ErasedComponentStorage.cast(erased, Component);
                    ptr.deinit(allocator);
                    allocator.destroy(ptr);
                }
            }).deinit,
        };
    }

    ...
```

Here `initErasedStorage` is a way for us to say:

> Hey! I anticipate storing `total_rows` of `Component` values in a table, please allocate that for me and give me a `ErasedComponentStorage` value in return.

The first two lines create a pointer where we can store our `ComponentStorage(Component)` struct value itself (not the items inside of it), and initialize `new_ptr.*` with a value:

```zig
var new_ptr = try entities.allocator.create(ComponentStorage(Component));
new_ptr.* = ComponentStorage(Component){ .total_rows = total_rows };
```

Then we create the `ErasedComponentStorage` value, giving it the pointer `*ComponentStorage(Component)` (and erasing that type in the process) `.ptr = new_ptr,`, and create our `deinit` helper:

```zig
.deinit = (struct {
    pub fn deinit(erased: *anyopaque, allocator: Allocator) void {
        var ptr = ErasedComponentStorage.cast(erased, Component);
        ptr.deinit(allocator);
        allocator.destroy(ptr);
    }
}).deinit,
```

`.deinit =` is setting the `deinit` field of `ErasedComponentStorage` to a value. The `(struct { ... }).deinit,` part is just creating an anonymous struct with a `deinit` function in it so we can pick it back out, this is some syntactual cruft [Zig currently requires](https://github.com/ziglang/zig/issues/1717) for writing function expressions.

You'll see that what the function does is pretty simple, though: It takes that `erased: *anyopaque` pointer, casts it back to the typed value `*ComponentStorage(Component)` (since within this function we know what the `Component` type is) and then calls `ptr.deinit(allocator)` which is just a standard method on the `ComponentStorage` struct so it has a chance to free any memory it allocated, before ultimately we ask the allocator `allocator.destroy(ptr)` to destroy the pointer where we're storing that struct value `ComponentStorage(Component)` we allocated earlier.

Now if we have an `*ErasedComponentStorage` value, we can call it's `.deinit` function and it knows how to cast back to the appropriate pointer type before freeing everything. We'll reuse this pattern to do other generic operations on component storage later.

## Managing entity IDs / pointers

At this point we've got:

* `Entities` (our database)
* `ArchetypeStorage` (a table)
* `ErasedComponentStorage` and `ComponentStorage` - the columns in a table

It's time to actually represent entities! We'll do so with just an ID:

```zig
/// An entity ID uniquely identifies an entity globally within an Entities set.
pub const EntityID = u64;
```

We need a way to know which table an entity is stored in, and which row in that table it's component values are located at. You may be tempted to think that `EntityID` could just be _that information_, but remember than when we add or remove a component from an entity it will _move_ between `ArchetypeStorage` tables! When that happens, it's nice if other user code referencing that `EntityID` can stay oblivious to that - so we'll store a mapping of entity IDs to `Pointer` values:

```zig
 pub const Entities = struct {
     ...
+    counter: EntityID = 0,
+
+    /// A mapping of entity IDs (array indices) to where an entity's component values are actually
+    /// stored.
+    entities: std.AutoHashMapUnmanaged(EntityID, Pointer) = .{},

+    /// Points to where an entity is stored, specifically in which archetype table and in which row
+    /// of that table. 
+    pub const Pointer = struct {
+        archetype_index: u16,
+        row_index: u32,
+    };

     pub fn deinit(entities: *Entities) void {
+        entities.entities.deinit(entities.allocator);
         ...
     }
     ...
 };
```

Remember how I said earlier it was important that the mapping of table names -> tables (`Entities.archetypes`) is an _array hash map_, not a regular hash map? That's because we can index directly into it! Say we're given an arbitrary `EntityID` and want to find it's component values, first we would find out which table/row the entity points to via a hash map lookup:

```zig
const ptr: Pointer = entities.entities.get(entity_id).?;
```

Now we know exactly which table and row it's stored in, and can lookup the table, or component values, with simple O(1) array access operations. e.g. to get the archetype table the entity is stored in:

```zig
var archetype = entities.archetypes.entries.get(ptr.archetype_index);
```

Here, `entities.archetypes.entries` is our `AutoArrayHashMapUnmanaged` mapping table names to their `ArchetypeStorage` - but we access the array inside the hash map directly instead of using a hash map lookup.

## Creating an entity

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/169890414-750717b6-f49d-4b9e-af82-08fcf5a663fe.png"><img src="https://user-images.githubusercontent.com/3173176/169889926-8872e7a0-8b98-4353-953f-8691a1b9f738.png"></a>

To create new entities, we'll use an `Entities.new` method that returns a new entity ID by incrementing a global counter in our database:

```zig
 pub const Entities = struct {
     ...

+    /// Returns a new entity.
+    pub fn new(entities: *Entities) !EntityID {
+        const new_id = entities.counter;
+        entities.counter += 1;
+        return new_id;
+    }
 };
```

Initially, an entity will have no components, and thus we'll put it into that special "void" archetype mentioned earlier (this just gives us a guarantee that an entity is _always_ residing in an archetype, even if it has _no components_ - a property that will come in handy later):

```zig
     /// Returns a new entity.
     pub fn new(entities: *Entities) !EntityID {
         const new_id = entities.counter;
         entities.counter += 1;

+        var void_archetype = entities.archetypes.getPtr(void_archetype_hash).?;
+        const new_row = try void_archetype.new(new_id);

         return new_id;
     }
```

`void_archetype` here is, of course, `ArchetypeStorage` (a database table). We're invoking `void_archetype.new(new_id)` to reserve a row in the _void archetype table_, which will be done using this:

```zig
 pub const ArchetypeStorage = struct {
     ...
+    entity_ids: std.ArrayListUnmanaged(EntityID) = .{},

     pub fn deinit(storage: *ArchetypeStorage) void {
         for (storage.components.values()) |erased| {
             erased.deinit(erased.ptr, storage.allocator);
         }
+        storage.entity_ids.deinit(storage.allocator);
         storage.components.deinit(storage.allocator);
     }

+    pub fn new(storage: *ArchetypeStorage, entity: EntityID) !u32 {
+        // Return a new row index
+        const new_row_index = storage.entity_ids.items.len;
+        try storage.entity_ids.append(storage.allocator, entity);
+        return @intCast(u32, new_row_index);
+    }
 };
```

Now `ArchetypeStorage` maintains a mapping of _rows in the table_ (`entity_ids` indices) to the _entity ID_. This will come in handy later - the only important thing to note here is that this is _reserving_ a new row in the table where the entity can live, but it's not actually _allocating the storage for that entity's component values_ yet.

Back over to `Entities` (the database), we need to record which archetype (table), and row number in that table, that the entity ID actually points to (remember-entity IDs are merely pointers to a specific table and row):

```zig
     /// Returns a new entity.
     pub fn new(entities: *Entities) !EntityID {
         const new_id = entities.counter;
         entities.counter += 1;

         var void_archetype = entities.archetypes.getPtr(void_archetype_hash).?;
         const new_row = try void_archetype.new(new_id);
+        const void_pointer = Pointer{
+            .archetype_index = 0, // void archetype is guaranteed to be first index
+            .row_index = new_row,
+        };
+
+        entities.entities.put(entities.allocator, new_id, void_pointer) catch |err| {
+            void_archetype.undoNew();
+            return err;
+        };

         return new_id;
     }
```

Remember how `void_archetype.new` from earlier _reserves_ a row in the table? Well, what happens if we reserved that row, but then fail to record the pointer (`entities.entities.put` OOMs)? In this case, our table has reserved a row for the entity to-be, but we don't have enough memory to record which table/row the entity ID points to. So we need to *undo* that reservation to ensure our table doesn't have an unused (reserved) row. `undoNew` does exactly that:

```zig
 pub const ArchetypeStorage = struct {
     ...

    pub fn new(storage: *ArchetypeStorage, entity: EntityID) !u32 {
        // Return a new row index
        const new_row_index = storage.entity_ids.items.len;
        try storage.entity_ids.append(storage.allocator, entity);
        return @intCast(u32, new_row_index);
    }

+    pub fn undoNew(storage: *ArchetypeStorage) void {
+        _ = storage.entity_ids.pop();
+    }
 };
```

Since the call to `new` merely appended a value to `entity_ids`, we only need to `pop` the last value off in order to undo the call to `new` and effectively unreserve the row we last reserved.

At this point, the following test works:

```zig
test "ecs" {
    const allocator = testing.allocator;

    var world = try Entities.init(allocator);
    defer world.deinit();

    const player = try world.new();
    _ = player;
}
```

(A copy of the full code at this point [is available here](https://gist.github.com/slimsag/477f9f4c68667e71fbe584a700cfd87d) and you can run it using `zig test ecs.zig`)

## Working with component storage

As we work with component storage (table columns, where component values for a single type `T` are stored contiguously in memory) we're going to need some helper functions. The first one is to swap remove a value from a column:

```zig
 pub fn ComponentStorage(comptime Component: type) type {
     return struct {
         ...
         data: std.ArrayListUnmanaged(Component) = .{},

+        pub fn remove(storage: *Self, row_index: u32) void {
+            if (storage.data.items.len > row_index) {
+                _ = storage.data.swapRemove(row_index);
+            }
+        }
```

Note that `ComponentStorage` memory is lazily allocated, so we only remove if the table column does in fact have storage allocated.

Next up is a simple copy function, a specific row's value from `src` to `dst`:

```zig
+        pub inline fn copy(dst: *Self, allocator: Allocator, src_row: u32, dst_row: u32, src: *Self) !void {
+            try dst.set(allocator, dst_row, src.get(src_row));
+        }
```

And a helper to get the actual component value from a column:

```zig
+        pub inline fn get(storage: Self, row_index: u32) Component {
+            return storage.data.items[row_index];
+        }
```

Here we do not need to check the length of `storage.data.items`, because we assert the column must have that row.

## Working with type-ErasedComponentStorage

As we discussed earlier, we often won't have a typed `ComponentStorage(T)` and instead will have `ErasedComponentStorage`. We need a few more helpers to operate on the columns of a table, this time without knowing the underlying data type.

### Cloning ComponentStorage types

The first helper we need is the ability to create a new value of type `ComponentStorage(T)` when we don't know the actual type of `T`:

```zig
 /// A type-erased representation of ComponentStorage(T) (where T is unknown).
 pub const ErasedComponentStorage = struct {
     ptr: *anyopaque,
     ...
+    cloneType: fn (erased: ErasedComponentStorage, total_entities: *usize, allocator: Allocator, retval: *ErasedComponentStorage) error{OutOfMemory}!void,
```

The goal of this helper is purely to clone the type, let's see how it is implemented when when initializing erased component storage:

```zig
 pub const Entities = struct {
     ...

     pub fn initErasedStorage(entities: *const Entities, total_rows: *usize, comptime Component: type) !ErasedComponentStorage {
         ...

         return ErasedComponentStorage{
             ...
+            .cloneType = (struct {
+                pub fn cloneType(erased: ErasedComponentStorage, _total_rows: *usize, allocator: Allocator, retval: *ErasedComponentStorage) !void {
+                    var new_clone = try allocator.create(ComponentStorage(Component));
+                    new_clone.* = ComponentStorage(Component){ .total_rows = _total_rows };
+                    var tmp = erased;
+                    tmp.ptr = new_clone;
+                    retval.* = tmp;
+                }
+            }).cloneType,
         };
     }
```

Notably this _doesn't copy the actual values in the component storage_, it's just creating a new `ComponentStorage(T)` for us - just the `struct` value, as if we'd written `ComponentStorage(T){.total_rows = total_rows}`! This doesn't allocate storage for the rows, it just says we anticipate there will be that many.

### Copying component values between tables

The second helper we'll need is a way to copy a component value from one `ComponentStorage(T)` column row to another of the same type `T`, again when we don't know the underlying type and just have two `ErasedComponentStorage` we need to copy a value between:

```zig
 /// A type-erased representation of ComponentStorage(T) (where T is unknown).
 pub const ErasedComponentStorage = struct {
     ptr: *anyopaque,
     ...
+    copy: fn (dst_erased: *anyopaque, allocator: Allocator, src_row: u32, dst_row: u32, src_erased: *anyopaque) error{OutOfMemory}!void,
```

The implementation is simple:

```zig
 pub const Entities = struct {
     ...

     pub fn initErasedStorage(entities: *const Entities, total_rows: *usize, comptime Component: type) !ErasedComponentStorage {
         ...
         return ErasedComponentStorage{
             ...
+            .copy = (struct {
+                pub fn copy(dst_erased: *anyopaque, allocator: Allocator, src_row: u32, dst_row: u32, src_erased: *anyopaque) !void {
+                    var dst = ErasedComponentStorage.cast(dst_erased, Component);
+                    var src = ErasedComponentStorage.cast(src_erased, Component);
+                    return dst.copy(allocator, src_row, dst_row, src);
+                }
+            }).copy,
```

### Removing a row from a column

Removing a single component value from a column in a table looks as you'd expect:

```zig
 /// A type-erased representation of ComponentStorage(T) (where T is unknown).
 pub const ErasedComponentStorage = struct {
     ptr: *anyopaque,
     ...
+    remove: fn (erased: *anyopaque, row: u32) void,
```

The implementation is simple:

```zig
 pub const Entities = struct {
     ...

     pub fn initErasedStorage(entities: *const Entities, total_rows: *usize, comptime Component: type) !ErasedComponentStorage {
         ...
+        return ErasedComponentStorage{
+            ...
+            .copy = (struct {
+                pub fn copy(dst_erased: *anyopaque, allocator: Allocator, src_row: u32, dst_row: u32, src_erased: *anyopaque) !void {
+                    var dst = ErasedComponentStorage.cast(dst_erased, Component);
+                    var src = ErasedComponentStorage.cast(src_erased, Component);
+                    return dst.copy(allocator, src_row, dst_row, src);
+                }
+            }).copy,
```

```zig
 pub const Entities = struct {
     ...

     pub fn initErasedStorage(entities: *const Entities, total_rows: *usize, comptime Component: type) !ErasedComponentStorage {
         ...
         return ErasedComponentStorage{
             ...
+            .remove = (struct {
+                pub fn remove(erased: *anyopaque, row: u32) void {
+                    var ptr = ErasedComponentStorage.cast(erased, Component);
+                    ptr.remove(row);
+                }
+            }).remove,
```

## Removing an entire row from a table

When we want to remove an _entire row_ (all column values), we need to invoke `remove` on each column:

```zig
 pub const ArchetypeStorage = struct {
     ...
+    pub fn remove(storage: *ArchetypeStorage, row_index: u32) !void {
+        _ = storage.entity_ids.swapRemove(row_index);
+        for (storage.components.values()) |component_storage| {
+            component_storage.remove(component_storage.ptr, row_index);
+        }
    }
```

This also swap removes the row from the table `entity_ids` mapping of row indices -> entity ID.

## Adding components

Now on to the fun part: adding components to our entity. Suppose we want to do this:

```zig
 test "ecs" {
     ...
 
+    const Location = struct {
+        x: f32 = 0,
+        y: f32 = 0,
+        z: f32 = 0,
+    };

+    try world.setComponent(player, "Name", "jane"); // add Name component
+    try world.setComponent(player, "Location", Location{}); // add Location component
+    try world.setComponent(player, "Name", "joe"); // update Name component
 }
```

When we call `setComponent`, we may be updating the value of an existing component OR adding a new component to the entity. If the latter, we need to move the entity to a new archetype table (which may or may not exist!) - so the first thing we need to do is figure out: what archetype table is this entity currently in, and where does it need to be?

```zig
 pub const Entities = struct {
     ...

+    pub inline fn archetypeByID(entities: *Entities, entity: EntityID) *ArchetypeStorage {
+        const ptr = entities.entities.get(entity).?;
+        return &entities.archetypes.values()[ptr.archetype_index];
+    }

+    pub fn setComponent(entities: *Entities, entity: EntityID, name: []const u8, component: anytype) !void {
+        var archetype = entities.archetypeByID(entity);

+        const old_hash = archetype.hash;

+        var have_already = archetype.components.contains(name);
+        const new_hash = if (have_already) old_hash else old_hash ^ std.hash_map.hashString(name);
+   };
```

`archetypeByID` takes an entity ID and gives us an actual memory pointer to the `*ArchetypeStorage` table where the entity is stored. It does this by looking up the entity ID (`entities.entities.get(entity).?`) so we know which `archetype_index` it is stored in, and then simply returns a pointer to that table.

Then `setComponent` first finds out which archetype table the entity is stored in _now_, and determines if that table already has the component we're adding/updating the entity with. Recall how earlier we mentioned [why our archetype table names are hashes](#why-our-archetype-table-names-are-hashes-entities-move-between-tables) - the hash here is simply a hash of every component name stored by the archetype table.

### Creating new archetype tables

As noted earlier, when adding a new component to an entity (say going from components `(Location, Name)` -> `(Location, Name, Weapon)`) it will move from the old table to the new one. If the new one doesn't exist yet we need to create it. We know this because `new_hash` is the name of the table, encompassing all the component types it stores:

```zig
 pub const Entities = struct {
     ...
     pub fn setComponent(entities: *Entities, entity: EntityID, name: []const u8, component: anytype) !void {
         ...

+        var archetype_entry = try entities.archetypes.getOrPut(entities.allocator, new_hash);
+        if (!archetype_entry.found_existing) {
+            archetype_entry.value_ptr.* = ArchetypeStorage{
+                .allocator = entities.allocator,
+                .components = .{},
+                .hash = 0,
+            };
+            var new_archetype = archetype_entry.value_ptr;
```

Merely creating a new `ArchetypeStorage` table is not enough, though - we need to create storage columns in the table to store all of the existing components found on the entity `(Location, Name)` - by iterating the components in the old table, which we don't know the actual type of (they're `ErasedComponentStorage`, not `ComponentStorage(Location)`), so we use a `cloneType` helper (which we'll define later):

```zig
+            var column_iter = archetype.components.iterator();
+            while (column_iter.next()) |entry| {
+                var erased: ErasedComponentStorage = undefined;
+                entry.value_ptr.cloneType(entry.value_ptr.*, &new_archetype.entity_ids.items.len, entities.allocator, &erased) catch ...;
+                new_archetype.components.put(entities.allocator, entry.key_ptr.*, erased) catch ...;
+            }
```

And finally, create storage / a new column for the new `component` we're adding to the entity (`Weapon`):

```zig
+            // Create storage/column for the new component.
+            const erased = entities.initErasedStorage(&new_archetype.entity_ids.items.len, @TypeOf(component)) catch ...;
+            new_archetype.components.put(entities.allocator, name, erased) catch ...;
+
+            new_archetype.calculateHash();
+        }
+     }
...
```

You may have noticed we wrote `catch ...` in the snippets above, these are simply written as:

```zig
catch |err| {
    assert(entities.archetypes.swapRemove(new_hash));
    return err;
}
```

The reason for this is simple: If we fail to clone the storage columns, or add the new storage column, then we failed to create the archetype storage table! In this case, we need to clean up after ourselves so as to not leave the database in a bad state - by removing the entry we added to `entities.archetypes` earlier.

Finally, we implement `ArchetypeStorage.calculateHash`:

```zig
 pub const ArchetypeStorage = struct {
     ...
     hash: u64,

+    pub fn calculateHash(storage: *ArchetypeStorage) void {
+        storage.hash = 0;
+        var iter = storage.components.iterator();
+        while (iter.next()) |entry| {
+            const component_name = entry.key_ptr.*;
+            storage.hash ^= std.hash_map.hashString(component_name);
+        }
+    }
```

This simply walks over each `storage.components` entry (the columns in the table), and hashes the component type names.

### Making `Entities.setComponent` update component values

At this point our `setComponent` method finds the `archetype_entry` table that needs to be updated, and has created it if necessary:

```zig
 pub const Entities = struct {
     ...
     pub fn setComponent(entities: *Entities, entity: EntityID, name: []const u8, component: anytype) !void {
         ...
         var archetype_entry = try entities.archetypes.getOrPut(entities.allocator, new_hash);
         if (!archetype_entry.found_existing) {
             // ... creates new archetype table
         }
```

Now it's time to actually _update_ the table, putting our component values into it:

```zig
+        var current_archetype_storage = archetype_entry.value_ptr;
+
+        if (new_hash == old_hash) {
+            const ptr = entities.entities.get(entity).?;
+            try current_archetype_storage.set(ptr.row_index, name, component);
+            return;
+        }
```

Here, `current_archetype_storage` is going to be either the new storage table (if the entity moved from an old table to a new table), or the prior storage table (if we're just updating the value of a component that was already on the entity.) We then compare `new_hash == old_hash` and, if equal, that implies we're just updating the value of the existing component on the entity.

Now, if we're moving the entity to a new table, things are a bit more involved. First, we need to copy all component values for our entity from the _old archetype storage table_ to the _destination storage table_. We do this by creating a new row in the destination table, iterating each component value in the old row, and copying it over:

```zig
+        const new_row = try current_archetype_storage.new(entity);
+        const old_ptr = entities.entities.get(entity).?;
+
+        var column_iter = archetype.components.iterator();
+        while (column_iter.next()) |entry| {
+            var old_component_storage = entry.value_ptr;
+            var new_component_storage = current_archetype_storage.components.get(entry.key_ptr.*).?;
+            new_component_storage.copy(new_component_storage.ptr, entities.allocator, new_row, old_ptr.row_index, old_component_storage.ptr) catch |err| {
+                current_archetype_storage.undoNew();
+                return err;
+            };
+        }
```

We also need to update the table's mapping of `entity_ids` (row indices -> entity ID):

```zig
+        current_archetype_storage.entity_ids.items[new_row] = entity;
```

And since we only copied over the old components -> the new table row, we don't yet have the _new component_ in the _new table row_ - it's undefined memory at present. We update it:

```zig
+        current_archetype_storage.set(new_row, name, component) catch |err| {
+            current_archetype_storage.undoNew();
+            return err;
+        };
```

At this point, our entity would be in the new table! The new table has a new row with all of our component values, too! But the old table row still exists: we need to remove it.

### Removing the old table row, updating pointers

We'll use a swap removal (swapping the row that may be in the middle of the table somewhere, with the last row in the table, and finally decreasing the size of the table by one.)

```zig
+        var swapped_entity_id = archetype.entity_ids.items[archetype.entity_ids.items.len - 1];
+        archetype.remove(old_ptr.row_index) catch |err| {
+            current_archetype_storage.undoNew();
+            return err;
+        };
```

Notably, `archetype.remove` swap removes `old_ptr.row_index` from the table. But in doing so, our global mapping of `entities` entity ID -> entity ptr has become invalid! So we correct it:

```zig
+        try entities.entities.put(entities.allocator, swapped_entity_id, old_ptr);
```

Last but not least, the entity we were using `setComponent` on has moved to a new archetype table, and a new row. We update it's pointer in the global `entities` map:

```zig
+        try entities.entities.put(entities.allocator, entity, Pointer{
+            .archetype_index = @intCast(u16, archetype_entry.index),
+            .row_index = new_row,
+        });
+        return;
```

### Setting the value of a component in a table

Earlier in `Entities.setComponent` we had invoked `ArchetypeStorage.set`:

```zig
        if (new_hash == old_hash) {
            const ptr = entities.entities.get(entity).?;
            try current_archetype_storage.set(ptr.row_index, name, component);
            return;
        }
```

This function will just find the `ErasedComponentStorage` (column storage) for `name`, cast it to the type of `component` so we have `ComponentStorage(T)` and update `row_index` to have the value `component`:

```zig
 pub const ArchetypeStorage = struct {
     ...

+    pub fn set(storage: *ArchetypeStorage, row_index: u32, name: []const u8, component: anytype) !void {
+        var component_storage_erased = storage.components.get(name).?;
+        var component_storage = ErasedComponentStorage.cast(component_storage_erased.ptr, @TypeOf(component));
+        try component_storage.set(storage.allocator, row_index, component);
+    }
};
```

Over in `ComponentStorage`, we implement the `set` method - which is quite simple - if the data array isn't large enough (we haven't actually allocated storage for the row yet), then we allocate it to `undefined` memory - and finally we set `row_index` to the `component` value:

```zig
 pub fn ComponentStorage(comptime Component: type) type {
     return struct {
         total_rows: *usize,
         data: std.ArrayListUnmanaged(Component) = .{},

         const Self = @This();

+        pub fn set(storage: *Self, allocator: Allocator, row_index: u32, component: Component) !void {
+            if (storage.data.items.len <= row_index) try storage.data.appendNTimes(allocator, undefined, storage.data.items.len + 1 - row_index);
+            storage.data.items[row_index] = component;
+        }
    };
}
```

## Finally, we can create entities *and* add/update components on them!

`Entities.setComponent` is fully implemented! These lines from our test earlier now work:

```zig
    try world.setComponent(player, "Name", "jane"); // add Name component
    try world.setComponent(player, "Location", Location{}); // add Location component
    try world.setComponent(player, "Name", "joe"); // update Name component
```

A copy of the full code at this point [is available here](https://gist.github.com/slimsag/7c3d36a8324dd733fb0377b087ed057c) and you can run it using `zig test ecs.zig` as before.

## Getting component values

Getting component values is pretty simple:

```zig
 pub const Entities = struct {
     ...
+    pub fn getComponent(entities: *Entities, entity: EntityID, name: []const u8, comptime Component: type) ?Component {
+        var archetype = entities.archetypeByID(entity);
+
+        var component_storage_erased = archetype.components.get(name) orelse return null;
+
+        const ptr = entities.entities.get(entity).?;
+        var component_storage = ErasedComponentStorage.cast(component_storage_erased.ptr, Component);
+        return component_storage.get(ptr.row_index);
+    }
```

This finds the `archetype` table the entity is stored in, then finds the `components` column the named component is stored in, and finally casts the `ErasedComponentStorage` -> `ComponentStorage(Component)` so we can get the row value. Notably, this means _both the name of the component and the provided type must be correct_, or else undefined behavior could occur. This is a fatal flaw in our ECS implementation which we will fix!

## Removing components, entities

Removing components is similar to adding them (because the entity needs to move between ArchetypeStorage tables.) Removing entities is similar as well. The code is lengthy, and nearly identical, so we won't cover it here.

## Conclusions

By this point you have a relatively solid archetypal ECS. The full source code for this article [is available here](https://gist.github.com/slimsag/aecbf725896d2947459ba915fc9103a7
).

Notably, it is lacking the following which we'll cover in part 3:

* **Querying** of actual entites, iterators over components for a single entity, etc.
* **Type-safety**, as noted earlier if you pass the wrong name / component type it will result in undefined behavior!
* ... and more

[`mach/ecs` is available on GitHub](https://github.com/hexops/mach/tree/main/ecs), slightly ahead of this series and changing rapidly. Once it becomes stable, it will also be available as a standalone Zig library anyone can use in their own engine/game.

Follow [@machengine](https://twitter.com/machengine) on Twitter for updates, and join [our Matrix chat room](https://matrix.to/#/#ecs:matrix.org) for ECS discussion & to help us reach Mach 1.0.

## Support my work

If you like my work on [Mach engine](https://machengine.org), [zigmonthly.org](https://zigmonthly.org), etc. you can [sponsor me on GitHub](https://github.com/sponsors/slimsag).
