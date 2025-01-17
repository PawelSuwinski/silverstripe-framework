---
title: Upgrading to GraphQL 4
summary: Upgrade your Silverstripe CMS project to use graphQL version 4
---

# Upgrading to GraphQL 4

[alert]
You are viewing docs for a pre-release version of silverstripe/graphql (4.x).
Help us improve it by joining #graphql on the [Community Slack](https://www.silverstripe.org/blog/community-slack-channel/),
and report any issues at [github.com/silverstripe/silverstripe-graphql](https://github.com/silverstripe/silverstripe-graphql).
Docs for the current stable version (3.x) can be found
[here](https://github.com/silverstripe/silverstripe-graphql/tree/3)
[/alert]

The 4.0 release of `silverstripe/graphql` underwent a massive set of changes representing an
entire rewrite of the module. This was done as part of a year-long plan to improve performance. While
there is no specific upgrade path, there are some key things to look out for and general guidelines on how
to adapt your code from the 3.x release to 4.x.

Note that this document is aimed towards developers who have a custom graphql schema. If you are updating from graphql
3 to 4 but do not have a custom schema, you should familiarise yourself with the
[building the schema](/developer_guides/graphql/getting_started/building_the_schema) documentation, but you do not need to read this document.

In this section, we'll cover each of these upgrade issues in order of impact.

## GraphQL schemas require a build step

The most critical change moving from 3.x to 4.x affects the developer experience.
The key to improving performance in GraphQL requests was eliminating the overhead of generating the schema at
runtime. This didn't scale. As the GraphQL schema grew, API response latency increased.

To eliminate this overhead, the GraphQL API relies on **generated code** for the schema. You need to run a
task to build it.

`vendor/bin/sake dev/graphql/build schema=default`

Check the [building the schema](/developer_guides/graphql/getting_started/building_the_schema) documentation to learn about the
different ways to build your schema.

## The `Manager` class, the godfather of GraphQL 3, is gone

`silverstripe/graphql` 3.x relied heavily on the `Manager` class. This became a catch-all that handled
scaffolding, registration of types, running queries and middleware, error handling, and more. This
class has been broken up into two separate concerns:

* [`Schema`](api:SilverStripe\GraphQL\Schema\Schema) <- register your stuff here
* [`QueryHandlerInterface`](api:SilverSTripe\GraphQL\QueryHandler\QueryHandlerInterface) <- Handles GraphQL queries, applies middlewares and context.
  You'll probably never have to touch it.

### Upgrading

**before**
```yaml
SilverStripe\GraphQL\Manager:
  schemas:
    default:
      types: {}
      queries: {}
      mutations: {}
```

**after**
```yaml
SilverStripe\GraphQL\Schema\Schema:
  schemas:
    default:
      src:
        - app/_graphql # A directory of your choice
```

Add the appropriate yaml files to the directory. For more information on this pattern, see
the [configuring your schema](/developer_guides/graphql/getting_started/configuring_your_schema) section.

```
app/_graphql
  types.yml
  queries.yml
  mutations.yml
  models.yml
  enums.yml
  interfaces.yml
  unions.yml
```

## `TypeCreator`, `QueryCreator`, and `MutationCreator` are gone

A thorough look at how these classes were being used revealed that they were really just functioning
as value objects that effectively created configuration in a static context. That is, they had no
real reason to be instance-based. Most of the time, they can easily be ported to configuration.

### Upgrading

**before**
```php
class GroupTypeCreator extends TypeCreator
{
    public function attributes()
    {
        return [
            'name' => 'group'
        ];
    }

    public function fields()
    {
        return [
            'ID' => ['type' => Type::nonNull(Type::id())],
            'Title' => ['type' => Type::string()],
            'Description' => ['type' => Type::string()]
        ];
    }
}
```

**after**

**app/_graphql/types.yml**
```yaml
group:
  fields:
    ID: ID!
    Title: String
    Description: String
```

That's a simple type, and obviously there's a lot more to it than that. Have a look at the
[working with generic types](/developer_guides/graphql/getting_started/working_with_generic_types) section of the documentation
to learn more.

## Resolvers must be static callables

You can no longer use instance methods for resolvers. They can't be easily transformed into generated
PHP code in the schema build step. These resolvers should be refactored to use the `static` declaration
and moved into a class.

### Upgrading

Move your resolvers into one or many classes, and register them. Notice that the name of the method
determines what is being resolved, where previously that would be the name of a resolver class. Now,
multiple resolver methods can exist within a single resolver class.

**before**
```php
class LatestPostResolver implements OperationResolver
{
    public function resolve($object, array $args, $context, ResolveInfo $info)
    {
        return Post::get()->sort('Date', 'DESC')->first();
    }
}
```

**after**

**app/_graphql/config.yml**
```yaml
resolvers:
  - MyProject\Resolvers\MyResolverClassA
  - MyProject\Resolvers\MyResolverClassB
```

```php
class MyResolverClassA
{
    public static function resolveLatestPost($object, array $args, $context, ResolveInfo $info)
    {
        return Post::get()->sort('Date', 'DESC')->first();
    }
}
```

This method relies on [resolver discovery](/developer_guides/graphql/getting_started/working_with_generic_types/resolver_discovery),
which you can learn more about in the documentation.

Alternatively, you can [hardcode the resolver into your config](/developer_guides/graphql/getting_started/working_with_generic_types/resolver_discovery#field-resolvers).

**app/_graphql/queries.yml**
```yaml
latestPost:
  type: Post
  resolver: ['MyApp\Resolvers\MyResolvers', 'latestPost']
```

## `ScaffoldingProvider`s are now `SchemaUpdater`s

The `ScaffoldingProvider` interface has been replaced with [`SchemaUpdater`](api:SilverStripe\GraphQL\Schema\Interfaces\SchemaUpdater).
If you were updating your schema with procedural code, you'll need to implement `SchemaUpdater`
and implement the [`updateSchema()`](api:SilverStripe\GraphQL\Schema\Interfaces\SchemaUpdater::updateSchema()) method.

### Upgrading

Register your schema builder, and change the code.

**before**
```yaml
SilverStripe\GraphQL\Manager:
  schemas:
    default:
      scaffolding_providers:
        - 'MyProject\MyProvider'
```

```php
class MyProvider implements ScaffoldingProvider
{
    public function provideGraphQLScaffolding(SchemaScaffolder $scaffolder)
    {
        // updates here...
    }
}
```

**after**
```yaml
SilverStripe\GraphQL\Schema\Schema:
  schemas:
    default:
      execute:
        - 'MyProject\MyProvider'
```

```php
class MyProvider implements SchemaUpdater
{
    public function updateSchema(Schema $schema): void
    {
        // updates here...
    }
}
```

[alert]
The API for procedural code has been **completely rewritten**. You'll need to rewrite all of the code
in these classes. For more information on working with procedural code, read the
[using procedural code](/developer_guides/graphql/getting_started/using_procedural_code) documentation.
[/alert]

## Goodbye scaffolding, hello models

In `silverstripe/graphql` 3.x, a massive footprint of the codebase was dedicated to a `DataObject`-specific API
called "scaffolding" that was used to generate types, queries, fields, and more from the ORM. In 4.x, that
approach has been replaced with a concept called **model types**.

A model type is just a type that is backed by a class that has awareness of its schema (like a `DataObject`!).
At a high-level, it needs to answer questions like:

* Do you have field X?
* What type is field Y?
* What are all the fields you offer?
* What operations do you provide?
* Do you require any extra types to be added to the schema?

### Upgrading

The 4.x release ships with a model type implementation specifically for DataObjects, which you can use
a lot like the old scaffolding API. It's largely the same syntax, but a lot easier to read.

**before**
```yaml
SilverStripe\GraphQL\Manager:
  schemas:
    default:
      scaffolding:
        types:
          SilverStripe\Security\Member:
            fields: '*'
            operations: '*'
          SilverStripe\CMS\Model\SiteTree:
            fields:
              title: true
              content: true
            operations:
              read: true

```

**after**

**app/_graphql/models.yml**
```yaml
SilverStripe\Security\Member:
  fields: '*'
  operations: '*'

SilverStripe\CMS\Model\SiteTree:
  fields:
    title: true
    content: true
  operations:
    read: true
```

## `DataObject` GraphQL field names are lowerCamelCase by default

The 3.x release of the module embraced an anti-pattern of using **UpperCamelCase** field names so that they could
map directly to the conventions of the ORM. This makes frontend code look awkward, and there's no great reason
for the Silverstripe CMS GraphQL server to break convention. In this major release, the **lowerCamelCase**
approach is encouraged.

### Upgrading

Change the casing in your queries.

**before**
```graphql
query readPages {
  edges {
    nodes {
     Title
     ShowInMenus
    }
  }
}
```

**after**
```graphql
query readPages {
  edges {
    node {
      title
      showInMenus
    }
  }
}
```

### `edges` no longer required

We don't have [cursor-based pagination](https://graphql.org/learn/pagination/) in Silverstripe CMS, so
the use of `edges` is merely for convention. You can eliminate a layer here and just use `nodes`, but `edges`
still exists for backward compatibility.

```graphql
query readPages {
  nodes {
    title
    showInMenus
  }
}
```

## `DataObject` type names are simpler

To avoid naming collisions, the 3.x release of the module used a pretty aggressive approach to ensuring
uniqueness when converting a `DataObject` class name to a GraphQL type name, which was `<vendorName><shortName>`.

In the 4.x release, the typename is just the `shortName` by default, which is based on the assumption that
most of what you'll be exposing is in your own app code, so collisions aren't that likely.

### Upgrading

Change any references to DataObject type names in your queries

**before**
`query SilverStripeSiteTrees {}`

**after**
`query SiteTrees {}`

[info]
If this new pattern is not compatible with your set up (e.g. if you use feature-based namespacing), you have full
control over how types are named. You can use the `type_formatter` and `type_prefix` on
[`DataObjectModel`](api:SilverStripe\GraphQL\Schema\DataObject\DataObjectModel) to influence the naming computation.
Read more about this in the [DataObject model type](/developer_guides/graphql/getting_started/working_with_dataobjects/dataobject_model_type#customising-the-type-name)
docs.
[/info]

## The `Connection` class has been replaced with plugins

In the 3.x release, you could wrap a query in the `Connection` class to add pagination features.
In 4.x, these features are provided via the new [plugin system](extending/plugins).

The good news is that all `DataObject` queries are paginated by default, and you shouldn't have to worry about
this. But if you are writing a custom query and want it paginated, check out the section on
[adding pagination to a custom query](/developer_guides/graphql/getting_started/working_with_generic_types/adding_pagination).

Additionally, the sorting features that were provided by `Connection` have been moved to a plugin dedicated to
`SS_List` results. Again, this plugin is applied to all `DataObject` classes by default, and will include all of their
sortable fields by default - though this is configurable. See the [query plugins](/developer_guides/graphql/getting_started/working_with_dataobjects/query_plugins)
section for more information.

### Upgrading

There isn't much you have to do here to maintain compatibility. If you prefer to have a lot of control over
what your sort fields are, check out the linked documentation above.

## Query filtering has been replaced with a plugin

The previous `QueryFilter` API has been vastly simplified in a new plugin. Filtering is provided to all
read queries by default, and should include all filterable fields including nested relationships - though
this is configurable. See the [query plugins](/developer_guides/graphql/getting_started/working_with_dataobjects/query_plugins)
section for more information.

### Upgrading

There isn't much you have to do here to maintain compatibility. If you prefer to have a lot of control over
what your filter fields are, check out the linked documentation above.

## Query permissions have been replaced with a plugin

This was mostly an internal API, and shouldn't be affected in an upgrade - but if you want more information
on how it works you can [read the permissions documentation](/developer_guides/graphql/getting_started/working_with_dataobjects/permissions).

## Enums are first-class citizens

In the 3.x release, there was no clear path to creating enum types, but in 4.x, they have a prime spot in the
configuration layer.

**before**

(A type creator that has been hacked to return an `Enum` singleton?)

**after**

**app/_graphql/enums.yml**
```yaml
Status:
  SHIPPED: Shipped
  CANCELLED: Cancelled
  PENDING: Pending
```

See the [Enums, unions, and interfaces](/developer_guides/graphql/working_with_generic_types/enums_unions_and_interfaces/#enum-types)
documentation for more information.
