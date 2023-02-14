# Investigating the cause of missing backlinks
## One
- OL42936104M
  - promise item
  - imported 2022-12-4
- 009nouvelleshist0000jean
  - scanned 2023-01-26 and links to work and edition
## Two


# Programming journey
I want to be able to find edition revisions to see how many time a book has been edited, and to compare creation dates on Open Library and Internet Archive. I have a theory that could help be confirmed or denied by this, but I don't want to manually get a lot of data on my own through copy/paste.

## Open a CSV file and read some data
The CSV file has tab-separated data like and is named `edition_ocaid.tsv`:
```
OL4844481M      zwischenromantik0007dahl
OL40285121M     zycietojednakstr0000stas
OL26937635M     zzzellibrodelsue0000wint
```

```python
import os
import csv
from pathlib import Path
CSV_FILE = os.environ["EDITION_OCAID_FILE"]

with Path(CSV_FILE).open(mode="r") as fp:
    csvreader = csv.reader(fp, delimiter="\t")
    for row in csvreader:
        print(row)

```

Output:
```python
python main.py
['OL42936104M', '009nouvelleshist0000jean']
...
['OL4844481M', 'zwischenromantik0007dahl']
['OL40285121M', 'zycietojednakstr0000stas']
```

## For each edition, connect to OL to get the revision
```python
import os
import csv
from pathlib import Path

from olclient import OpenLibrary, config

# Set in .env and load into the env via the shell, or docker-compose if using that.
ACCESS_KEY = os.environ["OL_ACCESS_KEY"]
SECRET_KEY = os.environ["OL_SECRET_KEY"]
CSV_FILE = os.environ["EDITION_OCAID_FILE"]

ol = OpenLibrary(credentials=config.Credentials(access=ACCESS_KEY, secret=SECRET_KEY))

with Path(CSV_FILE).open(mode="r") as fp:
    csvreader = csv.reader(fp, delimiter="\t")
    for row in csvreader:
        olid = row[0]
        edition = ol.Edition.get(olid=olid)
        print(f"{edition.olid}'s revision is: {edition.revision}") 
```

Output:
```
❯ python main.py
OL42936104M's revision is: 1
OL42928804M's revision is: 1
...
OL27385162M's revision is: 2
```

## Add some validation to ensure there's no OCAID and also get Internet Archive data too.
```python
import os
import requests
import csv
from pathlib import Path

# from typing import Any, Final
from olclient import OpenLibrary, config

# Set in .env and load into the env via the shell, or docker-compose if using that.
ACCESS_KEY = os.environ["OL_ACCESS_KEY"]
SECRET_KEY = os.environ["OL_SECRET_KEY"]
CSV_FILE = os.environ["EDITION_OCAID_FILE"]
IA_URL = "https://archive.org/metadata/%s"

ol = OpenLibrary(credentials=config.Credentials(access=ACCESS_KEY, secret=SECRET_KEY))

with Path(CSV_FILE).open(mode="r") as fp:
    csvreader = csv.reader(fp, delimiter="\t")
    for row in csvreader:
        olid = row[0]
        ocaid = row[1]
        edition = ol.Edition.get(olid=olid)

        if hasattr(edition, "ocaid"):
            print(f"{edition.olid} had an ocaid of {edition.ocaid}")

        print(f"{edition.olid}'s revision is: {edition.revision}") 

        ia_record = requests.get(IA_URL % ocaid).json()
        # `addeddate` looks like: "2023-01-12 04:57:21", so split it on a space
        # character and take the 0 index to get "2023-01-12"
        ia_creation_date = ia_record["metadata"]["addeddate"].split(" ")[0]
        print(ia_creation_date)
```

Outputs:
```
❯ python main.py
OL42936104M's revision is: 1
2023-01-26
OL42928804M's revision is: 1
...
2023-01-19
OL27385162M's revision is: 2
```

## Put the values in a dataclass so they're easier to work with
```python
import csv
import os
import requests

from dataclasses import dataclass
from datetime import datetime
from pathlib import Path

# from typing import Any, Final
from olclient import OpenLibrary, config

# Set in .env and load into the env via the shell, or docker-compose if using that.
ACCESS_KEY = os.environ["OL_ACCESS_KEY"]
SECRET_KEY = os.environ["OL_SECRET_KEY"]
CSV_FILE = os.environ["EDITION_OCAID_FILE"]
IA_URL = "https://archive.org/metadata/%s"
IA_DATE_FORMAT = "%Y-%m-%d"
OL_DATE_FORMAT = "%Y-%m-%dT%H:%M:%S.%f"


@dataclass
class Book:
    olid: str
    ol_revision: int
    ol_creation_date: datetime
    ocaid: str
    ia_creation_date: datetime



ol = OpenLibrary(credentials=config.Credentials(access=ACCESS_KEY, secret=SECRET_KEY))

with Path(CSV_FILE).open(mode="r") as fp:
    # Create a list of books
    books: list[Book] = []

    # Open the CSV file
    csvreader = csv.reader(fp, delimiter="\t")

    # A 'count' for testing.
    count = 0

    # Iterate through each row in the CSV.
    for row in csvreader:
        olid = row[0]
        ocaid = row[1]
        edition = ol.Edition.get(olid=olid)

        if hasattr(edition, "ocaid"):
            print(f"{edition.olid} had an ocaid of {edition.ocaid}")

        ol_revision = edition.revision
        # Get OL created date as a string, then convert to datetime format for
        # easier date comparison.
        created = edition.created["value"]
        ol_creation_date = datetime.strptime(created, OL_DATE_FORMAT)

        ia_record = requests.get(IA_URL % ocaid).json()
        # `addeddate` looks like: "2023-01-12 04:57:21", so split it on a space
        # character and take the 0 index to get "2023-01-12"
        created = ia_record["metadata"]["addeddate"].split(" ")[0]
        ia_creation_date = datetime.strptime(created, IA_DATE_FORMAT)

        # Create our book
        book = Book(
                olid=olid,
                ocaid=ocaid,
                ol_revision=ol_revision,
                ol_creation_date=ol_creation_date,
                ia_creation_date=ia_creation_date,
                )

        books.append(book)

        count +=1 
        if count >= 4:
            break

print("Here are the books")
for book in books:
    print(book)
```

