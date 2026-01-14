> [!CAUTION]
> This file contains the practical work solution, and this section specifically includes the required information and the solution.
> 
# SOLUTION / Lab 02: Binary Record Management with Table Schema

## Objective

Build a binary record management layer on top of the heap file system. This lab introduces structured records defined by a JSON schema.

Table Description (JSON)

Each table is described using a JSON schema in a file named ```schema.json```:


```json
{
  {
    "table_name": "Employee",
    "file_name" : "<file location>",
    "fields": [
      {"name": "id", "type": "int"},
      {"name": "name", "type": "char(20)"},
      {"name": "salary", "type": "float"}
    ]
  },
  {
    "table_name": "Dept",
    "file_name" : "<file location>",
    "fields": [
      {"name": "id", "type": "int"},
      {"name": "name", "type": "char(20)"},
      {"name": "Location", "type": "varchar(40)"}
    ]
  }
}
```
* int: 4 bytes
* float: 4 bytes
* char(n): fixed-length string
* varchar(n): 1 byte for the length  + n bytes content

## Tasks

### 1. Encode a record (dictionary → bytes)

```python
def encode_record(record_dict, table_name, schema) -> bytes:
    """
    Convert a Python dictionary record into binary form
    based on the JSON table description.
    """
  
```
A record should be provided in **Python dictionary form**, for example:

```python
record = {
    "id": 12,
    "name": "Alice",
    "age": 30
}
```

### 1. 1. Solution Encode a record (dictionary → bytes)

```python

import struct

def encode_record(record_dict, table_name, schema) -> bytes:
    table = next(t for t in schema if t["table_name"] == table_name)
    record_bytes = b""

    for field in table["fields"]:
        name = field["name"]
        ftype = field["type"]
        value = record_dict[name]

        if ftype == "int":
            record_bytes += struct.pack("i", value)

        elif ftype == "float":
            record_bytes += struct.pack("f", value)

        elif ftype.startswith("char"):
            size = int(ftype[5:-1])
            encoded = value.encode("utf-8")[:size]
            record_bytes += encoded.ljust(size, b"\x00")

        elif ftype.startswith("varchar"):
            size = int(ftype[8:-1])
            encoded = value.encode("utf-8")[:size]
            record_bytes += struct.pack("B", len(encoded))
            record_bytes += encoded.ljust(size, b"\x00")

    return record_bytes

```

### 2. Decode a record (bytes → dictionary)

```python
def decode_record(record_bytes, table_name, schema) -> dict:
    """
    Convert a binary record into a Python dictionary
    based on the JSON table description.
    """
    
```

### 2. 1. Solution Decode a record (bytes → dictionary)

```python

def decode_record(record_bytes, table_name, schema) -> dict:
    table = next(t for t in schema if t["table_name"] == table_name)
    record = {}
    offset = 0

    for field in table["fields"]:
        name = field["name"]
        ftype = field["type"]

        if ftype == "int":
            record[name] = struct.unpack("i", record_bytes[offset:offset+4])[0]
            offset += 4

        elif ftype == "float":
            record[name] = struct.unpack("f", record_bytes[offset:offset+4])[0]
            offset += 4

        elif ftype.startswith("char"):
            size = int(ftype[5:-1])
            raw = record_bytes[offset:offset+size]
            record[name] = raw.rstrip(b"\x00").decode("utf-8")
            offset += size

        elif ftype.startswith("varchar"):
            size = int(ftype[8:-1])
            length = struct.unpack("B", record_bytes[offset:offset+1])[0]
            offset += 1
            raw = record_bytes[offset:offset+size]
            record[name] = raw[:length].decode("utf-8")
            offset += size

    return record

```

### 3. Insert a Structured Record into the Heap File

```python
def insert_structured_record(table_name, schema, record_dict):
    """
    Encode a structured record and insert it into the heap file.
    """
```

### 3. 1. Solution Insert a Structured Record into the Heap File

```python
def insert_structured_record(table_name, schema, record_dict):
    table = next(t for t in schema if t["table_name"] == table_name)
    record_bytes = encode_record(record_dict, table_name, schema)

    with open(table["file_name"], "ab") as f:
        f.write(record_bytes)

```


