Q. 1 ->
What expression asserts that the distinct Nobel Prize categories catalogued by the "prizes" collection are the same as those catalogued by the "laureates"? Remember to explore example documents in the console via e.g. db.prizes.find_one() and db.laureates.find_one().

ans.

```python
assert set(db.prizes.distinct("category")) == set(db.laureates.distinct("prizes.category"))
```

Q. 2 -> Return a set of all such countries as countries.

ans.

```sql
# Countries recorded as countries of death but not as countries of birth
countries = set(db.laureates.distinct("diedCountry")) - set(db.laureates.distinct("bornCountry"))
print(countries)
```

Q. 3 -> Determine the number of distinct countries recorded as part of an affiliation for laureates' prizes. Save this as count.

ans.

```sql
# The number of distinct countries of laureate affiliation for prizes
count = len(db.laureates.distinct('prizes.affiliations.country'))
print(count)
```

Q. 4 -> In which countries have USA-born laureates had affiliations for their prizes?

```python
db.laureates.distinct("prizes.affiliations.country",{"bornCountry":"USA"})
```

ans. Australia, Denmark, United Kingdom, USA

Q. 5 -> Save a filter document criteria that, when passed to db.prizes.distinct, returns all prize categories shared by three or more laureates. That is, "laureates.2" must exist for such documents.
Save these prize categories as a Python set called triple_play_categories.
Confirm via an assertion that "literature" is the only prize category with no prizes shared by three or more laureates.

```sql
# Save a filter for prize documents with three or more laureates
criteria = {"laureates.2": {"$exists": True}}
# Save the set of distinct prize categories in documents satisfying the criteria
triple_play_categories = set(db.prizes.distinct("category", criteria))
# Confirm literature as the only category not satisfying the criteria.
assert set(db.prizes.distinct("category")) - triple_play_categories == {"literature"}
```

Q. 6 -> What is the approximate ratio of the number of laureates who won an unshared ({"share": "1"}) prize in physics after World War II ({"year": {"$gte": "1945"}}) to the number of laureates who won a shared prize in physics after World War II?

```python
db.laureates.count_documents({
    "prizes": {"$elemMatch": {
        "category": "physics",
        "share": "1",
        "year": {"$gte": "1945"}}}})

db.laureates.count_documents({
    "prizes": {"$elemMatch": {
        "category": "physics",
        "share": {"$ne":"1"},
        "year": {"$gte": "1945"}}}})
```

ans. 0.13

Q. 7 -> Save an $elemMatch filter unshared to count laureates with unshared prizes in categories other than ("not in") ["physics", "chemistry", "medicine"] in or after 1945.

Save an $elemMatch filter shared to count laureates with shared (i.e., "share" is not "1") prizes in categories other than ["physics", "chemistry", "medicine"] in or after 1945.

ans.

```sql
# Save a filter for laureates with unshared prizes
unshared = {
    "prizes": {"$elemMatch": {
        "category": {"$nin": ["physics", "chemistry", "medicine"]},
        "share": "1",
        "year": {"$gte": "1945"}
    }}}
# Save a filter for laureates with shared prizes
shared = {
    "prizes": {"$elemMatch": {
        "category": {"$nin": ["physics", "chemistry", "medicine"]},
        "share": {"$ne":"1"},
        "year": {"$gte": "1945"}
    }}}
ratio = db.laureates.count_documents(unshared) / db.laureates.count_documents(shared)
print(ratio)
```

Q. 8 -> You won't need the $elemMatch operator at all for this exercise.
Save a filter before to count organization laureates with prizes won before 1945. Recall that organization status is encoded with the "gender" field, and that dot notation is needed to access a laureate's "year" field within its "prizes" array.
Save a filter in_or_after to count organization laureates with prizes won in or after 1945.

ans.

```sql
# Save a filter for organization laureates with prizes won before 1945
before = {
    "gender": "org",
    "prizes.year": {"$lt": "1945"},
    }

# Save a filter for organization laureates with prizes won in or after 1945
in_or_after = {
    "gender": "org",
    "prizes.year": {"$gte": "1945"},
    }
n_before = db.laureates.count_documents(before)
n_in_or_after = db.laureates.count_documents(in_or_after)
ratio = n_in_or_after / (n_in_or_after + n_before)
print(ratio)
```

Q. 9 -> Evaluate the expression
db.laureates.count_documents({"firstname": Regex(\_**\_), "surname": Regex(\_\_**)})
in the console, filling in the blanks appropriately.

```python
db.laureates.count_documents({"firstname": Regex("^G"), "surname": Regex("^S")})
```

ans. 9

Q. 10 -> Use a regular expression object to filter for laureates with "Germany" in their "bornCountry" value.

Use a regular expression object to filter for laureates with a "bornCountry" value starting with "Germany".

Use a regular expression object to filter for laureates born in what was at the time Germany but is now another country.

Use a regular expression object to filter for laureates born in what is now Germany but at the time was another country.

ans.

```sql
from bson.regex import Regex
# Filter for laureates with "Germany" in their "bornCountry" value
criteria = {"bornCountry": Regex("Germany")}
print(set(db.laureates.distinct("bornCountry", criteria)))

from bson.regex import Regex
# Filter for laureates with a "bornCountry" value starting with "Germany"
criteria = {"bornCountry": Regex("^Germany")}
print(set(db.laureates.distinct("bornCountry", criteria)))

from bson.regex import Regex
# Fill in a string value to be sandwiched between the strings "^Germany " and "now"
criteria = {"bornCountry": Regex("^Germany " + "\(" + "now")}
print(set(db.laureates.distinct("bornCountry", criteria)))

from bson.regex import Regex
#Filter for currently-Germany countries of birth. Fill in a string value to be sandwiched between the strings "now" and "$"
criteria = {"bornCountry": Regex("now Germany" + "\)" + "$")}
print(set(db.laureates.distinct("bornCountry", criteria)))
```

Q. 11 -> Save a filter criteria that finds laureates with prizes.motivation values containing "transistor" as a substring. The substring can appear anywhere within the value, so no anchoring characters are needed.
Save to first and last the field names corresponding to a laureate's first name and last name (i.e. "surname") so that we can print out the names of these laureates.

ans.

```sql
from bson.regex import Regex
# Save a filter for laureates with prize motivation values containing "transistor" as a substring
criteria = {"prizes.motivation": Regex("transistor")}
# Save the field names corresponding to a laureate's first name and last name
first, last = "firstname", "surname"
print([(laureate[first], laureate[last]) for laureate in db.laureates.find(criteria)])
```
