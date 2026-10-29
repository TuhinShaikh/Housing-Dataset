# Star Wars ETL Project: MongoDB to SQL Server

## Project aim

Move starship data and its pilot relationships from MongoDB into related SQL Server tables using Python, pandas and SQLAlchemy.

The project follows **Extract → Transform → Load (ETL)**:

- **Extract:** retrieve starships and character records from MongoDB.
- **Transform:** prepare starship rows, match pilots and separate the relationships.
- **Load:** insert the prepared data into SQL Server and check the links.

## 1. Set up the project

The project uses three Python files:

| File | Purpose |
|---|---|
| `SW_Extract.py` | Inspect MongoDB documents, counts and DataFrame columns; test a SQL connection. |
| `connect_sql.py` | Create the SQL database and table structure. |
| `SW_Transform_and_Load.py` | Extract, transform, load and check the data. |

Keep the files in the same folder using these names, without downloaded numbered suffixes. Install the required Python packages:

```bash
python -m pip install pymongo pandas sqlalchemy pyodbc
```

MongoDB and SQL Server must be running, with ODBC Driver 18 installed. The uploaded SQL settings use a Windows named instance and integrated authentication; use your working server and authentication settings for your environment.

## 2. Inspect the MongoDB data

`SW_Extract.py` displays a sample document from each collection and counts the records:

```python
# Inspect an example document and count the collection
pprint(db["starships"].find_one())
print("Starships:", db["starships"].count_documents({}))
```

This helps identify the source fields and nested pilot lists before creating relational rows. The script also converts documents into DataFrames and displays their first rows and column names.

The uploaded scripts use lowercase `starwars`, `characters` and `starships`. Match these to your actual source names; your earlier working database used `StarWars`, `Characters` and `Starships`.

## 3. Extract the documents

The loader reads both collections into Python lists:

```python
# Connect to MongoDB and select the source database
client = MongoClient(mongo_uri)
db = client[database_name]

# Retrieve the starships and characters
starships_raw = list(db["starships"].find())
characters_raw = list(db["characters"].find())
```

`find()` retrieves the documents and `list()` collects them in Python memory. Starships provide the data to migrate; characters provide the names associated with pilot references.

The loader prints the number of documents retrieved so the source counts can be checked.

## 4. Transform the starships

The code creates one row for each starship and stores the rows in `starships_df`.

```python
# Assign SQL IDs and retain the original MongoDB IDs
for starship_id, ship in enumerate(starships_raw, start=1):
    mongo_id = str(ship["_id"])
    starship_id_by_mongo_id[mongo_id] = starship_id
```

- `enumerate(..., start=1)` assigns an integer ID to each starship.
- `str()` converts the original ObjectId into text for SQL storage.
- `ship.get()` retrieves fields such as name, model and manufacturer.
- `pd.DataFrame(starship_rows)` turns the prepared rows into a DataFrame.

The mapping dictionary keeps track of which SQL ID belongs to each MongoDB starship. Most specification fields remain text, preserving values such as `unknown` and crew ranges.

## 5. Match pilots to characters

The code builds a lookup dictionary using the existing character IDs:

```python
# Index characters by their MongoDB IDs
characters_by_mongo_id = {
    str(character["_id"]): character
    for character in characters_raw
}
```

For each starship's pilot reference, the code extracts the ID, converts it to text and finds the matching character:

```python
# Find the character associated with the pilot reference
pilot_mongo_id = str(pilot_mongo_id)
pilot = characters_by_mongo_id.get(pilot_mongo_id)
```

Successful matches become rows containing the starship ID, pilot ID and pilot name. Unmatched references are recorded in `unresolved_pilots` and skipped. Ships with no pilots produce no relationship rows.

`.drop_duplicates()` removes repeated starship–pilot pairs from the resulting DataFrame.

## 6. Remodel the pilot relationship

The intermediate pilot rows are separated into pilots and relationship rows:

```python
# Keep each pilot once, with their name
pilots_df = (
    starship_pilots_df[["pilot_mongo_id", "pilot_name"]]
    .drop_duplicates(subset="pilot_mongo_id")
)

# Keep the IDs that connect each starship to its pilots
links_df = starship_pilots_df[["starship_id", "pilot_mongo_id"]]
```

| Table | What it stores | Keys |
|---|---|---|
| `Starships` | Starship details and original MongoDB IDs. | Primary key: `StarshipID`; unique: `MongoID`. |
| `Pilots` | Each referenced pilot's ID and name. | Primary key: `PilotMongoID`. |
| `StarshipPilots` | The starship–pilot relationships. | Both columns are foreign keys and together form the primary key. |

A ship can have several pilots, and a pilot can fly several ships. The linking table represents this **many-to-many relationship**.

The pilot name depends on the pilot, rather than the starship–pilot pair, so it belongs in `Pilots`. This normalises the pilot relationship; it does not establish that every source field has been remodelled into 3NF.

## 7. Create the SQL database and tables

`connect_sql.py` connects through SQLAlchemy and creates `StarWarsDB` if necessary. It defines the tables using `Table()` and `Column()`.

```python
# Define the pilot table
pilots = Table(
    "Pilots",
    metadata,
    Column("PilotMongoID", String(50), primary_key=True),
    Column("PilotName", Unicode(150), nullable=False)
)
```

- `primary_key=True` identifies each row uniquely.
- `nullable=False` makes a value required.
- `ForeignKey()` requires a reference to an existing parent row.
- `PrimaryKeyConstraint()` prevents duplicate pairs in the linking table.
- `metadata.create_all(engine)` creates missing tables.

**The current `RESET_TABLES = True` setting drops and recreates the three tables.** Importing `engine` from this file executes that setup code too.

## 8. Load the prepared DataFrames

The loader renames columns to match the SQL schema, for example `starship_id` to `StarshipID`. It then deletes existing rows and inserts the prepared data:

```python
# Replace the current SQL rows in one transaction
with engine.begin() as conn:
    conn.execute(text("DELETE FROM StarshipPilots"))
    conn.execute(text("DELETE FROM Pilots"))
    conn.execute(text("DELETE FROM Starships"))

    starships_load.to_sql("Starships", conn, if_exists="append", index=False)
    pilots_load.to_sql("Pilots", conn, if_exists="append", index=False)
    links_load.to_sql("StarshipPilots", conn, if_exists="append", index=False)
```

Parent rows are inserted before their links to satisfy foreign keys. Deletes happen in the reverse order.

- `engine.begin()` commits these operations together on success or rolls them back on an error.
- `to_sql()` writes DataFrame rows into a SQL table.
- `if_exists="append"` adds rows to the existing table.
- `index=False` excludes the DataFrame index from the SQL columns.

The earlier table reset is separate from this transaction. Rerunning the loader replaces the existing SQL data.

## 9. Check the results

The script prints SQL row counts and joins the tables to show starships with their pilot names:

```sql
SELECT s.Name AS Starship, p.PilotName
FROM Starships s
JOIN StarshipPilots sp ON s.StarshipID = sp.StarshipID
JOIN Pilots p ON p.PilotMongoID = sp.PilotMongoID
ORDER BY s.Name;
```

| Check | Expected outcome |
|---|---|
| Starship count | SQL count matches the extracted starship count. |
| Unresolved pilots | Zero unmatched references before treating the migration as complete. |
| Pilot relationships | The joined output shows the expected pilots for each ship. |

Ships without pilots remain in `Starships` but are excluded from this inner-join output. These are checks performed by the code, not confirmed results from a new run.

The MongoDB connection is closed with `client.close()` at the end. This version loads starships and linked pilots only; full character details, films, planets and species are outside its SQL scope.

## References

- [PyMongo — Find documents](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/crud/query/find/)
- [pandas — DataFrame.to_sql](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_sql.html)
- [SQLAlchemy — Table definitions](https://docs.sqlalchemy.org/en/20/core/metadata.html)
- [SQLAlchemy — Connections and transactions](https://docs.sqlalchemy.org/en/20/core/connections.html)