Output:
```
❯ python main.py
Here are the books
Book(olid='OL42936104M', ol_revision=1, ol_creation_date=datetime.datetime(2022, 12, 4, 22, 28, 17, 752826), ocaid='009nouvelleshist0000jean', ia_creation_date=datetime.datetime(2023, 1, 26, 0, 0))
Book(olid='OL42928804M', ol_revision=1, ol_creation_date=datetime.datetime(2022, 12, 4, 20, 8, 27, 807854), ocaid='032christineface0000annm', ia_creation_date=datetime.datetime(2023, 1, 12, 0, 0))
Book(olid='OL42885374M', ol_revision=1, ol_creation_date=datetime.datetime(2022, 12, 4, 8, 1, 0, 832747), ocaid='0815trilogieinde0000hans', ia_creation_date=datetime.datetime(2023, 1, 19, 0, 0))
Book(olid='OL27385162M', ol_revision=2, ol_creation_date=datetime.datetime(2019, 10, 8, 14, 35, 36, 41044), ocaid='10000mileswith100000fran', ia_creation_date=datetime.datetime(2023, 1, 12, 0, 0))
```

## Now put it all together to make this worthwhile
We take the above code, and change it a bit so we can compare the dates, and also get some sort of summary so we can see how many books there were total, how many revision 1, and how many of those revision 1 books were created on Open Library before being created on Internet Archive.

```python
import csv
import os
import requests

from dataclasses import dataclass
from datetime import datetime
from pathlib import Path

# from typing import Any, Final
from olclient import OpenLibrary, config

# Set in .env and load into the env via the shell, or docker-compose if using that.
ACCESS_KEY = os.environ["OL_ACCESS_KEY"]
SECRET_KEY = os.environ["OL_SECRET_KEY"]
CSV_FILE = os.environ["EDITION_OCAID_FILE"]
IA_URL = "https://archive.org/metadata/%s"
IA_DATE_FORMAT = "%Y-%m-%d"
OL_DATE_FORMAT = "%Y-%m-%dT%H:%M:%S.%f"


@dataclass
class Book:
    olid: str
    ol_revision: int
    ol_creation_date: datetime
    ocaid: str
    ia_creation_date: datetime



ol = OpenLibrary(credentials=config.Credentials(access=ACCESS_KEY, secret=SECRET_KEY))

with Path(CSV_FILE).open(mode="r") as fp:
    # Create a list of books
    books: list[Book] = []

    # Open the CSV file
    csvreader = csv.reader(fp, delimiter="\t")

    # A 'count' for testing.
    count = 0

    # Iterate through each row in the CSV.
    for row in csvreader:
        olid = row[0]
        ocaid = row[1]
        edition = ol.Edition.get(olid=olid)

        if hasattr(edition, "ocaid"):
            print(f"{edition.olid} had an ocaid of {edition.ocaid}")

        ol_revision = edition.revision
        # Get OL created date as a string, then convert to datetime format for
        # easier date comparison.
        created = edition.created["value"]
        ol_creation_date = datetime.strptime(created, OL_DATE_FORMAT)

        ia_record = requests.get(IA_URL % ocaid).json()
        # `addeddate` looks like: "2023-01-12 04:57:21", so split it on a space
        # character and take the 0 index to get "2023-01-12"
        created = ia_record["metadata"]["addeddate"].split(" ")[0]
        ia_creation_date = datetime.strptime(created, IA_DATE_FORMAT)

        # Create our book
        book = Book(
                olid=olid,
                ocaid=ocaid,
                ol_revision=ol_revision,
                ol_creation_date=ol_creation_date,
                ia_creation_date=ia_creation_date,
                )

        books.append(book)

        count +=1 
        if count >= 100:
            break


# Total stuff up
total = len(books)
total_rev_1 = sum(1 for book in books if book.ol_revision == 1)

rev_1_and_ia_created_after_ol = 0

for book in books:
    if book.ol_revision == 1:
        if book.ol_creation_date < book.ia_creation_date:
            rev_1_and_ia_created_after_ol += 1

print(f"Total books: {total}")
print(f"Total revision 1 books: {total_rev_1}")
print(f"Total revision 1 books where the OL edition was created first: {total_rev_1}")
```

Output:
```
❯ python main.py
Total books: 4
Total revision 1 books: 3
Total revision 1 books where the OL edition was created first: 3
```
