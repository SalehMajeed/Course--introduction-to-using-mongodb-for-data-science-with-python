Q. 1 ->

```python
cursor = (db.laureates.find(
    projection={"firstname": 1, "prizes.year": 1, "_id": 0},
    filter={"gender": "org"})
 .limit(3).sort("prizes.year", -1))

project_stage = {"$project": {"firstname": 1, "prizes.year": 1, "_id": 0}}
match_stage = {"$match": {"gender": "org"}}
limit_stage = {"$limit": 3}
sort_stage = {"$sort": {"prizes.year": -1}}
```

What sequence pipeline of the above four stages can produce a cursor db.laureates.aggregate(pipeline) equivalent to cursor above?

ans. [match_stage, project_stage, sort_stage, limit_stage]

```python

```

Q. 2 -> Translate the above cursor cursor to an equivalent aggregation cursor, saving the pipeline stages to pipeline. Recall that the find collection method's "filter" parameter maps to the "$match" aggregation stage, its "projection" parameter maps to the "$project" stage, and the "limit" parameter (or cursor method) maps to the "$limit" stage.

ans.

```python
# Translate cursor to aggregation pipeline
pipeline = [
    {"$match": {"gender": {"$ne": "org"}}},
    {"$project": {"bornCountry": 1, "prizes.affiliations.country": 1}},
    {"$limit": 3}
]
for doc in db.laureates.aggregate(pipeline):
    print("{bornCountry}: {prizes}".format(**doc))
```

Q. 3 -> Save to pipeline an aggregation pipeline to collect prize documents as detailed above. Use Python's collections.OrderedDict to specify any sorting.

ans.

```python
from collections import OrderedDict
from itertools import groupby
from operator import itemgetter
original_categories = set(db.prizes.distinct("category", {"year": "1901"}))
# Save an pipeline to collect original-category prizes
pipeline = [
    {"$match": {"category": {"$in": list(original_categories)}}},
    {"$project": {"category": 1, "year": 1}},
    {"$sort": OrderedDict([("year", -1)])}
]
cursor = db.prizes.aggregate(pipeline)
for key, group in groupby(cursor, key=itemgetter("year")):
    missing = original_categories - {doc["category"] for doc in group}
    if missing:
        print("{year}: {missing}".format(year=key, missing=", ".join(sorted(missing))))
```

Q. 4 ->

```python
list(db.prizes.aggregate([
    {"$project": {"allThree": {"$setEquals": [____, ____]},
                  "noneThree": {"$not": {"$setIsSubset": [____, ____]}}}},
    {"$match": {"$nor": [{"allThree": True}, {"noneThree": True}]}}]))
```

Which values fill the blanks?

ans. "$laureates.share", ["3"], ["3"], "$laureates.share"

Q. 5 -> Fill out pipeline to determine the number of prizes awarded (at least partly) to organizations. To do this, you'll first need to $match on the "gender" that designates organizations.
Then, use a field path to project the number of prizes for each organization as the "$size" of the "prizes" array. Recall that to specify the value of a field "<my_field>", you use the field path "$<my_field>".
Finally, use a single group {"\_id": None} to sum over the values of all organizations' prize counts.

ans.

```python
# Count prizes awarded (at least partly) to organizations as a sum over sizes of "prizes" arrays.
pipeline = [
    {"$match": {"gender": "org"}},
    {"$project": {"n_prizes": {"$size": "$prizes"}}},
    {"$group": {"_id": None, "n_prizes_total": {"$sum": "n_prizes"}}}
]

print(list(db.laureates.aggregate(pipeline)))
```

Q. 6 -> Make the $group stage output a document for each prize year (set "\_id" to the field path for year) with the set of categories awarded that year.
Given your intermediate collection of year-keyed documents, $project a field named "missing" with the (original) categories not awarded that year. Again, mind your field paths!
Use a $match stage to only pass through documents with at least one missing prize category.
Finally, add sort documents in descending order.

ans.

```python
from collections import OrderedDict
original_categories = sorted(set(db.prizes.distinct("category", {"year": "1901"})))
pipeline = [
    {"$match": {"category": {"$in": original_categories}}},
    {"$project": {"category": 1, "year": 1}},
    # Collect the set of category values for each prize year.
    {"$group": {"_id": "$year", "categories": {"$addToSet": "$category"}}},
    # Project categories *not* awarded (i.e., that are missing this year).
    {"$project": {"missing": {"$setDifference": [original_categories, "$categories"]}}},
    # Only include years with at least one missing category
    {"$match": {"missing.0": {"$exists": True}}},
    # Sort in reverse chronological order. Note that "_id" is a distinct year at this stage.
    {"$sort": OrderedDict([("_id", -1)])},
]
for doc in db.prizes.aggregate(pipeline):
    print("{year}: {missing}".format(year=doc["_id"],missing=", ".join(sorted(doc["missing"]))))
```

Q. 7 -> You can assume (and check!) that the following is true:

assert all(isinstance(v, str) for v in db.laureates.distinct("bornCountry"))

ans.

