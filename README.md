# Searching-for-OP_CAT
This repository is relating my work to search for the first OP_CAT transaction on the bitcoin blockchain main network

## Run a bitcoin node
I already had a node ready but it wasn't running since a few weeks, reindexing takes some time so.... 
During that time I searched through the google BigQuery public instance of the bitcoin blockchain, I used a query, to get a first glimpse at the dataset (Since OP_CAT is deactivated, I searched for the hexcode which is "7e" ) 

## First try
The query :
```
SELECT
  inputs.index AS input_index,
  inputs.spent_transaction_hash,
  inputs.spent_output_index,
  inputs.script_asm AS input_script_asm,
  inputs.script_hex AS input_script_hex,
  outputs.index AS output_index,
  outputs.script_asm AS output_script_asm,
  outputs.script_hex AS output_script_hex,
  outputs.required_signatures,
  outputs.type AS output_type,
  outputs.addresses AS output_addresses,
  outputs.value AS output_value,
  `hash` AS transaction_hash,
  block_timestamp,
  is_coinbase
FROM
  `bigquery-public-data.crypto_bitcoin.transactions`,
  UNNEST(inputs) as inputs,
  UNNEST(outputs) as outputs
WHERE
  (inputs.script_hex LIKE '%7e%' OR outputs.script_hex LIKE '%7e%')
  AND block_timestamp < TIMESTAMP('2011-01-01 00:00:00')
  AND NOT is_coinbase
ORDER BY
  block_timestamp ASC
```

### Export result to JSON

### Run a script
This script parse the json find in order to find the first OP_CAT and the first OP_DUP (for validation of the parsing algorithm) transaction of the dataset

```
import json
from bitcoinlib.scripts import Script

# Function to check for OP_CAT in the script

def contains_op_cat(script_hex):
    try:
        # Parse the script from its hexadecimal string format
        script = Script.parse_hex(script_hex)
        # Serialize the script to get the list of operations as bytes
        script_operations = script.serialize_list()
        # Check if any operation is the OP_CAT opcode
        # OP_CAT is represented by the byte value 0x7e
        return any(op == b'\x7e' for op in script_operations)
    except Exception as e:
        print(f"Error parsing script: {e}")
        return False

def contains_op_dup(script_hex):
    try:
        # Parse the script from its hexadecimal string format
        script = Script.parse_hex(script_hex)
        # Serialize the script to get the list of operations as bytes
        script_operations = script.serialize_list()
        # Check if any operation is the OP_DUP opcode
        # OP_DUP is represented by the byte value 0x76
        return any(op == b'\x76' for op in script_operations)
    except Exception as e:
        print(f"Error parsing script: {e}")
        return False

# Initialize variable to keep track of the first occurrence
first_op_cat_tx = None

# Read the entire JSON file as a JSON array
with open('dump.json', 'r') as jsonfile:
    data = json.load(jsonfile)  # This loads the entire JSON array into the variable 'data'

# Iterate over each transaction in the JSON array
for transaction in data:
    input_script_hex = transaction['input_script_hex']
    output_script_hex = transaction['output_script_hex']
    
    # Check both input and output scripts for OP_CAT
    if contains_op_cat(input_script_hex) or contains_op_cat(output_script_hex):
        block_timestamp = transaction['block_timestamp']
        # Compare timestamps to find the earliest transaction with OP_CAT
        if first_op_cat_tx is None or block_timestamp < first_op_cat_tx['block_timestamp']:
            first_op_cat_tx = transaction

# Output the result
if first_op_cat_tx:
    print(f"The first transaction hash with OP_CAT is: {first_op_cat_tx['transaction_hash']}")
else:
    print("No transaction with OP_CAT found.")

# Initialize variable to keep track of the first occurrence
first_op_dup_tx = None

# Iterate over each transaction in the JSON array
for transaction in data:
    input_script_hex = transaction['input_script_hex']
    output_script_hex = transaction['output_script_hex']
    
    # Check both input and output scripts for OP_DUP
    if contains_op_dup(input_script_hex) or contains_op_dup(output_script_hex):
        block_timestamp = transaction['block_timestamp']
        # Compare timestamps to find the earliest transaction with OP_DUP
        if first_op_dup_tx is None or block_timestamp < first_op_dup_tx['block_timestamp']:
            first_op_dup_tx = transaction

# Output the result
if first_op_dup_tx:
    print(f"The first transaction hash with OP_DUP is: {first_op_dup_tx['transaction_hash']}")
else:
    print("No transaction with OP_DUP found.")
```
And the results :
```
No transaction with OP_CAT found.
The first transaction hash with OP_DUP is: 6f7cf9580f1c2dfb3c4d5d043cdbb128c640e3f20161245aa7372e9666168516
```
# Left to do :
- [x] Finish indexing the bitcoin node
- [ ] Write the script that access directly the data of the local bitcoin node
- [ ] Was OP_CAT really alive on mainnet once ?
