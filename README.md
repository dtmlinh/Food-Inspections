# Food Inspections Web Service Project

## Project Description
This project is a web service that implements:

- POST requests to load and process data from food inspections and tweets for restaurants in the Chicago area, which then is stored in a `PostgreSQL` database using `pyscopg2`:
	- food inspections:
		- can be loaded individually from json format or loaded in bulk from csv format
		- are checked for duplicative data and would be handled appropriately
	- tweets:
		- are checked if they match any restaurants in the database, by name (matched by checking 1,2,3, and 4 gram words from tweet text and compare with restaurant name) or location (matched by comparing tweet lon/lat and restaurant lon/lat)

- GET requests to configure database/transaction settings, process loaded data, and return query results:
	- database/transaction settings:
		- configurable options: reset DB, reset transaction size, abort/rollback transactions, enable bulk-loading
	- process loaded data:
		- add/remove table indexes (see note 1 below for details)
		- identify and link restaurants that are the same: record linkage from name, address, city, state, and zip with or without blocking and/or indexes (see note 2 below for details)
	-  return query results:
		- given a restaurant id, return all of its associated inspections
		- given an inspection id, return the restaurant info (incl. all linked restaurants)
		- given an inspection id, return all of its associated tweet keys

Contributors: Linh Dinh, Matthew Mauer

Frameworks/tools/packages used: `bottle`, `pyscopg2-binary` (or just `pyscopg2`), [`textdistance`](https://pypi.org/project/textdistance/), [`strsimpy`](https://pypi.org/project/strsimpy/)

## Usage
### DB initialization
To reset your database run `server/schema/drop.sql` and `server/schema/create.sql`.

### Server
To start the server, simply run `python3 server.py` in the `server` directory. There are a series of configuration parameters that can be passed to the server, to see them run `python3 server.py --help`. The server by default will run on localhost and port 30235. After running the server you should be able to visit `http://localhost:30235/hello` and see a message "Hello, World!" to verify that your web service is running (or run `curl http://localhost:30235/hello` in terminal). Note that you will need to get your database connection information working to start, which involves creating a `server.conf` file that follows the format in `server.conf.example`. 

### Send Requests
While the server is running, you can send GET and POST requests to the server by running `curl` in another terminal or visit `http://localhost:30235/...`). Some example use cases below:

- POST requests: 
	- to send POST requests from json or csv files, we need to use helper scripts `client.py` in the `client` directory (to see the configuration parameters for `client.py` run `python3 client.py --help`)
	- E.g., 
		- `python3 client.py -i inspections-example.json`, `python3 client.py -i inspections-example.csv --load bulk`, `python3 client.py -t tweets-example.json`
		- `python3 client.py -i inspections-example.json -t tweets-example.json`, `python3 client.py -i inspections-example.csv --load bulk --index post -t tweets-example.json`
		- `python3 client.py -i inspections-example.json --clean --halt` 

- GET requests: 
	- Directly from terminal: 
		- `curl http://localhost:30235/restaurants/1`, `curl http://localhost:30235/restaurants/by-inspection/<inspection_id>`, `curl -i http://localhost:30235/tweets/<inspection_id>`
		- `curl -i http://localhost:30235/bulkload/<your-file.csv>`
		- `curl -i http://localhost:30235/reset`, `curl -i http://localhost:30235/txn/2`, `curl -i http://localhost:30235/abort`, , `curl -i http://localhost:30235/buildidx`

![](food.gif)

### Notes

1. Indexes and tweet matching:

- The indexes (a hash index on upper(name) and a GIST index on location were used) brought significant performance gain for tweet matching.
- In all contexts, bulk loading took less than 1/100th the time to complete compared with transactional loading. And, it appears that as the transaction size expands, the load time slighlty diminishes.
- While we expected to see the load time of inspections/restaurants increase with the addition of an index (esp index_pre) due to the extra work the database must do to update the index for every entry, we actually saw only very small or negligible performance losses. This may be because a hash index and a Rtree index of points (GIST's impementation in this case) are both very cheap to update.

2. Text edit distance model for record linkage:

- two restaurants are considered a match if ALL of the conditions below are met (note: all strings are converted to uppercase before any comparisons):
	- their states are exact match
	- the first digit of their zip-codes are the same and the Jaro score between 2 zip-codes >= 0.7
	- their weighted similarity score based on name and address >= 0.875

- the weighted similarity score based on name and address between 2 restaurants is calculated as below:
	- we apply Jaro score for name, which accounts for 50% of the weighted similarity score
	- we split address into street number and street name
	- we apply Jaro Winkler score for street number, which accounts for 25% of the weighted similarity score
	- we apply Jaro Winkler score for street name, which accounts for 25% of the weighted similarity score

3. Blocking and record linkage performance:

- Enable blocking of the restaurant data to limit the candidate set of potential pairs (set `python3 server.py -s`)
- Within a block/umbrella set, indexes can be enabled to improve performance. 
- Rules to create a new primary record for all matched/linked restaurants: record with the most common street number & restaurant type & city, and the longest restaurant name & street name

## Structure of the software

1. directory `server`: contains scripts to implement the main web service

- `server.py`: contains functions for the main web/app service
- `db.py`: contains the data access object/layer that will interface with the DBMS
- `schema/create.sql` and `schema/drop.sql`: sql codes to initialize an empty DB
- `match_records.py`: contains helper functions for calculating text edit distances and creating a linear model for determining record matches for use in `db.py`

2. directory `client`: contains helper scripts to use the web service
	- `client.py` and `loader.py`: contains helper scripts to send POST requests from json or csv file to the web service
	- `inpsections-example.csv`, `inpsections-example.json`, `tweets-example.json`: example data files to test with `client.py`
