
# Mystery City Recommender

*Author: Sebastiaan Hemmer*

*October 12, 2019*

## 1. Introduction
My imaginary traveling agency ‘*Sacramentum*’ has a unique value proposition; we offer mystery trips to different US cities without our clients knowing their destination in advance.

Our clients purchase tickets and are only provided information regarding the climate of a destination so they know what clothes they should pack. At the airport, our clients are given an envelope which contains the tickets to their destination. Their destination is drawn completely random at this moment.

### 1.1 Problem definition
Recently, we have been dealing with an increasing amount of clients that were unsatisfied with the mystery location they have traveled to. Because this is impacting our business, we are looking for solutions to improve our customer satisfaction. Therefore, we have decided to design a recommender system that takes user input of any number of cities that they rate positively and then recommends a mystery location that will most likely be rated positively by our client.

To determine the profile of our clients, we will make use of the Foursquare API to cluster different US cities based on certain features. These features shall mainly consist of the number of different types of venues and attractions in a city, which will be combined with available geographical and demographic open data.

Because we will be using unlabelled data and do not have access to a training/testing dataset, we are limited to using unsupervised learning techniques.

## 2. Data sources and cleaning
As mentioned in the previous section, we will make use of the Foursquare API and combine it with both geographical and demographic data that are publicly available. To determine an accurate client profile, we will require a sufficient amount of features to make distinctions between different cities but should not use too many features as it will overfit and overcomplicate our model. Therefore, we have selected the following data from different sources.

### 2.1 Data sources
*Demographical data*

