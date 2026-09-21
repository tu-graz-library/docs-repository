# InvenioRDM OpenSearch Stale Document Cleanup

```bash
#!/bin/bash

psql_start_cmd=$1
db_table_name=$2
index_alias_name=$3

# get all available IDs from the database
declare -A db_ids

while IFS= read -r id; do
    [[ -n "$id" ]] && db_ids["$id"]=1
done < <(
    $psql_start_cmd -Atc \
        "SELECT id FROM $db_table_name"
)


# iterate through the opensearch entries and remove the ones
# where the ID is not present in the DB
search_after=''

while true; do

    if [[ -z "$search_after" ]]; then
        response=$(curl -sS -X POST \
            "http://localhost:9200/$index_alias_name/_search" \
            -H 'Content-Type: application/json' \
            -d '{
                "size": 1000,
                "query": {"match_all": {}},
                "sort": [{"_id": "asc"}]
            }')
    else
        response=$(curl -sS -X POST \
            "http://localhost:9200/$index_alias_name/_search" \
            -H 'Content-Type: application/json' \
            -d "$(jq -n \
                --argjson after "$search_after" \
                '{
                    size: 1000,
                    query: {match_all: {}},
                    sort: [{_id: "asc"}],
                    search_after: $after
                }')"
        )
    fi

    count=$(jq '.hits.hits | length' <<< "$response")

    [[ "$count" -eq 0 ]] && break

    while IFS= read -r id; do
        [[ -z "$id" ]] && continue

        if [[ -z "${db_ids[$id]+x}" ]]; then
            echo "Stale: $id"

            curl -sS -X DELETE \
                "http://localhost:9200/$index_alias_name/_doc/$id"

            echo
        fi
    done < <(jq -r '.hits.hits[]._id' <<< "$response")

    search_after=$(jq -c '.hits.hits[-1].sort' <<< "$response")
done
``` 

## Purpose

This script is intended to keep an InvenioRDM OpenSearch index consistent with the PostgreSQL database.

Its main purpose is:

> **Find OpenSearch documents whose IDs no longer exist in PostgreSQL and delete those stale documents from OpenSearch.**

In InvenioRDM, PostgreSQL is the authoritative source for record data, while OpenSearch contains a searchable representation of those records.

Conceptually:

```text
PostgreSQL                         OpenSearch
-----------                        ----------
record A  ──────────────────────►  record A
record B  ──────────────────────►  record B
record C  ──────────────────────►  record C
                                   record X  ← stale
                                   record Y  ← stale
```

The script identifies `record X` and `record Y` as stale and removes them from OpenSearch.

Normally this script should not be needed because the services remove both opensearch and database records, but for some entities the services are not yet implemented (e.g. vocabularies).

## Requirements

The script works only in environments that have access to both opensearch and database of the running instance. This is the case with the TU Graz infrastructure.

---

## Overall Process

The script performs four main steps:

1. Read all record IDs from PostgreSQL.
2. Store those IDs in a Bash associative array.
3. Iterate through all documents in the OpenSearch index in batches of 1,000.
4. Delete any OpenSearch document whose `_id` is not present in PostgreSQL.

---

## 1. Script Arguments

The script is intended to accept three arguments:

```bash
psql_cmd_start=$1
db_table_name=$2
index_alias_name=$3
```

These represent:

| Argument | Purpose                                      |
| -------- | -------------------------------------------- |
| `$1`     | PostgreSQL command used to execute the query |
| `$2`     | PostgreSQL table containing the record IDs   |
| `$3`     | OpenSearch index or alias to inspect         |

For example, conceptually:

```bash
./sync_index.sh "psql -U inveniordm" "award_metadata" "awards"
```

---

## 2. Collect IDs from PostgreSQL

The script creates a Bash associative array:

```bash
declare -A db_ids
```

It then queries PostgreSQL for all available IDs:

```sql
SELECT id FROM <table>
```

Each returned ID is stored in the associative array:

```bash
db_ids["abc"]=1
db_ids["def"]=1
db_ids["ghi"]=1
```

This gives the script an efficient way to check whether an ID exists in the database.

For example:

```bash
if [[ -z "${db_ids[$id]+x}" ]]; then
    ...
fi
```

means:

> Check whether `$id` exists in the set of IDs retrieved from PostgreSQL.

Using an associative array is useful here because the script may need to check a very large number of OpenSearch documents.

---

## 3. Iterate Through OpenSearch

The script queries OpenSearch using:

```json
{
    "size": 1000,
    "query": {
        "match_all": {}
    },
    "sort": [
        {
            "_id": "asc"
        }
    ]
}
```

This means:

* retrieve up to 1,000 documents;
* don't apply any filtering;
* sort documents by `_id`.

---

## 4. Compare OpenSearch IDs with PostgreSQL IDs

For every OpenSearch document, the script extracts:

```bash
jq -r '.hits.hits[]._id'
```

It then checks whether that ID exists in the PostgreSQL ID set.

Conceptually:

```text
OpenSearch document
        │
        ▼
     extract _id
        │
        ▼
Is _id present in PostgreSQL?
       / \
     yes  no
      │    │
      ▼    ▼
    keep  delete
```

---

## 5. Delete Stale Documents

If an OpenSearch ID isn't found in PostgreSQL:

```bash
if [[ -z "${db_ids[$id]+x}" ]]; then
```

the script considers it stale.

It prints:

```text
Stale: <id>
```

and sends a DELETE request:

```bash
curl -sS -X DELETE \
    "http://localhost:9200/<index>/_doc/<id>"
```

---

## 6. Important Assumption

The script relies on:

```text
PostgreSQL `id`
        =
OpenSearch `_id`
```

This should be verified for the specific InvenioRDM index being cleaned.

If the OpenSearch `_id` does not correspond directly to the PostgreSQL record ID, the script would need to extract the appropriate ID from `_source` instead.
