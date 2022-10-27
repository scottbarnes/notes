Open Library Future Projects
============================

# Get book ratings as a bulk request
In `openlibrary/core/models.py`, around line 1158, there's a loop through `self.docs` to get the ratings for each document. @cdrini suggests making that one bulk request. See [comment](https://github.com/internetarchive/openlibrary/pull/7052#discussion_r1006299780)

# Add LibreVox entries
@cdrini suggested scraping, but as Internet Archive hosts LibreVox, maybe we can just ask for access to an API or dump? Or at least get permission to scrape at some low rate limit?
