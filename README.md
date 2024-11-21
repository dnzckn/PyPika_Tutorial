# PyPika Tutorial: Slicing, Averaging, and Preserving Order

This tutorial demonstrates how to use PyPika to create SQL queries that involve:
- Basic selection and filtering (slicing).
- Aggregation (averaging).
- Combining selection, averaging, and ensuring the order is maintained for other functions that depend on ordering.

## Step 1: Installation and Imports

First, ensure that you have PyPika installed:

```bash
pip install pypika
```

Now, import PyPika and any other required packages:

```python
from pypika import Table, Field, functions as fn
import sqlite3  # To simulate the database
import pandas as pd
import numpy as np
```

## Step 2: Creating a Sample Database

We will create a small SQLite database with a table named "ds1" and populate it with a realistic 3D dataset.

The dataset represents a grid mesh where `x` and `y` span from -1 to 1, and `z` is a Gaussian response as a function of `x` and `y` with added noise. We will also have `a` representing 10 copies with different noise levels simulating repeat measurements, and `b` representing two states that flip the sign of `z`.

```python
# Establish a connection to an in-memory SQLite database
connection = sqlite3.connect(':memory:')
cursor = connection.cursor()

# Create the "ds1" table
cursor.execute('''
CREATE TABLE ds1 (
    x REAL,
    y REAL,
    z REAL,
    a INTEGER,
    b INTEGER
);
''')

# Generate sample data
x_vals = np.linspace(-1, 1, 5)
y_vals = np.linspace(-1, 1, 5)
a_vals = range(1, 11)  # 10 copies with different noise
b_vals = [1, -1]  # Two states flipping the sign of z

sample_data = []
for x in x_vals:
    for y in y_vals:
        base_z = np.exp(-(x**2 + y**2))  # Gaussian response
        for a in a_vals:
            noise = np.random.normal(0, 0.1)  # Adding noise
            for b in b_vals:
                z = base_z * b + noise
                sample_data.append((x, y, z, a, b))

# Insert the sample data into "ds1"
cursor.executemany('INSERT INTO ds1 VALUES (?, ?, ?, ?, ?)', sample_data)
connection.commit()
```

## Step 3: Slicing and Averaging Using PyPika

### Example 1: Simple Selection
Select columns `x`, `y`, `z`, `a`, and `b` from `ds1` where `a == 1`.

```python
# Define the table
ds1 = Table('ds1')

# Build the query
query = ds1.select(ds1.x, ds1.y, ds1.z, ds1.a, ds1.b).where(ds1.a == 1)

# Print the query
print("SQL Query:")
print(query)

# Execute the query
result_df = pd.read_sql_query(str(query), connection)
print("Simple Selection Result:")
print(result_df)
```

### Example 2: Aggregating with Averaging
Select columns `x`, `y`, and calculate the average of `z` where `a == 1`. The result should be grouped by `x` and `y`.

```python
# Build the query for aggregation
query = (
    ds1
    .select(ds1.x, ds1.y, fn.Avg(ds1.z).as_('avg_z'))
    .where(ds1.a == 1)
    .groupby(ds1.x, ds1.y)
    .orderby(ds1.x, ds1.y)
)

# Print the query
print("SQL Query:")
print(query)

# Execute the query
result_df = pd.read_sql_query(str(query), connection)
print("Aggregation with Averaging Result:")
print(result_df)
```

### Example 3: Combining Slicing, Averaging, and Preserving Order
Select `x`, `y`, and `avg(z)` where `a == 1`, and ensure the result preserves the order of `x` and `y`.

```python
# Build the combined query
query = (
    ds1
    .select(ds1.x, ds1.y, fn.Avg(ds1.z).as_('avg_z'))
    .where(ds1.a == 1)
    .groupby(ds1.x, ds1.y)
    .orderby(ds1.x, ds1.y)  # Ensure the ordering is maintained
)

# Print the query
print("SQL Query:")
print(query)

# Execute the query
result_df = pd.read_sql_query(str(query), connection)
print("Combined Selection and Averaging with Order Preserved:")
print(result_df)

# Verify that the ordering is maintained for further processing
print("Is the DataFrame sorted by ['x', 'y']?")
print(result_df.equals(result_df.sort_values(by=['x', 'y'])))
```

### Example 4: Averaging Across `a` and Selecting `b`
Select `x`, `y`, and `avg(z)` across all values of `a`, while also selecting only the rows where `b == 1`. The result should be grouped by `x`, `y`, and preserve the order.

```python
# Build the query for averaging across `a` and selecting `b`
query = (
    ds1
    .select(ds1.x, ds1.y, fn.Avg(ds1.z).as_('avg_z'))
    .where(ds1.b == 1)
    .groupby(ds1.x, ds1.y)
    .orderby(ds1.x, ds1.y)
)

# Print the query
print("SQL Query:")
print(query)

# Execute the query
result_df = pd.read_sql_query(str(query), connection)
print("Averaging Across 'a' with 'b' Selected Result:")
print(result_df)
```

In this final example, we use `.where(ds1.b == 1)` to filter the rows where `b` is equal to 1, and then calculate the average of `z` across all values of `a`, grouped by `x` and `y`. The result is ordered by `x` and `y` to maintain consistency for further processing.
