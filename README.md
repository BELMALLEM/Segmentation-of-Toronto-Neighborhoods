
# <br>                             Capstone Project: Toronto Neighborhood 

## <br>Introduction
In this project, we'll be required to **explore**, **segment**, and **cluster** the neighborhoods in the city of Toronto. However, the neighborhood data is not readily available on the Internet. What is interesting about the field of data science is that each project can be challenging in its unique way.
For the Toronto neighborhood data, a Wikipedia page exists that has all the information we need to explore and cluster the neighborhoods in Toronto. We'll be required to scrape that page and wrangle the data, clean it, and then read it into a pandas DataFrame so that it is in a structured format like the New York dataset.
Once the data is in a structured format, we can analyze our data to explore and cluster the neighborhoods in the city of Toronto.

<br>Before we get the data and start exploring it, let's download all the dependencies that we will need.


```python
import pandas as pd

import numpy as np
from bs4 import BeautifulSoup
import urllib.request
import requests

import json # library to handle JSON files
from geopy.geocoders import Nominatim # convert an address into latitude and longitude values
from pandas.io.json import json_normalize # tranform JSON file into a pandas dataframe

# Matplotlib and associated plotting modules
import matplotlib.cm as cm
import matplotlib.colors as colors

# import k-means from clustering stage
from sklearn.cluster import KMeans

#!conda install -c conda-forge folium=0.5.0 --yes # uncomment this line if you haven't completed the Foursquare API lab
import folium # map rendering library

print('Libraries imported.')
```

## <br>1. Scraping the Web to get the data

We'll scrape a wikipedia page where all needed informations about the neighborhoods in the city Toronto with their Postal Codes exist. https://en.wikipedia.org/wiki/List_of_postal_codes_of_Canada:_M
#### Load and explore the data

We define the targeted page to scrape and read it


```python
url = "https://en.wikipedia.org/wiki/List_of_postal_codes_of_Canada:_M"
pageContent = urllib.request.urlopen(url).read()
```

<br>Using the BeauifulSoup library we read the page as a manipulated object, then we extract the table where all the informations are


```python
soup = BeautifulSoup(pageContent)
TorontoTable = soup.table
```

<br>Now our job is to transform the gotten table to an array so we can manipulate it easily later.


```python
TorontoTableRows = TorontoTable.find_all('tr')

allRows = [trow.find_all('td') for trow in TorontoTableRows[1:]]
allRows = [x for x in allRows if x[1].get_text() != 'Not assigned\n']

i = -1                           # iterator on allRows
for row in allRows:
    i = i + 1
    if row:                      # if the row is not empty
        allRows[i] = [x.get_text()[:-1] for x in allRows[i]]  # we remplace the html tags with their string contents
        
allRows[:5]
```

<br>We'll split the principle list to three list one for Postal Codes, the second for Boroughs and the last for Negihborhoods


```python
postalCodes = [x[0] if x else '__' for x in allRows]
postalCodes[:10]
```


```python
Borough = [x[1] if x else '__' for x in allRows]
Borough[:10]
```


```python
Neighborhood = [x[2].replace(' /', ',') if x else '__' for x in allRows]
Neighborhood[:10]
```

<br>Let's visualize our three lists


```python
print("The three lists after cleaning them: ")
print("List of codes:      ", postalCodes[:10])
print("List of boroughs:   ", Borough[:5])
print("List of neighbors:  ", Neighborhood[:5])
```

-.- It's time to turn the list to a dataframe, so we'll analyze our data

#### Tranform the data into a *pandas* dataframe

The next task is essentially transforming this data of nested Python lists into a *pandas* dataframe. So let's start by creating an empty dataframe. Then fill the dataframe, and in the last time we sort the values to match with our Geospatial Coordinates.


```python
# dictionary of lists  
df = pd.DataFrame()
df['PostalCode'] = postalCodes
df['Borough'] = Borough
df['Neighborhood'] = Neighborhood
df = df.sort_values(by='PostalCode')
df = df.reset_index(drop=True)
df.head()
```

<br>Enable the cell below to get the needed GeoSpatial data to work with if you don't have them.

!wget https://cocl.us/Geospatial_data

<br>let's load our location data


```python
geoCord = pd.read_csv('Geospatial_Coordinates.csv')
geoCord.sort_values(by='Postal Code')
geoCord[:5]
```

<br>We need to add those data to our main dataframe.


```python
df['Latitude'] = geoCord['Latitude']
df['Longitude'] = geoCord['Longitude']
df[:5]
```

<br>The cell below is used to get the location of the city Toronto to be used later for showing the map, but as we know its location we disabled it and we let it to show how can we get the location of a certain place.