```python

{"bornCountry": {"$in": db.laureates.distinct("bornCountry")}}

{"$expr": {"$in": ["$bornCountry", db.laureates.distinct("bornCountry")]}}

{"$expr": {"$eq": [{"$type": "$bornCountry"}, "string"]}}

{"bornCountry": {"$type": "string"}}

```

All of the above

Q. 8 -> Use $unwind stages to ensure a single prize affiliation country per pipeline document.
Filter out prize-affiliation-country values that are "empty" (null, not present, etc.) -- ensure values are "$in" the list of known values.
Produce a count of documents for each value of "affilCountrySameAsBorn" (a field we've projected for you using the $indexOfBytes operator) by adding 1 to the running sum.

ans.

```python
key_ac = "prizes.affiliations.country"
key_bc = "bornCountry"
pipeline = [
    {"$project": {key_bc: 1, key_ac: 1}},
    # Ensure a single prize affiliation country per pipeline document
    {"$unwind": "$prizes"},
    {"$unwind": "$prizes.affiliations"},
    # Ensure values in the list of distinct values (so not empty)
    {"$match": {key_ac: {"$in": db.laureates.distinct(key_ac)}}},
    {"$project": {"affilCountrySameAsBorn": {
        "$gte": [{"$indexOfBytes": ["$"+key_ac, "$"+key_bc]}, 0]}}},
    # Count by "$affilCountrySameAsBorn" value (True or False)
    {"$group": {"_id": "$affilCountrySameAsBorn",
                "count": {"$sum": 1}}},
]
for doc in db.laureates.aggregate(pipeline): print(doc)
```

Q. 9 -> $unwind the laureates array field to output one pipeline document for each array element.
After pulling in laureate bios with a $lookup stage, unwind the new laureate_bios array field (each laureate has only a single biography document).
Collect the set of bornCountries associated with each prize category.
Project out the size of each category's set of bornCountries.

ans.

```python
pipeline = [
    # Unwind the laureates array
    {"$unwind": "$laureates"},
    {"$lookup": {
        "from": "laureates", "foreignField": "id",
        "localField": "laureates.id", "as": "laureate_bios"}},
    # Unwind the new laureate_bios array
    {"$unwind": "$laureate_bios"},
    {"$project": {"category": 1,
                  "bornCountry": "$laureate_bios.bornCountry"}},
    # Collect bornCountry values associated with each prize category
    {"$group": {"_id": "$category",
                "bornCountries": {"$addToSet": "$bornCountry"}}},
    # Project out the size of each category's (set of) bornCountries
    {"$project": {"category": 1,
                  "nBornCountries": {"$size": "$bornCountries"}}},
    {"$sort": {"nBornCountries": -1}},
]
for doc in db.prizes.aggregate(pipeline): print(doc)
```

Q. 10 ->

```python
from operator import itemgetter

print(max(docs, key=itemgetter("years")))
print(min(docs, key=itemgetter("years")))
{'firstname': 'Rita', 'surname': 'Levi-Montalcini', 'years': 103.0}
{'firstname': 'Martin Luther', 'surname': 'King Jr.', 'years': 39.0}
```

ans.

```python
{"$project": {"years": 1, "firstname": 1, "surname": 1, "_id": 0}}
```

Q. 11 -> In your aggregation pipeline pipeline, use the "gender" field to limit results to people (that is, not organizations).
Count prizes for which the laureate's "bornCountry" is not also the "country" of any of their affiliations for the prize. Be sure to use field paths (precede a field name with "$") when appropriate.

ans.

```python
pipeline = [
    # Limit results to people; project needed fields; unwind prizes
    {"$match": {"gender": {"$ne": "org"}}},
    {"$project": {"bornCountry": 1, "prizes.affiliations.country": 1}},
    {"$unwind": "$prizes"},
    # Count prizes with no country-of-birth affiliation
    {"$addFields": {"bornCountryInAffiliations": {"$in": ["$bornCountry", "$prizes.affiliations.country"]}}},
    {"$match": {"bornCountryInAffiliations": False}},
    {"$count": "awardedElsewhere"},
]
print(list(db.laureates.aggregate(pipeline)))
```

Q. 12 -> Construct a stage added_stage that filters for laureate "prizes.affiliations.country" values that are non-empty, that is, are $in a list of the distinct values that the field takes in the collection.
Insert this stage into the pipeline so that it filters out single prizes (not arrays) and precedes any test for membership in an array of countries. Recall that the first parameter to <list>.insert is the (zero-based) index for insertion.

ans.

```python
pipeline = [
    {"$match": {"gender": {"$ne": "org"}}},
    {"$project": {"bornCountry": 1, "prizes.affiliations.country": 1}},
    {"$unwind": "$prizes"},
    {"$addFields": {"bornCountryInAffiliations": {"$in": ["$bornCountry", "$prizes.affiliations.country"]}}},
    {"$match": {"bornCountryInAffiliations": False}},
    {"$count": "awardedElsewhere"},
]
# Construct the additional filter stage
added_stage = {"$match": {"prizes.affiliations.country": {"$in": db.laureates.distinct("prizes.affiliations.country")}}}
# Insert this stage into the pipeline
pipeline.insert(3, added_stage)
print(list(db.laureates.aggregate(pipeline)))
```
