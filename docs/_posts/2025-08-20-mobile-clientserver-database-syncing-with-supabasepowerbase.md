---
layout: post
published: true
date: 2025-08-20
title: Mobile Client/Server Database Syncing with Supabase+PowerBase
---
One of the most challenging problems in mobile development is keeping the state of your client synced with the server. If you're creating any sort of app where the state of what you're doing needs to be synced across multiple users, then you've likely hit this problem.

Originally there was Firebase. Firebase is NoSQL and quickly starts becoming expensive. However Firebase was really the only solution for a while.

Now with Supabase + PowerSync you can have a proper SQL database that syncs flawlessly. Previously you could do this with just PowerSync, but when it came down to it, you had to interact with a cursor to fetch all of the values, then when you want to combine all of your entities into the final model there was tons of mapping that needed to go on. Room Persistence has solved most of these issues for years by letting you query foreign key entities and have the embedded object all within one class and query. This is a huge improvement over what PowerSync makes you do. But is there a way to integrate both the power of PowerSync and Room together?

I managed to find a way to do it, but there are some risks that make it brittle as well as perform certain tasks in a specific order.

1.  Room needs to create its tables before PowerSync initializes otherwise the tables will never get created.
    
2.  You need the paths of both databases to point to the same database file.
    
3.  You need to be using the new Rust client on PowerSync as well as `RawTable`'s.
    
4.  You need to update the in-memory Room `room_table_modification_log` for triggers to work.
    

## 1\. Create Room's tables before PowerSync initializes

