# Lab: Persisting the People List with Room

## Abbreviations used in this document

| Abbreviation | Full term |
|---|---|
| UI | User Interface |
| SDK | Software Development Kit |
| DB | Database |
| DAO | Data Access Object |
| SQL | Structured Query Language |
| KSP | Kotlin Symbol Processing (Room's annotation-processor toolchain) |

**Project for this lab:** `MobileProgramming26/labs/JetPackNavigatePeopleTypeSafe` — the same project from the previous labs, migrated in place so the hardcoded people list is replaced by a real on-device database via **Room**.
**You will need:** `JetPackNavigatePeopleTypeSafe` already building and running from the previous labs, and a running emulator or physical device (API level 23 or higher — see section 6 for why).

By the end of this lab you will have replaced a hardcoded `val people = listOf(...)` with a real SQLite database that survives the app being fully closed and reopened — without changing anything about the two screens themselves.

---

## Table of contents

- [Learning objectives](#learning-objectives)
- [Recap: where we left off](#recap-where-we-left-off)
- [1. Why a database, and why Room specifically](#1-why-a-database-and-why-room-specifically)
- [2. Turn `Person` into a Room `@Entity`](#2-turn-person-into-a-room-entity)
- [3. Write the DAO: `PersonDao`](#3-write-the-dao-persondao)
- [4. Write the database class: `PeopleDatabase`](#4-write-the-database-class-peopledatabase)
- [5. Gradle setup: KSP and Room's dependencies](#5-gradle-setup-ksp-and-rooms-dependencies)
- [6. Checkpoint: the build will fail — here's why, and the fix](#6-checkpoint-the-build-will-fail--heres-why-and-the-fix)
- [7. Wire the database into `MainActivity`](#7-wire-the-database-into-mainactivity)
  - [7.1 Seed the database on first run](#71-seed-the-database-on-first-run)
  - [7.2 Read the list reactively](#72-read-the-list-reactively)
  - [7.3 What you should *not* need to touch](#73-what-you-should-not-need-to-touch)
- [8. Checkpoint: run it, and prove it actually persists](#8-checkpoint-run-it-and-prove-it-actually-persists)
- [9. If your existing UI tests still pass, read this](#9-if-your-existing-ui-tests-still-pass-read-this)
- [Wrap-up](#wrap-up)
- [Self-check](#self-check)
- [Exercise](#exercise)

---

## Learning objectives

By the end of this lab, you should be able to:

1. Explain why a hardcoded Kotlin list can't survive the app being closed, and what a real database adds that it can't.
2. Turn a plain `data class` into a Room `@Entity`, and write a `@Dao` interface with both `Flow`-returning and `suspend` query methods.
3. Explain why reading a Room query as a `Flow` and collecting it with `collectAsState` is the same reactive shape as `remember { mutableStateOf(...) }`.
4. Diagnose and fix a `minSdk` manifest-merge failure caused by adding a library with a higher minimum SDK requirement than your project.

---

## Recap: where we left off

`JetPackNavigatePeopleTypeSafe` currently defines its people like this, directly in `MainActivity.kt`:

```kotlin
data class Person(
    val id: Int,
    val name: String,
    val age: Int
)

val people = listOf(
    Person(id = 1, name = "Ada", age = 36),
    Person(id = 2, name = "Bob", age = 24),
    Person(id = 3, name = "Cleo", age = 41),
    Person(id = 4, name = "Dan", age = 19)
)
```

This is a plain Kotlin `val` — recreated from scratch, identical, every single time the app's process starts. There is no way for an edit to this list to survive the app being closed, because nothing is ever written to disk.

**Today's goal:** replace this hardcoded list with a real on-device SQLite database, using **Room** (Jetpack's recommended way to use SQLite from Android), so the data genuinely persists.

---

## 1. Why a database, and why Room specifically

A previous lab persisted a single saved value (a chosen language) using Jetpack **DataStore (Preferences)**. DataStore is built for flat key → value pairs — a handful of settings or flags. It has no concept of "a row," "a table," or "find me the record whose id is 3."

What you need today is different: a **collection of same-shaped records** (people), where you need "give me all of them" and "give me the one with this id." That's exactly what a relational database — and Room specifically — is for.

Room always has the same three pieces:

1. **`@Entity`** — a data class describing one row of one table.
2. **`@Dao`** (Data Access Object) — an interface listing the queries you want to run. Room generates the implementation for you at compile time.
3. **`@Database`** — ties your `@Entity` classes and `@Dao` interfaces together into one database you can actually get an instance of at runtime.

You'll add exactly one of each below.

---

## 2. Turn `Person` into a Room `@Entity`

Move `Person` out of `MainActivity.kt` into its own file, `Person.kt`, and add two Room annotations:

```kotlin
package com.example.jetpacknavigatepeopletypesafe

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "people")
data class Person(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val name: String,
    val age: Int
)
```

Delete the old `data class Person(...)` declaration from `MainActivity.kt` once this file exists.

**What changed and why:**
- `@Entity(tableName = "people")` — this class now describes one row of a table named `"people"`.
- `@PrimaryKey(autoGenerate = true)` — `id` is now the table's primary key, and the database assigns the next integer automatically whenever you insert a `Person` without specifying one. This is why `id` needs a default value now (`= 0`) — you'll construct people like `Person(name = "Ada", age = 36)`, with no `id` at all, later in this lab.
- Everything else is unchanged — `name: String` and `age: Int` work exactly as before; nowhere else in the code that reads `person.name` or `person.age` needs to change.

✅ **Checkpoint:** the project won't build yet (Room's annotations aren't on the classpath). That's expected — continue to the next sections before trying to build.

---

## 3. Write the DAO: `PersonDao`

Create `PersonDao.kt`:

```kotlin
package com.example.jetpacknavigatepeopletypesafe

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface PersonDao {
    @Query("SELECT * FROM people ORDER BY id")
    fun getAll(): Flow<List<Person>>

    @Query("SELECT * FROM people WHERE id = :personId")
    suspend fun getById(personId: Int): Person?

    @Query("SELECT COUNT(*) FROM people")
    suspend fun count(): Int

    @Insert
    suspend fun insertAll(people: List<Person>)
}
```

**Notice this is an `interface` with no method bodies.** `@Dao` tells Room's annotation processor to generate a class implementing this interface, whose method bodies actually run the SQL in each `@Query`/`@Insert` annotation. You never write these method bodies by hand.

**`:personId` in the query string** is a bind parameter — Room matches it to the `personId: Int` function parameter *by name*, and checks this at compile time. Misspell one or the other and the project won't build.

**Why `getAll()` returns a `Flow` but `getById`/`count`/`insertAll` are plain `suspend fun`:**
- `Flow<List<Person>>` is a **standing subscription** — Room watches the table and automatically emits a new list every time any write touches it. Use this when you want to keep displaying up-to-date data without ever writing a manual "refresh" call.
- `suspend fun` is a **one-off** read or write — "run this once, off the main thread, give me the result." Use this when you want exactly one answer right now (you'll use `count()` this way in section 7.1).

---

## 4. Write the database class: `PeopleDatabase`

Create `PeopleDatabase.kt`:

```kotlin
package com.example.jetpacknavigatepeopletypesafe

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Person::class], version = 1, exportSchema = false)
abstract class PeopleDatabase : RoomDatabase() {
    abstract fun personDao(): PersonDao

    companion object {
        @Volatile
        private var instance: PeopleDatabase? = null

        fun getInstance(context: Context): PeopleDatabase {
            return instance ?: synchronized(this) {
                instance ?: Room.databaseBuilder(
                    context.applicationContext,
                    PeopleDatabase::class.java,
                    "people.db"
                ).build().also { instance = it }
            }
        }
    }
}
```

**What each piece does:**
- `@Database(entities = [Person::class], version = 1, exportSchema = false)` — lists every `@Entity` this database contains (just `Person`, today).
- `abstract fun personDao(): PersonDao` — also generated by Room; it's how you get a usable `PersonDao` back out of the database.
- The `companion object` with `@Volatile`/`synchronized`/`?:` is a standard thread-safe lazy-singleton pattern (not Room-specific) — it guarantees the whole app only ever opens one connection to `"people.db"`.

---

## 5. Gradle setup: KSP and Room's dependencies

**In the top-level `build.gradle.kts`**, add the KSP plugin alongside the ones already there:

```kotlin
plugins {
    id("com.android.application") version "8.13.2" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
    id("org.jetbrains.kotlin.plugin.serialization") version "2.0.21" apply false
    id("com.google.devtools.ksp") version "2.0.21-1.0.28" apply false
}
```

**In `app/build.gradle.kts`**, apply the plugin and add Room's dependencies:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("org.jetbrains.kotlin.plugin.serialization")
    id("com.google.devtools.ksp")
}

dependencies {
    // ... your existing dependencies, unchanged ...
    implementation("androidx.room:room-runtime:2.8.5")
    implementation("androidx.room:room-ktx:2.8.5")
    ksp("androidx.room:room-compiler:2.8.5")
}
```

**Why `ksp(...)` instead of `implementation(...)` for `room-compiler`:** that dependency is the annotation processor itself — code that runs *during the build* to generate your DAO implementation, not code your app needs at runtime. `ksp(...)` tells Gradle to run it during compilation and keep it out of the APK.

**The KSP version string has two parts.** `2.0.21-1.0.28` means "built against Kotlin compiler 2.0.21, KSP framework release 1.0.28." The first part must match your project's Kotlin version exactly (`2.0.21`, already used everywhere in this project) — otherwise the build fails with a version-mismatch error before it compiles any of your code.

Sync Gradle.

---

## 6. Checkpoint: the build will fail — here's why, and the fix

Run:

```bash
./gradlew assembleDebug
```

❌ **This will fail**, with an error that looks like this:

```
> Task :app:processDebugMainManifest FAILED
AndroidManifest.xml Error:
	uses-sdk:minSdkVersion 21 cannot be smaller than version 23 declared in library
	[androidx.room:room-runtime-android:2.8.5] .../room-runtime/AndroidManifest.xml
	as the library might be using APIs not available in 21
	Suggestion: use a compatible library with a minSdk of at most 21,
		or increase this project's minSdk version to at least 23,
		or use tools:overrideLibrary="androidx.room" to force usage (may lead to runtime failures)
```

**What this means:** `room-runtime` 2.8.5 declares, in its own manifest, that it requires API level 23 (Android 6.0) or higher — it genuinely calls platform APIs that don't exist on older versions. This project's `minSdk` is currently 21, so the build refuses to continue rather than produce an app that might crash on those older devices.

**The fix — raise `minSdk` to 23**, in `app/build.gradle.kts`:

```kotlin
defaultConfig {
    applicationId = "com.example.jetpacknavigatepeopletypesafe"
    minSdk = 23   // was 21 — androidx.room:room-runtime-android:2.8.5 requires 23+
    // ... unchanged otherwise ...
}
```

**Do not use the `tools:overrideLibrary="androidx.room"` suggestion from the error message.** That suppresses the *check*, not the actual incompatibility — the app could still crash at runtime on an older device. Raising `minSdk` is the honest fix: it makes the project's declared requirements match what it actually needs.

✅ **Checkpoint:** after this change, `./gradlew assembleDebug` should succeed. Look for a `kspDebugKotlin` task in the build output, running before `compileDebugKotlin` — that's Room generating your DAO implementation. (If you're curious what it generated, look in `app/build/generated/ksp/debug/kotlin/` for a file named `PersonDao_Impl.kt`.)

---

## 7. Wire the database into `MainActivity`

### 7.1 Seed the database on first run

The original four people don't disappear from the code — they become a **one-time seed**, inserted only if the table is currently empty. Near the top of `MainActivity.kt` (replacing the old `val people = listOf(...)`), add:

```kotlin
private val seedPeople = listOf(
    Person(name = "Ada", age = 36),
    Person(name = "Bob", age = 24),
    Person(name = "Cleo", age = 41),
    Person(name = "Dan", age = 19)
)
```

Then, inside `MainActivity.onCreate`, build the database and pass the DAO down into your composable hierarchy:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val db = PeopleDatabase.getInstance(applicationContext)
        val dao = db.personDao()

        setContent {
            PeopleNavHost(dao = dao)
        }
    }
}
```

`PeopleNavHost` now needs to accept that `dao` parameter, and run the seed check once per composition:

```kotlin
@Composable
fun PeopleNavHost(dao: PersonDao) {
    val navController: NavHostController = rememberNavController()

    LaunchedEffect(Unit) {
        if (dao.count() == 0) {
            dao.insertAll(seedPeople)
        }
    }

    // ... continued in 7.2 ...
}
```

`LaunchedEffect(Unit)` runs this check exactly once per fresh composition. Because the key is the constant `Unit`, it never re-runs on recomposition — only when `PeopleNavHost` is composed anew (effectively, once per app launch).

### 7.2 Read the list reactively

Still inside `PeopleNavHost`, replace any reference to the old hardcoded `people` with:

```kotlin
val people by dao.getAll().collectAsState(initial = emptyList())
```

`dao.getAll()` returns a `Flow<List<Person>>`. `.collectAsState(initial = emptyList())` is the bridge that turns any `Flow` into something a composable can read with `by` — exactly the same `by` property delegation you've used since the very first lab of this course, here delegating to a state object that Compose automatically refreshes every time Room emits a new list.

`initial = emptyList()` gives `people` something to be for the brief moment before the `Flow`'s first real emission arrives. In practice this means the list renders empty for a fraction of a frame on startup, then fills in — not something a user will notice.

### 7.3 What you should *not* need to touch

`PeopleListScreen` and `PersonDetailsScreen` — the actual screen composables — should need **zero changes**. If you find yourself editing either of them to make this work, something has gone wrong: both composables only ever receive plain values and callbacks (`people`/`onPersonClick`, `person`/`onBackClick`), never the database or `NavController` directly. The whole migration should be contained inside `PeopleNavHost` plus the three new/changed files above.

Your finished `PeopleNavHost` should look like this:

```kotlin
@Composable
fun PeopleNavHost(dao: PersonDao) {
    val navController: NavHostController = rememberNavController()

    LaunchedEffect(Unit) {
        if (dao.count() == 0) {
            dao.insertAll(seedPeople)
        }
    }

    val people by dao.getAll().collectAsState(initial = emptyList())

    NavHost(navController = navController, startDestination = PeopleList) {
        composable<PeopleList> {
            PeopleListScreen(
                people = people,
                onPersonClick = { person ->
                    navController.navigate(PersonDetails(personId = person.id))
                }
            )
        }
        composable<PersonDetails> { backStackEntry ->
            val details: PersonDetails = backStackEntry.toRoute()
            val person = people.first { it.id == details.personId }
            PersonDetailsScreen(
                person = person,
                onBackClick = { navController.popBackStack() }
            )
        }
    }
}
```

You'll need these new imports in `MainActivity.kt`:

```kotlin
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
```

---

## 8. Checkpoint: run it, and prove it actually persists

Build and install:

```bash
./gradlew installDebug
```

For a true first-run test, clear the app's data first:

```bash
adb shell pm clear com.example.jetpacknavigatepeopletypesafe
adb shell am start -n com.example.jetpacknavigatepeopletypesafe/.MainActivity
```

✅ **Checkpoint 1:** the app shows the same four people as always — Ada (36), Bob (24), Cleo (41), Dan (19). Tap one, confirm the details screen, tap Back. Behavior is identical to before this lab.

Now, the real test — **this is the one that actually proves persistence, not just that the list looks right on a first launch:**

```bash
adb shell am force-stop com.example.jetpacknavigatepeopletypesafe
adb shell am start -n com.example.jetpacknavigatepeopletypesafe/.MainActivity
```

✅ **Checkpoint 2:** the list comes back exactly the same — same four people, same order, **no duplicates**. If you saw duplicates, your seed `LaunchedEffect` is re-inserting every launch — check that `dao.count() == 0` is actually being checked, and that you're not accidentally calling `insertAll` unconditionally.

**If your build fails or the app crashes, check in this order:**
1. Did you raise `minSdk` to 23 and sync Gradle (section 6)?
2. Did both `build.gradle.kts` files get the KSP plugin?
3. Are `room-runtime`, `room-ktx`, and `ksp("androidx.room:room-compiler:2.8.5")` all present in `app/build.gradle.kts`'s `dependencies` block?
4. Does `Person.kt` have both `@Entity(tableName = "people")` and `@PrimaryKey(autoGenerate = true)` on `id`, with `id: Int = 0` (a default value)?
5. Is `PeopleNavHost` actually receiving `dao: PersonDao` as a parameter, and is `MainActivity.onCreate` passing one in?

---

## 9. If your existing UI tests still pass, read this

If you completed the UI testing lab and still have `PeopleNavHostTest.kt` from that project, try running it now:

```bash
./gradlew connectedDebugAndroidTest
```

It should still pass — all 8 tests, unmodified. This might be surprising: the data source just went from "instantly available, in memory" to "a `Flow` from a real database, with an async seed check on startup" — strictly more asynchronous than before.

**Why it isn't flaky:** `createAndroidComposeRule` automatically waits for Compose to be "idle" — no pending recomposition, no pending `LaunchedEffect` coroutines still running — before any assertion is allowed to run. The seed `LaunchedEffect` and the `collectAsState` read are both tracked by this idling mechanism automatically; the test framework genuinely waits for the async work to settle before checking what's on screen. You never had to add a wait or a sleep anywhere.

---

## Wrap-up

A hardcoded `val people = listOf(...)` is recreated identically every time an app's process starts — there's no way for any change to it to survive, because nothing is ever written anywhere durable. Room gives you a structured, queryable, on-device database for exactly this kind of problem. Today's migration touched three small files (`Person.kt` annotated, plus new `PersonDao.kt` and `PeopleDatabase.kt`) and about a dozen lines inside `PeopleNavHost` — `PeopleListScreen` and `PersonDetailsScreen` never changed at all.

---

## Self-check

1. Why can't DataStore (Preferences) hold "a list of people" the way it held one saved language string in an earlier lab?
2. What generates the actual implementation of the `PersonDao` interface, and when does that happen — compile time or runtime?
3. Why does `getAll()` return `Flow<List<Person>>` instead of a plain `suspend fun getAll(): List<Person>`? What would you lose if you used the plain version instead?
4. What is `dao.count() == 0` inside `LaunchedEffect(Unit)` guarding against? Why does it only matter on the very first launch?
5. What exact error did adding Room produce in section 6, and why is raising `minSdk` the right fix instead of `tools:overrideLibrary`?

---

## Exercise

Add a new query to `PersonDao`:

```kotlin
@Query("SELECT * FROM people WHERE age > :minAge")
fun getAdults(minAge: Int): Flow<List<Person>>
```

Then wire a simple toggle into `PeopleListScreen` (or a separate screen) that switches between showing everyone and showing only people over 30. Confirm the list updates reactively, with no manual refresh code anywhere — the same way `getAll()` already does.