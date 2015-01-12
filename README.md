The Batch Pattern
=================

The batch pattern allows for a tradeoff to be made between the number of queries
required versus the amount of memory required when processing large amounts of 
data that is coming from multiple tables of a relational database.

## An Example Problem

We are tracking a database of music.  We might have tables
for:
 - Artists
 - Albums
 - Songs

If we were to want to display a list of artists and the albums they released,
how might we do this?

### N+1 Technique

We could request and iterate through all artists.  For each artist iterated,
we could then request and iterate through all albums associated with the
individual artist.

In terms of actually coding that, a simple way to do this processing might be:

```php
$artist_sql = <<<ARTIST_SQL
SELECT *
FROM artists
ORDER BY last_name, first_name
ARTIST_SQL;

$album_sql = <<<ALBUM_SQL
SELECT *
FROM albums
WHERE artist_id = ?artist_id?
ORDER BY title
ALBUM_SQL;

$artist_results = $db->query($artist_sql);
while ($artist = $artist_results->fetch()) {
    echo "{$artists['last_name']}, {$artists['first_name']}\n";

    $album_results = $db->query($album_sql, ['artist_id' => $artist['id']]);
    while ($album = $album_results->fetch()) {
        echo "> {$album['title']}\n";
    }
}
```

The problem with this solution is that it does not scale.  Known as the
"N+1 Problem", for every artist added to the database, another query is going
to need to be executed to retrieve that artist's albums.  Even with prepared
statements, each query executed will result in more latency during the network
roundtrip plus the overhead of executing the prepared statement one more time.

The "N+1 problem" is amplified even further when there are more tables that
need to be queried per iteration.  Think about if you wanted to get a list of
songs for each of the albums iterated.  For each artist, you need to execute
1 query to retrieve the list of albums plus `M` queries to retrieve the songs
(1 query for each of the artist's `M` albums).  Overall, you end up running
`N*M` queries, which with a large number of artist and album records will become
a problem quickly.

### Bulk Loading Technique

An alternative that allows the "N+1 problem" to be avoided is to load the needed
data in bulk:

```php
$artist_sql = <<<ARTIST_SQL
SELECT *
FROM artists
ORDER BY last_name, first_name
ARTIST_SQL;

$album_sql = <<<ALBUM_SQL
SELECT *
FROM albums
ORDER BY title
ALBUM_SQL;

$album_results = $db->query($album_sql);
$albums_by_artist_id = [];
while ($album = $album_results->fetch()) {
    $albums_by_artist_id[$album['artist_id']][$album['id']] = $album;
}

$artists = $db->query($artist_sql);
while ($artist = $artists->fetch()) {
    echo "{$artists['last_name']}, {$artists['first_name']}\n";

    if (!isset($albums_by_artist_id[$artist['id']])) {
        continue;
    }

    foreach ($albums_by_artist_id[$artist['id']] as $album) {
        echo "> {$album['title']}\n";
    }
}
```

This solution, like the N+1 solution, works well for small data sets.  However,
as the data set grows, the amount of memory the bulk load solution requires
grows in a non-scaleable way.  The amount of memory consumed for such a solution
can easily get into the hundreds of megabytes and even into the gigabytes, depending on the data set size.

## A Solution: The Batch Pattern

The batch pattern allows for a tradeoff to be made between the number of queries
required versus the amount of memory required when processing large amounts of
data that is coming from multiple tables of a relational database.

The batch pattern works by breaking the large data sets into smaller data sets.
Each smaller data set is then processed using the bulk loading technique.

### Generating Batches

In Postgres a simple and efficient way to generate batches is to create a
temporary table containing the primary IDs of the primary data element and an
auto-generated batch number:

```php
$batch_seq_sql = <<<SEQUENCE_SQL
CREATE TEMPORARY SEQUENCE artist_batches_seq
CACHE 10
SEQUENCE_SQL;

$batch_table_sql = <<<TABLE_SQL
CREATE TEMPORARY TABLE artist_batches
(
    batch_num INTEGER NOT NULL
        DEFAULT nextval('artist_batches_seq'::regclass) / ?batch_size?::integer,
    artist_id INTEGER NOT NULL
)
TABLE_SQL;

$batch_populator_sql = <<<POPULATOR_SQL
INSERT INTO artist_batches
(
    artist_id
)
SELECT artist_id
FROM artists
ORDER BY last_name, first_name
POPULATOR_SQL;

$db->query($batch_seq_sql);
$db->query($batch_table_sql, ['batch_size' => 25]);
$db->query($batch_populator_sql);
```
