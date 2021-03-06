## Why this post

Basically, my analytical toolbox consists of a few basic things:

1.  Programming Languages (`R` and `python`)
2. Storage   (`MySQL` and `MongoDB`)
3. Visualization and Reporting ([Tableau Desktop](http://www.tableausoftware.com/))

If I even remotely resembled a ***Data Scientist***, other tools listed above would include the all things command line, Hadoop, d3, and some familiarity with AWS.  With that said, you can do some really cool research with the tools listed above.  

Recently, Tableau announced that with version 8.1, the software would be able to talk to `R`.  This is a great first step at attacking a popular feature request, but the company, in my opinion, missed the point.   The large majority of Tableau users leverage the software to **visualize** data, not clean and recode datasets.

In truth, the new integration with R is pretty cool.  Users can execute R commands on data accessed in Tableau.  My big beef with this feature is that the company already released an API for python.  Simply, a python user could create Tableau datasets (Extracts) programmatically from any dataset, web or otherwise. This means they could access any API they wanted, modify, recode, and fit models to the data ***BEFORE*** bringing it into Tableau.  This is ***EXACTLY*** what I wanted for R.  

Nonetheless, I thought it would be fun to build upon my last post and highlight how you **could** use Tableau to access your Google Analytics data and then use `R` to do some basic analysis not native to the software.  I will talk about the downside this approach later, but it is certainly worth highlighting the capabilities.



### Setup

Right now, Tableau Desktop only works in a Windows environment but will be available for Mac users in an upcoming release (finally!!!).  To be honest, the setup is pretty straightforward.

Obviously you will need to have R installed. I am using version 3.0.1.  From there, ensure that you have the `Rserve` package installed.


```r
install.packages("Rserve")
```


Let's load the package.  NOTE:  You only have to run the `install.packages` command above if the library is not already on your system.


```r
library("Rserve")
```


The last step is that you fire up the server.  Tableau will use this service to pass commands (and data) back and forth to R.


```r
Rserve()
```



Because you leave `R` running in the background, I typically will just fire up a new command line session and let the server run in the background.  For example, here is what my terminal session looks like:

![terminal](rserve.png)


If you want a better, more formal introduction to getting R and Tableau talking, you should refer to the [official help page](http://onlinehelp.tableausoftware.com/v8.1/pro/online/en-us/help.htm#r_connection_manage.html).


### Tableau and Google Analytics

Now let's connect Tableau to the Google Analytics API for our school's website.  When you fire up Tableau, on the `Connect to Data` page, select `Google Analytics` from the left-hand menu.

![connect](ga-connect.png)

This will bring up a dialog box asking you to authenticate with the Google Analytics API.  We did this programatically in the last blog, but it's all the same thing conceptually.

![connect](ga-login.png)

Say `Sign In`, and on the next page, `Accept` that Tableau will programtically access our GA data.

![accept](tableau-app.png)


### Let's analyze some data

As promised in the title, I mentioned that we would be doing something beyond basic reporting. There are a ton awesome questions we can ask from the GA API, but for this post, we are going to grab some basic web stats aggregated by page for January through October 2013.

![data](ga-data.png)

If you think in SQL, then the query is something like.....


```r
SELECT page, bounces, entrances, exits, pageviews, time on page, visits
FROM Google
WHERE date >= 1/1/2013 and date <= 10/31/13
GROUP BY page
```



Being able to quickly and easily access our GA data from inside Tableau is awesome, but you need to be mindful of how it hits the API.  For basic exploration, it's not a big deal, but if you are using Tableau to study your school's website and make decisions, you should really figure out whether or not your data are being [sampled](https://support.google.com/analytics/answer/1042498?hl=en).



#### A quick summary of our data

Before performing any sort of detailed analysis, you really should take some time to evaluate the data beforehand.  Visuals are great, but sometimes we just need to look at the raw data ([e.g. Tableau feature request](http://community.tableausoftware.com/ideas/2685))

Here is how I setup my report; our summary stats by page.

![tableau-report](tableau-report.png)

Given our URL structure, we retain the search parameters in the page path, which means each search request is a different page. Because I am more interested in the root page, let's standardize our page paths.  

You should be able to copy and paste the code below into Tableau's calculated field box.  The `//` symbols are how we can include comments (and document!) our calculations.


```r
// This function cleans the URLs of any search parameters
// You might not need this function, but hopefully it helps
IF CONTAINS([Page], "?")
THEN LEFT([Page], FIND([Page], "?"))
ELSE [Page]
END
```


You might want to calculate the basic KPI's that Google does for us automatically.  Reference [this page](https://developers.google.com/analytics/devguides/reporting/core/dimsmets) for help and further discussion on the calculations.  For example:


```r
// Bounce Rate
[Bounces] / [Visits]

// Entrance Rate
[Entrances] / [Visits]

// Avg Time on Page
[Time on Page] / [Pageviews]
```


However, to keep things easy, I am going to use the basic data.

#### Cluster Analysis

Be forewarned, creating `R` code in Tableau is a nightmare.  I **highly** recommend writing your code externally and simply sourcing it in Tableau.  

The code below is a function that takes the data from Tableau, puts it into a dataframe, standardizes the values, and then uses [kmeans](http://en.wikipedia.org/wiki/K-means_clustering) clustering to find 3 groups of similar pages.  We will group the pages on 3 metrics (bounces, entrances, and exits).  

I put this function in a script called `cluster.r`.


```r
cluster_tableau = function(.arg1, .arg2, .arg3) {
    # put the tableau data into a dataframe
    df = data.frame(v1 = .arg1, v2 = .arg2, v3 = .arg3)
    # scale the data
    df = scale(df)
    # 3 clusters using kmean
    clus = kmeans(df, 3)
    # return the cluster assignments
    return(as.character(clus$cluster))
}
```



Next we have to call the code from Tableau, which we do using a calculated field.  

![calc](cluster-calc.png)

That's it. Now, you can insert the cluster field into your table to slice and dice the segments.


## Moving on

This post was intended to highlight what's currently possible by calling R from within Tableau.  As mentioned above, this doesn't really make my workflow easier.  In addition, there has been alot of discussion of predictive analytics on the #emchat hashtag.  We ***could*** fit models to our Tableau data with R.  That said, the modeling process is much easier in R.  Hopefully in the near future, Tableau will release a package that let's us do our advanced analysis in R, and then save Tableau datasets for visualization and reporting.



