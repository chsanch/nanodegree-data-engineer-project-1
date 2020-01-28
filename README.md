# Sparkify 
This a project to apply data modeling with Postgres and build an ETL
pipeline using Python.

## Project Description
A startup called Sparkify wants to analyze the data
they've been collecting on songs and user activity on their new music streaming
app. The analytics team is particularly interested in understanding what songs
users are listening to. Currently, they don't have an easy way to query their
data, which resides in a directory of JSON logs on user activity on the app, as
well as a directory with JSON metadata on the songs in their app.

### Datasets
We will use two datasets: _Songs_ and _Logs_

#### Songs dataset
Each file on the path `data/song_data` will contain the following JSON object:

```javascript
{
    "num_songs": 1,
    "artist_id": "AR7G5I41187FB4CE6C",
    "artist_latitude": null,
    "artist_longitude": null,
    "artist_location": "London, England",
    "artist_name": "Adam Ant",
    "song_id": "SONHOTT12A8C13493C",
    "title": "Something Girls",
    "duration": 233.40363,
    "year": 1982
}

```
#### Logs dataset
Each file on the path `data/log_data` will contain a list of the following object:

```javascript
{
    "artist": null,
    "auth": "Logged In",
    "firstName": "Walter",
    "gender": "M",
    "itemInSession": 0,
    "lastName": "Frye",
    "length": null,
    "level": "free",
    "location": "San Francisco-Oakland-Hayward, CA",
    "method": "GET",
    "page": "Home",
    "registration": 1540919166796.0,
    "sessionId": 38,
    "song": null,
    "status": 200,
    "ts": 1541105830796,
    "userAgent": "\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.143 Safari/537.36\"",
    "userId": "39"
}
```

### Database schema
For this project we will use the following database schema on PostgreSQL:

#### Table Artists
```sql
artists (
    artist_id character varying PRIMARY KEY,
    name character varying,
    location character varying,
    latitude numeric,
    longitude numeric
);

```
#### Table Songplays
```sql
songplays (
    songplay_id SERIAL PRIMARY KEY,
    start_time timestamp without time zone,
    user_id integer,
    level character varying,
    song_id character varying,
    artist_id character varying,
    session_id integer,
    location character varying,
    user_agent character varying
);


```
#### Table Songs
```sql
songs (
    song_id character varying PRIMARY KEY,
    title character varying,
    artist_id character varying,
    year integer,
    duration numeric
);

```
#### Table Time
```sql
time (
    time_id SERIAL PRIMARY KEY,
    start_time timestamp without time zone,
    hour integer,
    day integer,
    week integer,
    month integer,
    year integer,
    weekday character varying
);

```
#### Table Users
```sql
users (
    user_id character varying PRIMARY KEY,
    first_name character varying,
    last_name character varying,
    gender character varying,
    level character varying
);
```
### Process
#### Database
To create the database and tables we have a script: `create_tables.py` which will import all the queries stored in `sql_queries` and will execute them.
First it will drop all existing tables and the database, then it will create the database and each table.

Each table has an index (Primar Key) which will be assigned from the data obtained on the ETL process, on the case of the table `time` we are using the data type `serial primary key` to allow postgres auto increment the `time_id` value for every record inserted.

We also are using the clause `ON CONFLICT DO NOTHING` on all the insert queries to avoid the script to raise an error if there is any conflict.

#### ETL
The script `etl.py` provides all the functions to perform the ETL process.

##### Songs dataset
For the song dataset it will execute the function `process_song_file` on each .json file located on the `data/song_data` folder. This function will extract data for the the `artists` and `songs` tables.

For the songs table it will retrieve the following list:
```python
song_data = df[['song_id','title','artist_id','year','duration']].fillna(0).values[0].tolist()
```

For the artists table it will retrieve the following list:
```python
artist_data = df[['artist_id','artist_name', 'artist_location', 'artist_latitude', 'artist_longitude']].fillna(0).values[0].tolist()
```

In both extraction methods we are using the Pandas Dataframe [method](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.fillna.html) `fillna(0)` to replace all NaN elements with 0s.

##### Logs dataset
To obtain data from the logs dataset, the script will filter the records on each .json file using the following method:

```python
    df = df[df.page == "NextSong"] 
```

also it will convert the timestamp value (`ts` attribute) to datetime with the following method:
```python
    t = pd.to_datetime(df['ts'], unit='ms')
```

this will allow us to obtain the data for the `time` table:
```python
time_data = (t, t.dt.hour, t.dt.day, t.dt.week, t.dt.month, t.dt.year,t.dt.weekday) 
```

To obtain data for the `users` table the script will execute:
```python
user_df = df[['userId','firstName','lastName', 'gender', 'level']].drop_duplicates() 
```
the `drop_duplicates()` method will allow it to obtain unique user records.

And finally, for the `songplays` table the script will first perform a search to obtain the `song_id` and the `artist_id` using the data provided by the `log` json file.
To store a _human readable_ `start_time` the script will transform the `ts` attribute to a datetime value with the following method:
```python
    df['ts'] = pd.to_datetime(df['ts'], unit='ms')
```
Once the `song_id` and `artist_id` attributes are retrieved the script will store the data into the `songplays` table.

## Project Instructions
### Create database and tables

- Run `python create_tables.py` to create the database and tables.
- Optional: Run `test.ipynb` Juppyter notebook to confirm the creation of the tables with the correct columns.

### ETL Process
After creating the database and tables we can start the ETL process:
- Run `python etl.py` 
This will extract data from all the .json files located at the `data` folder and store it on the Postgres database. 

