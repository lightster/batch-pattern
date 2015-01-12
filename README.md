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
can easily get into the hundreds of megabytes and even into the gigabytes,
depending on the data set size.

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
        DEFAULT (nextval('artist_batches_seq'::regclass) / ?batch_size?::integer) + 1,
    artist_id INTEGER NOT NULL
)
TABLE_SQL;

$batch_populator_sql = <<<POPULATOR_SQL
INSERT INTO artist_batches
(
    artist_id
)
SELECT id
FROM artists
ORDER BY last_name, first_name
POPULATOR_SQL;

$batch_index_sql = <<<INDEX_SQL
CREATE UNIQUE INDEX batch_num_id_idx
    ON artist_batches
    USING btree (batch_num, artist_id);
CREATE UNIQUE INDEX id_idx
    ON artist_batches
    USING btree (artist_id);
INDEX_SQL;

$db->query($batch_seq_sql);
$db->query($batch_table_sql, ['batch_size' => 250]);
$db->query($batch_populator_sql);
$db->query($batch_index_sql);
```

### Processing Batches

Once the batches are generated, we can start iterating through the batches
fairly easily by grabbing the minimum and maximum batch numbers:

```php
$batch_num_sql = <<<BATCH_NUM_SQL
SELECT
    MIN(batch_num) AS min_batch_num,
    MAX(batch_num) AS max_batch_num
FROM artist_batches
BATCH_NUM_SQL;

$batch_nums = $db->getRow($batch_num_sql);
$min_batch = $batch_nums['min_batch_num'];
$max_batch = $batch_nums['max_batch_num'];

if ($min_batch) {
    for ($batch_num = $min_batch; $batch_num <= $max_batch; $batch_num++) {
        $this->processBatch($batch_num);
    }
}
```

### Process a Single Batch

Now that we are iterating through the batches, it is time we do something
with a batch.  We can adapt some of the logic we used in the bulk loading
technique to work with the batch pattern:

```php
// private function processBatch($batch_num)

$artist_sql = <<<ARTIST_SQL
SELECT *
FROM artists
WHERE EXISTS (
        SELECT 1
        FROM artist_batches batches
        WHERE batches.batch_num = ?batch_num?
            AND batches.artist_id = artists.id
    )
ORDER BY last_name, first_name
ARTIST_SQL;

$album_sql = <<<ALBUM_SQL
SELECT *
FROM albums
WHERE EXISTS (
        SELECT 1
        FROM artist_batches batches
        WHERE batches.batch_num = ?batch_num?
            AND batches.artist_id = albums.artist_id
    )
ORDER BY title
ALBUM_SQL;

$album_results = $db->query($album_sql, ['batch_num' => $batch_num]);
$albums_by_artist_id = [];
while ($album = $album_results->fetch()) {
    $albums_by_artist_id[$album['artist_id']][$album['id']] = $album;
}

$artists = $db->query($artist_sql, ['batch_num' => $batch_num]);
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

## Choose Your Tradeoff

Once the batch processing is in place, the tradeoff between the number of queries
versus memory consumption can be fine tuned.

Fine tuning is as simple as benchmarking timing and memory consumption statistics
when changing the size of the batches.  In most cases, the limitation is
available memory.  In this case, we can decrease the size of the batches until
each batch can safely run without hitting the memory limit.

Keep in mind that setting a batch size to infinity is very similar to using
the "Bulk Loading Technique", in which everything is loaded into memory.  Setting
the batch size to one is very similar to using the "N+1 Technique", which
requires individual queries for every primary data element.