This will be our starting dataset, we will select the top 100 largest US cities by population from the available dataset on ODS ([source](https://public.opendatasoft.com/explore/embed/dataset/1000-largest-us-cities-by-population-with-geographic-coordinates/table/?sort=-rank)), which will provide us information about the:
* Name of the city
* Population
* Coordinates
* Growth of population

*Geographical data*

We will then combine our demographical data with geographical data from open NOAA datasets, that include data such as:
* Elevation
* Average annual temperature
* Average annual precipitation

*Foursquare API*

Lastly, we will make use of the Foursquare API to include venue data. We have selected the following features of importance:
* Top 3 most popular venue categories (in 5km radius from center)
* Total number of venues in a 5km radius from the center

### 2.2 Data collection
First of all the ODS dataset was downloaded and converted into a data frame with 1000 rows and 4 features. To speed up further calculations, only the top 100 cities based on population were selected and the remaining 900 were dropped. Then, the coordinate vectors were split into separate latitude and longitude values. These values were used in a function that calls the Google Maps API and retrieves the elevation in meters at that particular coordinate. The result of this function was applied to the original data frame. 

Next, NOAA's dataset of US 1981-2010 Annual Climate Normals was downloaded. For every weather station in the dataset, the annual average temperature in Celcius and precipitation in Millimeters were determined. To merge the climate data with the existing dataset, a function was written that would take the latitude and longitude values of each city and then calculate the distance to every weather station using the Haversine formula. All stations outside of a 20 km radius from the city centers were dropped and the temperature and precipitation were averaged for all remaining weather stations. The resulting vector was then applied to the existing data frame.

Another function was written that connects to the Foursquare API and fetches venue details within a 5 km radius around the city centers. Both the count of venues as well as the three most occurring venue categories were extracted and added to the original data frame. The final result was an *nxm* data frame with n=100 cities and m=10 features.

### 2.3 Data cleaning
Most data that was collected during the previous steps had already been properly extracted and required minimal cleaning. Out of 10 available features, 8 were already in a numerical format without any missing values. Two features remained, the 'State' column was not particularly interesting for our model so only the top 3 venue categories were deemed relevant. To be able to incorporate this feature into our model, the Natural Language Toolkit (nltk) was used to create bags of words for each category. These bags were then ready for further processing.

## 3. Methodology
After collecting all of the required data and merging it into a single data frame, some explanatory data analysis was performed to understand possible correlations between different features. It was found that there were some obvious relationships between several features. There appeared to be an apparent strong negative correlation between the latitude and average temperature as well as a strong positive correlation between the longitude and average precipitation. The first relationship is very obvious as the temperature naturally increases when moving further north (and thus obtaining a larger latitude value) due to the tilt of the earth. However, the second relationship was less expected and meant that the average precipitation increased when the longitude value decreased, meaning a move towards the east. It was concluded that western US cities are, on average, much drier than their eastern counterparts.

Furthermore, there appears to be a moderate negative relationship between elevation and average precipitation. It was concluded that higher cities could be dryer on average, compared to cities located near sea level. Finally, a slight correlation was found between average precipitation and annual population growth. This shows that wetter cities could be less favorable to move to compared to dryer cities. However, other factors that are not accounted for might be responsible so it is too early to conclude.

### 3.1 Feature selection
Now that a general overview of the dataset was acquired, it was possible to select the relevant features that would be used in our recommender system. Of the available 10 features, 7 were deemed relevant for comparison between samples. Because the latitude and longitude values should not influence the model, these were omitted from the feature set. Furthermore, as mentioned in the previous step the state column would not be of interest as we are offering nationwide trips. The remaining 7 features were selected to be used in our model:
* Growth from 2000 to 2013
* Population
* Elevation
* Average temperature
* Average precipitation
* Count of venues
* Top three of venue categories

To be able to use the bags of words that were created in the previous step, a count vectorizer was used that split the bags into separate columns and counted the occurrence for each sample. The resulting matrix was combined with the selected feature set and normalized using a standard scaler. Together, they form a normalized array of the feature set that was ready for ingestion into our recommender system.

### 3.2 Modelling approach
Finally, it is possible to make use of this neat data to make some recommendations. Several approaches to designing recommender systems have recently been explored, the most extensive research has gone into two popular methods: Content-based and Collaborative filtering (CF). Because CF methods rely on the user-item interaction matrix, they are subject to a cold-start problem where an accurate recommendation requires interaction between many users and items but this is impossible if the service has not been launched yet. This problem is the reason why the content-based approach will be used for the initial design of our recommender system as it only compares similar items instead of user-item interaction. To quantify the similarity between items, a similarity value must be calculated that will be used to obtain the most similar items.

Several methods are available to quantify the similarity between items, popular metrics such as Euclidian distance or Cosine similarity are suitable for usage in recommender systems. The main advantage of the Euclidian distance metric is that it is possible to determine the absolute nearest neighbor of an item based on the magnitude between the two samples. However, it is unable to take into account the direction of the two vectors and cannot distinguish between positive and negative values. It might find the nearest neighbor for any particular sample but both vectors could be pointing in the opposite direction making them very dissimilar. 

This is where the Cosine similarity metric is most useful as it determines the cosine of the angle between two vectors by calculating the dot product and dividing over the product of their magnitudes. The main advantage of this metric is that it is excellent in determining the angle between two vectors. However, it is unable to distinguish the magnitude between them and thus, does not always determine the absolute nearest neighbor. For our recommender system, it was decided that angular similarity is valued above absolute distance because we want to avoid making wrong recommendations due to opposing directions of vectors. Therefore, we will accept that we will not necessarily find the closest match. This is actually desirable as it will help our system recommend cities that will have more or less extreme feature values but with the same general relationship between them. Consequently, the Cosine similarity metric is selected as the primary method for our recommender engine.

## 4. Results
Based on the determined requirements, a recommender engine was designed that takes a list of any number of cities that a user has enjoyed visiting in the past as input. The recommender system will process these cities and determine a user profile which represents the average of the feature values that were input into the system. This vector is then compared to the feature set and the cosine similarity between the user profile and all available cities in our dataset is calculated. The result is an array with similarity scores for every city in our dataset. This array is then filtered to exclude cities that the user has already traveled to in the past. Finally, the remaining cities are sorted based on their similarity scores and the top 5 cities with the highest similarity scores are output from the recommender engine. The similarity score ranges between [-100%, +100%], where -100% means that both vectors have complete opposide directions, 0% means orthogonal directions and +100% means both vectors share the same direction.

## 5. Discussion
After testing our recommender system with multiple different inputs it becomes clear that the engine seems to be operating as expected. Usually, the top recommendation has a similarity score of anywhere between 70-80%, which is considered quite reasonable for a cold-start system. Because of the scope and workload of collecting data for this project, the number of features was kept to a minimum. To improve the quality of recommendations, additional relevant features could be appended to the system. However, care must be taken that using too many features will increase the chance of overfitting the model, leading to too generalized results.

Additionally, results could be improved by assigning a weighting factor to the cities that users have previously visited as prior visits do not necessarily implicit a preference for any of those cities. Furthermore, another improvement opportunity lies in adding a weighting factor to each feature in the user profiles as some features might be more important to a user than others. However, both options require additional user input which is not always desirable. Therefore, a trade-off must be made between the required input and the quality of results.

Finally, the largest improvement to our recommender system will be possible after it is launched and a large amount of users has interacted with it. Based on reviews of recommended cities, a user-item interaction matrix can be created which opens up possibilities for collaborative filtering methods that improve recommendations based on other users. It is shown that recommender systems using a CF approach are almost always more accurate compared to content-based methods.

## 6. Conclusion
The problem defined at the start of this project was that we were looking for a method that would improve our customer satisfaction by building a system that recommends a city that is expected to be rated positively by our client. We decided to design a recommender system that would take any number of cities that our clients have rated positively in the past, use that information to build a user profile, and then recommend a mystery location that will most likely be rated positively too.

Concluding, we determined that it is certainly possible to build such a recommender system based on publicly available open datasets using a content-based approach. We have shown that the cold-start problem to collaborative-filtering methods can be partially mitigated by starting with a content-based method by using similarity metrics between items. For research purposes, this system provided sufficient results. However, this system is far from industry acceptable standards and improvements are required to achieve sufficient quality for real-life usage. It is assumed that higher quality recommendations could be made by adding relevant features and by building better user profiles by implementing weighing factors for each feature. However, further research is required to determine which modifications will achieve higher quality recommendations. Finally, the best results will be obtained by users interacting with the recommender engine online and appending a CF approach to the system as it has been shown that this approach will almost always provide more accurate recommendations.
