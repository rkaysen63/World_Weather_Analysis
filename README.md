# World_Weather_Analysis

<p align="center">
  <img src="weather_data/US_heatmap_cloudiness.png" width="800">
</p>

## Table of Contents
* [Overview](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#overview)
* [Resources](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#resources)
* [Results](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#results)

## Overview:
"PlanMyTrip", is an online travel app that helps users to locate lodging anywhere in the world.  They want to improve their user interface by allowing users to define their preferred temperature range, in order to generate a map of hotels in cities around the world meeting that criteria.  The map will have interactive markers that will allow the user to select and view the following information about each marker location: hotel, city, country, weather description and maximum temperature. 

## Resources

* APIs:
  * https://cloud.google.com/maps-platform
  * https://openweathermap.org/api
* Software: Python 3.7.9 in Jupyter Notebook interface
* Lesson Plan: UTA-VIRT-DATA-PT-02-2021-U-B-TTH, Module 6 Challenge

## Results:
### Weather Database

<p align="center">
  <img src="Weather_Database/city_data_df.png" width="800">
</p>

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

  * Weather data by city was gathered by iterating through the list of cities and making an API request for each city's weather data from the OpenWeatherAPI.

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

* The weather data was converted into a DataFrame, city_data_df and export to a file (CSV), "WeatherPy_Database.csv" for future use.  
  * The array of dictionaries was converted into a Pandas DataFrame.

        import pandas as pd
        city_data_df = pd.DataFrame(city_data)

  * Then the output file, "WeatherPy_Database.csv", which will become the weather database, was created from the DataFrame above.
        
        output_data_file = "WeatherPy_Database.csv"
        city_data_df.to_csv(output_data_file, index_label="City_ID")

### Customer Travel Destinations Map

<p align="center">
  <img src="Vacation_Search/WeatherPy_vacation_map.png" width="800">
</p>

* In order to create the *Customer Travel Destinations Map*, a filtered DataFrame based on user temperature preferences was created from filtering the city_data using the `.loc` method. But first, pandas, requests, gmap, the Google API key and the database, "WeatherPy_Database.csv" were imported.  

       city_data_df = pd.read_csv("../Weather_Database/WeatherPy_database.csv")  
       
* The user was asked to provide his/her minimum and maximum preferred temperatures.

        min_temp = float(input("What is the minimum temperature you would like for your trip? "))
        max_temp = float(input("What is the maximum temperature you would like for your trip? "))
        
        preferred_cities_df = city_data_df.loc[(city_data_df["Max Temp"] <= max_temp) & (city_data_df["Max Temp"] >= min_temp)]

* Before moving forward, the preferred_cities_df was checked for empty rows using `preferred_cities_df.count()`.  "Country" had three empty rows which were subsequently dropped using `.dropna()`.  `clean_df = preferred_cities_df.dropna()`

* A new DataFrame was created to store the hotel names along with the city, country, maximum temperature and coordinates.

        hotel_df = clean_df[["City", "Country", "Max Temp", "Current Description", "Lat", "Lng"]].copy()
        hotel_df["Hotel Name"] = ""
        
    At this point, the hotel column was empty.  The hotels within a 5000 meter radius of the latitude, longitude pairs were added by iterating through the hotel_df and making a request from Google Directions API.  Note that `try` and `except` are use to prevent the loop from stopping if a hotel isn't found.

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
  
    The hotel_df was checked and found to be clean.  In other words, none of the rows were missing hotels.  The "clean" DataFrame, `clean_hotel_df = hotel_df` was then exported to an output file, "WeatherPy_vacation.csv" for use later to develop the Travel Itinerary Map.
    
* From the `clean_hotel_df`, a *Customer Travel Destinations Map* was created that includes markers with an information box that opens when selected that provides the hotel name, city, country, current weather and maximum temperature information.    

        info_box_template = """
        <dl>
        <dt>Hotel Name</dt><dd>{Hotel Name}</dd>
        <dt>City</dt><dd>{City}</dd>
        <dt>Country</dt><dd>{Country}</dd>
        <dt>Current Weather</dt><dd>{Current Description} and {Max Temp} ??F</dd>
        </dl> 
        """
        
        hotel_info = [info_box_template.format(**row) for index, row in clean_hotel_df.iterrows()]
        locations = clean_hotel_df[["Lat", "Lng"]]
        fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
        marker_layer = gmaps.marker_layer(locations, info_box_content=hotel_info)
        fig.add_layer(marker_layer)
        fig

### Customer Travel Itinerary Map

<p align="center">
  <img src="Vacation_Itinerary/WeatherPy_travel_map.png" width="800">
</p>

The user was interested in a Florida vacation and an itinerary was created for travel between four Florida cities.  

* "WeatherPy_vacation.csv," was imported as vacation_df, as well as pandas, requests, gmap, and the Google API key#. From vacation_df, DataFrames were created for each city in the itinerary using the `.loc` on the "City" names.

      vacation_start = vacation_df.loc[vacation_df["City"] == "Saint Pete Beach"]
      vacation_end = vacation_df.loc[vacation_df["City"] =="Saint Pete Beach"]
      vacation_stop1 = vacation_df.loc[vacation_df["City"] =="Jacksonville"]
      vacation_stop2 = vacation_df.loc[vacation_df["City"] =="Labelle"] 
      vacation_stop3 = vacation_df.loc[vacation_df["City"] =="Palmetto"] 
    
* Then latitude-longitude pairs as tuples were retrieved from each city DataFrame using the to_numpy function and list indexing.

      start = vacation_start["Lat"].to_numpy()[0], vacation_start["Lng"].to_numpy()[0]
      end = vacation_end["Lat"].to_numpy()[0], vacation_end["Lng"].to_numpy()[0]
      stop1 = vacation_stop1["Lat"].to_numpy()[0], vacation_stop1["Lng"].to_numpy()[0]
      stop2 = vacation_stop2["Lat"].to_numpy()[0], vacation_stop2["Lng"].to_numpy()[0]
      stop3 = vacation_stop3["Lat"].to_numpy()[0], vacation_stop3["Lng"].to_numpy()[0]

* A direction layer map was created using gmaps and the start and end latitude-longitude pairs with stop1, stop2, and stop3 as the waypoints.

      import gmaps.datasets
      gmaps.configure(api_key=g_key)
      itinerary = gmaps.directions_layer(start, end, waypoints=[stop1, stop2, stop3], travel_mode='DRIVING')
      fig.add_layer(itinerary)
      fig

* In order to create a marker layer map between the four cities the four city DataFrames were combined into a single DataFrame, itinerary_df, using the concat() function.  The remaining code to add markers is similar to the code found in the section *Customer Travel Destinations Map* above, except that the data set has changed to four specific cities identified by the combined DataFrame, itinerary_df.

      destination_frames = [vacation_start, vacation_stop1, vacation_stop2, vacation_stop3, vacation_end]
      itinerary_df = pd.concat(destination_frames,ignore_index=True)

      info_box_template = """

      <dl>
      <dt>Hotel Name</dt><dd>{Hotel Name}</dd>
      <dt>City</dt><dd>{City}</dd>
      <dt>Country</dt><dd>{Country}</dd>
      <dt>Current Weather</dt><dd>{Current Description} and {Max Temp} ??F</dd>
      </dl> 
      """

      hotel_info = [info_box_template.format(**row) for index, row in itinerary_df.iterrows()]
      locations = itinerary_df[["Lat", "Lng"]]
      fig = gmaps.figure(center=(30.0, 31.0), zoom_level=1.5)
      marker_layer = gmaps.marker_layer(locations, info_box_content=hotel_info)
      fig.add_layer(marker_layer)
      fig

### Scatter Plots and Linear Regression

<p align="center">
  <img src="weather_data/LinRegressNHemi.png" width="500">
</p>

Image shown above is a scatterplot with linear regression showing the correlation of Maximum Temperature versus City Latitudes in the Northern Hemisphere.  The r value of this linear regression model is -0.9035, r^2 = 0.82 or 82%, which indicates that the linear regression fits the observed data very well.  

First I imported linregress from scipy.stats.  Then I created a function to perform the linear regression on the weather data and plot a regression line and equation with the data.  Then I created the data set.  For the example above, I created a DataFrame for weather data for cities in the Northern Hemisphere using `.loc` to locate only cities with latitudes >= 0.  Then 

    city_data_df = pd.read_csv("weather_data/cities.csv")

    import matplotlib.pyplot as plt
    from scipy.stats import linregress

    def plot_linear_regression(x_values, y_values, title, y_label, text_coordinates):

        # Run regression on hemisphere weather data.
        (slope, intercept, r_value, p_value, std_err) = linregress(x_values, y_values)

        # Calculate the regression line "y values" from the slope and intercept.
        regress_values = x_values * slope + intercept
    
        # Get the equation of the line.
        line_eq = "y = " + str(round(slope,2)) + "x + " + str(round(intercept,2))
    
        # Create a scatter plot and plot the regression line.
        plt.scatter(x_values,y_values)
        plt.plot(x_values,regress_values,"r")
    
        # Annotate the text for the line equation.
        plt.annotate(line_eq, text_coordinates, fontsize=15, color="red")
        plt.xlabel('Latitude')
        plt.ylabel(y_label)
        plt.title(title)
        plt.show()         
        
    # Create DataFrame to hold weather data for cities in Northern Hemisphere
    northern_hemi_df = city_data_df.loc[(city_data_df["Lat"] >= 0)]
        
    # Linear regression on the Northern Hemisphere Max Temps
    x_values = northern_hemi_df["Lat"]
    y_values = northern_hemi_df["Max Temp"]
    
    # Call the function.
    plot_linear_regression(x_values, y_values,  
        "Linear Regression on the Northern Hemisphere \nfor Maximum Temperature", 'Max Temp',(10,40))
        
    # Show results
    linregress(x_values,y_values)
        

[Back to the Table of Contents](https://github.com/rkaysen63/World_Weather_Analysis/blob/master/README.md#table-of-contents)
