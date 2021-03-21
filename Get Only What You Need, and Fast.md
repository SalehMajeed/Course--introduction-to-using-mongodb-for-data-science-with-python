Q. 1 -> Which projection(s) will fetch ONLY the laureates' full names and prize share info? I encourage you to experiment with the console and re-familiarize yourself with the structure of laureate collection documents.

```python
list(db.laureates.find(filter = {"category":"physics","year":"1903"},projection={"prizes":1,"_id":"0"}))
```

ans. {"firstname": 1, "surname": 1, "prizes.share": 1, "\_id": 0}

Q. 2 -> First, use regular expressions to fetch the documents for the laureates whose "firstname" starts with "G" and whose "surname" starts with "S".

In the previous step, we fetched all the data for all the laureates with initials G.S. This is unnecessary if we only want their full names!
Use projection and adjust the query to select only the "firstname" and "surname" fields.

Now the documents you fetched contain only the relevant information!
Iterate over the documents, and for each document, concatenate the first name and the surname fields together with a space in between to obtain full names.
ans.

```sql
# Find laureates whose first name starts with "G" and last name starts with "S"
docs = db.laureates.find(
       filter= {"firstname" : {"$regex" : "^G"},
                  "surname" : {"$regex" : "^S"}  })
# Print the first document
print(docs[0])

# Use projection to select only firstname and surname
docs = db.laureates.find(
        filter= {"firstname" : {"$regex" : "^G"},
                 "surname" : {"$regex" : "^S"}  },
	projection= {"firstname":1,"surname":1,"_id":0}  )
# Print the first document
print(docs[0])

# Use projection to select only firstname and surname
docs = db.laureates.find(
       filter= {"firstname" : {"$regex" : "^G"},
                "surname" : {"$regex" : "^S"}  },
   projection= ["firstname", "surname"]  )
# Iterate over docs and concatenate first name and surname
full_names = [doc["firstname"] + " " + doc["surname"]  for doc in docs]
# Print the full names
print(full_names)
```

Q. 3 -> Save a list of prizes (prizes), projecting out only the "laureates.share" values for each prize.
For each prize, compute the total share as follows:
Initialize the variable total_share to 0.
Iterate over the laureates for each prize, converting the "share" field of the "laureate" to float and adding the reciprocal of it (that is, 1 divided by it) to total_share.

ans.

```python
# Save documents, projecting out laureates share
prizes = db.prizes.find({}, ["laureates.share"])
# Iterate over prizes
for prize in prizes:
    # Initialize total share
    total_share = 0
    # Iterate over laureates for the prize
    for laureate in prize["laureates"]:
        # add the share of the laureate to total_share
        total_share += 1 / float(laureate["share"])
    # Print the total share
    print(total_share)
```

Q. 4 -> This block prints out the first five projections of a sorted query. What "sort" argument fills the blank?

```python
docs = list(db.laureates.find(
    {"born": {"$gte": "1900"}, "prizes.year": {"$gte": "1954"}},
    {"born": 1, "prizes.year": 1, "_id": 0},
    sort=____))
for doc in docs[:5]:
    print(doc)
```

ans. [("prizes.year", 1), ("born", -1)]

Q. 5 -> Complete the definition of all_laureates(prize). Within the body of the function:
Sort the "laureates" list of the prize document according to the "surname" key.
For each of the laureates in the sorted list, extract the "surname" field.
The code for joining the last names into a single string is already written for you.
Take a look at the console to make sure the output looks like what you'd expect!

Find the documents for the prizes in the physics category, sort them in chronological order (by "year", ascending), and only fetch the "year", "laureates.firstname", and "laureates.surname" fields.

Now that you have the prizes, and the function to extract laureates from a prize, print the year and the names of the laureates (use your all_laureates() function) for each prize document.
ans.

```python
from operator import itemgetter
def all_laureates(prize):
  # sort the laureates by surname
  sorted_laureates = sorted(prize["laureates"], key=itemgetter("surname"))
  # extract surnames
  surnames = [laureate["surname"] for laureate in sorted_laureates]
  # concatenate surnames separated with " and "
  all_names = " and ".join(surnames)
  return all_names
# test the function on a sample doc
print(all_laureates(sample_prize))

from operator import itemgetter
def all_laureates(prize):
  # sort the laureates by surname
  sorted_laureates = sorted(prize["laureates"], key=itemgetter("surname"))
  # extract surnames
  surnames = [laureate["surname"] for laureate in sorted_laureates]
  # concatenate surnames separated with " and "
  all_names = " and ".join(surnames)
  return all_names
# find physics prizes, project year and first and last name, and sort by year
docs = db.prizes.find(
           filter= {"category":"physics"},
           projection= ["year","laureates.firstname","laureates.surname"],
           sort= [("year", 1)])

from operator import itemgetter
def all_laureates(prize):
  # sort the laureates by surname
  sorted_laureates = sorted(prize["laureates"], key=itemgetter("surname"))
  # extract surnames
  surnames = [laureate["surname"] for laureate in sorted_laureates]
  # concatenate surnames separated with " and "
  all_names = " and ".join(surnames)
  return all_names
# find physics prizes, project year and name, and sort by year
docs = db.prizes.find(
           filter= {"category": "physics"},
           projection= ["year", "laureates.firstname", "laureates.surname"],
           sort= [("year", 1)])
# print the year and laureate names (from all_laureates)
for doc in docs:
  print("{year}: {names}".format(year=doc['year'], names=all_laureates(doc)))
```