To do this I created a `RoomAsyncCallback` that has a `Job` inside of it that completes only when Room is done initializing. It looks like this (take note of the subtle use of [explicit backing fields](https://github.com/Kotlin/KEEP/blob/explicit-backing-fields/proposals/explicit-backing-fields.md))

```
class RoomAsyncCallback : RoomDatabase.Callback() {
  val isReady: Job
    field = Job()

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

As we can see from above, when `onCreate` is done, we call our callback which will let PowerSync know that we are ready to start. Likewise we do the same for `onOpen`. It is my understanding that when a Room database is being created for the first time, only `onCreate` gets called.

After we create the callback we need to integrate it by doing so

```
val callback = RoomAsyncCallback()
val roomDatabase = MyDatabaseBuilder.builder()
      .setDriver(BundledSQLiteDriver())
      .addCallback(callback)
      .setQueryCoroutineContext(Dispatchers.IO)
      .build()
```

### Note: It is important that the callback is a single instance. We will be using this callback class for PowerSync also.

Finally we can create our PowerSync instance. Since we have to rely on the callback being done first, I created my database using something like this

```
class PowerSyncDatabaseProvider(
  private val callback: RoomAsyncCallback,
  private val factory: DatabaseDriverFactory, // Standard DatabaseDriverFactory per platform
  private val scope: CoroutineScope,
  private val path: DatabasePath // We'll figure out how to get this later
) {
  val database: Deferred<PowerSyncDatabase>
    field = CompletableDeferred()
  
  init {
    scope.launch {
      callback.isReady.join()
      val database = PowerSyncDatabase(
        factory = factory,
        schema = schema,
        dbFilename = path.name,
        dbDirectory = path.directory
      )
      this.database.complete(database)
    }
  }
}
```

## 2\. The database path needs to be the same for Room and PowerSync

It comes as no surprise that if you want to use both Room and PowerSync then you need to be pointing to the same database. In order to keep track of the directory and name, I use a holder class `DatabasePath`

```
data class DatabasePath(val absolutePath: String) {
  val name: String = absolutePath.substringAfterLast("/")
  val directory = absolutePath.substring(0, absolutePath.length - name.length)
}
```

On Android this is easy to get

```
fun provideAndroidDatabasePath(context: Context): DatabasePath {
  return DatabasePath(context.getDatabasePath("name.db").absolutePath)
}
```

Then we can use it both for Room and PowerSync

```
class AndroidMyDatabaseBuilder(
  private val context: Context, 
  private val path: DatabasePath
) : MyDatabaseBuilder {
  override fun builder(): RoomDatabase.Builder<MyDatabase> {
    return Room.databaseBuilder(context, path.absolutePath)
  }
}
 
...

val powerSyncDatabase = PowerSyncDatabase(
  factory = factory,
  schema = schema,
  dbFilename = path.name,
  dbDirectory = path.directory
)
```

However on iOS it becomes a little trickier. When looking at the documentation of the `PowerSyncDatabase` constructor for `dbDirectory` there's this little tidbit:

> Optional database file directory path. This parameter is ignored for iOS.

Well if the directory is ignored for iOS, then what is the directory? The Room Persistence KMP documentation for iOS says to do this

```
fun getDatabaseBuilder(): RoomDatabase.Builder<AppDatabase> {
    val dbFilePath = documentDirectory() + "/my_room.db"
    return Room.databaseBuilder<AppDatabase>(
        name = dbFilePath,
    )
}

private fun documentDirectory(): String {
  val documentDirectory = NSFileManager.defaultManager.URLForDirectory(
    directory = NSDocumentDirectory,
    inDomain = NSUserDomainMask,
    appropriateForURL = null,
    create = false,
    error = null,
  )
  return requireNotNull(documentDirectory?.path)
}
```

However after trying that and after long time debugging and seeing that the database is empty but supposedly synced, I wondered if there's another location that's using the database for PowerSync. After drilling down into the source code of PowerSync where the database is being created, I eventually find usage of a separate library `co.touchlab.sqliter.DatabaseFileContext` that is providing us the directory of the iOS database.

```
val path = DatabaseFileContext.databasePath("name.db", null)
val databasePath = DatabasePath(path)
```

Or if you want to just copy the code like I did

```
fun provideIosDatabasePath(): DatabasePath {
  return DatabasePath(databaseDirPath() + "/name.db")
}

fun databaseDirPath(): String = iosDirPath("databases")

private fun iosDirPath(folder: String): String {
  val paths =
    NSSearchPathForDirectoriesInDomains(NSApplicationSupportDirectory, NSUserDomainMask, true)
  val documentsDirectory = paths[0] as String

  val databaseDirectory = "$documentsDirectory/$folder"

  val fileManager = NSFileManager.defaultManager()

  if (!fileManager.fileExistsAtPath(databaseDirectory))
    fileManager.createDirectoryAtPath(databaseDirectory, true, null, null)

  return databaseDirectory
}
```

Which we can then pass to `RoomDatabase.Builder`

```
class IosEventDatabaseBuilder(private val path: DatabasePath) : EventDatabaseBuilder {
  override fun builder(): RoomDatabase.Builder<EventDatabase> {
    return Room.databaseBuilder(path.absolutePath)
  }
}
```

## 3\. You need to be using the Rust client and `RawTable`'s

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
              true -> database.connect(connector, options = SyncOptions(true)) // Important!
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

## 4\. Update the Room modification log

At that point we are just about ready for this to all work. We have the tables, entities, schemas, etc. all created. Everything is linked to everything else but now if we subscribe to our `Flow<FullProfile>` and modify a value on the server, it doesn't seem to update that new value. What's even more interesting is that if we query the local database, we can see that the value has been updated. What gives?

The reason why it doesn't update is because under the hood, Room using an invalidation tracker. If Room is the one that changes the values then it performs various methods in the invalidation tracker to trigger the appropriate refreshes per table.

After doing some digging into the source code, it looks like we can access it. What we need to do is listen to the changes on PowerSync, then for each updated table we invalidate it in the Room in-memory table, and finally call `InvalidationTracker.refreshAsync()`. Seems pretty simple. Here's what that looks like

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

The table id in the invalidation tracker comes from the ordering of the `@Database(entities = [...])` that you did above. So if you keep the order of your PowerSync schema the same as the order of your Room database entities then you can properly get the indexed from our PowerSync schema.

After all this is setup, you'll have PowerSync syncing and Room emitting. After running this, I haven't run into any issues with foreign key relations which is great.