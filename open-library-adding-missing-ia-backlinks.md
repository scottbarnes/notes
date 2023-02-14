Adding to Open Library books where IA links to OL but OL has no OCAID
=====================================================================

# Conditions
Books for which:
1. Internet Archive has an item that links to an Open Library edition;
2. There is only one ISBN 13 for that Internet Archive item; and.
3. The Open Library edition has no OCAID.

# Questions
- Should this compare ISBN 13s too to ensure they are identical? Or better, any reason not to?

# Shape
- Need a script that does the tasks below
- Script should allow importing into a DB; bonus if it can run every minute from cron, detect if it's running, and detect if anything should be added to DB
- Script should allow reading from DB automatically when run. Bonus points if cron again.

# Tasks
- For each item in list....
- Get edition
- Check if current OCAID exists; if it does, log and bail.
- Check if proposed OCAID is in `source_records`; if it is, log but don't bail.
- Add ocaid
- Add OCAID to `source_records`.
- Save with comment "Adding OCAID and source_record: {ocaid}"
- Print success or failure so that resumption is easier if failure.
- Log success, or failure with message, to a CSV I guess?
- Make it easy to have the source either be a DB query or a report file.
- Needs retry thing for requests, with exponential backoff.

# Import to SQLite
- running with `-populate` should fill SQLite with entries from `filename` as specified in `.env`.
<!-- - keys -->
<!--   - `edition_id: str` -->
<!--   - `ocaid: str` -->
<!--   - `status: int` -->
    - 0 = needs adding, 1 = added by this script; 2 = added by something else.
- bonus: this should de-duplicate somehow (either by querying to see what's already in there and removing it from the list of items to add, or maybe by a unique constraint and INSERT or IGNORE into table?)
- bonus: use attrs or pydantic to validate input before shoving into SQLlite

# Read from SQLite
<!-- - SQLite file specified via `.env` -->
- running with `-add` will cause to:
  - query SQLite for any that `need_adding = True`
  - iterate through results to add based on tasks above
  - update so `needs_adding = False`
  - or maybe this should run from docker and not cron. if docker, use a timer when it starts. Docker compose always restarts.

# To do still
- Always enable `PRAGMA journal_mode=WAL;'`

# Expanding Backlink bot
- Getting the cookie:
  - `http POST https://openlibrary.org/account/login username==scottreidbarnes+scottbot@gmail.com password==password_here 'Content-Type:application/x-www-form-urlencoded'`
  - Why does it work for my non-bot user? `http POST https://openlibrary.org/account/login username=srb36 password=\!abc`
  - Note the escapes in the above password.
  - Didn't work with a bot account?
  - I can't seem to authenticate with my bot account, but a but account is required, or API user, or admin. :(
- Import query: `http POST https://openlibrary.org/api/import/ia identifier==whatsgreatphonic00harc require_marc==true bulk_marc==false 'Cookie:session=/people/scott365bot%2C2023-01-20T02%3A26%3A33%2C4819b%246a6ea4ed169e1c9df5ee18a77ac1bbd8;Path=/'`


