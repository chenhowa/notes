# Element IR: Scalable Boolean Text Search

## Information Retrival (From Documents of Text)

This field was traditionally separate from Databases, in Academia, Industry, and Products. But web search made this a big deal due to search and advertising for documents, because it exploded the number of documents that were both digital AND had interesting data.

## Comparing IR and DBMS

The expected results have different semantics - IR is imprecise, DBMS is precise. IR has keyword search, while DMBS uses things like SQL. IR has unstructured data, while DBMS has structured data. IR has read-mostly data, with very few updates/inserts, that are done in batches. DBMS expects quite a few udpates. IR pages through top *k* results per page. DMBS generates the full answer.

Can we look at documents and see that the data is actually structured?

## IR Architecture Stack

* Search String Modifier
* Ranking Algorithm
* The Generated Query - needs to be very fast
* The Access Method
* Since no update/inserts, the OS can take care of the storage

### "Bag of Words" Model

IR Typically thinks the data is just a big bag of words (set, no structure) What can we do to improve this?

* *Stop Words* - some words have no meaning, so they are not included in the bag.
  * "the"
  * HTML tags like "H1"
* *Stemming* - use langauge-specific rule to convert certain words to a basic form
  * "surfing", "surfed" --> "surf"
  * Have to do this on a per-language basis
  * Lots of open-source libraries

### Boolean Text Search 

Find all docs that match a boolean expression - "Windows" AND ("Glass" OR "Door") AND NOT "Microsoft". You can see that the query terms were "stemmed"/"stopped"

"text index" - in IR, it is a logical schema (table) over the document, with a B+ tree on this logical schema, that is typically stored in files in a regular OS file system

#### Simple Relational Text Index

Given a corpus of text files, where each file has columns (docID [the key], content), we create an "inverted file" table that has columns (stemmed/stopped term, docID). The inverted file has something like an "Alternative 3" index on term where you have lists of docIDs (*in sorted order*) at the bottom of the index. This allows for single-word queries

To handle Boolean Logic, OR is the union of two docID lists. AND is the intersection of two docID lists. AND NOT is just set subtraction between two docIDs lists. OR NOT is the union of a union with term 1 and NOT term2. NOT is all the docIDs that don't contain the term.

OR NOT and NOT are pretty expensive, and are typically not allowed (or are ignored) in IR engines.

Boolean Search in SQL is very simple. It's just a bunch of JOINs on the inverted file table.

#### What if you want phrases?

Augment the Inverted file to have columns (term, docID, position in doc). Now terms can be stored multiple times per document. Make a new index on term, but the lists are now (docID, position), sorted by "position". Then to get the phrase "Happy" AND "Days", find docs with "Happy" and "Days" where the positions of the two are 1 off, and merge the lists. Observe that sorting by "position" is important for this, for obvious reasons.

A similar thing works for `term1` NEAR `term2`, where you try to figure out if *position diff* < *k*

Another thing you might want to do is split up a term into "Trigrams" (substrings of length 3). So "Database" becomes [ "dat", "ata", "tab", ... and so on ]. This allows for pattern search (looking for "data" means finding "dat" and "ata" ). It also lets you do approximate matches - for example, "databake" matches on quite a few trigrams, so it's close. Without trigrams, it would be hard to do approximate matches, since the data you'd have to scan through (even just a word) would be hard to find, since you'd have to scan the entire inverted table for terms that were close to the search term. Trigrams seem to work much better in English than 4-grams and 5-grams

#### What if you want the document content?

Recall that we have the tables InvertedFile(term, position, docID), and Files(docID, URL, content). If we built B+Tree indexes on InvertedFile.term and Files.docID, we can do a join between the list of docIDs and Files.docID, to generate (docID, content) pairs. Also, this join can be done lazily, since search results are viewed just 1 page at a time. This improves performance still further. Note that since search results typically return such large result sets, intermediate results are *never materialized* (put on disk). So the output is done entirely as a lazy stream.

But how can we sort by ranking without generating all the output?

#### Updates and Text Search

Text Search engines are designed to be query-mostly. Deletes and mofiications are rare. So updates are postponed

* Multiple indexes may be used -- 1 for old data, 1 for 2-day old data, 1 for 1-day old data, 1 for deleted data 2 days ago, and so on. Thus, the updated data is not necessarily immediately merged. The results of searching the indexes can then be merged sometime later, in a batch (deletions are subtracted from the final result). It's just not worth the IO's to be keeping the tables all together and up-to-date. This method is called *log-structured merge indices*.
* You can build other indicies on free machines, and then replace the existing indices with the newly constructed indicies.
* Since deletes and modfications are rare, compression can be used to speed index lookups.

#### Additional Tricks for Search Results

* Ranking
* Visualization of document clustering (related documents)
* Compression
* Dealing with misspellings, acronyms, and synonyms
* How to write a good web crawler
* Search engine optimization (don't be vulnerable to web spam)
* Customization for individual users

We only discuss ranking in more depth

IR lingo:

* Corpus - collection of documents
* Term - searchable unit
* index: mechanism to map terms to documents
* Inverted File = Postings File - file containing terms and associated postings list
* Postings list - list of pointers to documents

### Calculating Rank of Search Results

Classical IR Ranking was done with the vector space model. So we treat each document as a vector, where every unit vector is a term. Hence, every document is an array of possible terms. *Then a query is itself just a short document*, and you can rank all documents with a distance function to the query document

What is the right distance function? There are some problems:

* Two long docs will be more similar to each other (far from the origin) than to short docs (near origin).
  * Solution - normalize each vector to be a unit vector. Now every document is a point on the unit sphere. Now the cosine of the angle between two vectors serves as a good ranking function
* Not all dimensions in a document are equally important
  * We want to favor repeated terms in a document
  * We want to favor unusual words in the document.
  * We do this with (Term Frequency) x (Inverse Document Frequency) = TD x IDF
    * docTermRank = occurences of t in d x (log (total # of docs / # docs with this term))
    * We let docTerm Rank be the component of the term in the overall vector

If we want to find the ranking quickly, we need to make additional indexes. We have a new table TermInfo (term, numDocs), and we update the Inverted File schema to be InvertedFile(term, docID, DocTermRank float), which caches the ranking, which will be updated occasionally

####  Assigning Ranking

* Only rank results that match the search query (heuristic).
  * It's possible that a document exists that has the highest rank that doesn't match the search query
  * We could do better by considering results that *mostly* match the search query, by dropping part of the query
* For each docID that matches, compute cosine distance to query, and rank accordingly. But sorting is expensive! We don't want to have to sort 10 million results out-of-core.

How do we make the sort go fast? We parallelize them, as will be discussed in the next lecture.






