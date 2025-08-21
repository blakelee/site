---
layout: post
published: true
date: 2025-08-20
title: Syncing Room Persistence with PowerSync in Kotlin Multiplatform
---
![](/media/supasync.png)

One of the most challenging problems in mobile development is keeping client state synced with the server. If you're building an app where multiple users need to see the same state, you’ve probably run into this.

## A Quick Look Back

Originally, we had Firebase. It worked, but being NoSQL it quickly became messy and expensive. For a while, it was the only option.

Then came PowerSync, which gave us sync on top of a proper SQL database. The catch? You had to fetch rows with cursors and manually map them into usable models.

Meanwhile, Room Persistence solved this problem years ago: foreign keys, embedded entities, and type-safe queries that “just work.” The natural question is—can we get PowerSync and Room working together?

The answer: yes, but with a few caveats. Let’s walk through it.

## Integration Requirements

To make PowerSync and Room cooperate, you’ll need to ensure:

1.  Room tables are created before PowerSync initializes
    
2.  Both point to the same database path
    
3.  You’re using the new Rust client + RawTable API
    
4.  Room’s in-memory modification log is updated when PowerSync changes data
    

Let’s break each one down.

## 1\. Create Room Tables Before PowerSync

Room must initialize its schema first. To coordinate this, I wrote a `RoomAsyncCallback` that signals when Room is ready:

```
class RoomAsyncCallback : RoomDatabase.Callback() {
  val isReady: Job = Job()

  override fun onCreate(connection: SQLiteConnection) {
    super.onCreate(connection)
    isReady.complete()
  }

  override fun onOpen(connection: SQLiteConnection) {
    super.onOpen(connection)
    isReady.complete()
  }
}
```

Attach it to your Room builder:

```
val callback = RoomAsyncCallback()
val roomDatabase = MyDatabaseBuilder.builder()
  .setDriver(BundledSQLiteDriver())
  .addCallback(callback)
  .setQueryCoroutineContext(Dispatchers.IO)
  .build()
```

### Note: It is important that the callback is a single instance. We will be using this callback class for PowerSync also.

Then, wait for Room before spinning up PowerSync:

```
class PowerSyncDatabaseProvider(
  private val callback: RoomAsyncCallback,
  private val factory: DatabaseDriverFactory,
  private val scope: CoroutineScope,
  private val path: DatabasePath
) {
  val database: Deferred<PowerSyncDatabase> = CompletableDeferred()

  init {
    scope.launch {
      callback.isReady.join()
      val db = PowerSyncDatabase(
        factory = factory,
        schema = schema,
        dbFilename = path.name,
        dbDirectory = path.directory
      )
      database.complete(db)
    }
  }
}
```

## 2\. Point Both to the Same Database Path

Both libraries must share the same underlying file. Here’s a small helper:

```
data class DatabasePath(val absolutePath: String) {
  val name = absolutePath.substringAfterLast("/")
  val directory = absolutePath.removeSuffix(name)
}
```

On Android:

```
fun provideAndroidDatabasePath(context: Context): DatabasePath {
  return DatabasePath(context.getDatabasePath("name.db").absolutePath)
}
```

However on iOS it becomes a little trickier. When looking at the documentation of the `PowerSyncDatabase` constructor for `dbDirectory` there's this little tidbit:

> Optional database file directory path. This parameter is ignored for iOS.

After digging through the source, I found this

```
fun provideIosDatabasePath(): DatabasePath {
  return DatabasePath(databaseDirPath() + "/name.db")
}

fun databaseDirPath(): String = iosDirPath("databases")

private fun iosDirPath(folder: String): String {
  val paths = NSSearchPathForDirectoriesInDomains(
    NSApplicationSupportDirectory, NSUserDomainMask, true
  )
  val documentsDirectory = paths[0] as String
  val databaseDirectory = "$documentsDirectory/$folder"

  val fileManager = NSFileManager.defaultManager()
  if (!fileManager.fileExistsAtPath(databaseDirectory)) {
    fileManager.createDirectoryAtPath(databaseDirectory, true, null, null)
  }
  return databaseDirectory
}
```

Or if you want to use `co.touchlab.sqliter.DatabaseFileContext`

```
val path = DatabaseFileContext.databasePath("name.db", null)
val databasePath = DatabasePath(path)
```

Which we can then pass to `RoomDatabase.Builder`

```
class [Android|Ios]DatabaseBuilder(private val path: DatabasePath) : EventDatabaseBuilder {
  override fun builder(): RoomDatabase.Builder<EventDatabase> {
    return Room.databaseBuilder(path.absolutePath)
  }
}
```

## 3\. Use the Rust Client + RawTables

When connecting to the client with the Supabase connector I use a lifecycle but you're welcome to use any method you choose. However, when you do connect, you need to be using the Rust implementation. My code is as follows

```
class PowerSyncDatabaseConnection(
  private val powerSyncDatabaseProvider: PowerSyncDatabaseProvider,
  private val foregroundState: ForegroundState,
  private val connector: SupabaseConnector,
  private val scope: CoroutineScope
) {
  init {
    scope.launch {
      val database = powerSyncDatabaseProvider.database.await()
      launch {
        foregroundState.isForeground
          .collect { isForeground ->
            when (isForeground) {
              true -> database.connect(connector, options = SyncOptions(true))
              false -> database.disconnect()
            }
          }
      }
    }
  }
}
```

### Declaring the entities

