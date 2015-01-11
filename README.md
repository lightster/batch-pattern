The Batch Pattern
=================

Developing applications that make use of a relational database system can 
involve the need to pull large amounts of data from multiple tables.

For example, we might be tracking a database of music.  We might have tables
for:
 - Artists
 - Albums
 - Songs

If we were to want to display a list of artists and the albums they released,
how might we do this?

## N+1 Solution

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

$artists = $db->query($artist_sql);
while ($artist = $artists->fetch()) {
    echo "{$artists['last_name']}, {$artists['first_name']}\n";

    $albums = $db->query($album_sql, ['artist_id' => $artist['id']])
    while ($album = $albums->fetch()) {
        echo "> {$album['title']}\n";
    }
}
```
