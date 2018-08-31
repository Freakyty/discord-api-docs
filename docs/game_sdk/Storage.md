# Storage

> danger
> The Discord Store is still in a beta period. All documentation and functionality can and will change.

We've been told that people playing games want to save their progress as they go, allowing them to come back where they left off and continue their epic journey of power.

Yeah, roguelikes. Even you.

If you need to save some data to some disks on your players' computers (and eventually some disks on those **BIG COMPUTERS** in the cloud), look no further! We've got the manager for you.

Discord's storage manager lets you save data mapped to a key for easy reading, writing, and deleting both synchronously and asynchronously. It's saved to a super special directory, the Holy Grail of file mappings, that's unique per Discord user—need to worry about your little brother overwriting your save file. In the future, Discord will keep _really_ close eyes on that directory and sync any changes to the data to cloud storage.

Creating this manager will also spawn an IO thread for async reads and writes, so unless you really want to be blocking, you don't need to be!

## Data Models

###### FileStat Struct

| name         | type   | description                                  |
| ------------ | ------ | -------------------------------------------- |
| Filename     | string | the name of the file                         |
| Size         | UInt64 | the size of the file                         |
| LastModified | UInt64 | timestamp of when the file was last modified |

## Read

Reads data synchronously from the game's allocated save file into a buffer. The file is mapped by key value pairs, and this function will read data that exists for the given key name.

Returns a `UInt32`.

###### Parameters

| name | type   | description                        |
| ---- | ------ | ---------------------------------- |
| name | string | the key name to read from the file |
| data | byte[] | the buffer to read into            |

## ReadAsync

Reads data asynchronously from the game's allocated save file into a buffer.

Returns a `Discord.Result` and a `byte[]` containing the data via callback.

###### Parameters

| name | type   | description                        |
| ---- | ------ | ---------------------------------- |
| name | string | the key name to read from the file |

###### Example

```cs
storeManager.ReadAsync("high_score", (Discord.Result result, byte[] data) =>
{
  if (result == Discord.Result.OK) {
    LoadHighScore(data);
  }
});
```

## ReadAsyncPartial

Reads data asynchronously from the game's allocated save file into a buffer, starting at a given offset and up to a given length.

Returns a `Discord.Result` and `byte[]` containing the data via callback.

###### Parameters

| name   | type   | description                          |
| ------ | ------ | ------------------------------------ |
| name   | string | the key name to read from the file   |
| offset | UInt64 | the offset at which to start reading |
| length | UInt64 | the length to read                   |

###### Example

```cs
storeManager.ReadAsyncPartial("high_score", 10, 8, (Discord.Result result, byte[] data) =>
{
  if (result == Discord.Result.OK) {
    LoadHighScore(data);
  }
});
```

## Write

Writes data synchronously to disk, under the given key name.

Returns `void`.

###### Parameters

| name | type   | description                 |
| ---- | ------ | --------------------------- |
| name | string | the key name to write under |
| data | byte[] | the data to write           |

###### Example

```cs
storageManager.Write("high_score", Encoding.UTF8.GetBytes("9999"));
```

## WriteAsync

Writes data asynchronously to disk under the given keyname.

###### Parameters

| name | type   | description                 |
| ---- | ------ | --------------------------- |
| name | string | the key name to write under |
| data | byte[] | the data to write           |

###### Example

```cs
storageManager.WriteAsync("high_score", Encoding.UTF8.GetBytes("9999"), Discord.Result result => {
  if (result == Discord.Result.OK) {
    Console.WriteLine("Wrote data");
  }
});
```

## Delete

Deletes written data for the given key name.

Returns `void`.

###### Parameters

| name | type   | description            |
| ---- | ------ | ---------------------- |
| name | string | the key name to delete |

###### Example

```cs
storageManager.Delete("high_score");
// Because you cheated. Jerk.
```

## Exists

Checks if data exists for a given key name.

Returns `bool`.

###### Parameters

| name | type   | description           |
| ---- | ------ | --------------------- |
| name | string | the key name to check |

###### Example

```cs
var highScore = storageManager.Exists("high_score");
if (!highScore) {
  Console.WriteLine("Couldn't find any highscore for you. Did you cheat? Jerk.");
}
```

## Stat

Returns file info for the given key name.

Returns a `Filestat`.

###### Parameters

| name | type   | description                    |
| ---- | ------ | ------------------------------ |
| name | string | the key name get file data for |

###### Example

```cs
var file = storageManager.Stat("high_score");
Console.WriteLine("File {0} is {1} in size and was last edited at {2}", file.Name, file.Size, file.LastModified);
```

## Count

Returns the count of files, for iteration.

Returns an `Int32`.

###### Parameters

None

###### Example

```cs
var numFiles = storageManager.Count();
for (int i = 0; i < numFiles; i++) {
  Console.WriteLine("We're at file {0}", i);
}
```

## StatIndex

Returns file info for the given index when iterating over files.

Returns a `FileStat`.

###### Parameters

| name  | type  | description                    |
| ----- | ----- | ------------------------------ |
| index | Int32 | the index to get file data for |

###### Example

```cs
var numFiles = storageManager.Count();
for (int i = 0; i < numFiles; i++) {
  var file = storageManager.StatIndex(i);
  Console.WriteLine("File is {0}", file.Name);
}
```

## Example: Saving, Reading, Deleting, and Checking Data

```cs
var discord = new Discord.Discord(clientId, Discord.CreateFlags.Default);
var storageManager = discord.GetStorageManager();

// Create some nonsense data
var contents = new byte[20000];
var random = new Random();
random.NextBytes(contents);

// Write the data asynchronously
storageManager.WriteAsync("foo", contents, res =>
{
    // Get our list of files and iterate over it
    for (int i = 0; i < storageManager.Count(); i++)
    {
        var file = storageManager.StatIndex(i);
        Console.WriteLine("file: {0} size: {1} last_modified: {2}", file.Filename, file.Size, file.LastModified);
    }

    // Let's read just a small chunk of data from the "foo" key
    storageManager.ReadAsyncPartial("foo", 400, 50, (result, data) =>
    {
        Console.WriteLine("partial contents of foo match {0}", Enumerable.SequenceEqual(data, new ArraySegment<byte>(contents, 400, 50)));
    });

    // Now let's read all of "foo"
    storageManager.ReadAsync("foo", (result, data) =>
    {
        Console.WriteLine("length of contents {0} data {1}", contents.Length, data.Length);
        Console.WriteLine("contents of foo match {0}", Enumerable.SequenceEqual(data, contents));

        // We just read it, but let's make sure "foo" exists
        Console.WriteLine("foo exists? {0}", storageManager.Exists("foo"));

        // Now delete it
        storageManager.Delete("foo");

        // Make sure it was deleted
        Console.WriteLine("post-delete foo exists? {0}", storageManager.Exists("foo"));
    });
});
```