```python
# The latitude and longitude of the city Toronto
latitude = 43.653963
longitude = -79.387207

# Using geocode we can get the Location of the city Toronto
#address = 'Toronto, Ontario'

#geolocator = Nominatim(user_agent="ny_explorer")
#location = geolocator.geocode(address)
#latitude = location.latitude
#longitude = location.longitude
#print('The geograpical coordinate of Manhattan are {}, {}.'.format(latitude, longitude))
```

#### <br>Create a map of New York with neighborhoods superimposed on top.


```python
# create map of New York using latitude and longitude values
map_toronto = folium.Map(location=[43.653963, -79.387207], zoom_start=10)

# add markers to map
for lat, lng, borough, neighborhood in zip(df['Latitude'], df['Longitude'], df['Borough'], df['Neighborhood']):
    label = '{}, {}'.format(neighborhood, borough)
    label = folium.Popup(label, parse_html=True)
    folium.CircleMarker(
        [lat, lng],
        radius=5,
        popup=label,
        color='blue',
        fill=True,
        fill_color='#3186cc',
        fill_opacity=0.7,
        parse_html=False).add_to(map_toronto)  
    
map_toronto
```

**Folium** is a great visualization library. Feel free to zoom into the above map, and click on each circle mark to reveal the name of the neighborhood and its respective borough.

<h1>2. Analyze Our Data</h1>

We got the data we need, now it's time to analyze them so after we can cluster the neighborhoods.


```python
print('This is the different Boroughs:')
for element in df['Borough'].unique():
    print('       + ', element)
```

<br>Next, we are going to start utilizing the Foursquare API to explore the neighborhoods and segment them.

#### Define Foursquare Credentials and Version

We define our Foursquare ID and Secret code that we get after registering in their site


```python
CLIENT_ID = 'YWHDVDQAXIHWG4NAP2O5VY1E52ICIENPII2GZNL00SOMAJIV' # your Foursquare ID
CLIENT_SECRET = 'ZTDT0SR3EA1WFL2Y3FUIFV31WEEGATRFACQYPWYVNSL42MVI' # your Foursquare Secret
VERSION = '20180605' # Foursquare API version
radius = 500
LIMIT = 100

print('Your credentails:')
print('CLIENT_ID: ' + CLIENT_ID)
print('CLIENT_SECRET:' + CLIENT_SECRET)
```

#### <br>Now, let's get the top 100 venues that are in Marble Hill within a radius of 500 meters.

We define a function that will get for each neighborhood its venues, using the Foursquare API, the name of the neighborhood, and its location.


```python
def getNearbyVenues(names, latitudes, longitudes, radius=500):
    
    venues_list=[]
    for name, lat, lng in zip(names, latitudes, longitudes):
        print(name)
            
        # create the API request URL
        url = 'https://api.foursquare.com/v2/venues/explore?&client_id={}&client_secret={}&v={}&ll={},{}&radius={}&limit={}'.format(
            CLIENT_ID, 
            CLIENT_SECRET, 
            VERSION, 
            lat, 
            lng, 
            radius, 
            LIMIT)
            
        # make the GET request
        results = requests.get(url).json()["response"]['groups'][0]['items']
        
        # return only relevant information for each nearby venue
        venues_list.append([(
            name, 
            lat, 
            lng, 
            v['venue']['name'], 
            v['venue']['location']['lat'], 
            v['venue']['location']['lng'],  
            v['venue']['categories'][0]['name']) for v in results])

    nearby_venues = pd.DataFrame([item for venue_list in venues_list for item in venue_list])
    nearby_venues.columns = ['Neighborhood', 
                  'Neighborhood Latitude', 
                  'Neighborhood Longitude', 
                  'Venue', 
                  'Venue Latitude', 
                  'Venue Longitude', 
                  'Venue Category']
    
    return(nearby_venues)
```

#### <br>Now write the code to run the above function on each neighborhood and create a new dataframe called *Toronto_venues*.


```python
Toronto_venues = getNearbyVenues(names=df['Neighborhood'],
                                   latitudes=df['Latitude'],
                                   longitudes=df['Longitude']
                                  )
```

#### <br>Let's check the size of the resulting dataframe

Let's check how many venues were returned for each neighborhood


```python
print(Toronto_venues.shape)
Toronto_venues.head()
```

#### <br>Let's find out how many unique categories can be curated from all the returned venues


```python
print('There are {} uniques categories.'.format(len(Toronto_venues['Venue Category'].unique())))
```

## <br>3. Analyze Each Neighborhood

