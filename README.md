# Daytime fill model

## Objective
To create a model which can predict speed on the roads of Portland. The objective was to come up with a solution which handles the fresh GPS data fallback scenarios, on which many services in Mapbox pancake depend.   
## Data
The data for each road in Portland in [GeoJSON](http://geojson.org/) format is provided and is in a human-readable geographic data format. One example can be viewed as below: 
```json
{
"type":"Feature",
"geometry":{
      "type":"LineString",
      "coordinates":[[-121.121805,45.606604],[-121.121312,45.607024],[-121.121027,45.607257],[-121.12084,45.607422],[-121.120588,45.607628],[-121.120276,45.607834],[-121.119874,45.608146],[-121.119606,45.608315],[-121.119413,45.608442],[-121.11901,45.608589],[-121.118699,45.608641],[-121.118195,45.608686],[-121.11701,45.608615],[-121.11555,45.608379],[-121.114874,45.60827],[-121.113571,45.608067],[-121.112487,45.607913]]
      },
"properties":{   
      "id":"f5192971!7,5195262!0",
      "refs":["702618838","36175616","36179046","36212728","36187460","36183216","36200908","36223627","702619163","36210593","36184460","36187183","36191000","36208730","36178984","36183865","3624137549"],
      "oneway":1,
      "highway":"tertiary",
      "speed":58
      }
}
```
Each line can be visualized in a tool such as [geojson.io](http://geojson.io/). 
Understanding the features of data:
- [name](https://wiki.openstreetmap.org/wiki/Key:name)
  - the name of the road
- [highway](https://wiki.openstreetmap.org/wiki/Key:highway)
  - the type of road (ie: motorway vs residential)
- [oneway](https://wiki.openstreetmap.org/wiki/Key:oneway)
  - whether or not the road is oneway or bidirectional
- refs
  - a list of node ids that make up the road
- speed
  - the predicted daytime prevailing speed
  - a speed of -1 indicates a road we did not have enough data to make a prediction for

## Defining Single Numeric Metric
Setting single value matric makes easier to compare the models. As here the problem demands prediction of a numeric value basically it is a regression problem and residual error would be a good measure to understand how far the model's prediction is from the true value. Specifically, [Reduced mean squared error](https://medium.com/human-in-a-machine-world/mae-and-rmse-which-metric-is-better-e60ac3bde13d) was used for evaluation of models.
![render](https://user-images.githubusercontent.com/2561578/37561759-f8435128-2a13-11e8-942e-15440fb1e0fe.png)

## Strategy 
Out of the data provided, 68 % is labeled(or had speed values). The labeled data is split into 80-20 proportion to use as test and train datasets. After doing initial data processing the data had following features: `id`, `refs`, `highway`, `speed`. Initially started with simpler models to predict the speed. This approach is without machine learning modeling, which helps in understanding the data, feature dependency on each other and to come up with the accurate pipeline to reach the objective.

### Model 1- Overall average speed
This is model just averages the speed of available data points and returns the value.
Drawback: The hierarchy between the roads  (or any other feature) is totally ignored hence, high error.

### Model 2- Categorical average speed
This model considers the categorical feature- `highway`, and returns the average speed of all roads belonging the category of the test data point. For example, the categorical average for motorway is 50 km/hr, the model predicts the same speed for any test point belonging to that category.
Drawback: This model blindly considers the extreme outliers hence the predicted value has a considerably high error.

### Model 3- Categorical mode speed
Like model 2 this model also considers the categorical feature- `highway`, and returns the most frequent speed value of the roads belonging the category of the test data point. For example, the mode value for motorway category is 54 km/hr, the model predicts the same speed for any test point belonging to that category.
Drawback: Relatively less error as the `highway` type and the extreme outliers are ignored. But again, this model has a high bias 

### Model 4- One-way vs two-way road segregation analysis
This model applies separate algorithms for predicting the speed of one-way and two-way roads. For one-way roads, it again used the categorical mode value as model 3 performed better than model 2.
For two-way roads, the density of the difference between the speeds of roads going to opposite direction looked like this:
![figure_1](https://user-images.githubusercontent.com/2561578/37562712-45d59618-2a2c-11e8-9a11-7563075da9e0.png)

Hence the algorithm just returns the speed of the opposite direction. Considerably better performance than previous models.
Drawback: In real-time depending on opposite lane's speed is highly unreliable.  Whereas, after discussing with @morgan, it was cleared that the two-way roads are considered separate one-way roads going in `forward` and `reverse` directions.

The performance of above models in terms of RMSE is represented as below:
![mp1](https://user-images.githubusercontent.com/2561578/37562993-a2a23944-2a33-11e8-9b95-42309a232ed1.png)

Now, after having basic data engineering it was time to dive deep into Open Street Map feature's domain. 
### Feature Engineering
#### Curvature
Even though all the data points (roads), in the dataset were of `type`  [LineString](https://tools.ietf.org/html/rfc7946#appendix-A.2), consider the following  [lineString](https://tools.ietf.org/html/rfc7946#appendix-A.2) examples. 
![1](https://user-images.githubusercontent.com/2561578/37563086-09829db4-2a36-11e8-8c6b-39df16c79226.jpeg)

Naturally, the curvature of the road affects the speed hence for each data point, the curvature was calculated.
#### Neighbouring Roads
The road network can be viewed as a directed graph. Hence while predicting the speed of an edge(road), the neighboring edge's(connected roads) speed should not be ignored. A hypothesis was developed to consider the effect of the category to which the neighbor belongs. Following algorithm was developed for this purpose:

1. Calculate the average speed for each of the `highway` category.
2. Rank each category according to the average speed. Highest speed category ranks 1st and lowest speed category ranks last.
3. Calculate the weights according to the average speed. (Used weighted average for this purpose) 
4. Consider all the neighboring edges of an edge `e`, and consider two new features `increasing` and `decreasing`. 
5. `increasing` is the sum of weights of all the edges which have category ranks less or equal to that of edge `e`.
6. `decreasing` is the sum of weights of all the edges which have category ranks more than that of edge `e`.
Following figure illustrates the algorithm to compute `increasing` and `decreasing`.
![2](https://user-images.githubusercontent.com/2561578/37563578-398f42a4-2a41-11e8-93f8-fc5622894315.png)
Hence the data had these features: `highway`, `curvature`, increasing`, and `decreasing`.
### Model 5- Random Forest



