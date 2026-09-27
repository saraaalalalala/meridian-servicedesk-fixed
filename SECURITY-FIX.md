# Meridian Service Desk — SQL Injection Fix (503M Lab 1)

This repository contains the Meridian Service Desk source code with the SQL
injection vulnerability from Lab 1 fixed.

## The vulnerability

The agent-only Advanced Ticket Search (`GET /tickets/search/?q=`) built its SQL
query by concatenating the user-supplied `q` value straight into the `WHERE`
clause:

```python
# src/helpdesk/views/staff.py  (function advanced_ticket_search) — BEFORE
sql = (
    "SELECT id, title, submitter_email, status "
    "FROM helpdesk_ticket "
    "WHERE title LIKE '%" + term + "%' "
    "ORDER BY created DESC"
)
with connection.cursor() as cursor:
    cursor.execute(sql)
    rows = cursor.fetchall()
```

Because the search term sits inside a single-quoted string with no escaping, an
attacker (any support agent) can break out of the string and rewrite the query.
This allowed, all through the search box:

- returning tickets that the search term does not match (`' OR id=7 -- -`)
- dumping the whole ticket table (`' OR '1'='1' -- -`)
- reading `auth_user` usernames and password hashes via `UNION SELECT`
- recovering the admin password hash blind, one character at a time
  (boolean-based and time-based, using `pg_sleep`)

## The fix

The query is now parameterized. The search term is passed to the database as a
bound parameter, so it is always treated as data and can never change the
structure of the query. The `LIKE` wildcards are added to the value, not to the
SQL string.

```python
# src/helpdesk/views/staff.py  (function advanced_ticket_search) — AFTER
sql = (
    "SELECT id, title, submitter_email, status "
    "FROM helpdesk_ticket "
    "WHERE title LIKE %s "
    "ORDER BY created DESC"
)
pattern = "%" + term + "%"
with connection.cursor() as cursor:
    cursor.execute(sql, [pattern])
    rows = cursor.fetchall()
```

After the fix, every payload above returns 0 rows (the injection string is
treated as a literal title that matches nothing), while a genuine search term
still returns its ticket. A normal apostrophe (e.g. `q=O'Brien`) is handled as
data instead of causing an error.

## Running it

Same as upstream:

```
docker compose -f lab/docker-compose.yml up --build
```

Then open http://127.0.0.1:5001 and sign in as `rmasri` / `Agent1234!`.

---
Fix applied for 503M Software Security, Lab 1 — SQL Injection.
