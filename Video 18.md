# More on IR Ranking

## Parallelizing the Search Result Calculations

We can partition InvertedFile by docIDs. Each machine could have all the docIDs that hash to x - say, "14". To calculate search results, each machine calculates its top 10 docs, and then there's one final merge at a central node.

We can partition InvertedFile by term. Each machine could have all the terms that hash to "14". In that case, given a query, each machine looks up the terms from the query that it has (if any), and then the results are merged at a central node, where ranking results are then calculated.

Which method is better? What are the pros and cons?

Problem with partitioning by term

- some machines have to work harder because they're terms are more popular among searchers. Load balancing can be difficult.

Problem with partitioning by document

- asking queries on a very rare term will still require all machines to work on it, even though most machines may not have the term within its documents. It's not that pricey though.

Hence, typically partitioning is done by *document id*. There is an additional join with a snippet of the doc that is shown to the user, using index nested loops join (parallel, over many nodes). Typically ranking is done *before* the join with the docs

## Defining the quality of a non-Boolean Answer

If only the top k answers are retrieved, there are two quality metrics

* Precision - the number of correct retrieved answers / total retrieved
  * The percent of retrieved answers that are correct
* Recall - the number of correct retrieved answers / total correct
  * The percent of correct answers that were retrieved

Typically the user click-throughs are analyzed to estimate the number of correct retreived answers

## Phrases and Proximity Ranking

The query "The Who" is hard to query well based on single terms, since "The" and "Who" appear everywhere. We would rather find the phrase "The Who". You could do this by indexing on 2-word runs in a document (bigrams), and give higher ranking to bigram matches. This can be generalized to n-grams.

But what about getting documents based on the number of words apart? To do proximity ranking, we add a position field to the InvertedFile, as discussed earlier

## Additional Tricks

* Query expansion, suggestions - use search history to add things to the query to get more results
* Fix mispellings
* Document expansion - add terms to a doc before putting it in the inverted file, by classifying them "english", "japanese", etc. in the document.
* Adjust a term's rank by its font, its position in the document (title, header), etc.

Google's search engine has had these tricks, and more, optimized in it for years and years and years.


## Hypertext Ranking

In the web, documents can link to each other. This extra information, this document graph, can be used to help with search.

* If you are important, and you link to me, then I'm important
* This can be done with a recursive computation that requires linear algebra...details not important
* PageRank - recursively share node weight with its neighbors, and continue across the entire graph until the ranks stabilize

## Random Asides

* The web's dictionry of terms is huge. Numerals, codes, mispellings, langauges
* Web spam - trying to get top-rated by optimizing for a particular search engine, by putting popular terms on the page. Or programatically generate a bunch of web sites that are linked to each other, to create a cluster of documents that point to the desired page, to increase the page's ranking
* Can cache the query results of popular queries
* Users don't usually notice minor differences in answers across different devices and times, when the same query is made

## Building a Web Crawler

This was skipped in the video

## Data Visualization

A popular medium

* Commercial products - Tableau, QlikTEch
* Open-source packages - D3, Processing
* Conferences - InfoVis, EuroVis, VAST

So how do you build good data visualizations? How do you *specify* a data visualization?

51:30