### 4. Read All Structured Records from the Heap File

```python
def read_all_structured_records(table_name, schema):
    """
    Retrieve and decode all structured records from the heap file.
    """
```

### 4. 1. Solution Read All Structured Records from the Heap File

```python
def read_all_structured_records(table_name, schema):
    table = next(t for t in schema if t["table_name"] == table_name)

    records = []
    record_size = 0

    for field in table["fields"]:
        ftype = field["type"]
        if ftype in ("int", "float"):
            record_size += 4
        elif ftype.startswith("char"):
            record_size += int(ftype[5:-1])
        elif ftype.startswith("varchar"):
            record_size += 1 + int(ftype[8:-1])

    with open(table["file_name"], "rb") as f:
        while chunk := f.read(record_size):
            records.append(decode_record(chunk, table_name, schema))

    return records

```

# Lab 03 — Simple Query Processor over Heap File

## Objective
Build a minimal SQL-like query processor operating on structured records stored in the heap file.

### Supported Query Model
- **SELECT queries**:

```sql
SELECT field_list FROM table_name WHERE field = value
```

## INSERT Queries

- **INSERT queries**:

```sql
INSERT INTO table_name (field1, field2, ...) VALUES (value1, value2, ...)
```

### Examples

```sql
-- Select all fields
SELECT * FROM Employee;

-- Select specific fields with condition
SELECT name, salary FROM Employee WHERE id = 3;

-- Insert a new record
INSERT INTO Employee (id, name, salary) VALUES (4, 'Alice', 4500);
```

## Tasks

### 1. Parse SELECT Queries

```python
def parse_select_query(query, schema) -> dict:
    """
    Parse a simple SELECT query into a structured dictionary.
    Example output:
    {
        "fields": ["name", "salary"],
        "table": "Employee",
        "condition": {"field": "id", "value": 3}
    }
    """
```
### 1. 1. Solution Parse SELECT Queries

```python
def parse_select_query(query, schema) -> dict:
    """
    SELECT name, salary FROM Employee WHERE id = 3
    """
   
```

### 2. Parse INSERT Queries

```python
def parse_insert_query(query, schema) -> dict:
    """
    Parse a simple INSERT query into a structured dictionary.
    Example output:
    {
        "table": "Employee",
        "fields": ["id", "name", "salary"],
        "values": [4, "Alice", 4500]
    }
    """
    
```

### 2. 1. Solution Parse INSERT Queries

```python
def parse_insert_query(query, schema) -> dict:
    """
    INSERT INTO Employee (id, name, salary) VALUES (4, 'Alice', 4500)
    """
 ```


### 3. Execute Queries

```python
def execute_query(query, schema):
    """
    Execute a SELECT or INSERT query on the structured records stored in the heap file.
    """
    
```

### 3. 1. Solution Execute Queries

```python
## Mysqli Query
$result = mysqli_query($conn, $sql);

## Query
$result = $conn->query($sql);

## PDO Query
$stmt = $pdo->query($sql);

## Prepare Query
$stmt = $pdo->prepare($sql);
$stmt->execute();
          |
Execute ---

```


### 3. 2. Solution Execute Queries

```python
def execute_query(query, schema):
    """
    query = query.strip().upper()

    if query.startswith("SELECT"):
        parsed = parse_select_query(query, schema)
        records = read_all_structured_records(parsed["table"], schema)

        # Apply WHERE condition if exists
        if parsed.get("condition"):
            field = parsed["condition"]["field"]
            value = parsed["condition"]["value"]
            records = [r for r in records if r[field] == value]

        # Select specific fields
        if parsed["fields"] != ["*"]:
            records = [
                {f: r[f] for f in parsed["fields"]}
                for r in records
            ]

        return records

    elif query.startswith("INSERT"):
        parsed = parse_insert_query(query, schema)

        record = dict(zip(parsed["fields"], parsed["values"]))
        insert_structured_record(parsed["table"], schema, record)

        return "Record inserted successfully"

    else:
        raise ValueError("Unsupported query type")

    """
    
```

### Optional
Update the code to support AND conditions and comparison operators (>, <, ...)
