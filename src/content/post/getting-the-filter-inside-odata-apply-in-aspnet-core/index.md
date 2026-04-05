---
author: "Alfero Chingono"
title: "Getting the Filter Inside OData $apply in ASP.NET Core"
date: 2025-11-04T17:03:29Z
draft: true
description: "How I ended up solving a subtle OData problem in ASP.NET Core: applying a filter nested inside $apply so count headers stay accurate for grouped queries."
slug: getting-the-filter-inside-odata-apply-in-aspnet-core
tags: [
"dotnet",
"c-sharp",
"aspnet-core",
"odata",
"api"
]
categories: [
"Development",
"ASP.NET Core"
]
image: ""
---

I ran into a small OData problem recently that took just long enough to be annoying.

The request looked like this:

```http
GET /api/v1/participants?$apply=filter(userId eq c925c935-c1ad-4f59-9f68-07456da19d8b)/groupby((role,event/status),aggregate($count as count))&$count=true
```

At first glance it feels like a normal filtering problem. It is not.

The interesting part is that the `filter(...)` is nested inside `$apply`, so `options.Filter` is `null` even though the request absolutely does contain a filter.

That mattered because the frontend was sending grouped stats queries like this and still expected an accurate `odata-count` header back:

```typescript
const path = `participants?$apply=filter(userId eq ${userId})/groupby((role,event/status),aggregate($count as count))&$count=true`
```

Once that count is wrong, the client can still render data, but it has lost the real total behind the grouped result.

## The shape of the problem

The generic controller logic looked like this:

```csharp
[HttpGet("")]
[ProducesResponseType(StatusCodes.Status200OK)]
[EnableQuery(MaxExpansionDepth = 3)]
public virtual IQueryable<TEntity> GetAll(ODataQueryOptions<TEntity> options)
{
    if (Request?.Query.ContainsKey("$count") == true &&
        "true".Equals($"{Request.Query["$count"]}", StringComparison.OrdinalIgnoreCase))
    {
        var queryable = Repository.GetAll();

        if (options?.Filter != null)
        {
            queryable = options.Filter.ApplyTo(queryable, new ODataQuerySettings())
                                    .Cast<TEntity>();
        }
        else if (options?.Apply != null)
        {
            // Need the nested filter from $apply here.
        }

        Response.Headers["odata-count"] = queryable.Count().ToString();
    }

    return Repository.GetAll();
}
```

The normal `$filter=...` case was easy.

The `$apply=filter(...)/groupby(...)` case was the one that needed a closer look.

## Why `options.Filter` stays null

This is the key detail.

OData does not treat a `filter(...)` inside `$apply` as the same thing as a top-level `$filter` query option.

Instead, `$apply` is parsed as a transformation pipeline.

That means the filter is sitting inside the apply tree, not in `options.Filter`.

Once I framed it that way, the right place to inspect was much clearer:

```csharp
options.Apply.ApplyClause.Transformations
```

From there, you can look for a `FilterTransformationNode`.

## The final implementation was better than the original idea

My first instinct was to just extract the `FilterClause` and figure out the rest later.

The final implementation in my project went one step further: it added a small OData extension layer so the controller code could stay clean.

```csharp
public static IQueryable<TEntity> ApplyTo<TEntity>(
    this FilterTransformationNode filterNode,
    IQueryable<TEntity> queryable,
    ODataQueryOptions<TEntity> options)
    where TEntity : class
{
    if (filterNode != null)
    {
        var binder = new FilterBinder();
        var filterExpression = binder.BindFilterClause<TEntity>(filterNode.FilterClause, options);
        queryable = queryable.Where(filterExpression);
    }

    return queryable;
}
```

That changed the problem from:

"How do I manually unpack this nested clause?"

to:

"How do I make `FilterTransformationNode` behave like a first-class query option in my own code?"

I like that version better.

## The important part was binding the clause back to LINQ

The real work was not locating the nested filter. It was turning it back into something the repository query could execute.

In the final code, the helper uses `FilterBinder` and wraps the bound expression into a lambda:

```csharp
public static Expression<Func<TEntity, bool>> BindFilterClause<TEntity>(
    this FilterBinder binder,
    FilterClause filterClause,
    ODataQueryOptions<TEntity> options)
{
    var entityType = typeof(TEntity);
    var settings = new ODataQuerySettings();
    var context = new QueryBinderContext(options.Context.Model, settings, entityType);
    var expression = binder.Bind(filterClause.Expression, context);

    var parameter = Expression.Parameter(
        entityType,
        filterClause.RangeVariable.Name.Replace("$", ""));

    var visitor = new ParameterReplacerVisitor(parameter);
    var lambdaBody = visitor.Visit(expression);

    return Expression.Lambda<Func<TEntity, bool>>(lambdaBody, parameter);
}
```

That parameter replacement step is what made the solution feel complete rather than experimental.

It is not only finding the nested filter.

It is turning it into something the base `IQueryable` can actually use.

## The controller ended up staying simple

Once the extension methods existed, the generic controller logic became pretty clean:

```csharp
if (options?.Filter != null)
{
    queryable = options.Filter.ApplyTo(queryable, new ODataQuerySettings())
                            .Cast<TEntity>();
}
else if (options?.Apply != null)
{
    var filterNode = options.Apply.ApplyClause.Transformations
        .OfType<FilterTransformationNode>()
        .FirstOrDefault();

    if (filterNode != null)
    {
        queryable = filterNode.ApplyTo(queryable, options);
    }
}

Response.Headers["odata-count"] = queryable.Count().ToString();
```

That is the version I trust more than the earlier idea of treating this as a one-off parsing trick.

It keeps the special-case logic close to the actual query shape you care about:

- top-level `$filter` uses the built-in `FilterQueryOption`
- nested `filter(...)` inside `$apply` uses `FilterTransformationNode`

Same intent. Different entry points.

## What this fixed in practice

This was not theoretical cleanup.

It was needed so grouped stats endpoints could return both:

- the grouped aggregate data
- the original filtered total via `odata-count`

That is useful when the client wants grouped cards, charts, or summaries but still needs to know how many entities matched the filter before grouping.

Without this fix, `$count=true` on an `$apply=filter(...)/groupby(...)` request would not respect the nested filter logic in the controller-side count header.

## The real lesson for me

The biggest mental shift was stopping the question from being:

"Why is OData ignoring my filter?"

and turning it into:

"Which part of the query tree owns this filter?"

Once I did that, the behavior made sense.

Top-level `$filter` becomes `options.Filter`.
Nested `filter(...)` inside `$apply` becomes a transformation node.

That distinction explains the whole problem.

## My takeaway

If you need accurate count behavior for grouped OData queries, extracting the filter from `$apply` is only the first step.

The better solution is to make that `FilterTransformationNode` executable against your `IQueryable` in the same way your top-level `$filter` already is.

That was the final shape of the solution in my project:

- find the `FilterTransformationNode`
- bind its `FilterClause`
- apply it to the base query
- compute `odata-count` from the filtered query before returning the unmaterialized result

The parser already knows where the filter is.

The real implementation detail is deciding how to turn that transformation node back into a predicate your query pipeline can use.
