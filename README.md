Collaborative Project with team: Anas Alanqar, Emily Oh, Ethan Hong, Suna Lee, Juan Aldunate

## End-to-End Data Pipeline Project
In this project, I collaborated with the team to build an end-to-end ETL pipeline that serves data collected using API to data warehouse on Snowflake to serve real-time dashboard built on Tableau, providing information on US flight delays and weather.


## Overview

Bash script **run_pipeline.sh** orchestrates whole process from Python layer (for data collection and backfilling), Spark layer (preprocessing and joining), to Snowflake layer (for OLAP data loading & query).

<img src="pipeline.png" width="600" height="400">

* Each of the process are encapsulated in functions, to make pipeline more atomic. 
* On any error triggered during the pipeline that makes pipeline to break, it automatically run **`rollback()`** function to remove all the created files and directories.  
  * ***(The repository includes a `rollback.sh` script for manual rollbacks when needed. Additionally, the pipeline automatically triggers a rollback upon detecting an error, requiring no manual intervention.)***
  
#### Snowflake  
* SQL query executed by Bash loads data into Snowflake, creates table and views for aggregated measures, which is used on the Tableau layer. 

<p float="left">
  <img src="snowflake-1.png" width="400" height="300" />
  <img src="snowflake-2.png" width="400" height="300" />
</p>

#### Tableau 
* Tableau connects to Snowflake, and live connetor enables Tableau to refresh dashboard real-time without breaking any pipeline, when source data is updated. 
[Link to Tableau Dashboard](https://public.tableau.com/views/TeamProjectFlightDelayandWeatherDashboard/Story1)
<img src="dashboard.png" width="400" height="300">

## Dependencies

### Python libraries

* run_pipeline.sh includes following lines to install dependencies, but you can also manually set them up. 

~~~shell
pip install kagglehub tqdm geopy pyspark findspark pandas meteostat requests bs4
~~~

### Frameworks

This projects requires the following frameworks

* Spark
* Snowflake / Snowsql
* Python / pipenv (or any other python virtual env)

## About Data

Considering the pipeline running on the instance or virtual environment, all of the source data used in the project directly requests data from url (using `pandas`, or `request`), rather than saving CSV file locally.

Only CSV files saved in the repository are `airport_whitelist.csv` and `codes_to_carrier.csv`, which will be explained in ***Supplementarty data sources to backfill data*** section.

* ### Airport info data

  * https://raw.githubusercontent.com/lxndrblz/Airports/main/airports.csv

  * The project firectly requests data through url from pandas

    ~~~python
    url = "https://raw.githubusercontent.com/lxndrblz/Airports/main/airports.csv"
    df = pd.read_csv(url)
    df.to_csv("airports.csv", index=False)
    ~~~

* ### Flight dataset

  * 'Add data source'

    ```python
    path = kagglehub.dataset_download(
        "yuanyuwendymu/airline-delay-and-cancellation-data-2009-2018"
    )
    ```

* ### Weather dataset

  * **About Meteostat API**: 

    * Meteostat has a pre-derined library **Hourly**, **Daily** to fetch hourly / daily weather data for selected location (station), for select time rage. 

    ~~~python
    from meteostat import Hourly, Daily
    from meteostat import Stations
    
    data = Hourly(id, start, end)
    data = data.fetch()
    ~~~

  * The project collect weather data using **get_weather.py**

  * The script uses Meteostat API to fetch weather for designated region, and date. 

    ```python
    def get_weather_data(iata, start, end):
    
        icao = code_map[iata]
    
        stations = Stations()
        stations = stations.fetch()
    
        select_station = stations[stations["icao"] == icao]
        id = select_station.index.values[0]
    
        data = Hourly(id, start, end)
        data = data.fetch()
        # make weather_data directory if not exists
    
        return data
    ```



## Supplementarty data sources to backfill data

Some of the data sources above are incomplete, including huge portion of Nulls in fields with important information. Also, some information are not included in the source data above. Therefore, we utilize external data sources (either directly requesting through url, or scraping from html source) to create a mapping table.

* **Airline codes to carrier mapping table**

  * Source : https://en.wikipedia.org/wiki/List_of_airlines_of_the_United_States

  * Used following codes to create carrier code - carrier name mapping table

  * ~~~python
    if os.path.exists("codes_to_carrier.csv"):
        print("🔄Loading carrier codes from codes_to_carrier.csv")
        codes_to_carrier = pd.read_csv("codes_to_carrier.csv")
    else:  # Collect data from wikipedia
        print("🚀Downloading carrier codes from wikipedia")
        carrier_codes_url = (
            "https://en.wikipedia.org/wiki/List_of_airlines_of_the_United_States"
        )
        response = requests.get(carrier_codes_url)
        soup = BeautifulSoup(response.content, "html.parser")
    
        airline_tables = soup.findAll("table", class_="wikitable sortable")
        codes_to_carrier = {}
    
        for table in airline_tables:
            airlines = table.findAll("tr")[1:]
            for i in airlines:
                iata, icao, name = (i.findAll("td")[j].text.strip() for j in range(2, 5))
                codes_to_carrier[iata] = name
    
        # 2 missing carriers manually
        codes_to_carrier["VX"] = "Virgin America"
        codes_to_carrier["EV"] = "ExpressJet Airlines"
    
        codes_to_carrier = pd.DataFrame(
            codes_to_carrier.items(), columns=["Code", "Carrier"]
        )
    
        codes_to_carrier.to_csv("codes_to_carrier.csv", index=False)
    ~~~

* **Backfilling missing state, city names with latitude and longitude**

  * Source: geopy.geocoders Nominatim Python library.

  * Used following codes to input latitude, and longitude to get city / state name, for airports with Null in the fields.

  * ~~~python
    from geopy.geocoders import Nominatim
    geolocator = Nominatim(user_agent="my_app", timeout=3)
    
        def get_city_state_country(lat, lon):
            location = geolocator.reverse((lat, lon), language="en")
            if location:
                address = location.raw.get("address", {})
                return (
                    address.get("city", None),
                    address.get("state", None),
                    address.get("country", None),
                    address.get("town", None),
                )
            return None, None, None, None
    ~~~

* **Airport Whitelist**

  * Original source data includes more than 300 airports, even including minor airports or private owned airports, inflating amount of process the pipeline has to go through. 
  * Therefore, using US airline traffic statistics & Null ratio of weather data, we created whitelist for Airports and subsetted data.
  * Whitelist is included in airport_whiteliest.csv

---

