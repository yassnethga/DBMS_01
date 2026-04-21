<!-- Bild einbetten: ![Alternativer Text](relativer/pfad/zur/datei) -->
![SQLite Logo](assets/sqlite3_logo.gif)

# DBMS Praktikum 1 — The Filesystem as a Database

**Course:** Databases · THGA Bochum
**Duration:** 60 minutes
**Instructor:** Stephan Bökelmann · [sboekelmann@ep1.rub.de](mailto:sboekelmann@ep1.rub.de)

---

## Learning Objective

By the end of this exercise you will be able to explain why a general-purpose tool — the filesystem combined with shell utilities — can in principle answer *any* question about stored data, but becomes increasingly unwieldy as queries grow more complex.

You will formulate the same three queries in both the shell and in SQL using SQLite, and directly experience how a dedicated query language

- reduces cognitive load,
- eliminates error-prone boilerplate code, and
- enables optimisation,

without adding new *capabilities* — but by absorbing unnecessary complexity internally.

**After completing this exercise you should be able to answer the following questions independently:**
- What is the difference between an *imperative* and a *declarative* query?
- What structural advantages does a relational database offer over loose files?
- When is the overhead of a database infrastructure worth introducing?

---

## Prerequisites

Before you begin, verify that all required tools are available on your system. Run each command and check the output.

**bash**
```bash
bash --version
```
> You should see a version string starting with `GNU bash, version 5` or higher. On macOS, the pre-installed version may be 3.x — in that case install a current version via Homebrew (`brew install bash`).

**awk**
```bash
awk --version
```
> You should see output mentioning `GNU Awk` followed by a version number. Any version from 4.x onward is sufficient.

**sqlite3**
```bash
sqlite3 --version
```
> You should see a version number such as `3.45.0 ...`. SQLite 3.x is pre-installed on most Linux distributions and macOS. On Debian/Ubuntu it can be installed with `sudo apt install sqlite3`.

**grep and sort** (part of GNU coreutils)
```bash
grep --version | head -1
sort --version | head -1
```
> You should see two lines, each mentioning `GNU grep` and `GNU coreutils` respectively. These are available on every standard Linux installation.

If any of the above commands return `command not found`, resolve the installation before continuing.

