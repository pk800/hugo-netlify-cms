---
title: A beginners’ guide to brewing with Chemex
date: 2017-01-05T15:04:10.000Z
description: >-
  Brewing with a Chemex probably seems like a complicated, time-consuming
  ordeal, but once you get used to the process, it becomes a soothing ritual
  that's worth the effort every time.
image: /img/blog-chemex.jpg
extra: asdfasdf
---
Entity linking is a core Knowledge Graph service, so let's learn how to use it.<!--more-->

{{% figure src="/img/2018-cer.jpg" class="inliner" %}}

One of the most exciting features we've been working on is \[entity
linking]({{<ref "2018-entity-linking.md">}}), the ability to identify and link
text entities to concepts in a Knowledge Graph. In this short tutorial, we'll
explore a concrete example---how to identify chemical substances in text and
link those references to their respective concepts in DBPedia.

Gathering Data

For each chemical substance we want to identify, we need two things: the
identifier in DBPedia and a list of name variations. Fortunately, DBPedia
provides us with ways of easily acquiring both.

First, let's load the [necessary
data](http://wiki.dbpedia.org/downloads-2016-10) into Stardog.

```bash
stardog-admin db create -o strict.parsing=false -n dbpedia \ 
                           nif_text_links_en.ttl.bz2 \
                           instance_types_transitive_en.ttl.bz2
```

Finding identifiers is easy, since DBPedia has an explicit category type,
`ChemicalSubstance`.

```sparql
PREFIX dbpo: <http://dbpedia.org/ontology/>

SELECT DISTINCT ?substance
WHERE {
    ?substance rdf:type dbpo:ChemicalSubstance .
}
LIMIT 5
```

```
+--------------------------------------------------------+
|                       substance                        |
+--------------------------------------------------------+
| http://dbpedia.org/resource/Hydrochloric_acid          |
| http://dbpedia.org/resource/Sulfuric_acid              |
| http://dbpedia.org/resource/Citric_acid                |
| http://dbpedia.org/resource/Boron_trifluoride          |
| http://dbpedia.org/resource/Ammonia                    |
+--------------------------------------------------------+
```

One way of gathering name variations is by analysing the anchor text used in
hyperlinked knowledge bases such as Wikipedia.

> [Hydrochloric acid](http://en.wikipedia.org/wiki/Hydrochloric_acid)
> regeneration or [HCl](http://en.wikipedia.org/wiki/Hydrochloric_acid)
> regeneration refers to a chemical process for the reclamation of bound and
> unbound HCl from metal chloride solutions.

Given the previous sentence and hyperlinks, we can infer that hydrochloric acid
can be identified by at least two different names: `Hydrochloric acid` and
`HCl`.

DBPedia contains detailed information about text links of Wikipedia pages, and
since those pages are linked to DBPedia concepts, we can use this data to find
name variations of chemical substances. For example, here are the five most
used anchor texts for hydrochloric acid:

```sparql
PREFIX dbpr: <http://dbpedia.org/resource/>
PREFIX its: <http://www.w3.org/2005/11/its/rdf#>
PREFIX nif: <http://persistence.uni-leipzig.org/nlp2rdf/ontologies/nif-core#>

SELECT ?text (COUNT(*) AS ?count)
WHERE {
    []  its:taIdentRef dbpr:Hydrochloric_acid ;
        nif:anchorOf ?text .
}
GROUP BY ?text 
ORDER BY DESC(?count)
LIMIT 5
```

```
+----------------------|-------+
|         text         | count |
+----------------------|-------+
| "hydrochloric acid"  | 872   |
| "HCl"                | 58    |
| "Hydrochloric acid"  | 50    |
| "hydrochloric"       | 38    |
| "hydrochloric acids" | 5     |
+----------------------|-------+
```

With this information in hand, we are able to generate a dataset of chemical
substances and their most common names.

```sparql
PREFIX dbpo: <http://dbpedia.org/ontology/>
PREFIX its: <http://www.w3.org/2005/11/its/rdf#>
PREFIX nif: <http://persistence.uni-leipzig.org/nlp2rdf/ontologies/nif-core#>

SELECT ?text ?substance (count(*) as ?count)
WHERE {
    []  its:taIdentRef ?substance ;
        nif:anchorOf ?text .
    ?substance rdf:type dbpo:ChemicalSubstance .

    FILTER (strlen(?text) > 1)
}
GROUP BY ?text ?substance
HAVING (?count > 1)
ORDER BY DESC(?count)
```

```
+------------------|----------------------------------------------------|-------+
|       text       |                     substance                      | count |
+------------------|----------------------------------------------------|-------+
| "carbon dioxide" | http://dbpedia.org/resource/Carbon_dioxide         | 3870  |
| "quartz"         | http://dbpedia.org/resource/Quartz                 | 1997  |
| "ammonia"        | http://dbpedia.org/resource/Ammonia                | 1981  |
| "sulfur"         | http://dbpedia.org/resource/Sulfur                 | 1819  |
| "glucose"        | http://dbpedia.org/resource/Glucose                | 1777  |
| "ethanol"        | http://dbpedia.org/resource/Ethanol                | 1737  |
| "methane"        | http://dbpedia.org/resource/Methane                | 1599  |
| "ATP"            | http://dbpedia.org/resource/Adenosine_triphosphate | 1526  |
...
+------------------|----------------------------------------------------|-------+
```

Let's save the results of the query to a `csv` file, and proceed to the next
steps.

```bash
stardog query dbpedia -f CSV query.sparql > substances.csv
```

## Identifying Entities

Now that we have a list of substances and their names, we need to tell Stardog
how to identify them in text. For this, we need to create an OpenNLP [name
finder](https://opennlp.apache.org/docs/1.8.4/manual/opennlp.html#tools.namefind).
Since we have an extensive list of name variations, and the domain is quite
restricted, it makes sense to create a simple dictionary-based name finder,
which simply consists of a tokenized list of names.

Note, by the way, that the following code snippets require
[stardog:server](https://www.stardog.com/docs/#_using_maven) dependencies,
and omit some Java boilerplate for clarity.

First, let's load a tokenizer and the previously created CSV.

```java
Tokenizer tokenizer;
try (InputStream model = Files.newInputStream(Paths.get("opennlp/en-token.bin"))) {
  tokenizer = new TokenizerME(new TokenizerModel(model));
}

Reader in = new FileReader("substances.csv");
Iterable<CSVRecord> records = CSVFormat.DEFAULT.withHeader().parse(in);
```

The [tokenizer
model](https://www.stardog.com/docs/#_entity_extraction_and_linking) should be
the same one we're going to use when configuring Stardog later on.

Creating an OpenNLP dictionary name finder is easy: we tokenize each substance
name and add an entry to the dictionary.

```java
Dictionary dictionary = new Dictionary();

for (CSVRecord record : records) {
  String[] tokens = tokenizer.tokenize(record.get("text"));
  dictionary.put(new StringList(tokens));
}

dictionary.serialize(new FileOutputStream("opennlp/en-ner-substances.dict"));
```

## Creating a Linker

The next step is to create a [dictionary
linker](https://www.stardog.com/docs/#_dictionary) by defining a mapping between
names and entity identifiers (i.e., IRIs). Using the tokenizer and CSV from the
previous section, a dictionary linker can be created as follows:

```java
ImmutableMultimap.Builder<String, IRI> links = ImmutableMultimap.builder();

for (CSVRecord record : records) {
  String[] tokens = tokenizer.tokenize(record.get("text"));
  String text = String.join(" ", tokens);
  links.put(text, Values.iri(record.get("substance")));
}

DictionaryLinker.Linker linker = new DictionaryLinker.Linker(links.build());
linker.to(new File("opennlp/substances.linker"));
```

Once again, it's important to always use the exact same tokenizer model,
otherwise linking might not work as expected.

## Putting it all together

Now that we created a name finder and a linker, let's create a new database that
is able to use both. First, we need to make sure that we have all the [necessary
files](https://www.stardog.com/docs/#_entity_extraction_and_linking) in the
`opennlp` folder: `en-ner-substances.dict`, `substances.linker`, `en-token.bin`,
and `en-sent.bin`.

Creating the database requires one extra parameter, the location of the
`opennlp` folder.

```bash
stardog-admin db create -o docs.opennlp.models.path=opennlp/ -n substances
```

## Linking Entities

Now we are ready to use Stardog's entity linking capabilities. The simplest way
to access this functionality is by using query evaluation via our [SPARQL
service](https://www.stardog.com/docs/#_sparql). We provide some text, which
will be analysed by the name finder and linker, returning the findings as query
results.

```sparql
prefix docs: <tag:stardog:api:docs:>

SELECT * {

  SERVICE docs:entityExtractor {
    []  docs:text ?text ;
        docs:mention ?mention ;
        docs:entity ?entity ;
        docs:mode docs:Dictionary
    }
    
    VALUES ?text { "When hydrochloric acid is mixed or reacted with limestone, it produces calcium chloride, a type of salt used to de-ice roads"}
}
```

```
+-------------------------------------------------------|---------------------|-----------------------------------------------+
|                            text                       |       mention       |                    entity                     |
+-------------------------------------------------------|---------------------|-----------------------------------------------+
| "When hydrochloric acid is mixed or reacted with ..." | "hydrochloric acid" | http://dbpedia.org/resource/Hydrochloric_acid |
| "When hydrochloric acid is mixed or reacted with ..." | "salt"              | http://dbpedia.org/resource/Halite            |
| "When hydrochloric acid is mixed or reacted with ..." | "salt"              | http://dbpedia.org/resource/Sodium_chloride   |
| "When hydrochloric acid is mixed or reacted with ..." | "calcium chloride"  | http://dbpedia.org/resource/Calcium_chloride  |
+-------------------------------------------------------|---------------------|-----------------------------------------------+
```

Entity linking is also acessible through our
[BITES](https://www.stardog.com/docs/#_structured_data_extraction) subsystem,
where the input is a document that will be ingested and analysed by Stardog.

```bash
echo "When hydrochloric acid is mixed or reacted with limestone, it produces calcium chloride, a type of salt used to de-ice roads" > document.txt
```

```bash
./stardog doc put --rdf-extractors dictionary substances document.txt
Successfully put document in the document store: tag:stardog:api:docs:substances:document.txt
```

The results of the analysis are available in the document's named graph.

```sparql
PREFIX dcterms: <http://purl.org/dc/terms/>

SELECT * {
    GRAPH <tag:stardog:api:docs:blog_dictlink:document.txt> {
        [] rdfs:label ?mention ;
           dcterms:references ?entity .
    }
}
```

```
+---------------------|-----------------------------------------------+
|       mention       |                    entity                     |
+---------------------|-----------------------------------------------+
| "hydrochloric acid" | http://dbpedia.org/resource/Hydrochloric_acid |
| "salt"              | http://dbpedia.org/resource/Halite            |
| "salt"              | http://dbpedia.org/resource/Sodium_chloride   |
| "calcium chloride"  | http://dbpedia.org/resource/Calcium_chloride  |
+---------------------|-----------------------------------------------+
```

## Learn More

In this short tutorial we've learned how to create dictionary-based name finders
and linkers. Details about extra features and parameters can be found in
Stardog's
[manual](https://www.stardog.com/docs/#_entity_extraction_and_linking).

In future blog posts we'll explore more advanced features, such as learning
custom-trained name finders, and automatic linking to large knowledge bases.
Stay tuned!
