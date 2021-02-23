# **VRPTW with Advanced Merged RDBSCAN**


![](https://ahamove.com/wp-content/uploads/2018/10/Web-c%C5%A9-sang-web-m%E1%BB%9Bi.jpg)

# Agenda
  1. Introduction
  2. DBSCAN
  3. Recursive DBSCAN ( RDBSCAN )
  4. VRPTW with Advanced Merge RDBSCAN
  5. Deployment with AWS Serveless



# 1 Introduction

## a. Vehicle Routing Problem at Ahamove.
Vehicle routing problem ( VRP ) is a combinatory optimization and integer programming problem in which there are a finite but very large ( million ) numbers of combination of fleet of drivers and way points. And the task is to find out an optimized combination which minimizes an objective such as total travel distances or number of used drivers while must travel through all way points once and satisfy constraints such as the maximum travel distance of a driver, or due time delivery at a way point.


At Ahamove, in each defined interval time of Sameday Service, we have a set of orders need to be delivered in same day. The available drivers on our platform are  exactlly the fleet of drivers mentioned above. Then, we have a task that need to match orders to drivers in which a driver can be assigned many orders with different sequence. The VRP is used here to minimize the total distanced. In addition, Ahamove' s partners/custormers also face VRP as they have to deliver products from their hubs to different stores in same day but at different available receiving time, time windows, using their own finite fleet of drivers.<br>
The more information about VRP at Ahamove can be read [here](https://sites.google.com/ahamove.com/opstech-featuring/features/vehicle-routing-problem?authuser=2#h.p_xOflMdDv_anF) 

## b. OR tool and need of clustering orders.
To find solutions for a VRP, we need to model the problem to a suitable formula form and an optimizer solver to solve that. There are many available open source and commercial solvers, but [Or tool](https://developers.google.com/optimization) which is an open source software for combinatorial optimization problem is our choice. OR-Tools is developed by Google AI, and  won three over five categories at 2020 MiniZinc Challenge, the international constraint programming competition. Moreover, OR-Tools supports python language which could be adapted by team easily.

![ort-picture](https://developers.google.com/optimization/images/routing/cvrp_solution.svg)

ORT is strong but still limited compared with a demand of big logistics companies. The solver performs well under three hundreds sample points for our constraint VRP, but degrades quickly with larger samples. To bypass this limitation, we can pre cluster way points to smaller groups, then solve them with OR-Tools one by one. There is a work from [GOGOVAN](https://towardsdatascience.com/improving-operations-with-route-optimization-4b8a3701ca39) that they develope a recursive DBSCAN algorithm to cluster way points to smaller groups and solve their VRP with Time Window problem with them. The claimed result is very expressive when the gap between global solutions by pure OR over all points and combined local solutions by RDBSCAN is properly small.


Motivated by above algorithm, we apply and update this RDBSCAN algorithm for Ahamove data sets. In addition, steps of preprocessing, post-processing and dealing with outlier points are not presented in above article, so we need to handle it ourself.


# 2 DBSCAN algorithm

## a. Clustering algorithms
- what is cluster algorithm ?
- Type of cluster algorithm ?
- Comparision between cluster algs ?
- Why DBSCAN suitable ?

Clustering is an unsupervised machine learing problem which automatically realize groups of data based on their features.
It is constrat to supervised problem where labels of trained data are given, and the algorithm learns from old data to predict right labels for new data. In the clustering problem, we may dont know the number of groups, there are algorithms can detect groups without knowing the number of groups in advanced. Normally, data lies in multi-dimension space, but we use data in 2d space to visulaize, and it is also the space of positions of our way points.


![](https://media.geeksforgeeks.org/wp-content/uploads/k-means-copy.jpg)

Clustering algorithms can be classified as : center based algorithms ( KMean), denstiy based algorithms ( DBSCAN ), distribution based algorithms (Gaussian Mixture Model). To compare cluster algorithms, we can refere from [sklearn work](https://scikit-learn.org/stable/auto_examples/cluster/plot_cluster_comparison.html#sphx-glr-auto-examples-cluster-plot-cluster-comparison-py)

DBSCAN connects areas of high example density into clusters. This allows for arbitrary-shaped distributions as long as dense areas can be connected. Our way points are 2D feature data ( latitude and longtitude ), so by drawing in 2d axis, we can visualize the cluster trending easily. Points are densed in center and spare at remote areas around city center. If we try to cluster points to groups, there would be outliers that intuitively should not in any groups. From this property of our data, DBSCAN seem to be best candidate for clustering

![](image of ahamove points at HCM)



## b. DBSCAN property.


RDBSCAN starts with a point, searching for neighbor points within a radius of epsilon, if no of neighbor points are over a threshold , this point are considered a core point of a cluster, and continue to search with neighbor points. We see that epsilon affects to density level of clusters and min points affect to noise level determination of the algorithm. If the min points is large, then many points can be clasified as noises, otherwise noised points are considered fewer.
We can see the graph of no of cluster vs radius of searching neighbor ( radius ~ epsilon ). if the epsilon is low, there are large part of points not be clustered. If it is too high, clusters will be merged. So the graph has bell curve.

![final test limit bm](https://drive.google.com/uc?export=view&id=1-m4LbiOCpPjfcEqYnUjVAN8q9dW1n6zR)

![final test limit bm](https://drive.google.com/uc?export=view&id=1Hi6Es4Z51Guskysa-ouya9s4_iDxdoCN)

For our problem, we want to have a maximum no of clusters which are well separated, and it should corresponding with crital epsilon of the peak point in the above bell curve. 
When we draw a curve of standard deviation of cluster sizes versus epsilon, we also recieve the fairly low std at peak points.
 
To find peak points, we can simply gird search with a proper small step size ( or use gradient descent )

# 3 Recursive DBSCAN


## a. Algorithm


The recursive DBSCAN is a recursive search of best radius to get lowest standard deviation of cluster sizes achieved from naive DBSCAN. If we have 300 points, we prefer ( 180, 120 ) clusters over ( 200, 50, 50 ) clusters.

As DBSCAN always results in outlier points, so we don't try control it by setting min points to a constant. We just change radius ( epsilon ) in each DBSCAN call. The algorithm as below

```
Data: all points (lat, long)
min_points = const
max_points_threshold = const
max_radius = 1000m
final_cluster = []
Recursive_RDBSCAN: points, max_radius
  best_cluster_size = 999999
  best_cluser == ∅
  for a_step_20_m from 20 to max_radius:
    clusters <- DBSCAN(points, a_step_20_m, min_points, algorithm ='ball tree', metric ='haversine')
    if no_of_clusters >= 2:
      if std_cluster_size < best_cluster_size:
        best_cluster_size <- std_cluster_size
        best_cluster <- clusters
      end if
    end if
  end for
  if best_cluser == ∅
    return points
  for each_cluster in best_cluster:
    if len(each_cluster) > max_points_threshold:
      sub_clusters <- Recursive_RDBSCAN(cluster_points, cluster_radius)
      final_clusters <- final_clusters U sub_clusters
    else:
      final_clusters <- final_clusters U clusters
    end if
  end for

optimized_solution = or_api_with_cluster(final_clusters)
return optimized_solution
```

## b. Pre and post process


Pre-process

- Use DBSCAN and preset Radius to detect points are so far from our intereset region such as HCM. These points are left out of algorithm.

![final test limiqwt bm](https://drive.google.com/uc?export=view&id=1NRRqJyMzo_iBTu53f23U0ropIPLKwCWm)

- Detect dense clusters that could be hundreds of orders just in radius of few meters ( under 20m ) 


![dense cluster](https://drive.google.com/uc?export=view&id=193s-mE2I7Xq_WJuyqXV5KSf-fVZRpwvI)

Post-process
- Merge DBSCAN outliers with achieved clusters by nearest cluster center merging.
- Use recursive 2 group k-mean algorithm to divide large size clusters to smaller clusters which max size is under our defined threshold.


## c. Result and Analysis


Below is the result of clusters with RDBSCAN. Clusters are well densed. However, there are still lot of small clusters. It is nature of our dataset, as we can't find a set of radius that cluster data too large size groups. There are small clusters with similar radius blended between big clusters, so when we increase the radius of searching point those clusers are merged together. Therefore, there are always small clusters with few big clusters at middle.

![rdba dense](https://drive.google.com/uc?export=view&id=1XK3PwlxfQTfrkYOU8ZwYLYo_3YYsvKNH)

If number of small clusters are high, it could degrade the performance of VRP, we will exam and find solutions for it in next section.

**Peformance increasing**
- As we tune best radius by grid search, it could slow down our algorithm. However, we note that in first grid search, we already run over all possible radius for all points; therefore, we can apply dynamics programing with memorizaion to look up the cluster result of sub-point at each radius



# 4 VRPTW with Advanced Merge RDBSCAN

## a. VRPTW
- VRPTW is a VRP with a constraint of delivery time that means drivers have to travel to a waypoint in a permisable interval of time called time window. For example, a waypoint could just available from 8h to 17h to recevie products, while others  are avaiable from 17h to 20h.
- For our dataset, we define timewindow intervals like (7 -18h), ( 17-20h). And they are uniformly distributed to waypoints.

## b. VRPTW with RDBSCAN, RKmean and pure OR.

Besides using RDBSCAN to divide waypoints to small groups, we also use Recursive Kmean (RKmean) and random division to compare results. In RKmean, we apply Kmean recursively to sub-clusers to achieve sub clusters whose sizes are under our 200 threshold .For small datasets, we just use pure OR API to solve VRPTW for all points, and its result if achievable is best solution as it is global optimized solution.

**Benchmark of RDBSCAN with RKmean and pure OR.**

![](https://drive.google.com/uc?export=view&id=1_HwMMAHZvX6hSvf7Ntt9B-srV-Pnqfxo)

The sample sizes for benchmark are 500,750,1000 and 1200. The solution of RKmean is much better RDBSCAN at total distance and vehicle used. We suspect it is due to a large amount of small size clusers resulted from RDBSCAN compared to RKmean. The global optimized solution from pure OR is far better than RKmean and RDBSCAN, but it is just available to 1000 waypoints. At 1250 points, OR API dosen't response.


**Benchmark of RDBSCAN with RKmean and random division.**

![](https://drive.google.com/uc?export=view&id=1UaV_npO3ynfFeHT3XrU_huVOhb5wox4g)

The sample sizes for benchmark are 500,1000,1500,2000,2500 and 3000. The solution of RKmean is still better than significant better than RDBSCAN and random division. Also, compared to random division, two clusters show better at total distance and worst at vehicle used. It is reasonable as RDBSCAN and RKmean results out small clusters than random division so they use more drivers/vehicles. However, as group are more densed so clusters' solutions have optimized distance compared with randomness.

**Require of merging small cluster**

We observe that if clusters are densed but much smaller than the threshould, 200 points, they will consume much larger drivers. Therefore, we should cluster to big group or merge them together. Clustering points to big groups by RDBSCAN is difficult as it is nature of our data that small clusters are distributed around city and we cann't tune a big radius of searching to merge them by RDBSCAN. We prefer an advanced merging method to solve this issue.

## c. VRPTW with Advanced Merged RDBSCAN (AM-RDBSCAN), RKmean and pure OR.


**Advaned merged of clusters from RDBSCAN.**

RDBSCAN returns small densed clusters, so we can take center of clusters and number of points ( weight) to represent for each cluster. We now have a set of center points and corresponding weights. The merging problem becomes VRP that we need travel to all points with minimum number of drivers, which means minimal final merged-clusters, and satisfy the capacitated contraint that total weight through a route ( merged cluster ) must smaller than the 200 points threshold. By converting to capacitated constraint VRP, we can automatically merge close sub-clusters together and ensure the merged size under our control. 

**Benchmark of AM-RDBSCAN, with RKmean and pure OR.**

![](https://drive.google.com/uc?export=view&id=1zE3gT4x3cBjxCOfu6lqNg_gnTERJo-VY)

The sample sizes for benchmark are 500,750,1000 and 1200. The results are really impressive, as the total distances of merged RDBSCAN follow pure OR distance, and even the total vehicles used are fewer. The OR API is no surprised not response at 1200 points, why AM-RDBSCAN seems be linear when no of points increase. The  RKmean solutions now are significant worse than AM-RDBSCAN, and we will exam it with larger sample tests.

**Benchmark of AM-RDBSCAN with RKmean and randomness division.**

![](https://drive.google.com/uc?export=view&id=1hOWWHv7UC5lpqPNxdH8vsRzwCDnSMFl1)

The sample sizes for benchmark are 500,1000,1500,2000,2500 and 3000. The solution of AM-RDBSCAN improves as we expect. While the total distances are fewer than RKMEAN, the total used drivers are significant lower.

**Conclusion** : The advanced merged algorithm really works and improve our original RDBCAN as its performance follows gloable OR solution and better RKmean in our test. 

# 5 Deployment and API.

## a. Deployment with serverless

Following the serveless architecure of team for the ease of deploying and scaling , the AM-RDBSCAN is deployed on AWS serveless.

## b. API
- **HTTP Request**
```
Post request at: https://pq82nnwzc7.execute-api.us-east-1.amazonaws.com/staging/rdbscan
```
- **Parameters**

|Parameter|Type   |Required   |Description|
|:---|:---|:---|:---|
|Points| 2D array  | Yes  | 2d array of points in form (lat,long). Ex:<br>`{"points" : [[10.7132507,106.6167047],[10.6132507,106.5167047],[10.8132507,106.7167047]]}`|

- **Response**
```
Status-Code: 200 OK
Code400: points are not in 2d array form/Missing required input
Code500: internal error when running rdbscan
```

|Parameter|Type   |Description|
|:---|:---|:---|
|clusters| 3D array<br>[] possible| array of 2d array of points in form (lat,long). Ex: 2 clusters of 2 and 1 points <br>`[ [[10.7132507,106.6167047],[10.6132507,106.5167047]], [[10.8132507,106.7167047]] ]}`|
|clusters_indice| 2D array <br> [] possible| 2d array indice of above cluster's points. Ex: points' indice of 3 cluster <br>`[[2,3],[0],[1,4]]`|
|noises| 2D array <br> [] possible| 2d array of points in form (lat,long). Ex: 3 noise points <br>`[ [10.7132507,106.6167047],[10.6132507,106.5167047],[10.8132507,106.7167047] ]}`|
|noises_indice| 1D array <br> [] possible| 1d array indice of noise points. Ex: Indice of 3 noise points <br>`[33,2,1]`

```python

```
