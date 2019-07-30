# More on IR Ranking

## Parallelizing the Search Result Calculations

We can partition InvertedFile by docIDs. Each machine could have all the docIDs that hash to x - say, "14". To calculate search results, each machine calculates its top 10 docs, and then there's one final merge at a central node.

We can partition InvertedFile by term. Each machine could have all the terms that hash to "14". In that case, given a query, each machine looks up the terms from the query that it has (if any), and then the results are merged at a central node, where ranking results are then calculated.

Which method is better? What are the pros and cons?

Problem with partitioning by term

- some machines have to work harder because they're terms are more popular among searchers. Load balancing can be difficult.

Problem with partitioning by document

- asking queries on a very rare term will still require all machines to work on it, even though most machines may not have the term within its documents

22:00

