---
layout: post-light-feature
title: "Getting started with Azure Search"
description: "When implementing search in a product; Azure Search seems to be the way forward."
category: articles
tags: [azure, search]
date: 2015-02-04
image: 
  feature: search.jpg
---

Last week, we decided we needed to get search working in the backend for one of our products. We had a rather small database that needed to be searchable, so my initial thought was to simply do this with a simple database query over the tables involved.

Then I remembered that Azure had recently released their hosted search solution into preview a few weeks back. So I decided to give it a try.

After a few setbacks with creating a search service in the Azure Portal (something the PM for the product resolved for me after a question on Stack Overflow), I spent 3 hours going from nothing to a deployed solution.

At the moment, Azure Search has a REST-based API. There are no mentions of a .NET SDK on the website, but a third-party implementation is available:

{% highlight html linenos %}
PM> Install-Package RedDog.Search
{% endhighlight %}

This is a [great package](https://github.com/reddog-io/RedDog.Search) that clearly mirrors the REST API, so I had no trouble going on. First you'll need to get your API key, and store it in your configuration somewhere (I put it under `<appSettings>`):

{% highlight xml linenos %}
<appSettings>
    <add key="AzureSearch-Endpoint" value="{serviceName}" />
    <add key="AzureSearch-Key" value="{key}" />
    <add key="AzureSearch-Analyzer" value="no.lucene" />
</appSettings>
{% endhighlight %}

Then you need to decide which fields to put in your index. The index defines the model of your search, and typically you would create a separate index for each type of query you want to run. I wanted to search our question database, so creating the index was easy-peasy:

{% highlight csharp linenos %}
public async Task EnsureIndexesExistAsync()
{
    using (var connection = ApiConnection.Create(
    	this._configuration.AzureSearch.Endpoint, 
    	this._configuration.AzureSearch.Key))
    using (var client = new IndexManagementClient(connection))
    {
        var analyzer = this._configuration.AzureSearch.Analyzer;
        var index = await client.GetIndexAsync("questions");

        if (index.IsSuccess) return;

        await client.CreateIndexAsync(
            new Index("questions")
                .WithStringField(
                	"id", 
                	f => f.IsKey().IsRetrivable())
                .WithStringField(
                	"title", 
                	f => f.IsSearchable().Analyzer(analyzer))
                .WithStringCollectionField(
                	"alternatives", 
            		f => f.IsSearchable().Analyzer(analyzer))
                .WithStringCollectionField(
                	"explanations", 
                	f => f.IsSearchable().Analyzer(analyzer))
                .WithStringCollectionField(
                	"tags", 
                	f => f.IsFacetable().IsSearchable().Analyzer(analyzer)));
    }
}
{% endhighlight %}

There is only one thing that stuck out here, namely that the **key fields** in the search index **has to be strings**. I only figured that out because `RedDog.Search` only takes a string when you are creating an `IndexOperation` (in the next snippet).

The cool part about the code above is the `Analyzer()` call. The analyzer tells Azure Search how it should interpret the text it indexes. In my case, the text is in Norwegian, and the analyzer affects e.g. how stemming of words in queries work. E.g. if you in search for "numbers" with the default analyzer, it will also find results for "number" and "numbering". You need to explicitly set the language, as the rules for which endings can be removed from words is different from language to language. After you've set the analyzer, no extra work is required to make searches like this work.

The `IsFacetable()` call on the tags lets you pivot your results on tags. This is a cool feature that I didn't have a need for at the moment, but I decided to create the index properly in case I changed my mind later.

This piece of code is only run when my web service starts up. When I want to index a question:

{% highlight csharp linenos %}
public async Task StoreAsync(Question question)
{
    using (var connection = ApiConnection.Create(
    	this._configuration.AzureSearch.Endpoint, 
    	this._configuration.AzureSearch.Key))
    using (var client = new IndexManagementClient(connection))
    {
        await client.PopulateAsync(
            "questions",
            this.QuestionToIndexOperation(question));
    }
}

private IndexOperation QuestionToIndexOperation(Question question)
{
    var operation = new IndexOperation(
    	IndexOperationType.MergeOrUpload, 
    	"id", 
    	question.Id.ToStringInvariant());
    
    operation.WithProperty("title", question.Title);

    if (question.Alternatives.Any())
    {
        operation.WithProperty("alternatives", 
        	question.Alternatives.Select(a => a.Text));
        operation.WithProperty("explanations", 
        	question.Alternatives.Select(a => a.Explanation));
    }

    if (question.Tags.Any())
    {
        operation.WithProperty("tags", 
        	question.Tags.Select(t => t.Name));
    }

    return operation;
}
{% endhighlight %}

Now, searching this index is quite easy. I am only interested in the keys for my questions, I can efficiently look them up locally and convert them to DTOs when I know the relevant question IDs:

{% highlight csharp linenos %}
public async Task<IEnumerable<int>> SearchAsync(string query)
{
    using (var connection = ApiConnection.Create(
    	this._configuration.AzureSearch.Endpoint, 
    	this._configuration.AzureSearch.Key))
    using (var client = new IndexQueryClient(connection))
    {
        var results = await client.SearchAsync(
        	"questions", 
        	new SearchQuery(query).Select("id"));
        return results.Body.Records
        	.Select(r => int.Parse(r.Properties["id"].ToString()));
    }
}
{% endhighlight %}

In total, implementing this solution &mdash; including a front-end AngularJS search &mdash; took me a grand total of 3 hours. Not bad for a service that is free for small solutions and is still in preview. [Try it out today](http://azure.microsoft.com/en-us/services/search/).

<small>[Feature image by n8.laverdure on Flickr.](https://www.flickr.com/photos/n8laverdure/3737446721/in/photolist-6GgoGz-5dCKf-99Hfrw-bBRmtC-uXVW-bQGaiB-6URbF8-bQGc1g-568KgN-6o7Soi-o5jh8c-7mzoV-kbHutU-8BwM5v-84Zre3-57SFVF-6bnfSQ-bQL1dg-bBRkQw-4NJBha-hrmWmx-fwfhg2-fr4gn7-nKfGH3-nsNv1j-cm1iZ-5BLsVE-nM7ZfW-bQKZxF-84ZqTh-8kP1uB-8kRPx5-8kNCu2-8kNCt8-8C4YkK-84ZqRG-hABg8Q-7iETXY-eZfSZ-aCMtE3-5jUiVY-4qkNK5-eZYd8H-eUQt5h-5dDGMt-8syFGf-7Vkpud-beq4TM-p91Hvd-47S3zd)</small>