> **Screenshot 1:** Take a screenshot of your terminal showing all four successful version checks and insert it here.
>
> [`(https://github.com/yassnethga/DBMS_01/blob/6d009b5bc23d848590b217ff4a6300fe7eee704d/Terminal.png)`](https://github.com/yassnethga/DBMS_01/blob/844be71d3f5cbb55d6a0433b5669a3f5104aa93e/Terminal.png)

---

## Setup — Generating the Sample Dataset

Run the following script to create the sample dataset. It will produce a directory called `sensordata/` containing CSV files that simulate temperature readings from four sensors over 30 days.

```bash
mkdir -p sensordata && cd sensordata

sensors=("T01" "T02" "T03" "T04")
start=$(date -d "2026-03-01" +%s 2>/dev/null || date -j -f "%Y-%m-%d" "2026-03-01" +%s)

for i in $(seq 0 29); do
  day=$(date -d "@$((start + i * 86400))" +%Y-%m-%d 2>/dev/null || \
        date -r "$((start + i * 86400))" +%Y-%m-%d)
  for sensor in "${sensors[@]}"; do
    file="${sensor}_${day}.csv"
    echo "timestamp,sensor_id,location,value_celsius" > "$file"
    for h in 06 12 18; do
      value=$(awk "BEGIN{printf \"%.1f\", 15 + rand()*15}")
      echo "${day}T${h}:00:00,${sensor},bochum,${value}" >> "$file"
    done
  done
done

cd ..
echo "Dataset ready: $(ls sensordata/ | wc -l) files created."
```

> You should see the message `Dataset ready: 120 files created.` — one file per sensor per day (4 sensors × 30 days = 120 files).

Verify the result:
```bash
ls sensordata/ | head -8
cat sensordata/T01_2026-03-01.csv
```
> You should see a list of filenames following the pattern `<SENSOR>_<DATE>.csv`, and the contents of one file should look like this:
> ```
> timestamp,sensor_id,location,value_celsius
> 2026-03-01T06:00:00,T01,bochum,23.4
> 2026-03-01T12:00:00,T01,bochum,19.7
> 2026-03-01T18:00:00,T01,bochum,27.1
> ```

> **Screenshot 2:** Take a screenshot showing the output of `ls sensordata/ | head -8` and the contents of one CSV file, and insert it here.
>
>(https://github.com/yassnethga/DBMS_01/blob/e09fb1af2b6fb67cc275d10c1198a61198057e0a/terminal%202%20.png)

### What does the script do, line by line?

```bash
mkdir -p sensordata && cd sensordata
```
Creates the target directory (the `-p` flag suppresses errors if it already exists) and changes into it. The `&&` ensures the second command only runs if the first succeeded.

```bash
sensors=("T01" "T02" "T03" "T04")
```
Defines a bash array of four sensor identifiers. These will be used as both the sensor name in the data and as part of each filename.

```bash
start=$(date -d "2026-03-01" +%s 2>/dev/null || date -j -f "%Y-%m-%d" "2026-03-01" +%s)
```
Converts the start date into a Unix timestamp — the number of seconds since 1970-01-01 00:00:00 UTC. The two alternative `date` syntaxes handle Linux (`-d`) and macOS (`-j -f`) respectively; `2>/dev/null` silently discards the error from whichever variant does not apply.

```bash
for i in $(seq 0 29); do
  day=$(date -d "@$((start + i * 86400))" +%Y-%m-%d ...)
```
Iterates 30 times (days 0 through 29). For each iteration the Unix timestamp is advanced by one day (86 400 seconds = 24 h × 60 min × 60 s) and converted back into a human-readable date string in `YYYY-MM-DD` format.

```bash
  for sensor in "${sensors[@]}"; do
    file="${sensor}_${day}.csv"
    echo "timestamp,sensor_id,location,value_celsius" > "$file"
```
For every day, the inner loop iterates over all four sensors. It opens a new file (the `>` operator creates or overwrites it) and writes the header line.

```bash
    for h in 06 12 18; do
      value=$(awk "BEGIN{printf \"%.1f\", 15 + rand()*15}")
      echo "${day}T${h}:00:00,${sensor},bochum,${value}" >> "$file"
    done
```
For each sensor, three readings per day are recorded — at 06:00, 12:00, and 18:00. The temperature value is generated with `awk`: `rand()` returns a pseudo-random number in [0, 1), scaled to the range [15.0, 30.0) and formatted with one decimal place. The `>>` operator *appends* each reading to the existing file (in contrast to `>`, which would overwrite it).

---

### Excursus: Why is it called CSV?

**CSV** stands for **Comma-Separated Values**. The name describes the format exactly: each row of data is a plain text line, and the individual fields within that row are separated by commas. The first row is conventionally a header line that names the columns.

```
timestamp,sensor_id,location,value_celsius
2026-03-01T06:00:00,T01,bochum,23.4
```

The format has no official standard body and no strict specification, which is both its greatest strength and its most common source of headaches. It is universally readable — any text editor, spreadsheet application, programming language, and database tool can open a CSV file — and it requires no special software to create. A CSV file is simply text.

The comma as a delimiter is a convention, not a rule. Depending on locale and application, you will encounter tab-separated (`.tsv`), semicolon-separated (common in German-speaking countries, where the comma is used as a decimal separator), or pipe-separated (`|`) variants. The underlying concept is always the same.

The limitations of CSV become apparent quickly in practice: there is no type information (every field is a string until a program interprets it otherwise), no native support for nested structures, no schema enforcement, and no standard way to represent special characters or newlines within a field. CSV is, in a sense, the simplest possible data format — and for exactly that reason it serves as an excellent starting point for understanding why richer formats and dedicated storage systems were invented.

---

## Importing the Data into SQLite

Before solving the tasks, load all CSV data into a SQLite database. SQLite stores the entire database in a single file (`sensors.db`) and requires no server process.

```bash
sqlite3 sensors.db <<'EOF'
CREATE TABLE IF NOT EXISTS readings (
  timestamp     TEXT,
  sensor_id     TEXT,
  location      TEXT,
  value_celsius REAL
);
EOF

for f in sensordata/*.csv; do
  tail -n +2 "$f" | sqlite3 sensors.db \
    ".separator ','" \
    ".import /dev/stdin readings"
done

echo "Import complete."
```

> You should see `Import complete.` without any error messages. Verify the import with:
> ```bash
> sqlite3 sensors.db "SELECT COUNT(*) FROM readings;"
> ```
> You should see `360` — 120 files × 3 readings each.

> **Screenshot 3:** Take a screenshot showing the successful execution of the import script and the result of the `COUNT(*)` query, and insert it here.
>
> [(https://github.com/yassnethga/DBMS_01/blob/7a8d7b5ed8e87c28084dcc0b4c36b6ef0782b6fd/termin%203.png)](https://github.com/yassnethga/DBMS_01/blob/bbb52dcaf8f324f5e5c18343f5bb26e2ab34e194/termin%203.png)

---

## Task 1 — List all readings from sensor T02, sorted by date

### Shell

Using only standard shell tools, print every data row (no header) recorded by sensor `T02`, sorted chronologically.

<details>
<summary>Solution – Shell</summary>

```bash
grep -h "T02" sensordata/T02_*.csv \
  | grep -v "^timestamp" \
  | sort -t',' -k1,1
```

**Syntax explained:**

| Part | Meaning |
|---|---|
| `grep -h "T02"` | Searches all listed files for lines containing `T02`. The `-h` flag suppresses the filename prefix that `grep` would otherwise prepend to each matching line. |
| `sensordata/T02_*.csv` | Glob pattern: the shell expands this to all files whose name starts with `T02_` and ends with `.csv`. |
| `\|` | Pipe: forwards the standard output of the left-hand command as standard input to the right-hand command. |
| `grep -v "^timestamp"` | `-v` inverts the filter — only lines that do *not* start with `timestamp` are passed through. This removes any stray header lines. |
| `sort -t',' -k1,1` | Sorts the lines; `-t','` sets the comma as field delimiter, `-k1,1` sorts exclusively by the first field (the ISO timestamp). |

> You should see 90 lines (30 days × 3 readings), all from sensor T02, ordered from `2026-03-01` to `2026-03-30`.

</details>

### SQLite

```sql
SELECT *
FROM   readings
WHERE  sensor_id = 'T02'
ORDER  BY timestamp;
```

<details>
<summary>Solution – SQLite</summary>

```bash
sqlite3 sensors.db <<'EOF'
SELECT *
FROM   readings
WHERE  sensor_id = 'T02'
ORDER  BY timestamp;
EOF
```

**Syntax explained:**

| Clause | Meaning |
|---|---|
| `SELECT *` | Selects all columns of the result rows. The `*` is a wildcard for "all columns". |
| `FROM readings` | Names the table to read from. |
| `WHERE sensor_id = 'T02'` | Filter condition: only rows where the `sensor_id` column contains exactly `'T02'` are returned. String literals in SQL are enclosed in single quotes. |
| `ORDER BY timestamp` | Sorts the result in ascending order by the `timestamp` column. Because timestamps are stored in ISO format `YYYY-MM-DDTHH:MM:SS`, lexicographic order equals chronological order. |

> You should see the same 90 rows as in the shell solution. Notice how the intent is immediately readable. The database handles sorting internally and can use an index if one exists — the shell solution re-reads every matching file from disk on every invocation.

</details>

> **Screenshot 4:** Take a screenshot showing the output of the Task 1 SQLite query (the first and last few rows are sufficient), and insert it here.
>
> https://github.com/yassnethga/DBMS_01/blob/ec0a6f80c4ac7b531f9d84b1cdfe707ee61cc6a1/terminal%204.png

### Questions for Task 1

Answer the following questions in your own words and add your answers directly below each question.

**Question 1.1:** Why is `grep -v "^timestamp"` needed in the shell solution even though the files are already filtered with `grep -h "T02"`? Could this step be omitted? Justify your answer.

> *Your answer:*
     grep -h "T02" still includes header lines from each CSV file because every file contains its own header.
     `grep -v "^timestamp"` is needed to remove these headers.
     Without it, multiple header rows would appear in the output, making it messy and less readable. Therefore, it should
     not be omitted.

**Question 1.2:** The shell solution uses `sensordata/T02_*.csv` as a file pattern, even though `grep -h "T02"` already filters for `T02`. Why is the file pattern still important — and what would happen if you used `sensordata/*.csv` instead?

> *Your answer:*
     The file pattern `sensordata/T02_*.csv` ensures that only files for sensor T02 are processed.
     Even though `grep -h "T02"` filters the correct lines, using `sensordata/*.csv` would load all sensor files, which is          inefficient and unnecessary.
     Therefore, the file pattern is important because it limits input files and improves performance.

**Question 1.3:** The SQL solution uses `ORDER BY timestamp` even though `timestamp` is stored as type `TEXT`. Why does chronological sorting still work correctly? Under what condition would it fail?

> *Your answer:*
    The timestamps are stored in ISO format (YYYY-MM-DDTHH:MM:SS), which means they are lexicographically ordered.
    Because of this format, alphabetical (text) sorting produces the same result as chronological sorting.
    It would fail if the timestamp format were different, for example DD-MM-YYYY, where text order does not match time order.
---

## Task 2 — Find all readings above 25 °C, for any sensor, in March 2026

### Shell

Print `timestamp`, `sensor_id`, and `value_celsius` for every measurement that exceeds 25.0 °C and was recorded in March 2026.

<details>
<summary>Solution – Shell</summary>

```bash
grep -rh "2026-03" sensordata/ \
  | grep -v "^timestamp" \
  | awk -F',' '$4 > 25.0 {print $1, $2, $4}'
```

**Syntax explained:**

| Part | Meaning |
|---|---|
| `grep -rh "2026-03"` | `-r` searches a directory recursively; `-h` suppresses filenames in the output. All lines containing the substring `2026-03` are matched. |
| `grep -v "^timestamp"` | Removes header lines — a safety net since headers do not contain a date string, but good practice regardless. |
| `awk -F','` | Starts `awk` and sets the comma as the field separator (`-F`). `awk` processes each input line individually. |
| `'$4 > 25.0 {print $1, $2, $4}'` | The pattern `$4 > 25.0` is a condition: the action `{print ...}` is only executed when the fourth field (1-indexed) is greater than 25.0. `$1`, `$2`, `$4` reference the first, second, and fourth fields respectively. |

> You should see a variable number of lines depending on the randomly generated values — typically somewhere between 80 and 130 results. Each line contains the timestamp, sensor ID, and the temperature that exceeded 25 °C.

Caveats: floating-point comparison in `awk` works here, but breaks silently if the value format changes. The pipeline reads all 120 files even though the filename already encodes the date — there is no way to skip irrelevant files without additional logic.

</details>

### SQL

```sql
SELECT timestamp, sensor_id, value_celsius
FROM   readings
WHERE  value_celsius > 25.0
  AND  timestamp LIKE '2026-03-%'
ORDER  BY value_celsius DESC;
```

<details>
<summary>Solution – SQLite</summary>

```bash
sqlite3 sensors.db <<'EOF'
SELECT timestamp, sensor_id, value_celsius
FROM   readings
WHERE  value_celsius > 25.0
  AND  timestamp LIKE '2026-03-%'
ORDER  BY value_celsius DESC;
EOF
```

**Syntax explained:**

| Clause | Meaning |
|---|---|
| `SELECT timestamp, sensor_id, value_celsius` | Selects only three of the four columns explicitly — rather than using `*`. This reduces the amount of data transferred and makes the intent clearer. |
| `WHERE value_celsius > 25.0` | Numeric comparison: because `value_celsius` is declared as type `REAL`, SQLite guarantees that this value is always treated as a floating-point number. |
| `AND timestamp LIKE '2026-03-%'` | `LIKE` is a pattern match: `%` stands for any sequence of characters. This expression matches all timestamps that start with `2026-03-`. `AND` combines both conditions — both must be satisfied simultaneously. |
| `ORDER BY value_celsius DESC` | Sorts in descending order (`DESC`) by temperature — the highest value appears first. Without `DESC`, the default is ascending (`ASC`). |

> You should see the same rows as in the shell solution, this time sorted from the highest to the lowest temperature. With an index on `value_celsius` or `timestamp`, the database could skip irrelevant rows entirely.

</details>

> **Screenshot 5:** Take a screenshot showing the output of the Task 2 SQLite query and insert it here.
>
> `[insert screenshot]`

### Questions for Task 2

**Question 2.1:** The shell solution filters by date using `grep -rh "2026-03"`. What problem could arise if a sensor value happened to contain the string `2026-03` — for example as part of an error note? How does the SQL solution handle this problem?

> *Your answer:*

**Question 2.2:** The SQL solution uses `timestamp LIKE '2026-03-%'` for the date filter instead of a proper date function. Name one advantage and one disadvantage of this approach.

> *Your answer:*

**Question 2.3:** The SQL solution returns results sorted by `ORDER BY value_celsius DESC`. The shell solution does not include this sorting. Extend the shell solution to also sort by temperature in descending order and write your command here.

> *Your answer (extended shell command):*

---

## Task 3 — Per-sensor statistics: min, max, and average temperature

### Shell

For each of the four sensors, compute the minimum, maximum, and average value across all 30 days.

<details>
<summary>Solution – Shell</summary>

```bash
for sensor in T01 T02 T03 T04; do
  echo -n "$sensor: "
  grep -h "$sensor" sensordata/${sensor}_*.csv \
    | grep -v "^timestamp" \
    | cut -d',' -f4 \
    | awk '
        BEGIN { min=9999; max=-9999; sum=0; n=0 }
        { v=$1+0; if(v<min) min=v; if(v>max) max=v; sum+=v; n++ }
        END { printf "min=%.1f  max=%.1f  avg=%.1f\n", min, max, sum/n }
      '
done
```

**Syntax explained:**

| Part | Meaning |
|---|---|
| `for sensor in T01 T02 T03 T04; do ... done` | Loop over an explicit list of values. The variable `$sensor` takes one of the four values in each iteration. |
| `echo -n "$sensor: "` | Prints the sensor name; `-n` suppresses the trailing newline so that the `awk` output appears on the same line. |
| `cut -d',' -f4` | Extracts only the fourth field (`-f4`) using the comma as delimiter (`-d','`). Passes only the temperature values downstream. |
| `BEGIN { min=9999; max=-9999; sum=0; n=0 }` | An `awk` block executed before the first input line. Initialises variables for minimum, maximum, sum, and counter. |
| `{ v=$1+0; if(v<min) ... }` | Executed for every line. `$1+0` forces numeric interpretation. Min and max are updated; sum and counter are incremented. |
| `END { printf ... }` | An `awk` block executed after the last input line. Outputs the computed statistics in a formatted string. `%.1f` formats a floating-point number with one decimal place. |

> You should see four lines, one per sensor, each showing minimum, maximum, and average temperature. Because the values are randomly generated, the exact numbers will differ each time the dataset is created — but min and max should fall within [15.0, 30.0) and the average should be close to 22.5.

This works — but it requires four separate passes over the data, one loop iteration per sensor, and a non-trivial `awk` script that most readers need a moment to parse.

</details>

### SQL

```sql
SELECT sensor_id,
       MIN(value_celsius) AS min_temp,
       MAX(value_celsius) AS max_temp,
       ROUND(AVG(value_celsius), 1) AS avg_temp
FROM   readings
GROUP  BY sensor_id
ORDER  BY sensor_id;
```

<details>
<summary>Solution – SQLite</summary>

```bash
sqlite3 sensors.db <<'EOF'
SELECT sensor_id,
       MIN(value_celsius) AS min_temp,
       MAX(value_celsius) AS max_temp,
       ROUND(AVG(value_celsius), 1) AS avg_temp
FROM   readings
GROUP  BY sensor_id
ORDER  BY sensor_id;
EOF
```

**Syntax explained:**

| Clause | Meaning |
|---|---|
| `MIN(value_celsius) AS min_temp` | `MIN()` is an *aggregate function*: it computes the smallest value within a group of rows. `AS min_temp` assigns a readable alias to the result column. |
| `MAX(value_celsius) AS max_temp` | Analogous to `MIN()`, returns the largest value. |
| `ROUND(AVG(value_celsius), 1) AS avg_temp` | `AVG()` computes the arithmetic mean. `ROUND(..., 1)` rounds the result to one decimal place. |
| `GROUP BY sensor_id` | The key clause: it partitions all rows into groups — one group per distinct value in `sensor_id`. Each aggregate function is then computed *within* each group, not across the entire table. |
| `ORDER BY sensor_id` | Sorts the four result rows alphabetically by `sensor_id`. |

> You should see exactly four rows — one per sensor — with columns `sensor_id`, `min_temp`, `max_temp`, and `avg_temp`. A single declarative statement replaces the loop, the `grep`, the `cut`, and the entire `awk` aggregation. The database processes all rows in a single pass.

</details>

> **Screenshot 6:** Take a screenshot showing the output of the Task 3 SQLite query — the four rows with sensor statistics — and insert it here.
>
> `[insert screenshot]`

### Questions for Task 3

**Question 3.1:** The `awk` solution initialises `min=9999` and `max=-9999`. What would happen if all temperature values in the dataset were greater than 9999? How could the initialisation be made more robust?

> *Your answer:*

**Question 3.2:** The SQL solution uses `GROUP BY sensor_id`. What would the query return *without* this clause — i.e. if you ran `SELECT sensor_id, MIN(value_celsius), MAX(value_celsius), ROUND(AVG(value_celsius), 1) FROM readings`? Try it and describe the result.

> *Your answer:*

**Question 3.3:** Extend the SQL query with an additional column `COUNT(*) AS num_readings` that shows the total number of measurements for each sensor. Write the complete extended query here.

> *Your answer (extended SQL query):*

---

## Reflection

After completing all three tasks, answer the following questions:

**Question A — Writing effort:**
Which approach was easier to write correctly on the first try? Explain which properties of each language contributed to this.

> *Your answer:*

**Question B — Extensibility:**
What would you need to change in the shell solution if a fifth sensor `T05` were added? What about the SQL solution? Which approach scales better — and why?

> *Your answer:*

**Question C — Performance:**
The shell solution reads files from disk on every invocation. A database can cache frequently queried data in memory. What does this mean for performance with 10 000 sensors and multi-year measurement data?

> *Your answer:*

**Question D — Declarative vs. imperative:**
SQL is called a *declarative* language: you describe *what* you want, not *how* to compute it. Bash/awk, by contrast, are *imperative*: you write step by step how the result is to be computed. In which of the three tasks did you feel this difference most clearly? Justify your choice.

> *Your answer:*

> **Screenshot 7:** Take a final screenshot of your terminal showing the SQLite prompt with a query of your own invention on the `readings` table — one you came up with yourself that goes beyond the tasks above — and insert it here.
>
> `[insert screenshot]`

---

> **Key takeaway:** The filesystem *can* answer every query — but it delegates all the complexity to you. A database management system absorbs that complexity internally and exposes a language that is shaped exactly around the operations its data model supports.
