---
date:
  created: 2023-08-26
---

# WhereToLive.LA
I made a website to map new rental listings in Los Angeles County based on spreadsheet data.
<!-- more -->
---

I've been looking to move into a new place for a while now and have been amassing resources and websites that I can peruse through to find the right place.
One of the moderators on [the /r/LArentals subreddit](https://www.reddit.com/r/LARentals/) posts [a spreadsheet](https://docs.google.com/spreadsheets/d/1gBLt73zziGg41IUS3FdqU4ddULWV5sFaSRawLyO5YyY/edit#gid=80625330) of new rental properties every week.

As you can see, [it's quite detailed and organized.](https://user-images.githubusercontent.com/28774550/188333676-11846918-2c1d-4ebd-aa92-897fc2f18dfa.png)

You can filter by any column to narrow down the results list. I absolutely loved that; Zillow, RedFin, etc. don't have as quite a granular filter as this spreadsheet does.

But I was a little miffed that I kept having to open new tabs and paste the street address into Google to find out where in LA the property resided. At 350+ rows, that's a LOT of browser tabs and LA County is such a massive sprawl that I simply can't visualize where every property is.

## The Idea
So I thought, "why not just visually map every street address on Google Maps?" And then that idea became "the map should be filterable in all the same ways as the spreadsheet."
So I wanted a filterable map. And that's how I fell down the rabbit hole. Using a combination of:

* BeautifulSoup 4
* Dash Leaflet
* Dash Bootstrap Components
* GeoPy
* ImageKit
* Pandas

I made an interactive map that displays and filters all the rental properties listed. Eventually, [I expanded that to for-sale listings under $1,000,000](https://wheretolive.la/buy) (those still exist in LA somehow) since that same person also posts a spreadsheet with that information too.
You can view [my source code](https://github.com/perfectly-preserved-pie/larentals) on GitHub. The website is live at [https://wheretolive.la](https://wheretolive.la).

### Data Handling
Because the spreadsheet was already in CSV form, Pandas was an obvious choice here. I could simply just read it into a dataframe and add columns and manipulate the data in whatever way I needed to.

### Geocoding
I first needed some kind of API that I could feed street addresses from the spreadsheet and it would spit out the assoicated coordinates. I intially spun up an instance of Nominatim because it was free and easy and tried that. Unfortunately, quite a few addresses just simply wouldn't resolve in Nominatim but resolved just fine with the Google Maps API. So I switched over to Google Maps, which provides [a generous free tier](https://mapsplatform.google.com/pricing/) ($200/month).
I haven't had any issues since then; it handled every address I threw at it and returned me accurate coordinates. 

I created a quick function to return coordinates based on a provided street address:

```python
# Create a function to get coordinates from the full street address
def return_coordinates(address, row_index):
    try:
        geocode_info = g.geocode(address, components={'administrative_area': 'CA', 'country': 'US'})
        lat = float(geocode_info.latitude)
        lon = float(geocode_info.longitude)
    except Exception as e:
        lat = NaN
        lon = NaN
        logger.warning(f"Couldn't fetch geocode information for {address} (row {row.Index} of {len(df)}) because of {e}.")
    logger.success(f"Fetched coordinates {lat}, {lon} for {address} (row {row.Index} of {len(df)}).")
    return lat, lon
```
You'll see that I had to add `CA` and `US` as conditions within `components`; this ensures that all results are restricted to California, USA. I was getting some coordinates that were all the way in China, some in Africa, and some in the middle of the Atlantic Ocean.
I'm not entirely sure why, but part of the reason for that is that there are cities with the same names in other states/countries. For example: Lancaster, MA.

Putting these constraints in ensures that I get the right coordinates for the California city.

Then I iterated over every row with that function:

```python
# Iterate through the dataframe and fetch coordinates for rows that don't have them
# If the Coordinates column is already present, iterate through the null cells
# Similiar to above, we can use the presence of the Coordinates column as a proxy for Longitude and Latitude; all 3 should exist together or none at all
# This assumption will reduce the number of API calls to Google Maps
if 'Coordinates' in df.columns:
    for row in df['Coordinates'].isnull().itertuples():
        print(f"Grabbing coordinates for row #{row.Index}...")
        coordinates = return_coordinates(df.at[row.Index, 'Full Street Address'])
        df.at[row.Index, 'Latitude'] = coordinates[0]
        df.at[row.Index, 'Longitude'] = coordinates[1]
        df.at[row.Index, 'Coordinates'] = coordinates[2]
# If the Coordinates column doesn't exist (i.e this is a first run), create it using df.at
elif 'Coordinates' not in df.columns:
    for row in df.itertuples():
        print(f"Grabbing coordinates for row #{row.Index}...")
        coordinates = return_coordinates(df.at[row.Index, 'Full Street Address'])
        df.at[row.Index, 'Latitude'] = coordinates[0]
        df.at[row.Index, 'Longitude'] = coordinates[1]
        df.at[row.Index, 'Coordinates'] = coordinates[2]
```
### Filters
So I had the map and markers ready, but how could I filter the markers depending on different variables? Luckily, that's what Dash does via "[callbacks](https://dash.plotly.com/basic-callbacks)". A box could be checked or a slider could be dragged, and that would cause an action on the backend to fire off. In my case, a checked box would need to change the dataframe; the dataframe is how the markers on the map are being populated. So if I wanted to see ONLY condos, I would need to query the dataframe for ONLY condos.
```python
@callback(
  Output(component_id='lease_geojson', component_property='children'),
  [
    Input(component_id='subtype_checklist', component_property='value'),
    Input(component_id='pets_radio', component_property='value'),
    Input(component_id='terms_checklist', component_property='value'),
    Input(component_id='garage_spaces_slider', component_property='value'),
    Input(component_id='rental_price_slider', component_property='value'),
    Input(component_id='bedrooms_slider', component_property='value'),
    Input(component_id='bathrooms_slider', component_property='value'),
    Input(component_id='sqft_slider', component_property='value'),
    Input(component_id='yrbuilt_slider', component_property='value'),
    Input(component_id='sqft_missing_radio', component_property='value'),
    Input(component_id='yrbuilt_missing_radio', component_property='value'),
    Input(component_id='garage_missing_radio', component_property='value'),
    Input(component_id='ppsqft_slider', component_property='value'),
    Input(component_id='ppsqft_missing_radio', component_property='value'),
    Input(component_id='furnished_checklist', component_property='value'),
    Input(component_id='security_deposit_slider', component_property='value'),
    Input(component_id='security_deposit_missing_radio', component_property='value'),
    Input(component_id='pet_deposit_slider', component_property='value'),
    Input(component_id='pet_deposit_missing_radio', component_property='value'),
    Input(component_id='key_deposit_slider', component_property='value'),
    Input(component_id='key_deposit_missing_radio', component_property='value'),
    Input(component_id='other_deposit_slider', component_property='value'),
    Input(component_id='other_deposit_missing_radio', component_property='value'),
    Input(component_id='listed_date_datepicker', component_property='start_date'),
    Input(component_id='listed_date_datepicker', component_property='end_date'),
    Input(component_id='listed_date_radio', component_property='value'),
    Input(component_id='laundry_checklist', component_property='value'),
  ]
)
# The following function arguments are positional related to the Inputs in the callback above
# Their order must match
def update_map(subtypes_chosen, pets_chosen, terms_chosen, garage_spaces, rental_price, bedrooms_chosen, bathrooms_chosen, sqft_chosen, years_chosen, sqft_missing_radio_choice, yrbuilt_missing_radio_choice, garage_missing_radio_choice, ppsqft_chosen, ppsqft_missing_radio_choice, furnished_choice, security_deposit_chosen, security_deposit_radio_choice, pet_deposit_chosen, pet_deposit_radio_choice, key_deposit_chosen, key_deposit_radio_choice, other_deposit_chosen, other_deposit_radio_choice, listed_date_datepicker_start, listed_date_datepicker_end, listed_date_radio, laundry_chosen):
  # Pre-sort our various lists of strings for faster performance
  subtypes_chosen.sort()
  df_filtered = df[
    subtype_function(subtypes_chosen) &
    pets_radio_button(pets_chosen) &
    terms_function(terms_chosen) &
    # For the slider, we need to filter the dataframe by an integer range this time and not a string like the ones aboves
    # To do this, we can use the Pandas .between function
    # See https://stackoverflow.com/a/40442778
    ((df.sort_values(by='garage_spaces')['garage_spaces'].between(garage_spaces[0], garage_spaces[1])) | garage_radio_button(garage_missing_radio_choice, garage_spaces[0], garage_spaces[1])) &
    # Repeat but for rental price
    # Also pre-sort our lists of values to improve the performance of .between()
    (df.sort_values(by='list_price')['list_price'].between(rental_price[0], rental_price[1])) &
    (df.sort_values(by='Bedrooms')['Bedrooms'].between(bedrooms_chosen[0], bedrooms_chosen[1])) &
    (df.sort_values(by='Total Bathrooms')['Total Bathrooms'].between(bathrooms_chosen[0], bathrooms_chosen[1])) &
    ((df.sort_values(by='Sqft')['Sqft'].between(sqft_chosen[0], sqft_chosen[1])) | sqft_radio_button(sqft_missing_radio_choice, sqft_chosen[0], sqft_chosen[1])) &
    ((df.sort_values(by='YrBuilt')['YrBuilt'].between(years_chosen[0], years_chosen[1])) | yrbuilt_radio_button(yrbuilt_missing_radio_choice, years_chosen[0], years_chosen[1])) &
    ((df.sort_values(by='ppsqft')['ppsqft'].between(ppsqft_chosen[0], ppsqft_chosen[1])) | ppsqft_radio_button(ppsqft_missing_radio_choice, ppsqft_chosen[0], ppsqft_chosen[1])) &
    furnished_checklist_function(furnished_choice) &
    security_deposit_function(security_deposit_radio_choice, security_deposit_chosen[0], security_deposit_chosen[1]) &
    pet_deposit_function(pet_deposit_radio_choice, pet_deposit_chosen[0], pet_deposit_chosen[1]) &
    key_deposit_function(key_deposit_radio_choice, key_deposit_chosen[0], key_deposit_chosen[1]) &
    other_deposit_function(other_deposit_radio_choice, other_deposit_chosen[0], other_deposit_chosen[1]) &
    listed_date_function(listed_date_radio, listed_date_datepicker_start, listed_date_datepicker_end) &
    laundry_checklist_function(laundry_chosen)
  ]
  ```
  
You can see here that the callback performs [various dataframe options (filtering, comparing strings, etc.)](https://github.com/perfectly-preserved-pie/larentals/blob/master/pages/lease_page.py#L40-L198) and puts the results in a `df_filtered` variable. I then iterate through `df_filtered` to generate the markers & their associated popups, as you'll see below.

### Mapping
Now that I had a list of coordinates, I needed a way to actually _display_ the points on the map. This led me to [Folium](http://python-visualization.github.io/folium/), however I wasn't too happy with the look and soon moved on to [Dash by Plotly](https://github.com/plotly/dash). Even then I still wasn't satisified with any of the map types. Heatmaps, chloropeths, etc. all were too complex for what I wanted: a simple marker with a table of the property's characteristics (rent price, garage spaces, address, etc.). My search led me to [Dash-Leaflet](https://dash-leaflet.herokuapp.com/) which was perfect. Not only did it look good but nearby points could all be part of a cluster group that would expand and shrink as the user zoomed the map in or out:

[Example](https://i.imgur.com/czbdxpQ.mp4)

I wanted each marker to show the property details, so I created a function to return HTML code for the marker's popup:
```python
# Define HTML code for the popup so it looks pretty and nice
def popup_html(row):
    i = row.Index
    street_address=df['Full Street Address'].at[i] 
    mls_number=df['Listing ID'].at[i]
    mls_number_hyperlink=df['bhhs_url'].at[i]
    mls_photo = df['MLS Photo'].at[i]
    lc_price = df['List Price'].at[i] 
    price_per_sqft=df['Price Per Square Foot'].at[i]                  
    brba = df['Br/Ba'].at[i]
    square_ft = df['Sqft'].at[i]
    year = df['YrBuilt'].at[i]
    garage = df['Garage Spaces'].at[i]
    pets = df['PetsAllowed'].at[i]
    phone = df['List Office Phone'].at[i]
    terms = df['Terms'].at[i]
    sub_type = df['Sub Type'].at[i]
    listed_date = pd.to_datetime(df['Listed Date'].at[i]).date() # Convert the full datetime into date only. See https://stackoverflow.com/a/47388569
    furnished = df['Furnished'].at[i]
    key_deposit = df['DepositKey'].at[i]
    other_deposit = df['DepositOther'].at[i]
    pet_deposit = df['DepositPets'].at[i]
    security_deposit = df['DepositSecurity'].at[i]
```

That function uses the Pandas dataframe row to populate different fields like rental price, garage spaces, terms, etc. and formats them into an HTML table for that specific marker:
![image](https://user-images.githubusercontent.com/28774550/188334715-20842be0-b171-4631-8c1b-cf908ee1715a.png)

Then, in my Dash callback, I iterate through the filtered dataframe `df_filtered` and add each row's coordinates & popup HTML to a list. Then I use Dash Leaflet's `dicts_to_geojson` function to convert each object in the list a GeoJSON object that can be displayed on the map:
```python
# Create an empty list for the markers
  markers = []
  # Iterate through the dataframe, create a marker for each row, and append it to the list
  for row in df_filtered.itertuples():
    markers.append(
      dict(
        lat=row.Latitude,
        lon=row.Longitude,
        popup=row.popup_html
        )
    )
  # Generate geojson with a marker for each listing
  geojson = dlx.dicts_to_geojson([{**m} for m in markers])
  ...
  # Generate the map
  return dl.GeoJSON(
    id=str(uuid.uuid4()),
    data=geojson,
    cluster=True,
    zoomToBoundsOnClick=True,
    superClusterOptions={ # https://github.com/mapbox/supercluster#options
      'radius': 160,
      'minZoom': 3,
    }
```
  
So now, whenever a user clicks on a marker, a popup appears with that property's details. Very handy to see what it's like at a glance.


### Web Scraping
  Because knowing _when_ a listing was posted is important (a listing from 9 months ago probably isn't going to be available) I wanted to get the "listed" date. That also led me to finding an MLS photo associcated with the property, so I figured I'd scrape that too and insert that photo into the HTML popup for the property.

```python
## Webscraping Time
# Create a function to scrape the listing's Berkshire Hathaway Home Services (BHHS) page using BeautifulSoup 4 and extract some info
def webscrape_bhhs(url, row_index):
    try:
        response = requests.get(url)
        soup = bs4(response.text, 'html.parser')
        # First find the URL to the actual listing instead of just the search result page
        try:
          link = 'https://www.bhhscalifornia.com' + soup.find('a', attrs={'class' : 'btn cab waves-effect waves-light btn-details show-listing-details'})['href']
          logging.info(f"Successfully fetched listing URL for {row_index}.")
        except AttributeError as e:
          link = None
          logging.warning(f"Couldn't fetch listing URL for {row_index}. Passing on...")
          pass
        # If the URL is available, fetch the MLS photo and listed date
        if link is not None:
          # Now find the MLS photo URL
          # https://stackoverflow.com/a/44293555
          try:
            photo = soup.find('a', attrs={'class' : 'show-listing-details'}).contents[1]['src']
            logging.info(f"Successfully fetched MLS photo for {row_index}.")
          except AttributeError as e:
            photo = None
            logging.warning(f"Couldn't fetch MLS photo for {row_index}. Passing on...")
            pass
          # For the list date, split the p class into strings and get the last element in the list
          # https://stackoverflow.com/a/64976919
          try:
            listed_date = soup.find('p', attrs={'class' : 'summary-mlsnumber'}).text.split()[-1]
            logging.info(f"Successfully fetched listed date for {row_index}.")
          except AttributeError as e:
            listed_date = pd.NaT
            logging.warning(f"Couldn't fetch listed date for {row_index}. Passing on...")
            pass
        elif link is None:
          pass
    except Exception as e:
      listed_date = pd.NaT
      photo = NaN
      link = NaN
      logging.warning(f"Couldn't scrape BHHS page for {row_index} because of {e}. Passing on...")
      pass
    return listed_date, photo, link 
```

Additionally, I also created a function to scrape the BHHS listing to check if the listing was still active. This is not a 100% accurate proxy, since realtors aren't required to post a listing on an MLS or at all, but it's good enough:
```python
# Create a function to check for expired listings based on the presence of a string
def check_expired_listing(url, mls_number):
  try:
    response = requests.get(url, timeout=5)
    soup = bs4(response.text, 'html.parser')
    # Detect if the listing has expired. Remove \t, \n, etc. and strip whitespaces
    try:
      soup.find('div', class_='page-description').text.replace("\r", "").replace("\n", "").replace("\t", "").strip()
      return True
    except AttributeError:
      return False
  except Exception as e:
    logger.warning(f"Couldn't detect if the listing for {mls_number} has expired because {e}.")
    return False
```   


## Challenges
At some point due to [performance issues](https://github.com/thedirtyfew/dash-leaflet/issues/168#issuecomment-1374404327) with `dl.MarkerClusterGroup` I switched to using `dl.GeoJSON` to generate the markers. Unforunately, this change meant that whenever the map is panned or zoomed, even just a tiny bit, any open popup immediately closes. 
To make things worse, clicking on a marker automatically pans the map to fit the resulting popup, but the MLS photo loads _after_ the popup and therefore the map pans _a second time_, closing the popup. You'd have to click a marker twice, sometimes even 3 or 4 times, to get the popup to stay. That leads to a frustrating UX (at least for me), because panning the map shouldn't close the popup. It's even worse on a mobile device (or any device with a small screen) because there's so little screen real estate to begin with.

Basically, the problem is that with `cluster=True`, the clusters (and markers) are redrawn whenever the viewport changes (by panning or zooming). That's what makes the popups close.

I brought up this issue on the official GitHub repo and happily, [Emil has solved this with a new version of Dash Leaflet](https://github.com/thedirtyfew/dash-leaflet/issues/180#issuecomment-1694490970). The popups stay open even when you pan or zoom the map, greatly reducing the UX friction. Thank you Emil!!

This was a long, long running bug in the project ([7 months!](https://github.com/perfectly-preserved-pie/larentals/pull/16)). It's finally been squashed and I couldn't be more relieved because it has bugged me the entire damn time. 

Anyways, I hope you get some use out of this website; it was a labor of love.
