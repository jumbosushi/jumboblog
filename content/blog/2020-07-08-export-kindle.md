+++
title = "Export data from Kindle vocab builder"
date = "2020-07-08"
slug = "export-kindle-vocab-builder"
+++

When I read books these days, I prefer e-books mainly due to the dictionary feature. In order to expose myself more to unfamiliar words, I sought ways to:
1. Export words from Kindle's vocab builder feature
2. Create a vim keybinding to import a random word from the list for my journals

Note: I use a Kindle Paperwhite. Other firmware might have a different configuration.

## Exporting from `vocab.db`

What I didn't know was that all the words you've ever looked up in Kindle are stored in a SQLite database file called `vocab.db`. Once you connect the Kindle to your computer, it will be located at `Kindle/system/vocabulary/vocab.db`.

Open the database file with the `sqlite3 vocab.db` command in the terminal.

You can try the `.schema` command to see table schemas:

```sql
sqlite> .schema

CREATE TABLE WORDS (
    id TEXT PRIMARY KEY NOT NULL,
    word TEXT,
    stem TEXT,
    lang TEXT,
    category INTEGER DEFAULT 0,
    timestamp INTEGER DEFAULT 0,
    profileid TEXT
);

CREATE INDEX wordidx ON WORDS (word);

CREATE TABLE LOOKUPS (
    id TEXT PRIMARY KEY NOT NULL,
    word_key TEXT,
    book_key TEXT,
    dict_key TEXT,
    pos TEXT,
    usage TEXT,
    timestamp INTEGER DEFAULT 0
);

CREATE INDEX lookupwordkey ON LOOKUPS (word_key);
CREATE INDEX lookupbookkey ON LOOKUPS (book_key);

CREATE TABLE BOOK_INFO (
    id TEXT PRIMARY KEY NOT NULL,
    asin TEXT,
    guid TEXT,
    lang TEXT,
    title TEXT,
    authors TEXT
);

CREATE TABLE DICT_INFO (
    id TEXT PRIMARY KEY NOT NULL,
    asin TEXT,
    langin TEXT,
    langout TEXT
);

CREATE TABLE METADATA (
    id TEXT PRIMARY KEY NOT NULL,
    dsname TEXT,
    sgcontextent INTEGER,
    profileid TEXT
);

CREATE TABLE VERSION (
    id TEXT PRIMARY KEY NOT NULL,
    dsname TEXT,
    version INTEGER
);
```

We're only interested in exporting the `word` column from the `words` table. The following command will export them to a `vocab.txt` file:

```txt
.output vocab.txt
select word from words;
.exit
```

After you're done, you can delete the `vocab.db` file from the Kindle. The device will create a new database on the next lookup.

## Importing new words from a text file

Here's a vimscript snippet to import a random vocab word from a text file with `<Leader>v` in normal mode:

```txt
function! LoadVocab()
  return systemlist('shuf -n 1 ~/vocab.txt')
endfunction

nmap <Leader>v "=LoadVocab()<C-M>p
```

I write my journal within Vim, and I've managed to integrate this function into my script that loads the daily journal template, which is working surprisingly well.

Feel free to reach out to me if you have any questions!