As we see we have edited our dataframe as we added extra columns gotten with *Foursquare* like Venue, its Location and Category


```python
# one hot encoding
Toronto_onehot = pd.get_dummies(Toronto_venues[['Venue Category']], prefix="", prefix_sep="")

# add neighborhood column back to dataframe
Toronto_onehot['Neighborhood'] = Toronto_venues['Neighborhood'] 

# move neighborhood column to the first column
fixed_columns = [Toronto_onehot.columns[-1]] + list(Toronto_onehot.columns[:-1])
Toronto_onehot = Toronto_onehot[fixed_columns]

Toronto_onehot
```

#### <br>Next, let's group rows by neighborhood and by taking the mean of the frequency of occurrence of each category


```python
Toronto_grouped = Toronto_onehot.groupby('Neighborhood').mean().reset_index()
Toronto_grouped
```

### <br>Let's confirm the new size


```python
Toronto_grouped.shape
```

## <br>Most common venues

We want to choose the most 5 common venues for each neighborhood.<br>
First, let's write a function to sort the venues in descending order.


```python
def return_most_common_venues(row, num_top_venues):
    row_categories = row.iloc[1:]
    row_categories_sorted = row_categories.sort_values(ascending=False)
    
    return row_categories_sorted.index.values[0:num_top_venues]
```

<br>Now let's create the new dataframe and display the top 10 venues for each neighborhood.


```python
num_top_venues = 10

indicators = ['st', 'nd', 'rd']

# create columns according to number of top venues
columns = ['Neighborhood']
for ind in np.arange(num_top_venues):
    try:
        columns.append('{}{} Most Common Venue'.format(ind+1, indicators[ind]))
    except:
        columns.append('{}th Most Common Venue'.format(ind+1))

# create a new dataframe
neighborhoods_venues_sorted = pd.DataFrame(columns=columns)
neighborhoods_venues_sorted['Neighborhood'] = Toronto_grouped['Neighborhood']

for ind in np.arange(Toronto_grouped.shape[0]):
    neighborhoods_venues_sorted.iloc[ind, 1:] = return_most_common_venues(Toronto_grouped.iloc[ind, :], num_top_venues)

neighborhoods_venues_sorted
```

## <br>4. Cluster Neighborhoods

Run *k*-means to cluster the neighborhood into 5 clusters.


```python
# set number of clusters
kclusters = 5

Toronto_grouped_clustering = Toronto_grouped.drop('Neighborhood', 1)

# run k-means clustering
kmeans = KMeans(n_clusters=kclusters, random_state=0).fit(Toronto_grouped_clustering)

# check cluster labels generated for each row in the dataframe
kmeans.labels_.shape
```

<br>Let's create a new dataframe that includes the cluster as well as the top 10 venues for each neighborhood.


```python
# add clustering labels
neighborhoods_venues_sorted.insert(0, 'Cluster Labels', np.int_( kmeans.labels_))

Toronto_merged = df

# merge toronto_grouped with toronto_data to add latitude/longitude for each neighborhood
Toronto_merged = Toronto_merged.join(neighborhoods_venues_sorted.set_index('Neighborhood'), on='Neighborhood')

Toronto_merged
```

<br> We got some rows with null values, and we need to delete them before continuing.


```python
Toronto_merged.dropna(axis='rows', how='any', inplace=True)
Toronto_merged.head()
```

<br>Finally, let's visualize the resulting clusters


```python
# create map
map_clusters = folium.Map(location=[latitude, longitude], zoom_start=11)

# set color scheme for the clusters
x = np.arange(kclusters)
ys = [i + x + (i*x)**2 for i in range(kclusters)]
colors_array = cm.rainbow(np.linspace(0, 1, len(ys)))
rainbow = [colors.rgb2hex(i) for i in colors_array]

# add markers to the map
markers_colors = []
for lat, lon, poi, cluster in zip(Toronto_merged['Latitude'], Toronto_merged['Longitude'], Toronto_merged['Neighborhood'], Toronto_merged['Cluster Labels'].astype(int)):
    label = folium.Popup(str(poi) + ' Cluster ' + str(cluster), parse_html=True)
    folium.CircleMarker(
        [lat, lon],
        radius=5,
        popup='Laurelhurst Park',
        line_color='#3186cc',
        fill=True,
        fill_color=rainbow[cluster-1],
        fill_opacity=0.7).add_to(map_clusters)
       
map_clusters
```

<a id='item1'></a>

### <br>Thank you for going to the end!

This notebook was created by [Marouane Belmallem](https://www.linkedin.com/in/marouane-belmallem/).!

<hr>
Feel free to use this notebook!
