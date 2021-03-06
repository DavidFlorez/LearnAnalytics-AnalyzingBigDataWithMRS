Scaling and deployment
================
Seth Mottaghinejad
2017-03-08

Once a model is built, we're usually interested in using it to make predictions on future data, a process sometimes referred to as **scoring**. This is not very different from how we used the model in the last section to make predictions on the test data using the `rxPredict` function. But future data may be sitting in a different environment from the one used to develop the model.

For example, let's say we want to regularly score new data as it arrives in a SQL Server. Without `RevoScaleR`, we may need to bring the new data out of the SQL Server (e.g. as a flat file or using an ODBC connection) into a machine with R installed. We would then load the data in R to score it and send the scores back to SQL Server. Moving data around is both inefficient and can often breaks security protocols around data governance. Additionally, if the data we are trying to score is too large to fit in the R session, we would also have to perform this process peace-wise using a chunk of the data each time. Microsoft R Server can eliminate all three problems by scoring new data in the machine the data is already sitting in. It uses the efficient `rxPredict` function which can score data without having to load it in an R session in its entirety.

Because `RevoScaleR` comes with parallel modeling and machine learning algorithms, we can also use them to develop models on much larger datasets sitting in SQL Server or on HDFS. This narrows the gap between the development and production environment considerably.

What is WODA?
-------------

In addition to offering scalable algorithms that run on large datasets, Microsoft R Server offers the ability to deploy those algorithms on multiple platforms with minimal changes to the code structure. This is referred to as WODA, which stands for **write once and deploy anywhere**. WODA is an abstraction layer. It allows the data scientist to develop code locally (on a single machine and using smaller datasets) but deploy it in environments such as Hadoop, Spark or SQL Server *without having to change the code too much and without having to know too much about what goes on under the hood in such environments when the code is deployed*.

More information on WODA can be found [here](https://msdn.microsoft.com/en-us/microsoft-r/scaler-distributed-computing).

Setting up the compute context
------------------------------

The way that `RevoScaleR` achieves WODA is by setting and changing the compute context. The **compute context** refers to the environment in which the computation is happening, which by default is set to the machine hosting the **local** R session (called the **client**), but can be change to a **remote** machine (such a a SQL server or a Hadoop/Spark cluster). At a low level, the same computation will run differently in different compute context, but produce the same results. Whenever we need to perform a computation remotely, we simply change the compute context to the remote environment. `RevoScaleR` functions are aware of the compute context at runtime and when the compute context is set to remote they will perform their computation remotely. This is how we can *take the computation to the data* instead of bringing the data to the computation. Other R functions are not compute-context-aware, however as we will see by using the `rxExec` function we can send any arbitrary R function to execute remotely.

Caveats about WODA
------------------

There are some important caveats about WODA that we need to point out:

-   1.  A few `RevoScaleR` functions only run in a local compute context: `rxSort`, `rxMerge`, `rxSplit` can be used on XDF data and a `data.frame` but they do not work in remote compute contexts like SQL Server or Spark. Fortunately, there are usually very efficient alternatives to these functions in those environments: we can simply use T-SQL to sort, merge, or split data in SQL Server and we can use Hive to do the same with data on HDFS. `rxFactors` is another function that does not work in a remote compute context, but as we saw earlier we can use `rxDataStep` along with the `factor` function to do the same transformation. None of the analytics functions in `RevoScaleR` have this limitation.
-   1.  Remote environment such as SQL Server and Spark can act as development environments (where code is developped from scratch and tested) but more often they act as production environments (where we can deploy mature R code on large datasets hosted in these environments). When we develop code, it is usually much easier to **first work with a sample of the data in a local compute context**, so we can test things and make mistakes without putting too much strain on the environment or having to deal with the complexities of working in a remote environment. Note that *local* here doesn't necessarily mean that we need the data on our laptops. It simply means a single machine. As we will see, both SQL Server and Spark an R instance that can be used to work locally (it is called a **stand-alone R Server** in SQL Server and the **edge node** in Spark). This means we can still keep the data in the confines of the environments but work in a local comute context to develop our R code. Once the code is tested and somewhat mature, WODA gives us a reassurance that without too many changes to the code, we can rerun it in a remote environment.
-   1.  Even if WODA make it possible to take code that we developed in a local compute context and run it in remote production environments, this doesn't mean that the code we put together in development phase should simply be put to production without a second pass at it. There are usually things we can do differently to make the code run more efficiency. Examples of this will be pointed out both the SQL Server and Spark to give users a better idea.
