---
versionFrom: 8.0.0
versionTo: 8.0.0
---

# Examine Events

_Examine events are ways to modify the data being indexed._

:::note
This document has been verified for Umbraco 8.

If you are using Umbraco 9 or later versions, please refer to the note on the [Examine documentation landing page](index.md) for more details.
:::

## TransformingIndexValues

The TransformingIndexValues event allows you to manipulate the data that will be indexed during an index operation. With this event you can add, remove or modify the data that is going into the index which can be helpful in many scenarios.

### Example

In the [Quick Start](Quick-Start/index.md) documentation you can see how to perform a search with Examine. That is great if you want to search between node names or you know that you always want to search for a specific field - e.g. `bodyText`.

However, what if you want to search through several different node types and search across many different fields, you will typically need to have a query that looks like this:

```csharp
var textFields = new[] { "title", "description",  "content", .... };
var results = searcher.CreateQuery("content")
                    .GroupedOr(textFields, searchTerm)
                    .Execute();
```

This can be simplified. Instead you can use TransformingIndexValues event (used to be called GatheringNodeData in Umbraco 7) to add a custom field to the indexed data to combine the data from several fields and then you can search on that one field. This is done via a [composer](../../../Implementation/Composing/index.md).

## Creating a TransformingIndexValues event

:::note
This example will build upon the Umbraco Starterkit as it is a good starting point for some content. Feel free to use your own site, but some content may need to be generated.
:::

So to add a TransformingIndexValues event we will add a controller that inherits from `IComponent`. Something like this:

```csharp
public class ExamineEvents : IComponent
{
    private readonly IExamineManager _examineManager;
    private readonly IUmbracoContextFactory _umbracoContextFactory;
    private readonly IScopeProvider _scopeProvider;

    public ExamineEvents(IExamineManager examineManager, IUmbracoContextFactory umbracoContextFactory, IScopeProvider scopeProvider)
    {
        _examineManager = examineManager;
        _umbracoContextFactory = umbracoContextFactory;
        _scopeProvider = scopeProvider;
    }

    public void Initialize()
    {
        if (!_examineManager.TryGetIndex(Constants.UmbracoIndexes.ExternalIndexName, out IIndex index))
           throw new InvalidOperationException($"No index found by name {Constants.UmbracoIndexes.ExternalIndexName}");

        //we need to cast because BaseIndexProvider contains the TransformingIndexValues event
        if (!(index is BaseIndexProvider indexProvider))
            throw new InvalidOperationException("Could not cast");

        indexProvider.TransformingIndexValues += IndexProviderTransformingIndexValues;
    }

    private void IndexProviderTransformingIndexValues(object sender, IndexingItemEventArgs e)
    {
        //will be added in next step
    }

    public void Terminate()
    {
        //unsubscribe during shutdown
       indexProvider.TransformingIndexValues -= IndexProviderTransformingIndexValues;
    }
}
```

You can read more about this [syntax and Components here](../../../Implementation/Composing/index.md). We can now add the logic to combine fields in the `IndexProviderTransformingIndexValues` method:

```csharp
if (e.ValueSet.Category == IndexTypes.Content)
{
    var combinedFields = new StringBuilder();

    foreach (var fieldValues in e.ValueSet.Values)
    {
        foreach (var value in fieldValues.Value)
        {
            if (value != null)
            {
                combinedFields.AppendLine(value.ToString());
            }
        }
    }

    //Accessing the Umbraco Cache code will be added in the next step.
    //combinedFields.AppendLine(GetBreadcrumb(e.ValueSet.Values["id"].FirstOrDefault()?.ToString()));

    //The TryAdd() method has been deprecated (Examine 3/Umbraco 10) and has been replaced by a clone/update process.
    //Old Method:
    e.ValueSet.TryAdd("combinedField", combinedFields.ToString());
    
    //New Method:
    //Clone the dictionary
    var updatedValues = e.ValueSet.Values.ToDictionary(x => x.Key, x => x.Value.ToList());
    
    //Add Values
    updatedValues.Add("combinedField", new List<object> { combinedFields.ToString() });
    
    //Update the item to utilize the cloned/updated dictionary
    e.SetValues(updatedValues.ToDictionary(x => x.Key, x => (IEnumerable<object>)x.Value));


    
}
```

So at this point we have done something along the lines of:
- Get the ExternalIndex
- Get the valueset for the ExternalIndex
- For each field, add the content to a new field called `combinedField`

Next is an optional step to highlight how you can access the Umbraco published cache during an indexing event. In this example, we will be adding the breadcrumb values to our combined field.

