---
layout: post-kaiqi
title: "Using GraphQL dotnet with Sitecore index search"
date: 2020-03-18
categories:
  - Sitecore
  - GraphQL
description: 
image: https://picsum.photos/2000/1200?image=3072
image-sm: https://picsum.photos/500/300?image=3072
---
# Using GraphQL dotnet with Sitecore index search

## What have been achieved in this POC

-   In this POC, a GraphQL endpoint has been build, which supports two top level queries for `country` and `destinations` respectively. We integrated the GraphQL endpoint with SOLR indexes to retrieve data. We also did the following performance optimizations:
    -   Moved filtering to ORM by using a QueryBuilder
    -   Solved `N + 1 problem` in queries using Data Loader
    -   Implemented field projection by dynamically build the expression

## Why we want to use GraphQL

-   There are too many blog posts out there for you if you want to know when to use GraphQL. Check [this blog](https://www.altexsoft.com/blog/engineering/graphql-core-features-architecture-pros-and-cons/) It's main advantages includes:
    -   Fetching data with a single API call.
    -   No over- and under-fetching problems.
    -   Tailoring requests to your needs.
    -   Validation and type checking out-of-the-box.
    -   Auto-generating API documentation.
    -   API evolution without versioning.

### GraphQL-Dotnet

-   It's also known that Sitecore JSS also provides a GraphQL implementation, and it is depending on [GraphQL-Dotnet](https://www.nuget.org/packages/GraphQL/2.0.0), which is also why it is the library that we are exploring in this POC. You should be able to find out the documentation [here](https://graphql-dotnet.github.io/docs/getting-started/introduction).

## Walk through of the POC

### Approach

#### GraphType First Approach

-   In this POC we are using a [GraphType First](https://graphql-dotnet.github.io/docs/getting-started/introduction#graphtype-first-approach) Approach. Which gives you access to all of the provided properties of your `GraphType`'s and `Schema`.

#### Multiple Queries

-   `Queries` needs to be defined to fetch data. Normally you would have a single root Query object, this can make the root query bloat with unrelated functionality. In this case, two top level queries have been set up, each of them targeting at a specific type of entity.
-   For better understanding, I would like to consider queries as the **entry point** of the graph. Or a **starting point** of retrieving the graph. When you fetching data from either of the query, you are going through the same graph, just from different starting point.
-   In really world example, I think it would be common for complex data structures to have multiple top level queries to have different entry points of the graph.

## Implementation

### Basic models setup

-   In this POC, we are using a very simple models. `Destination` and `Country`, where `Country` has a list of `Destinations`. It is not complex, but enough to test most of the functionalities. The `one-to-many` relationship between `Destination` and `Country` is also good to be used as an example to be optimized.

![GraphQLPOCModel](/assets/image/2020-03-18-GraphQL-with-Sitecore/GraphQLPOCModel.png)

#### Country

`CountryGraphType` have been setup to map to the `Country` class. It has a few properties includes `name`, `countryName`, `currencyCode` and `destinations`.
The `destinations` are a list of `DestinationGraphType`, which is the second model we are setting up.

```c#
public CountryGraphType(IFakeDb fakeDb, IDataLoaderContextAccessor accessor)
{
    Name = "Countries";
    Field(d => d.Name, nullable: true).Description("The name of the country item.");
    Field(d => d.CountryName, nullable: true).Description("The name of the country.");
    Field(d => d.CurrencyCode, nullable: true).Description("The currency code of the country.");

    Field<ListGraphType<DestinationGraphType>, IEnumerable<Destination>>()
        .Name("destinations")
        .Argument<StringGraphType>("name", "")
        .ResolveAsync(ctx => GetDestinationsAsync(ctx, fakeDb, accessor)
        );
}
```

#### Destination

`DestinationGraphType` have been setup to map to the `Destination` class.

The relationship back to the Country has not been setup due to simplicity concerns.

```c#
public DestinationGraphType()
{
    Name = "Destinations";
    Field(d => d.DestinationName, nullable: true).Description("The name of the destination item.");
    Field(d => d.Name, nullable: true).Description("The name of the destination.");
}
```

### Multiple top level queries

We also have dedicated query for each of the type above. The reason behind using multiple top level query is that to make the data retrieval more flexible. For example, when you only have a country level query, when you only want to get the info related to a specific destination, you still need to have the following query:

```json
query {
    countryTopQuery{
        countries{
            destinations(name: "Seoul"){
                name
            }
        }
    }
}
```

And the response will be full of of empty lists of destinations:

```json
{
    "data": {
        "countryTopQuery": {
            "countries": [
                {
                    "destinations": []
                }
                {
                    "destinations": []
                },
                ...
                {
                    "destinations": [
                        {
                            "name": "Seoul"
                        }
                    ]
                },
                ...
                {
                    "destinations": []
                }
            ]
        }
    }
}
```

But if you have a dedicated query just for destinations, you would have just enough data for your need.

#### Show me your code

The top level queries in this POC are as following.

```c#
public class DestinationQuery : ObjectGraphType
{
    public DestinationQuery(IFakeDb fakeDb)
    {
        Name = "destinationQuery";
        Field<ListGraphType<DestinationGraphType>>("destinations",
            arguments: new QueryArguments(
                new QueryArgument<StringGraphType> { Name = "name" }
            ),
            resolve: fakeDb.GetDestinations);
    }
}

public class CountryQuery : ObjectGraphType
{
    public CountryQuery(IFakeDb fakeDb)
    {
        Name = "countryQuery";

        Field<ListGraphType<CountryGraphType>>("countries",
            arguments: new QueryArguments(
                new QueryArgument<StringGraphType> { Name = "name" }
            ),
            resolve: fakeDb.GetCountries);
    }
}
```

The key of having multiple top level queries are still have a root query but have a empty `resolve`. [More info](https://graphql-dotnet.github.io/docs/getting-started/query-organization).

```c#
public class Query : ObjectGraphType
{
    public Query()
    {
        Name = "Query";
        Field<CountryQuery>("countryTopQuery", resolve: context => new { });
        Field<DestinationQuery>("destinationTopQuery", resolve: context => new { });
    }
}
```

### `N + 1 problem` in queries

It's common knowledge that having less hits to the db/indexes will result in a better performance in general. `N + 1` problem is a problem of firing up too many db/index/Api calls. For example, for a following data response, typically there will be `one` request for getting all countries info, for "Cook Islands", "India", "Canada" and "Ethiopia". And in the meantime, `four (N)` independent requests will be fired to get destinations for "Cook Islands", "India", "Canada" and "Ethiopia". Hence the `N + 1` problem.

![Query demo](/assets/image/2020-03-18-GraphQL-with-Sitecore/2020-03-17-21-27-54.png)

#### Data loader

[Data loader](https://graphql-dotnet.github.io/docs/guides/dataloader) is a good way of resolving this issue. And has been implemented by the GraphQL.Net library.

By `batching` the keys for loading items, in this case, `countryId`, one request will be called asynchronously at the end of the process with all the aggregated keys, in this case, the `countryId` of "Cook Islands", "India", "Canada" and "Ethiopia". So in total, there will be only two requests, one for all the countries, one for the destinations for those countries. Some [ASP.NET Core examples](https://github.com/fiyazbinhasan/GraphQLCore/tree/Part_X_DataLoader).

#### Show me your code

Let's get back in the code, you might already notice the `ResovleAsync` function call in the type `CountryGraphType`.

The second parameter of the function `GetOrAddCollectionBatchLoader` is a function which takes a list of Ids as input and returns a `Task<ILookup<Guid, Destination>>`. It is also the function that will be called when the data loader get all the Ids to filter by. Be aware that using `ILookup` is because one countryId will map to many `Destinations`. If one `Guid` here will map to one `Destination`, we would need a `IDictionary` here.

![The data loader](/assets/image/2020-03-18-GraphQL-with-Sitecore/The%20data%20loader.png)

You also need to make sure that when you are converting the `ILookup/IDictionary`, the key is the same for the `LoadAsync`. The library will use it to map the collections of results back to the entities. In this case, we are using `x => x.Country` as the key of the result(`x.Country` is actually of type `Guid`, represents the `id` of the `country` for the `destination`) which would map back to the `ctx.Source.Id` which is the id of the country we used to filter destinations by.
![The key for look up](/assets/image/2020-03-18-GraphQL-with-Sitecore/The%20key%20for%20Lookup.png)

As a result the following query

```json
query {
    countryTopQuery{
        countries{
            countryName
            currencyCode
            name
            destinations{
                name
            }
        }
    }
}
```

Has triggered the following SOLR queries

```
10920 22:26:04 INFO  Solr Query - ?q=*:*&rows=2147483647&fl=countryname_s,currencycode_s,_name,_group,_uniqueid,_datasource&fq=((_latestversion:(True) AND parsedlanguage_s:(english_australia)) AND _template:(9c7c84fe4c51469e9693ed3ac375dfda))&fq=_indexname:(jq_countries_web_index)

10920 22:26:04 INFO  Solr Query - ?q=(country_s:(b2669fd14a4a48e1b60084187870ac44) OR (country_s:(67865f242c1c42f49d22f691fcce4510) OR (country_s:.......(3fab15b2db8040c98ff657c35825b02e) OR (country_s:(2e61d08332de499a85b9b9269dff499f) OR country_s:(4cb2a8fc3ed34bac88561165ac983542)))))))&rows=2147483647&fq=parsedlanguage_s:(english_australia)&fq=_indexname:(jq_destinations_web_index)
```

One query for each type. The first one is for countries and the second one is for destinations filtered by their `countryIds`.

### Move filtering to ORM

I am little disappointed to see that, you have to deal with the arguments your self. In other words, you need to write the logic for the filtering on each of the collections. In other words, you have to declare the applicable arguments for a collection, and then, write your own logic to handle them. For example, when you want to filter items by their names.

#### QueryBuilder

A good practice is to do the filtering in SQL/Index queries, rather than loading all items in memory and then do the filtering, which ended up over-fetching useless data. So I reckon have a `queryBuilder` would be beneficial in this case. The `queryBuilder` is an object which holds all the filter related values. And we can pass it to the async method, which will be responsible to compose the query based on the `queryBuilder`.

#### Show me your code

For example, when you want to be able to filter the list of Destinations of a country by name, this is what you have to do":

![Query builder](/assets/image/2020-03-18-GraphQL-with-Sitecore/Query%20Builder.png)

Then, in the async method:
![Usage of the query builder](/assets/image/2020-03-18-GraphQL-with-Sitecore/Usage%20of%20the%20query%20builder.png)

-   Surely there are better ways to organize the logic using design patterns, this is just a POC. Don't judge me! :)

By doing this, we can let the ORM to handle the filtering, so that as long as you are using LINQ query that can be translated by the ORM, the filtering will happen in DB/index queries.

As a result, for the following GraphQL query

```json
query {
    countryTopQuery{
        countries{
            destinations(name:"Delhi"){
                name
            }
        }
    }
}
```

The following SOLR query will be fired to fetched the destinations whose name is `Delhi` with some other filters.

```
24656 22:33:16 INFO  Solr Query - ?q=((country_s:(b2669fd14a4a48e1b60084187870ac44) OR (country_s:.....................(2e61d08332de499a85b9b9269dff499f) OR country_s:(4cb2a8fc3ed34bac88561165ac983542)))) AND _name:(Delhi))&rows=2147483647&fq=parsedlanguage_s:(english_australia)&fq=_indexname:(jq_destinations_web_index)
```

### Field projection

While moving filtering in ORM reduced the over-fetching for the number of rows, field projection would help with reducing the number of columns for each row.

Because most of the ORM would support doing the field projection using a MemberAssignment expression such as the following format.

```c#
...select(c => new Country{
    CurrencyCode = c.CurrencyCode,
    Name = c.Name,
} )...
```

#### Expression Builder

Inspired by this [answer](https://stackoverflow.com/a/41391508), I write a helper for composing such expression dynamically with the fields you need to select give a map between the GraphQL fields and the actual member fields in the model.

#### Show me your code

```c#
private readonly IDictionary<string, string> _nameMapping;

public FakeDb(...)
{
    ...
    // Name in the GraphQL query, Name in the class
    _nameMapping = new Dictionary<string, string>
    {
        {"name", nameof(Country.Name)},
        {"countryName", nameof(Country.CountryName)},
        {"currencyCode", nameof(Country.CurrencyCode)},
        {"id", nameof(Country.Id)}
    };
}

public Expression<Func<T, T>> CreateExpression<T>(List<string> keys) where T : Country
{
    // the field that is used for the mapping in GraphQL
    keys.Add("id");

    var param = Expression.Parameter(typeof(T));

    var allAssignments = keys
        .Select(key =>
                Expression
                    .Bind(typeof(T).GetMember(_nameMapping[key]).First(),
                        Expression.PropertyOrField(param, _nameMapping[key])))
        .ToList();


    var body = Expression.MemberInit(
        Expression.New(typeof(T)), allAssignments.ToArray());

    var exp = Expression.Lambda(body, param);
    return (Expression<Func<T, T>>)exp;
}
```

When you are using it, you can just pass in the keys requested by the GraphQL query.

```c#
...
// filter out the keys that are not simply a column in db/indexes
var simpleFields = resolveContext.SubFields
    .Where(sf => !sf.Value.SelectionSet.Children.Any())
    .Select(sf => sf.Key).ToList();
...

result = searchContext.GetQueryable<Country>()
    .Filter(p)
    .Select(CreateExpression<Country>(simpleFields.ToList()))
    .ToList();
...

```

## More to solve

### Caching

We are using Akamai to caching most of our API calls. Obviously this is not as easy as caching for RESTFUL Apis where you can compose the cache key out of the URL and the parameters. [Here](https://developer.akamai.com/blog/2018/10/29/overview-graphql-query-parsing-and-caching-edge)'s Akamai's suggestion for caching GraphQL Queries.
Basically, remove all the whitespaces, order the arguments and then use the hashed value for the query as part of the cache key.

They also provided a dedicated product [API Gateway](https://developer.akamai.com/akamai-api-gateway) which has built-in functionalities for caching GraphQL queries. It charges based on traffic.

### Error detection

Error detection can also be tricky for the GraphQL queries since it would always return 200 and put the actually error inside of the response. Akamai's API gateway also have built-in functionality to detect the errors within the response. How to properly log those errors should also be investigated.

![Error in GraphQL](/assets/image/2020-03-18-GraphQL-with-Sitecore/Error&#32;in&#32;GraphQL.png)

### Huge post data

When a party trying to get more and more data using queries, the query itself will get bigger and bigger. The server might need more computing power the de serialize queried from POST data which could end up downgrading the performance of the endpoint. The concept of [Fragment](https://graphql.org/learn/queries/#arguments) in GraphQL might be helpful in this case.

### Security

Exposing a GraphQL endpoint can also introduce some security challenges. For example, one of the starting point of conducting a DDOS attack is to find a most time consuming/ in-efficient request on the website. With a GraphQL endpoint, it's not that hard to find one. We might need to looking at only whitelisting certain queries to the endpoint. And adding some protections in the API can also be a good idea.

### Json package issue

There is some inconsistency between the version of the Json package between Sitecore 8 and GraphQL.Net.

Happy coding! :)