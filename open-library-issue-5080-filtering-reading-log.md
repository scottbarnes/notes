Filter reading log
==================
[Issue #5080](https://github.com/internetarchive/openlibrary/issues/5080): Adding Search / Filter capability to Reading Log

# Questions
- Where should different things happen? Do we have a difference between data model (where we store data / what is store?) and a domain model (where we make queries and set values)?
- Changes to `get_users_logged_books`:
  - this is only called for the three shelves in `upstream/mybooks.py`, and then `process_logged_books` uses the `work_id` to create works with `web.ctx.site.get_many(work_keys)`, and then adds `created` and `edition_id` from the logged books query, and that's it. So it just needs those three for a non-filtered query, and _only_ the work ID for the filtered query, as there is no sorting for that one.
- Use SearchRespose.num_found to get the total.
- Now that I have got a ReadingLogWorkResponse coming out of bookshelves.py, I need to make everything else take it, and where it expects docs, I need to give it ReadingLogWorkResponse.works
- Display notifications about > 100 matches and >1000 books.
- Ensure `public_my_books_json` works.
- I need to start from the API endpoint with the delgate and write out what I want to do in a total list, and then see how I can logically break it up in the abstract.
- I can solve the circular import by not using `Work` in bookshelves.py. That would mean it should get its own dataclass to smuggle stuff out of there, and `ReadingLogWorks` can be used in `process_logged_books`.
- There is a bug when >100 results because numfound is > 100, and pagination is based on that, but the limit is 100 so the pagination is mistaken.
- Maybe `ReadingLogWorks` should be `ReadingLogData`, as it has a `.works`
- How much can I move into the template.render function? (So that one doesn't have to burrow through function after function?)
- `get_users_logged_books` can return two types depending on the input (i.e. if `q` is present). Then `process_logged_books` takes two types, and does different things for each.
- see if all the `get_want_to_read`, `get_currently_reading`, and `get_already_read` can be one function with a `kind` argument, and then just use f-strings to put than in like so: `bookshelf_id=Bookshelves.PRESET_BOOKSHELVES[f"{kind}"]`
- maybe I should just pass the `LoggedBookData` through and access the attributes off that, rather than passing its attributes as keywords. This would allow methods on LoggedBookData.
- How is "want-to-read" without a query returning 20 results, and 25 with a query?
- logged_edition and logged_data only used with JSON? Only add them then?
- need to try SHELVES with zero books.

# PR questions
- With the sets, maybe I could have three enums, and they could be in a set or list?
- Maybe what I need to do is think about what I'm trying to accomplish, and different ways that could be done in terms of data structures?
- What's the best way to make it so we know if someone is hitting the 30k book search limit?

# Latest problems
- With respect to redirects, I realized we can't take a diff (I think) because we don't know what's missing because it's a redirected work key, and what's missing because it's not matching work key.

# Plan for bookshelves and mybooks
- `mybooks.my_books_view()`:
  - handle GET.
  - `call MyBooksTemplate.render()`
- `mybooks.MyBooksTemplate.render` calls `get_works()`.
  - `get_works()` is what takes a `key` and then iterates through `self.KEYS` to find the correct function to use.
  - Goal: be able to type this end-to-end. Idea: maybe `MyBooksTemplate.READING_LOG_KEYS` could be an `enum`. Or a `dataclass`? Would it help if `self.KEYS` is a `dataclass` of the functions? Would that type?

# Things to consider
- Maybe make a `NewType` out of data for the items, because although `data` is a list of `<Work>`s, it's not just any works, but ones that have been through `process_logged_books`, and these are special work items with two new things. So I should have whatever accepts them (But what does? They're just returned up the chain to the render function it seems?) only accept the NewType.
- Need to install a plugin to find TODO items in my staged files or dirty files or whatever.
- Maybe move all the reading log stuff to `reading_log.py`
- Maybe `process_logged_books` can return the `ReadingLogWorkResponse`. Should be `ReadingLogWorks`. But it has to be in `get_users_logged_books` because I need to get the total results out of there.

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
```python
logged_books = [<Storage {'username': 'openlibrary', 'work_id': 13101191, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 33, 12, 661254), 'created': datetime.datetime(2022, 10, 2, 22, 33, 12, 661254)}>, <Storage {'username': 'openlibrary', 'work_id': 15298516, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 33, 11, 865206), 'created': datetime.datetime(2022, 10, 2, 22, 33, 11, 865206)}>, <Storage {'username': 'openlibrary', 'work_id': 20600, 'bookshelf_id': 1, 'edition_id': None, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 33, 4, 982721), 'created': datetime.datetime(2022, 10, 2, 22, 33, 4, 982721)}>, <Storage {'username': 'openlibrary', 'work_id': 118420, 'bookshelf_id': 1, 'edition_id': 20431791, 'private': None, 'updated': datetime.datetime(2022, 10, 2, 22, 32, 56, 323280), 'created': datetime.datetime(2022, 10, 2, 22, 32, 56, 323280)}>]
```

## `resp.docs` from `QueryResponse`
```python
docs=[{'key': '/works/OL118420W', 'title': 'Flatland', 'subtitle': 'a romance of many dimensions', 'edition_count': 1, 'first_publish_year': 1885, 'has_fulltext': True, 'public_scan_b': True, 'ia': ['flatlandaromanc01abbogoog'], 'ia_collection_s': 'americana', 'lending_edition_s': 'OL20431791M', 'lending_identifier_s': 'flatlandaromanc01abbogoog', 'author_key': ['OL20585A'], 'author_name': ['Edwin Abbott Abbott'], 'editions': {'numFound': 0, 'start': 0, 'numFoundExact': True, 'docs': []}}, {'key': '/works/OL15298516W', 'title': 'Through the looking-glass', 'edition_count': 1, 'first_publish_year': 1923, 'has_fulltext': True, 'public_scan_b': True, 'ia': ['aliceimspiegella00carrrich'], 'ia_collection_s': 'americana;burstein;internetarchivebooks', 'lending_edition_s': 'OL13517105M', 'lending_identifier_s': 'aliceimspiegella00carrrich', 'author_key': ['OL22098A'], 'author_name': ['Lewis Carroll'], 'editions': {'numFound': 0, 'start': 0, 'numFoundExact': True, 'docs': []}}]
```

# `ReadingLogWorkResponse` coming out of `get_users_logged_books`
```python
reading_log_work_response: ReadingLogWorkResponse(q='stephen king', page_size=25, total_results=10, works=[{'key': '/works/OL81633W'}, {'key': '/works/OL43861W'}, {'key': '/works/OL81632W'}, {'key': '/works/OL81597W'}, {'key': '/works/OL81589W'}, {'key': '/works/OL258902W'}, {'key': '/works/OL71058W'}, {'key': '/works/OL71078W'}, {'key': '/works/OL362699W'}, {'key': '/works/OL82563W'}])
```

## `works` as they were previously:
`works_old: [<Work: '/works/OL6037022W'>, <Work: '/works/OL54120W'>, <Work: '/works/OL8193488W'>]`
- This means the template used to get `<Work :'/works/OL1234W'>`, but now gets `<Storage {'key': '/works/OL1234W'}.`


```
reading_log_work_response: ReadingLogWorkResponse(q='stephen king', page_size=25, total_results=10, logged_books={'/works/OL81633W', '/works/OL81597W', '/works/OL258902W', '/works/OL82563W', '/works/OL43861W', '/works/OL81632W', '/works/OL81589W', '/works/OL71058W', '/works/OL362699W', '/works/OL71078W'}, works=[{'key': '/works/OL81633W'}, {'key': '/works/OL43861W'}, {'key': '/works/OL81632W'}, {'key': '/works/OL81597W'}, {'key': '/works/OL81589W'}, {'key': '/works/OL258902W'}, {'key': '/works/OL71058W'}, {'key': '/works/OL71078W'}, {'key': '/works/OL362699W'}, {'key': '/works/OL82563W'}])
```


```
solr_resp = SearchResponse(facet_counts=None, sort=None, docs=[{'key': '/works/OL81633W'}, {'key': '/works/OL43861W'}, {'key': '/works/OL81632W'}, {'key': '/works/OL81597W'}, {'key': '/works/OL258902W'}, {'key': '/works/OL81589W'}, {'key': '/works/OL71058W'}, {'key': '/works/OL71078W'}, {'key': '/works/OL362699W'}, {'key': '/works/OL82563W'}], num_found=10, solr_select='http://solr:8983/solr/openlibrary/select?fq=t'), ... raw_resp={'responseHeader': {'status': 0, 'QTime': 53, 'params': {'q': '({!edismax q.op="AND" qf="text alternative_title^20 author_name^20" bf="min(100,edition_count)" v=$workQuery})', 'spellcheck': 'true', 'fl': 'key', 'start': '0', 'fq': 'type:work', 'spellcheck.count': '3', 'rows': '25', 'workQuery': 'stephen king key:(\\/works\\/OL81633W OR ...)'}}}
```

"items" in reading_log.html, prior to any changes
```
[<Work: '/works/OL71078W'>, <Work: '/works/OL81633W'>, <Work: '/works/OL71058W'>, <Work: '/works/OL82563W'>, <Work: '/works/OL43861W'>, <Work: '/works/OL258902W'>, <Work: '/works/OL81589W'>, <Work: '/works/OL81597W'>, <Work: '/works/OL81632W'>, <Work: '/works/OL362699W'>]
```

`data` in `mybooks.py` prior to any changes
```
data is: [<Work: '/works/OL71078W'>, <Work: '/works/OL81633W'>, <Work: '/works/OL71058W'>, <Work: '/works/OL82563W'>, <Work: '/works/OL43861W'>, <Work: '/works/OL258902W'>, <Work: '/works/OL81589W'>, <Work: '/works/OL81597W'>, <Work: '/works/OL81632W'>, <Work: '/works/OL362699W'>]

```

`items` in `reading_log.html` when passing Solr results through:
```
items in reading_log.html: [<Storage {'key': '/works/OL16805415W', 'type': 'work', 'seed': ['/books/OL26369833M',
```
i.e. list of Storage dicts with {'key': '/works/OL123W', more stuff}


[item['key'].split('/')[-1] for item in items]

`solr_resp.docs`:
```python
[{'key': '/works/OL81633W', 'title': 'The Shining', 'edition_count': 77, 'first_publish_year': 1977, 'has_fulltext': True, 'public_scan_b': False, 'ia': ['shining0000king_w4g5', 'shining0000king', 'shining0000king_q3a8', 'skullofskeleton0000unse', 'shining00step', 'shining0000king_u6w1', 'shining06king', 'shiningking00king', 'shining00king', 'shiningk00king', 'shining00kingrich'], 'ia_collection_s': 'americana;china;delawarecountydistrictlibrary;inlibrary;internetarchivebooks;printdisabled', 'lending_edition_s': 'OL22312685M', 'lending_identifier_s': 'shining0000king_w4g5', 'cover_edition_key': 'OL23253707M', 'cover_i': 7013717, 'language': ['eng', 'spa', 'dut', 'jpn', 'pol', 'fre', 'por', 'ger'], 'author_key': ['OL2162284A'], 'author_name': ['Stephen King']}, {'key': '/works/OL43861W', 'title': 'Oedipus Rex', 'edition_count': 185, 'first_publish_year': 1715, 'has_fulltext': True, 'public_scan_b': True, 'ia'}]
```

`solr_works` are:
```python
solr_works are: [<Storage {'key': '/works/OL52237W', 'type': 'work', 'seed': ['/books/OL18798199M', '/books/OL18135656M', '/books/OL16679454M',]}]
```

# Order
- Complete works of Mark Twain, A taste of heaven, Men like Gods, Fight Club

<Storage {'key': '/works/OL3501730W', 'title': 'The Neon Bible', 'edition_count': 12, 'first_publish_year': 1989, 'has_fulltext': True, 'public_scan_b': False, 'ia': ['neonbible00tool', 'neonbible00tool_528'], 'ia_collection_s': 'china;inlibrary;internetarchivebooks;librarygenesis;printdisabled', 'lending_edition_s': 'OL2048765M', 'lending_identifier_s': 'neonbible00tool', 'cover_edition_key': 'OL2048765M', 'cover_i': 8372396, 'language': ['eng', 'spa', 'fre'], 'author_key': ['OL585131A'], 'author_name': ['John Kennedy Toole']}>


