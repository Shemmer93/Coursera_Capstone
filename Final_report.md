# Mystery City Recommender

*Author: Sebastiaan Hemmer*

*October 12, 2019*

## 1. Introduction
My imaginary traveling agency ‘*Sacramentum*’ has a unique value proposition; we offer mystery trips to different US cities without our clients knowing their destination in advance.

Our clients purchase tickets and are only provided information regarding the climate of a destination so they know what clothes they should pack. On the airport, our clients are given an envelope which contains the tickets to their destination. Their destination is drawn completely random at this moment.

### 1.1 Problem definition
Recently, we have been dealing with many customers that were unsatisfied with the mystery location they have traveled to. Because this is impacting our business, we are looking for solutions to improve our customer satisfaction. Therefore, we have decided to design a content-based recommender system that takes user input of 3-5 cities that they rate positively and then recommends a mystery location that will most likely be rated positively by our client.

In order to determine a profile of our clients, we will leverage the Foursquare API to cluster different US cities based on certain features. These features shall mainly consist of the amount of different types of venues and attractions in a city, as well as geographical and demographic data.

Because we will be using unlabelled data and do not have access to a training/testing dataset, we are limited to using unsupervised learning techniques.

## 2. Data sources and cleaning
As mentioned in the previous section, we will be leveraging the Foursquare API and combining it with both geographical and demographic data that are publicly available. In order to determine an accurate client profile, we will need enough features to make distinctions between different cities but not too many as that will overfit and overcomplicate our model. Therefore, we have selected the following data from different sources.

### 2.1 Data sources
*Demographical data*

This will be our starting dataset, we will select the top 100 largest US cities by population. We will use the available dataset on ODS ([source](https://public.opendatasoft.com/explore/embed/dataset/1000-largest-us-cities-by-population-with-geographic-coordinates/table/?sort=-rank)), which will provide us information about the:
-	Name of city
-	Population
-	Coordinates
- Growth of population

*Geographical data*

We will then combine our city demographical data with geographical data from open NOAA datasets, such as:
-	Elevation
-	Average annual temperature
-	Average annual precipitation

*Foursquare API*

Lastly, we will include the Foursquare API to include venue data. We have selected the following features of importance:
-	Top 3 most popular venue categories (in 5km radius from center)
-	Amount of venues in a 5km radius from the center

### 2.2 Data collection
First of all the ODS dataset was downloaded and converted into a dataframe with 1000 rows and 4 features. In order to speed up further calculation, only the top 100 cities based on population were filtered and the lower 900 were dropped. Then, the coordinates column was split into seperate columns for the Latitude and Longitude values. These values were used in a function that would call the Google Maps API and retrieve the elevation in meters at that particular coordinate. The result of this function was applied to the original dataframe. Next, NOAA's dataset of US 1981-2010 Annual Climate Normals was downloaded. For every weather station in the dataset, the annual average temperature in Celcius and percipitation in Millimeters were found. 

To merge the climate data with the existing dataset, a function was written that would take the Latitude and Longitude values of each city and then find the distance to every weather station using the Haversine formula. All stations outside a 20 km radius of the city centers were dropped and the average temperature and percipitation were taken for all remaining weather stations. The resulting vector was then applied to the existing dataframe.

Another function was written that would connect with the Foursquare API and fetch venue details within a 5km radius around the city centers. Both the count of venues as well as the three most occuring venue categories were extracted and added to the original dataframe. The end result was a nxm dataframe with n=100 cities and m=10 feeatures.

### 2.3 Explanatory data analysis
Now that the full dataset is collected and joined into a single dataframe, some explanatory data analysis was performed to understand possible correlations between different features. It was found that there were some obvious relationships between different variables. There was an apparent strong negative correlation between the Latitude and average temperature features and a strong positive correlation between the Longitude and average percipitation. The first relationship is very obvious as the temperature naturally increases when moving further north (and thus obtaining a larger latitude value) due to the tilt of the earth. However, the second relationship was less expected and meant that the average percipitation increased when the longitude value decreased, meaning a move towards the east. It was concluded that western US cities are, on average, much drier than their eastern counterparts.

Furthermore, there appears to be a moderate negative relationship between elevation and average percipitation. It was concluded that higher cities could be dryer on average, compared to cities located near sea level. Finally, a slight correlation was found between average percipitation and annual population growth. This shows that wetter cities could be less favourable to move to compared to dryer cities. However, other factors that are not accounted for might be responsible so it is too early to draw conclusions.

### 2.4 Data cleaning
Most data that was collected during the previous steps had been properly extracted without the need for any data cleaning. Out of 10 available features, 8 were already numerical columns without any missing values. Of the remaining two columns, the 'State' column was not particularely interesting for our model so only the top 3 venue categories remained. In order to be able to process this column, the Natural Language Toolkit (nltk) was used to create bags of words for each category. These bags were then ready for usage.

### 2.5 Feature selection
Of the remaining 9 features, 7 were deemed appropiate for comparison between samples. Because the latitude and longitude values should not influence the model, these were omitted from the feature set. To be able to use the bags of words that were created in the previous step, a count vectorizer was used that split the bags into seperate columns and counted the occurence for each sample. The resulting matrix was combined with the selected featureset and normalized using a standard scaler. Together, they form a normalized array of the featureset that was ready for ingestion into our recommender system.

## 3. Methodology
Finally, it's time to use all of this neat data to make some recommendations. Because the dataset consists of unlabeled data, it is not possible to train the model using a subset of the data as the final results are unknown. Consequently, unsupervised learning techniques such as clustering must be resorted to. Because of the nature of recommender systems, it is not approtiate to cluster similar samples together and asign a label to them. Instead, a similarity value must be acquired between the input and all other samples. Therefore, distance metrics such as the Euclidian distance or Cosine similarity are suitable for usage in a recommender system. The advantage of Euclidian distance is that it is possible to determine the absolute nearest neighbour of a sample. However, it is unable to take into account the direction of the two vectors and cannot distinquish between negative values. Furthermore, Cosine Similarity is very popular and widely adopted for usage in conjunction with textual data. Therefore, Cosine Similarity is selected as primary method for our recommender engine.

## 4. Results

## 5. Discussion

## 6. Conclusion