```csharp
private string GetBreadcrumb(string id)
{
    if (int.TryParse(idString, out var id))
    {
        using(var scope = _scopeProvider.CreateScope(autoComplete: true))
        {
            using (var umbracoContextReference = _umbracoContextFactory.EnsureUmbracoContext())
            {
                var nodeBeingIndexed = umbracoContextReference.UmbracoContext.Content.GetById(id);
                return nodeBeingIndexed.Ancestor != null ?  
                    string.Join(" ", nodeBeingIndexed.AncestorsOrSelf().Select(n => n.Name)) :
                    string.Empty;
            }
        }
    }
}
```

:::note
This example is here to highlight the user of the Scope Provider. A Scope is required for many things in Umbraco, from controlling database transactions to how service level events are handled to how nucache manages it's snapshots (and more). Although Umbraco tries to ensure there is a Scope available, this is not always the case, especially when working in background threads / events. As such, it is important to wrap calls to Umbraco Services/Contexts in a Scope, as without this unusual errors can occur, including the disposal of objects still required. *For more information on this, please see the answer to this [forum question](https://our.umbraco.com/forum/using-umbraco-and-getting-started/102676-triggering-index-rebuild-via-hangfire-causes-objectdisposedexception-in-nucache)*
:::

Before this works the component will have to be registered in a composer. If you already have a composer you can add it to that one, but for this example we will make a new composer:

```csharp
//This is a composer which automatically appends the ExamineEvents component
public class ExamineComposer : ComponentComposer<ExamineEvents>, IUserComposer
{
    // you could override `Compose` if you wanted to do more things, but if it's just registering a component there's nothing else that needs to be done.
}
```

We append it so it runs as the last one. Now if you start up your website and [rebuild your indexes](examine-management.md), then you can find the new field in the search results:

![Example of adding a Transforming Index Values field](images/transforming-index-values.png)

At this point you can create a query for only that field:

```csharp
var results = searcher.CreateQuery("content").Field("combinedField", searchTerm).Execute();
```

### Adding the path of the node as a searchable field into the index

Sometimes you might want to search descendants of a node rather than immediate children for a particular search term. In this instance, we need to write out the `path` property as a searchable field into the index. The `path` property is a comma-separated string. Due to the presence of the comma in the path, the field is not tokenized therefore it is not searchable. We will inject new
field with the path that has a comma removed this making it searchable. This can be done by adding the new field using to `IndexProviderTransformingIndexValues` method. So continuing from the above example the method will look like below.

```csharp
if (e.ValueSet.Category == IndexTypes.Content)
{
    var combinedFields = new StringBuilder();

    var pathBuilder = new StringBuilder();

    foreach (var fieldValues in e.ValueSet.Values)
    {
        foreach (var value in fieldValues.Value)
        {
            if (value != null)
            {
                combinedFields.AppendLine(value.ToString());
            }
        }
    }

    if (fieldValues.Key == "path")
    {
        foreach (var value in fieldValues.Value)
        {
            if (value != null)
            {
                var path = value.ToString().Replace(",", " ");
            }
        }
    }

    //Accessing the Umbraco Cache code will be added in the next step.
    //combinedFields.AppendLine(GetBreadcrumb(e.ValueSet.Values["id"].FirstOrDefault()?.ToString()));

    //The TryAdd() method has been deprecated (Examine 3/Umbraco 10) and has been replaced by a clone/update process.
    //Old Method:
    e.ValueSet.TryAdd("combinedField", combinedFields.ToString());
    e.ValueSet.TryAdd("searchPath", pathBuilder.ToString());
    
    //New Method
    //Clone the dictionary
    var updatedValues = e.ValueSet.Values.ToDictionary(x => x.Key, x => x.Value.ToList());
    
    //Add Values
    updatedValues.Add("combinedField", new List<object> { combinedFields.ToString() });
    updatedValues.Add("searchPath", new List<object> {  });
    
    //Update the item to utilize the cloned/updated dictionary
    e.SetValues(updatedValues.ToDictionary(x => x.Key, x => (IEnumerable<object>)x.Value));
   
    
}
```

The path of a node usually follows the format -1,1066,1234,1236 as an example where 1236 is the current node. In the code we split the path, loop through them and append it to a stringbuilder object, inserting a whitespace before and after the value. Finally we add the value to the stringbuilder object into a new field in the index.

Once this is done you can update your search query to search the descendants of a particular node. In this example the query searches all descendants of node id 1105 for the search term.

```c#
var results = searcher.CreateQuery("content").Field("searchPath","1105").And().Field("combinedField", searchTerm).Execute();
```