Q. 6 -> Find the original prize categories established in 1901 by looking at the distinct values of the "category" field for prizes from year 1901.
Fetch ONLY the year and category from all the documents (without the "\_id" field).Sort by "year" in descending order, then by "category" in ascending order.

ans.

```python
# original categories from 1901
original_categories = db.prizes.distinct("category", {"year": "1901"})
print(original_categories)
# project year and category, and sort
docs = db.prizes.find(
        filter={},
        projection = {"year":1,"category":1,"_id":0},
         sort= [("year", -1),("category",1)]
)
#print the documents
for doc in docs:
  print(doc)
```

Q. 7 -> Which of the following indexes is best suited to speeding up the operation db.prizes.distinct("category", {"laureates.share": {"$gt": "3"}})?

ans. [("laureates.share", 1), ("category", 1)]

Q. 8 -> Specify an index model that indexes first on category (ascending) and second on year (descending).
Save a string report for printing the last single-laureate year for each distinct category, one category per line. To do this, for each distinct prize category, find the latest-year prize (requiring a descending sort by year) of that category (so, find matches for that category) with a laureate share of "1".

ans.

```python
# Specify an index model for compound sorting
index_model = [("category", 1), ("year", -1)]
db.prizes.create_index(index_model)
# Collect the last single-laureate year for each category
report = ""
for category in sorted(db.prizes.distinct("category")):
    doc = db.prizes.find_one(
        {"category": category, "laureates.share": "1"},
        sort=[("year", -1)]
    )
    report += "{category}: {year}\n".format(**doc)
print(report)
```

Q. 9 -> Create an index on country of birth ("bornCountry") for db.laureates to ensure efficient gathering of distinct values and counting of documents
Complete the skeleton dictionary comprehension to construct n_born_and_affiliated, the count of laureates as described above for each distinct country of birth. For each call to count_documents, ensure that you use the value of country to filter documents properly.

ans.

```python
from collections import Counter
# Ensure an index on country of birth
db.laureates.create_index([("bornCountry", 1)])
# Collect a count of laureates for each country of birth
n_born_and_affiliated = {
    country: db.laureates.count_documents({
        "bornCountry": country,
        "prizes.affiliations.country": country
    })
    for country in db.laureates.distinct("bornCountry")
}
five_most_common = Counter(n_born_and_affiliated).most_common(5)
print(five_most_common)
```

Q. 10 -> How many documents does the following expression return?

list(db.prizes.find({"category": "economics"},
{"year": 1, "\_id": 0})
.sort("year")
.limit(3)
.limit(5))

ans. 5: the second call to limit overrides the first

Q. 11 -> Save to filter\_ the filter document to fetch only prizes with one or more quarter-share laureates, i.e. with a "laureates.share" of "4".
Save to projection the list of field names so that prize category, year and laureates' motivations ("laureates.motivation") may be fetched for inspection.
Save to cursor a cursor that will yield prizes, sorted by ascending year. Limit this to five prizes, and sort using the most concise specification.

ans.

```python
from pprint import pprint
# Fetch prizes with quarter-share laureate(s)
filter_ = {"laureates.share": "4"}
# Save the list of field names
projection = ["category","year", "laureates.motivation"]
# Save a cursor to yield the first five prizes
cursor = db.prizes.find(filter_, projection).sort("year").limit(5)
pprint(list(cursor))
```

Q. 12 -> Complete the function get_particle_laureates that, given page_number and page_size, retrieves a given page of prize data on laureates who have the word "particle" (use $regex) in their prize motivations ("prizes.motivation"). Sort laureates first by ascending "prizes.year" and next by ascending "surname".
Collect and save the first nine pages of laureate data to pages.

ans.

```python
from pprint import pprint

# Write a function to retrieve a page of data
def get_particle_laureates(page_number=1, page_size=3):
    if page_number < 1 or not isinstance(page_number, int):
        raise ValueError("Pages are natural numbers (starting from 1).")
    particle_laureates = list(
        db.laureates.find(
            {"prizes.motivation": {"$regex": "particle"}},
            ["firstname", "surname", "prizes"])
        .sort([("prizes.year", 1), ("surname", 1)])
        .skip(page_size * (page_number - 1))
        .limit(page_size))
    return particle_laureates
# Collect and save the first nine pages
pages = [get_particle_laureates(page_number=page) for page in range(1,9)]
pprint(pages[0])
```
