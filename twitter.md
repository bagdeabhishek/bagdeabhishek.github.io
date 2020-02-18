---
layout: default
title: twitter
permalink: /twitter/
---


Started with crawling 50mil in  5 days.

# Operating on data
The data I've crawled is huge and there is no way to work on it in memory (even after upgrading my main memory to 32gigs). So pandas is out of question. Pandas loads all the data in memory and then performs operations on it, our data size requires us to use more sophisticated solutions which generally are used in case of big data. 

## Apache spark
Apache spark has a very good dataframe API which can acutally speed up data access and processing. Though it is generally discouraged to use a single node apache spark cluster, I found the advantage of operating directly on data using DF API is significantly greater than loading data in memory and then working on it(which is anyways impossible due to amount of data we have). 

Setting up apache spark is fairly straightforward. Complete the setup add the postgres jdbc jar file in the appropriate location and you are good to go. Installing the pyspark package using will make things easier to work with if you prefer python instead of scala like me. The main issue I encountered is that, since the table I'm operating on is a single table with 50 millon rows. There is no parallelism present for the spark to take advantage of which caused ridiculously wait times. Initially these wait times caused a lot of timeouts and changing the settings fixed that. Still the time taken is ridiculous. 

To fix this problem I searched for ways to partition the table. Postgres allows you to partition the table using inheritance but in our case the table already exists and there is no easy way to partition table once created. The other option is using [pg\_partman](https://github.com/pgpartman/pg_partman). I'll configure the existing tables based on created\_at column so that the database is partitioned based on time intervals. This would ideally allow parallel access to the records and should speed up spark access.

# Plotting the network
Processing the crawled data in a user interactable form becomes challenging with the scale of data I had. The idea I had was to identify clusters in the twitter network and then do further processing on this information.

## Twitter network graph
Constructing the twitter network was done with the NetworkX graphing library in python. I converted the crawled data into list of tuples with one source and one destination node. I kept only the tweets which were retweets for this analysis. In general retweets are a stronger measure of endorsement compared to mentions. I've also observed that the more popular personalities' mentions carry more importance, these personalities rarely retweet. I might scale the edge weights in future keeping this creiteria in mind and do additional processing to get better clutering using gephi's clustering algorithm.

The function to costruct the graph is pretty simple. The graph we construct is a Directed graph where edge weight is the number of times a user A retweets user B. The python function below does the same simple operation and gives a Directed Graph G.
```python
def create_graph(ls_tup):
    G = nx.DiGraph()
    for dc in ls_tup:
        tfrom=dc['tweet_from']
        rt = dc['retweeted_status_user_handle']
        if G.has_edge(tfrom,rt):
            print(tfrom,rt,G[tfrom][rt]['weight'])
            G[tfrom][rt]['weight'] += 1
        else:
            G.add_edge(tfrom,rt,weight=1)
    return(G)
```
We can do a lot of preprocessing in python itself but i've found [Gephi](https://gephi.org/) to be a much better tool which is easier to use and operate on such large amounts of data. The visualizations are also way more engaging and it comes with various plugins to export the network in various formats. 
You can export this Graph easily using GEXF format which can be done using NetworkX library's _write\_gexf()_ function


## Gephi processing
Once you get the GEXF file you can run Gephi and import the data into the tool. Before you do that though, make sure you have installed Oracle's Java version. The difference between Oracle Java and the OpenJDK version is day and night, Gephi basically becomes unusable if the network is as large as I had. Once installed disable the anti-aliasing in settings to make the renders quicker as well.
### Pre-processing
Once you get the data into Gephi you can do some preprocessing, I did the following thigs to weed out un-important nodes

1. Calculate the weighted degrees of all the nodes. Trim the nodes with weighted degrees less than 2 (you can adjust this threhold to much higher)
2. Run the modularity algorithm tweaking the parameters to get less number of communities. You can now see major clusters that aris out of data. 
3. Trim out the clusters with less than say 500 nodes in them. This can be done using the partition count filter in Gephi.
4. Run the clustering algorithm once again on this reduced graph. Now you shoulds see the clusters clear enough. In my case all handles from a particular political party were in a cluster.

### Visualization
Once you have the data the next step is visualizing the data. If you don't want to visualize, go to data tab and select columns of interest in my case that would be the modularity class and export it as CSV or some other format. Visualization in Gephi is pretty cnofusing if you are new to it. For most of the purposes aimply running the Openord clustering algorithm will suffice. That's what I did, I tweaked the phases of Openord algorithm to give more time to expansion phase. Also increasing iterations will help in case of large networks, though it might take more time depending on the hardware you have. 

Once you are happy with the visualization render it and export it in any format you want. Hot-tip you can use the plugin (will add later) which wil generate the HTML code which renders using Javascript for browser friendly interactive rendering.











