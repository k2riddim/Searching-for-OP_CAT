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

# Now let's the real deal begin
## Run bitcoin node
## Python script to find OP_CAT :
Just to be 100% sure, the script search for OP_CAT and OP_DUP, and inside the output and the input.
```
from datetime import datetime
from blockchain_parser.blockchain import Blockchain
from bitcoinlib.scripts import Script
import binascii
import os

# Path to the Bitcoin Core blockchain data
blockchain_directory = '/mnt/d/Bitcoin/data/blocks'

# Create a Blockchain object
blockchain = Blockchain(blockchain_directory)

 
def contains_opcode(script, opcode):
    try:
        # Convert the script to its byte representation
        script = Script.parse_hex(script_hex)
        # Check if the opcode is present in the script bytes
        
        return opcode in script.commands
    except Exception as e:
        print(f"Error parsing script: {e}")
        return False

# Define the end date for parsing as January 1st, 2011 because OP_CAT was deprecated before this date
end_date = datetime(2011, 1, 1)

# Define opcodes
OP_CAT = 126  # 0x7e
OP_DUP = 118  # 0x76

# Initialize counters
transactions_output_with_op_cat = 0
transactions_output_with_op_dup = 0

# Initialize counters
transactions_input_with_op_cat = 0
transactions_input_with_op_dup = 0

# Loop through the ordered blocks up to the end_date
for block in blockchain.get_ordered_blocks(os.path.join(blockchain_directory, 'index'), cache='index-cache.pickle'):
    block_datetime = block.header.timestamp  
    if block_datetime >= end_date:
        break  # Stop processing if the block date is beyond the end date

    for tx in block.transactions:
        for no, output in enumerate(tx.outputs):
            try:
              # Parse the script
                script_hex = output.script.hex
                if isinstance(script_hex, bytes):
                    script_hex = binascii.hexlify(script_hex).decode()
                    script = Script.parse_hex(script_hex)
                if contains_opcode(script, OP_DUP):
                    print("Found OP_DUP in transaction {} output {}.".format(tx.hash, no))
                    transactions_output_with_op_dup += 1
                if contains_opcode(script, OP_CAT):
                    print("Found OP_CAT in transaction {} output {}.".format(tx.hash, no))
                    transactions_output_with_op_cat += 1
            except Exception as e:
                print(f"Error parsing script for transaction {tx.hash}: {e}")
    
    for tx in block.transactions:
        if tx.is_coinbase():
            continue    
        else:
            for no, input in enumerate(tx.inputs):
                try:
                # Parse the script
                    script_hex = input.script.hex
                    if isinstance(script_hex, bytes):
                        script_hex = binascii.hexlify(script_hex).decode()
                        script = Script.parse_hex(script_hex)
                    if contains_opcode(script, OP_DUP):
                        print("Found OP_DUP in transaction {} input {}.".format(tx.hash, no))
                        transactions_input_with_op_dup += 1
                    if contains_opcode(script, OP_CAT):
                        print("Found OP_CAT in transaction {} input {}.".format(tx.hash, no))
                        transactions_input_with_op_cat += 1
                except Exception as e:
                    print(f"Error parsing script for transaction {tx.hash}: {e}")

# Print summary
if transactions_output_with_op_cat == 0:
    print("No transactions output with OP_CAT found before {}.".format(end_date.strftime('%Y-%m-%d %H:%M:%S')))
else:
    print(f"{transactions_output_with_op_cat} transactions with OP_CAT found before {end_date.strftime('%Y-%m-%d %H:%M:%S')}")
if transactions_output_with_op_dup == 0:
    print("No transactions output with OP_DUP found before {}.".format(end_date.strftime('%Y-%m-%d %H:%M:%S')))
else:
    print(f"{transactions_output_with_op_dup} transactions with OP_DUP found before {end_date.strftime('%Y-%m-%d %H:%M:%S')}")
if transactions_input_with_op_cat == 0:
    print("No transactions input with OP_CAT found before {}.".format(end_date.strftime('%Y-%m-%d %H:%M:%S')))
else:
    print(f"{transactions_input_with_op_cat} transactions with OP_CAT found before {end_date.strftime('%Y-%m-%d %H:%M:%S')}")
if transactions_input_with_op_dup == 0:
    print("No transactions input with OP_DUP found before {}.".format(end_date.strftime('%Y-%m-%d %H:%M:%S')))
else:
    print(f"{transactions_input_with_op_dup} transactions with OP_DUP found before {end_date.strftime('%Y-%m-%d %H:%M:%S')}")
```
## Result
```
No transactions output with OP_CAT found before 2011-01-01 00:00:00.
163550 transactions with OP_DUP found before 2011-01-01 00:00:00
No transactions input with OP_CAT found before 2011-01-01 00:00:00.
No transactions input with OP_DUP found before 2011-01-01 00:00:00.
```

# Conclusion :
CAT WAS NEVER ALIVE ! #teamded
# To do :
- [x] Finish indexing the bitcoin node
- [x] Query public bitcoin blockchain from BigQuery
- [x] Parse the results searching for the first OP_CAT transaction
- [x] Write a script that access directly the data of the local bitcoin node
- [x] Run the script and share it and the results on this github
- [x] Answer the question "Was OP_CAT really alive on mainnet once ?"
