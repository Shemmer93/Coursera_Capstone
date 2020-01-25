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
First of all the ODS dataset was downloaded and converted into a dataframe with 1000 rows and 4 features. In order to speed up further calculation, only the top 100 cities based on population were selected and the lower 900 were dropped. Then, the coordinate vectors were split into seperate latitude and longitude values. These values were used in a function that calls the Google Maps API and retrieves the elevation in meters at that particular coordinate. The result of this function was applied to the original dataframe. 

Next, NOAA's dataset of US 1981-2010 Annual Climate Normals was downloaded. For every weather station in the dataset, the annual average temperature in Celcius and percipitation in Millimeters were determined. To merge the climate data with the existing dataset, a function was written that would take the latitude and longitude values of each city and then calculate the distance to every weather station using the Haversine formula. All stations outside of a 20 km radius from the city centers were dropped and the temperature and percipitation were averaged for all remaining weather stations. The resulting vector was then applied to the existing dataframe.

Another function was written that connects to the Foursquare API and fetches venue details within a 5 km radius around the city centers. Both the count of venues as well as the three most occuring venue categories were extracted and added to the original dataframe. The end result was a nxm dataframe with n=100 cities and m=10 feeatures.

### 2.3 Data cleaning
Most data that was collected during the previous steps had already been properly extracted and required minimal data cleaning. Out of 10 available features, 8 were already numerical columns without any missing values. Of the remaining two columns, the 'State' column was not particularely interesting for our model so only the top 3 venue categories remained. In order to be able to process this column, the Natural Language Toolkit (nltk) was used to create bags of words for each category. These bags were then ready for further processing.

## 3. Methodology
After collecting all of the required data and merging it into a single dataframe, some explanatory data analysis was performed to understand possible correlations between different features. It was found that there were some obvious relationships between several variables. There was an apparent strong negative correlation between the latitude and average temperature features and a strong positive correlation between the longitude and average percipitation. The first relationship is very obvious as the temperature naturally increases when moving further north (and thus obtaining a larger latitude value) due to the tilt of the earth. However, the second relationship was less expected and meant that the average percipitation increased when the longitude value decreased, meaning a move towards the east. It was concluded that western US cities are, on average, much drier than their eastern counterparts.

Furthermore, there appears to be a moderate negative relationship between elevation and average percipitation. It was concluded that higher cities could be dryer on average, compared to cities located near sea level. Finally, a slight correlation was found between average percipitation and annual population growth. This shows that wetter cities could be less favourable to move to compared to dryer cities. However, other factors that are not accounted for might be responsible so it is too early to draw conclusions.

### 3.1 Feature selection
Now that a general overview of the dataset was aquired, it was possible to select the relevant features that would be used in our recommender system. Of the available 10 features, 7 were deemed appropiate for comparison between samples. Because the latitude and longitude values should not influence the model, these were omitted from the feature set. Furthermore, as mentioned in the previous step the state column would not be of interested as we are offering nationwide trips. The remaining 7 features were selected to be used in our model:
* Growth from 2000 to 2013
* Population
* Elevation
* Average temperature
* Average percipitation
* Count of venues
* Top three venue categories

To be able to use the bags of words that were created in the previous step, a count vectorizer was used that split the bags into seperate columns and counted the occurence for each sample. The resulting matrix was combined with the selected featureset and normalized using a standard scaler. Together, they form a normalized array of the featureset that was ready for ingestion into our recommender system.

### 3.2 Modelling approach
Finally, it's now possible to make use of this neat data to make some recommendations. Several approaches to designing recommender systems have recently been explored, the most extensive research has gone into two popular methods: Content based and Collaborative filtering (CF). Because CF methods rely on the user-item interaction matrix, they are subject to a cold-start problem where an accurate recommendation requires interaction between many users and items but this is impossible if the service has not been launched yet. This problem is the reason why the content-based approach will be used for the initial design of our recommender system as it only compares similar items instead of user-item interaction. In order to quantify the similarity between items, a similarity value must be calculated that will be used to obtain the most similar items.

Several methods are available to quantify the similarity between items, popular metrics such as Euclidian distance or Cosine similarity are suitable for usage in recommender systems. The main advantage of the Euclidian distance metric is that it is possible to determine the absolute nearest neighbour of an item based on the magnitude between the two samples. However, it is unable to take into account the direction of the two vectors and cannot distinquish between positive and negative values. It might find the nearest neighbour for any particular sample, but both vectors could be pointing in the opposite direction making them very unsimilar. 

This is where the Cosine similarity metric is most useful, as it determines the cosine of the angle between two vectors by using the dot product and dividing over the product of the magnitudes. The main advantage of this metric is that it is excellent in determining the similar angle between two samples. However, it is unable to distinguish the magnitude between two vectors and thus, does not determine the absolute nearest neighbour. For our recommender system, it was decided that angular similarity is valued above absolute distance because we want to avoid making wrong recommendations due to opposing directions of vectors. Therefore, we will accept that we will not necessarily find the closest match. This is actually desirable as it  will help our system recommend cities that will have more or less extreme feature values but with the same general relationship between them. Consequently, the Cosine similarity metric is selected as primary method for our recommender engine.

## 4. Results
The resulting

## 5. Discussion

## 6. Conclusion

