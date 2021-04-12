# World_Weather_Analysis

<p align="center">
  <img src="weather_data/Fig1.png" width="500">
</p>

(swap out image for heatmap)

## Table of Contents
* [Overview](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#overview)
* [Resources](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#resources)
* [Results](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#results)

## Overview:
"PlanMyTrip", is an online travel app that helps customers locate lodging anywhere in the world.  They want to improve their user interface by allowing customers to define their preferred temperature range, in order to generate a map of hotels around the world that meet their criteria.  The map will have interactive markers that will allow the customer to select and view the following information about each marker location: hotel, city, country, weather description and maximum temperature. 

## Resources

* APIs:
  * https://cloud.google.com/maps-platform
  * https://openweathermap.org/api
* Software: Python 3.7.9 in Jupyter Notebook interface
* Lesson Plan: UTA-VIRT-DATA-PT-02-2021-U-B-TTH, Module 6 Challenge

## Results:
### Retrieve Weather Data

(insert image of city_data_df)

* Cities throughout the world were selected at random.  
  * The `numpy` dependency was imported and random latitude, longitude pairs were created using `numpy.random`.  

    import numpy as np  
        lats = np.random.uniform(low=-90.000, high=90.000, size=2000)  
        lngs = np.random.uniform(low=-180.000, high=180.000, size=2000)  
        lat_lngs = zip(lats, lngs)
        coordinates = list(lat_lngs) 
        
  * The `citipy` module was imported in order create a list of cities based on the latitude, longitude pairs.

        cities = []  
        for coordinate in coordinates:  
            city = citipy.nearest_city(coordinate[0], coordinate[1]).city_name  
            if city not in cities:  
                cities.append(city)

  * Then iterate through the list of cities and make an API request for each city to gather weather data.

        import requests
        from config import weather_api_key
        url = "http://api.openweathermap.org/data/2.5/weather?units=Imperial&APPID=" + weather_api_key
        
        city_data = []
        record_count = 1
        set_count = 1
        
        for i, city in enumerate(cities):
            if (i % 50 == 0 and i >= 50):
                set_count += 1
                record_count = 1
                
            city_url = url + "&q=" + city.replace(" ","+")
            print(f"Processing Record {record_count} of Set {set_count} | {city}")
            record_count += 1
            try:
                city_weather = requests.get(city_url).json()
                    city_lat = city_weather["coord"]["lat"]
                    city_lng = city_weather["coord"]["lon"]
                    city_max_temp = city_weather["main"]["temp_max"]
                    city_humidity = city_weather["main"]["humidity"]
                    city_clouds = city_weather["clouds"]["all"]
                    city_wind = city_weather["wind"]["speed"]
                    city_country = city_weather["sys"]["country"]
                    city_weather_description = city_weather["weather"][0]["description"]
                    
                    city_data.append({"City": city.title(),
                          "Country": city_country,
                          "Lat": city_lat,
                          "Lng": city_lng,
                          "Max Temp": city_max_temp,
                          "Humidity": city_humidity,
                          "Cloudiness": city_clouds,
                          "Wind Speed": city_wind,
                          "Current Description": city_weather_description})
            except:
                print("City not found. Skipping...")
                pass

* Create a DataFrame and export to a file (CSV).  
  * Convert the array of dictionaries to a Pandas DataFrame.

        import pandas as pd
        city_data_df = pd.DataFrame(city_data)

  * Create the output file (CSV), WeatherPy_Database.csv, which will become the weather database.
        
        output_data_file = "WeatherPy_Database.csv"
        city_data_df.to_csv(output_data_file, index_label="City_ID")

### Create A Customer Travel Destinations Map

(add screenshot with markers and info boxes.)

* In order to create a customer travel destinations map, a filtered DataFrame based on customer temperature preferres was created from filtering the city_data using the `.loc` method. But first, pandas, requests, gmap, the Google API key and the database, WeatherPy_Database,csv were imported.  

       city_data_df = pd.read_csv("../Weather_Database/WeatherPy_database.csv")  
       
  * Then the user was asked to provide his/her minimum and maximum preferred temperatures.

        min_temp = float(input("What is the minimum temperature you would like for your trip? "))
        max_temp = float(input("What is the maximum temperature you would like for your trip? "))
        
        preferred_cities_df = city_data_df.loc[(city_data_df["Max Temp"] <= max_temp) & (city_data_df["Max Temp"] >= min_temp)]

  * Before moving forward, the preferred_cities_df was checked for empty rows using `preferred_cities_df.count()`.  "Country" had three empty rows which were subsequently dropped using `.dropna()`.  `clean_df = preferred_cities_df.dropna()`

  * A new DataFrame was created to store the hotel names along with the city, country, maximum temperature and coordinates.

        hotel_df = clean_df[["City", "Country", "Max Temp", "Current Description", "Lat", "Lng"]].copy()
        hotel_df["Hotel Name"] = ""
        
    At this point, the hotel column is empty.  The hotels with 5000 meter radius will be added by iterating through the hotel_df and making a request from Google Directions API.  Note that the `try` and `except` are use to prevent the loop from stopping if a hotel isn't found.

            params = {
            "radius": 5000,
            "type": "lodging",
            "key": g_key
        }

        for index, row in hotel_df.iterrows():
            lat = row["Lat"]
            lng = row["Lng"]
            
            params["location"] = f"{lat},{lng}"
            
            base_url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json"    
            hotels = requests.get(base_url, params=params).json()  
            
            try:
                hotel_df.loc[index, "Hotel Name"] = hotels["results"][0]["name"]
            except (IndexError):
               print("Hotel not found... skipping.")
  
    The hotel_df is checked and found to be clean.  In other words, none of the rows were missing hotels.  The "clean" DataFrame, `clean_hotel_df = hotel_df` was then exported to an output file, "WeatherPy_vacation.csv" for use later to develop the Travel Itinerary Map.
    
  * From `clean_hotel_df`, a Customer Travel Destinations Map was created that includes markers with an information box pop-up that provides the hotel name, city, country, current weather and maximum temperature information.    

        info_box_template = """
        <dl>
        <dt>Hotel Name</dt><dd>{Hotel Name}</dd>
        <dt>City</dt><dd>{City}</dd>
        <dt>Country</dt><dd>{Country}</dd>
        <dt>Current Weather</dt><dd>{Current Description} and {Max Temp} °F</dd>
        </dl> 
        """
        
        hotel_info = [info_box_template.format(**row) for index, row in clean_hotel_df.iterrows()]
        locations = clean_hotel_df[["Lat", "Lng"]]
        fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
        marker_layer = gmaps.marker_layer(locations, info_box_content=hotel_info)
        fig.add_layer(marker_layer)
        fig

### Create a Customer Travel Itinerary Map


[Back to the Table of Contents](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#table-of-contents)
