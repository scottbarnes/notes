Filter reading log
==================
[Issue #5080](https://github.com/internetarchive/openlibrary/issues/5080): Adding Search / Filter capability to Reading Log

# Questions
- Is the strategy for uniting reading log data and Solr data going to be a huge performance problem with thousands of works? Quick way to add thousands of works to reading log?
- I've added no additional limits. I need to test to see if the filtering happens before any limits are imposed. I can see if I am filtering all by lowering the limit to 1 or 2 or 3 and then filtering to see what's in the results, I think.

# Actual plan
- self.processed_logged_books(): This should be changed to only query Solr and return a Solr work.
  - We should check if self.processed_logged_books does anything else beyond merely fetching the work (e.g. adding fields)
  - Check self.processed_logged_books is used elsewhere because we change the return values. (Answer: yes, though it was indirect. I need to test more thoroughly).
- If `q` is present, then pass that through with the query to Solr and also remove `limit` (?)

# Things to consider
- Possible make a `dataclass` that is something like:
```python
@dataclass
class UserReadingLogWork:
    """
    One reading log item to hold all the attributes to avoid requerying.
    """
    user: User
    rating: int
    read_status: str  # enum?
    logged_date: datetime
    logged_edition_id: str  # currently this is "/book/OL1234M" if it exists. It's in thte solr_work, but it may make some sense to take some values in there attributes. Need to see more thoroughly how this would get used.
    solr_work: dict[str, Any]  # I can maybe make this a bit better.
    other_mystery_things: str  # ?

    def __post_init__(self) -> None:
        self.user_rating = self.get_user_rating(self.user)

    @staticmethod
    def get_user_rating(user) -> str:
        """Set the user rating"""
        pass
```

# Implementation issues

## Uniting the data from the reading log DB and Solr
I have a question specifically about how I am uniting the data from the reading log DB. Specifically, I iterate through a pair of lists, as will be shown, and I am worried about the performance implications with large numbers of books.

In `bookshelves.py`, and specifically in [`get_users_logged_books`](https://github.com/internetarchive/openlibrary/blob/caf727acdc61ed49bec0bbcbd9e1f0cedb7a38c9/openlibrary/core/bookshelves.py#L173), there is a query to get logged books, and it returns `<Storage>` that look like:
```python
[<Storage {'username': 'openlibrary', 'work_id': 5688746, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 3, 17, 42, 47, 795770), 'created': datetime.datetime(2022, 10, 3, 17, 42, 47, 795770)}>, ...]
```
In `mybooks.py`, and specifically [`process_logged_books()`](https://github.com/internetarchive/openlibrary/blob/caf727acdc61ed49bec0bbcbd9e1f0cedb7a38c9/openlibrary/plugins/upstream/mybooks.py#L362), there is a query to `web.ctx.site.get_many(work_ids)` to get `<Work>` items, and then put the `logged_date` and `logged_edition` into those `<Work>` items:
```python
def process_logged_books(self, logged_books):
    work_ids = ['/works/OL%sW' % i['work_id'] for i in logged_books]
    works = web.ctx.site.get_many(work_ids)
    for i in range(len(works)):
        # insert the logged edition (if present) and logged date
        works[i].logged_date = logged_books[i]['created']
        works[i].logged_edition = (
            '/books/OL%sM' % logged_books[i]['edition_id']
            if logged_books[i]['edition_id']
            else ''
        )
    return works
```

This brings us to my concern.

The above relies on the results from `get_users_logged_books` and `web.ctx.site.get_many(work_ids)` being in the same order. In an attempt to cut back on the number of queries (Solr is later queried for each work anyway, for example), to limit the number of places we look for info, and to handle the filtering of reading log data by author and edition, I here query Solr and then unite the Solr and reading log data.

Note, I could also do this within `get_users_logged_books()` in `mybooks.py` so only one 'item' ever comes out for people to work with, and perhaps this could be the `dataclass`.

This creates two issues:
1. the lists are no longer in the same order, so they must be searched, which requires multiple partial passes; and
2. querying Solr breaks the sort order from `bookshelves.py` and requires re-sorting.
```python
def process_logged_books(
    self, logged_books: list[web.storage], q: str | None, sort_order=None
):
    """
    Get additional information for books in the reading log from Solr, and
    handle the sorting and filtering of that data.

    The ReadingLog db table only has stripped OL identifiers (e.g. 1234
    rather than OL1234W), so filtering a patron's reading log by title
    or author requires getting that information from Solr.

    However, because Solr in turn does not know when a book was added to
    the reading log, that information must here be added to the Solr
    results.
    """
    # Get logged_books from the ReadingLog db and use those IDs to search
    # Solr. The first query is when a partron is filtering their reading
    # log.
    work_ids = ['/works/OL%sW' % i['work_id'] for i in logged_books]
    if q:
        query = {'q': f'{q} key:({" OR ".join(work_ids)})'}
    else:
        query = {'q': f'key:({" OR ".join(work_ids)})'}

    resp = do_search(param=query, page=1, rows=RESULTS_PER_PAGE, sort=None)
    works = [web.storage(doc) for doc in resp.docs]

    # For each query result, find the correspondeing logged_book and get
    # the created datetime and the edition if known. This is necessary
    # because Solr has no way of knowing when books were added to the
    # reading log.
    for solr_work in works:
        work_id_in_logged_format = "".join(
            char for char in solr_work.key if char.isdigit()
        )
        # Performance: what about sorting both lists by work_id first, then
        # list.pop(idx) for each processed logged item?
        logged_book = next(
            (
                logged_book
                for logged_book in logged_books
                if logged_book.work_id == int(work_id_in_logged_format)
            ),
            None,
        )
        solr_work.logged_date = logged_book.created if logged_book else ''
        solr_work.logged_edition = (
            "/books/OL%sM" % logged_book.edition_id
            if (logged_book and logged_book.edition_id)
            else ''
        )

    if sort_order == "asc":
        works.sort(key=lambda x: x.logged_date, reverse=False)
    else:
        works.sort(key=lambda x: x.logged_date, reverse=True)

    return works
```

# Misc notes
- Order of adding to want-to-read items: Flatland, Travels, Looking Glass, Wonderland

# What's in side of things

## `logged_books` from `Bookshelves.get_users_logged_books()`
- They are works, but as `work_id` shows, they're missing `OL` and `W`.
```
[<Storage {'username': 'openlibrary', 'work_id': 13101191, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 33, 12, 661254), 'created': datetime.datetime(2022, 10, 2, 22, 33, 12, 661254)}>, <Storage {'username': 'openlibrary', 'work_id': 15298516, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 33, 11, 865206), 'created': datetime.datetime(2022, 10, 2, 22, 33, 11, 865206)}>, <Storage {'username': 'openlibrary', 'work_id': 20600, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 33, 4, 982721), 'created': datetime.datetime(2022, 10, 2, 22, 33, 4, 982721)}>, <Storage {'username': 'openlibrary', 'work_id': 118420, 'bookshelf_id': 1, 'edition_id': 20431791, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 32, 56, 323280), 'created': datetime.datetime(2022, 10, 2, 22, 32, 56, 323280)}>]
```

## `resp.docs` from `QueryResponse`
```
docs=[{'key': '/works/OL118420W', 'title': 'Flatland', 'subtitle': 'a romance of many dimensions', 'edition_count': 1, 'first_publish_year': 1885, 'has_fulltext': True, 'public_scan_b': True, 'ia': ['flatlandaromanc01abbogoog'], 'ia_collection_s': 'americana', 'lending_edition_s': 'OL20431791M', 'lending_identifier_s': 'flatlandaromanc01abbogoog', 'author_key': ['OL20585A'], 'author_name': ['Edwin Abbott Abbott'], 'editions': {'numFound': 0, 'start': 0, 'numFoundExact': True, 'docs': []}}, {'key': '/works/OL15298516W', 'title': 'Through the looking-glass', 'edition_count': 1, 'first_publish_year': 1923, 'has_fulltext': True, 'public_scan_b': True, 'ia': ['aliceimspiegella00carrrich'], 'ia_collection_s': 'americana;burstein;internetarchivebooks', 'lending_edition_s': 'OL13517105M', 'lending_identifier_s': 'aliceimspiegella00carrrich', 'author_key': ['OL22098A'], 'author_name': ['Lewis Carroll'], 'editions': {'numFound': 0, 'start': 0, 'numFoundExact': True, 'docs': []}}]
```

## `works` as they were previously:
`works_old: [<Work: '/works/OL6037022W'>, <Work: '/works/OL54120W'>, <Work: '/works/OL8193488W'>]`
- This means the template used to get `<Work :'/works/OL1234W'>`, but now gets `<Storage {'key': '/works/OL1234W'}.`