Now that we have the Rust client being used, we can declare our tables. Let's make a couple sample tables -- `Profile` and `Todo` to stick with the todo list theme in many examples. We'll keep everything simple, both of them will be the same locally on device and on the server. Finally we'll create a `FullProfile` that contains the `Profile` and the `Todo`'s associated with the profile

```
@Entity(
  tableName = "todo",
  indices = [
    Index(value = ["profile_id"])
  ],
  foreignKeys = [
    ForeignKey(
      entity = ProfileEntity::class,
      parentColumns = ["id"],
      childColumns = ["profile_id"],
      onDelete = ForeignKey.CASCADE,
      onUpdate = ForeignKey.CASCADE
    )
  ]
)
data class TodoEntity(
  @PrimaryKey val id: String,
  @ColumnInfo(name = "profile_id") val profileId: String,
  val title: String,
)

data class FullProfile(
  @Embedded val profile: ProfileEntity,
  @Relation(
    parentColumn = "id",
    entityColumn = "profile_id"
  )
  val todos: List<TodoEntity>
)
```

Next we'll want to set up the DAO's for each one. We'll do this before setting up the PowerSync table because there's a trick you can do to save you some time. First we create both DAO's per table and we make sure to have an insert operation in it. I'll also be including the full object embedding so we can get reactive updates with our foreign keys as well.

```
@Dao
interface ProfileDao {
  @Insert
  suspend fun insert(profile: ProfileEntity)

  @Query("SELECT * FROM profile")
  fun getProfile(): Flow<FullProfile>
}

@Dao
interface TodoDao {
  @Insert
  suspend fun insert(todo: TodoEntity)
}
```

After making these, adding the entities to Room, AND adding your DAO to the Room database, compile the app. You should now have access to `ProfileDao_Impl` and `TodoDao_Impl` which will be useful for creating the `RawTable`'s for PowerSync.

### Making the schemas

The schema's for our `RawTable`'s should match the entities exactly as we created them, which in turn means that the schema needs to be the same on Supabase as well. The basic table structure should look like this

```
val todoTable = RawTable(
  name = "todo",
  put = PendingStatement(
    sql = TODO(), // Come back to this
    parameters = listOf(
      PendingStatementParameter.Id,
      PendingStatementParameter.Column("profile_id"),
      PendingStatementParameter.Column("title"),
    )
  ),
  delete = PendingStatement(
    "DELETE FROM todo WHERE id = ?", listOf(PendingStatementParameter.Id)
  )
)
```

Now for the trick. Open up `TodoDao_Impl` and look for the `createQuery` function. You should see a value like this

``"INSERT OR ABORT INTO todo (`id`,`profile_id`,`title`) VALUES (?,?,?)"``

Copy that into the the sql block that we labeled todo.

After this, do the same for the `Profile` table then make the schema. The order of the schema matters here -- If in Room we have our `@Database` annotation look like this

```
@Database(
  entities = [ProfileEntity::class, TodoEntity::class]
)
```

Then we need our PowerSync schema to look the same

```
val schema: Schema = Schema(
  tables = listOf(
    profileTable,
    todoTable
  )
)
```

## 4\. Update Room’s Invalidation Tracker

At that point we are just about ready for this to all work. We have the tables, entities, schemas, etc. all created. Everything is linked to everything else but now if we subscribe to our `Flow<FullProfile>` and modify a value on the server, it doesn't seem to update that new value. What's even more interesting is that if we query the local database, we can see that the value has been updated. What gives?

The reason why it doesn't update is because under the hood, Room using an invalidation tracker. If Room is the one that changes the values then it performs various methods in the invalidation tracker to trigger the appropriate refreshes per table.

After doing some digging into the source code, it looks like we can access it. What we need to do is listen to the changes on PowerSync, then for each updated table we invalidate it in the Room in-memory table, and finally call `InvalidationTracker.refreshAsync()`. Here's what that looks like.

```
class PowerSyncDatabaseInvalidator(
  private val powerSyncDatabaseProvider: PowerSyncDatabaseProvider,
  private val roomDatabase: MyDatabase,
  private val scope: CoroutineScope
) {
  init {
    scope.launch {
      val powerSyncDatabase = powerSyncDatabaseProvider.database.await()
      val indexes = schema.rawTables.withIndex().associate { it.value.name to it.index }

      powerSyncDatabase.onChange(indexes.keys)
        .collect { updatedTables ->
          if (updatedTables.isNotEmpty()) {
            updatedTables
              .mapNotNull { updatedTable -> indexes[updatedTable] }
              .forEach { tableId ->
                roomDatabase.useWriterConnection { writer ->
                  writer.execSQL(
                    """
                      UPDATE room_table_modification_log
                      SET invalidated = 1
                      WHERE table_id = $tableId
                    """.trimIndent()
                  )
                }
              }

            roomDatabase.invalidationTracker.refreshAsync()
          }
        }
    }
  }
}
```

The table id in the invalidation tracker comes from the ordering of the `@Database(entities = [...])` that you did above. So if you keep the order of your PowerSync schema the same as the order of your Room database entities then you can properly get the index from our PowerSync schema.

Now, when PowerSync updates, Room’s reactive queries fire as expected—even with relations.

## Wrapping Up

By combining PowerSync for syncing and Room for relational modeling + reactive queries, you get the best of both worlds:

*   Reliable Supabase-powered sync
    
*   Clean Room-based data access
    

It’s not without risks (ordering, paths, internal logs), but once wired up, it feels seamless.