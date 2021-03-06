---
title: DASE Components Explained (Recommendation)
---

PredictionIO's DASE architecture brings the separation-of-concerns design
principle to predictive engine development. DASE stands for the following
components of an engine:

* **D**ata - includes Data Source and Data Preparator
* **A**lgorithm(s)
* **S**erving
* **E**valuator

Let's look at the code and see how you can customize the Recommendation engine
you built from the Recommendation Engine Template.

INFO: Evaluator will not be covered in this tutorial.

## The Engine Design

As you can see from the Quick Start, *MyRecommendation* takes a JSON prediction
query, e.g. `{ "user": "1", "num": 4 }`, and return a JSON predicted result.
In MyRecommendation/src/main/scala/***Engine.scala***, the `Query` case class
defines the format of such **query**:

```scala
case class Query(
  user: String,
  num: Int
) extends Serializable
```

The `PredictedResult` case class defines the format of **predicted result**,
such as

```json
{"itemScores":[
  {"item":22,"score":4.07},
  {"item":62,"score":4.05},
  {"item":75,"score":4.04},
  {"item":68,"score":3.81}
]}
```

with:

```scala
case class PredictedResult(
  itemScores: Array[ItemScore]
) extends Serializable

case class ItemScore(
  item: String,
  score: Double
) extends Serializable
```

Finally, `RecommendationEngine` is the *Engine Factory* that defines the
components this engine will use: Data Source, Data Preparator, Algorithm(s) and
Serving components.

```scala
object RecommendationEngine extends IEngineFactory {
  def apply() = {
    new Engine(
      classOf[DataSource],
      classOf[Preparator],
      Map("als" -> classOf[ALSAlgorithm]),
      classOf[Serving])
  }
  ...
}
```

### Spark MLlib

Spark's MLlib ALS algorithm takes training data of RDD type, i.e. `RDD[Rating]`
and train a model, which is a `MatrixFactorizationModel` object.

PredictionIO Recommendation Engine Template, which
*MyRecommendation* bases on, integrates this algorithm under the DASE
architecture. We will take a closer look at the DASE code below.

INFO: [Check this
out](https://spark.apache.org/docs/latest/mllib-collaborative-filtering.html) to
learn more about MLlib's ALS collaborative filtering algorithm.


## Data

In the DASE architecture, data is prepared by 2 components sequentially: *Data
Source* and *Data Preparator*. *Data Source* and *Data Preparator* takes data
from the data store and prepares `RDD[Rating]` for the ALS algorithm.

### Data Source

In MyRecommendation/src/main/scala/***DataSource.scala***, the `readTraining`
method of class `DataSource` reads, and selects, data from the *Event Store*
(data store of the *Event Server*) and returns `TrainingData`.

```scala
case class DataSourceParams(appId: Int) extends Params

class DataSource(val dsp: DataSourceParams)
  extends PDataSource[TrainingData,
      EmptyEvaluationInfo, Query, EmptyActualResult] {

  @transient lazy val logger = Logger[this.type]

  override
  def readTraining(sc: SparkContext): TrainingData = {
    val eventsDb = Storage.getPEvents()
    val eventsRDD: RDD[Event] = eventsDb.find(
      appId = dsp.appId,
      entityType = Some("user"),
      eventNames = Some(List("rate", "buy")), // read "rate" and "buy" event
      // targetEntityType is optional field of an event.
      targetEntityType = Some(Some("item")))(sc)

    val ratingsRDD: RDD[Rating] = eventsRDD.map { event =>
      val rating = try {
        val ratingValue: Double = event.event match {
          case "rate" => event.properties.get[Double]("rating")
          case "buy" => 4.0 // map buy event to rating value of 4
          case _ => throw new Exception(s"Unexpected event ${event} is read.")
        }
        // entityId and targetEntityId is String
        Rating(event.entityId,
          event.targetEntityId.get,
          ratingValue)
      } catch {
        case e: Exception => {
          logger.error(s"Cannot convert ${event} to Rating. Exception: ${e}.")
          throw e
        }
      }
      rating
    }
    new TrainingData(ratingsRDD)
  }
}
```

`Storage.getPEvents()` returns a data access object which you could use to
access data that is collected by PredictionIO *Event Server*.
`eventsDb.find(...)` specifies the events that you want to read. PredictionIO
automatically loads the parameters of *datasource* specified in
MyRecommendation/***engine.json***, including *appId*, to `dsp`.

In ***engine.json***:

```
{
  ...
  "datasource": {
    "params" : {
      "appId": 1
    }
  },
  ...
}
```

Each *rate* and *buy* user event data is read as `Rating`.
For flexibility, this Recommendation engine template is designed to support user ID and item ID in `String`.
Since Spark MLlib's `Rating` class assumes `Int`-only user ID and item ID, you have to define a new `Rating` class:  

```scala
case class Rating(
  user: String,
  item: String,
  rating: Double
)
```

`TrainingData` contains an RDD of all these `Rating` events. The class definition of `TrainingData` is:

```scala
class TrainingData(
  val ratings: RDD[Rating]
) extends Serializable {...}
```
and PredictionIO passes the returned `TrainingData` object to *Data Preparator*.

<!-- TODO
> HOW-TO:
>
> You may modify readTraining function to read from other datastores, such as MongoDB -  [link]
-->

INFO: You could [modify the DataSource to read custom events](reading-custom-events.html) other than the default **rate** and **buy**.

### Data Preparator

In MyRecommendation/src/main/scala/***Preparator.scala***, the `prepare` method
of class `Preparator` takes `TrainingData` as its input and performs any
necessary feature selection and data processing tasks. At the end, it returns
`PreparedData` which should contain the data *Algorithm* needs. For MLlib ALS,
it is `RDD[Rating]`.

By default, `prepare` simply copies the unprocessed `TrainingData` data to `PreparedData`:

```scala
class Preparator
  extends PPreparator[TrainingData, PreparedData] {

  def prepare(sc: SparkContext, trainingData: TrainingData): PreparedData = {
    new PreparedData(ratings = trainingData.ratings)
  }
}

class PreparedData(
  val ratings: RDD[Rating]
) extends Serializable
```

PredictionIO passes the returned `PreparedData` object to Algorithm's `train` function.

<!-- TODO
> HOW-TO:
>
> MLlib ALS limitation: user id, item id must be integer - convert [link]
-->

## Algorithm

In MyRecommendation/src/main/scala/***ALSAlgorithm.scala***, the two methods of
the algorithm class are `train` and `predict`. `train` is responsible for
training a predictive model. PredictionIO will store this model and `predict` is
responsible for using this model to make prediction.

### train(...)

`train` is called when you run **pio train**. This is where MLlib ALS algorithm,
i.e. `ALS.train`, is used to train a predictive model.


```scala
  def train(data: PreparedData): ALSModel = {
    ...
    // Convert user and item String IDs to Int index for MLlib
    val userStringIntMap = BiMap.stringInt(data.ratings.map(_.user))
    val itemStringIntMap = BiMap.stringInt(data.ratings.map(_.item))
    val mllibRatings = data.ratings.map( r =>
      // MLlibRating requires integer index for user and item
      MLlibRating(userStringIntMap(r.user), itemStringIntMap(r.item), r.rating)
    )

    // seed for MLlib ALS
    val seed = ap.seed.getOrElse(System.nanoTime)

    // If you only have one type of implicit event (Eg. "view" event only),
    // replace ALS.train(...) with
    //val m = ALS.trainImplicit(
      //ratings = mllibRatings,
      //rank = ap.rank,
      //iterations = ap.numIterations,
      //lambda = ap.lambda,
      //blocks = -1,
      //alpha = 1.0,
      //seed = seed)

    val m = ALS.train(
      ratings = mllibRatings,
      rank = ap.rank,
      iterations = ap.numIterations,
      lambda = ap.lambda,
      blocks = -1,
      seed = seed)

    new ALSModel(
      rank = m.rank,
      userFeatures = m.userFeatures,
      productFeatures = m.productFeatures,
      userStringIntMap = userStringIntMap,
      itemStringIntMap = itemStringIntMap)
  }
```

#### Working with Spark MLlib's ALS.train(....)

As mentioned above, MLlib's `Rating` does not support `String` user ID and item ID.
Its `ALS.train` thus also assumes `Int`-only `Rating`.

Here you need to map your String-supported `Rating` to MLlib's Integer-only `Rating`.
First, you can rename MLlib's Integer-only `Rating` to `MLlibRating` for clarity:

```
import org.apache.spark.mllib.recommendation.{Rating => MLlibRating}
```

You then create a bi-directional map with `BiMap.stringInt` which maps each String record to an Integer index.

```
val userStringIntMap = BiMap.stringInt(data.ratings.map(_.user))
val itemStringIntMap = BiMap.stringInt(data.ratings.map(_.item))
```
Finally, you re-create each `Rating` event as `MLlibRating`:

```
MLlibRating(userStringIntMap(r.user), itemStringIntMap(r.item), r.rating)
```


In addition to `RDD[MLlibRating]`, `ALS.train` takes the following parameters: *rank*, *iterations*, *lambda* and *seed*.

The values of these parameters are specified in *algorithms* of
MyRecommendation/***engine.json***:

```
{
  ...
  "algorithms": [
    {
      "name": "als",
      "params": {
        "rank": 10,
        "numIterations": 20,
        "lambda": 0.01,
        "seed": 3
      }
    }
  ]
  ...
}
```

PredictionIO will automatically loads these values into the constructor `ap`,
which has a corresponding case case `ALSAlgorithmParams`:

```scala
case class ALSAlgorithmParams(
  rank: Int,
  numIterations: Int,
  lambda: Double,
  seed: Option[Long]) extends Params
```

The `seed` parameter is an optional parameter, which is used by MLlib ALS algorithm internally to generate random values. If the `seed` is not specified, current system time would be used and hence each train may produce different reuslts. Specify a fixed value for the `seed` if you want to have deterministic result (For example, when you are testing).

`ALS.train` then returns a `MatrixFactorizationModel` model which contains RDD
data. RDD is a distributed collection of items which *does not* persist. To
store the model, you convert the model to `ALSModel` class at the end.
`ALSModel` is a persistable class that extends `MatrixFactorizationModel`.

> The detailed implementation can be found at
MyRecommendation/src/main/scala/***ALSModel.scala***


PredictionIO will automatically store the returned model, i.e. `ALSModel` in this case.


### predict(...)

`predict` is called when you send a JSON query to
http://localhost:8000/queries.json. PredictionIO converts the query, such as `{
"user": "1", "num": 4 }` to the `Query` class you defined previously.

The predictive model `MatrixFactorizationModel` of MLlib ALS, which is now
extended as `ALSModel`, offers a method called
`recommendProducts`. `recommendProducts` takes two parameters: user id (i.e.
the `Int` index of `query.user`) and the number of items to be returned (i.e. `query.num`). It
predicts the top *num* of items a user will like.

```scala
  def predict(model: ALSModel, query: Query): PredictedResult = {
    // Convert String ID to Int index for Mllib
    model.userStringIntMap.get(query.user).map { userInt =>
      // create inverse view of itemStringIntMap
      val itemIntStringMap = model.itemStringIntMap.inverse
      // recommendProducts() returns Array[MLlibRating], which uses item Int
      // index. Convert it to String ID for returning PredictedResult
      val itemScores = model.recommendProducts(userInt, query.num)
        .map (r => ItemScore(itemIntStringMap(r.product), r.rating))
      new PredictedResult(itemScores)
    }.getOrElse{
      logger.info(s"No prediction for unknown user ${query.user}.")
      new PredictedResult(Array.empty)
    }
  }
```

Note that `recommendProducts` returns the `Int` indices of items. You map them back to `String` with `itemIntStringMap` before they are returned.

> You have defined the class `PredictedResult` earlier.

PredictionIO passes the returned `PredictedResult` object to *Serving*.

## Serving

The `serve` method of class `Serving` processes predicted result. It is also
responsible for combining multiple predicted results into one if you have more
than one predictive model. *Serving* then returns the final predicted result.
PredictionIO will convert it to a JSON response automatically.

In MyRecommendation/src/main/scala/***Serving.scala***,

```scala
class Serving
  extends LServing[Query, PredictedResult] {

  override
  def serve(query: Query,
    predictedResults: Seq[PredictedResult]): PredictedResult = {
    predictedResults.head
  }
}
```

When you send a JSON query to http://localhost:8000/queries.json,
`PredictedResult` from all models will be passed to `serve` as a sequence, i.e.
`Seq[PredictedResult]`.

> An engine can train multiple models if you specify more than one Algorithm
component in `object RecommendationEngine` inside ***Engine.scala***. Since only
one `ALSAlgorithm` is implemented by default, this `Seq` contains one element.


Now you should have a good understanding of the DASE model. We will show you an
example of customizing the Data Preparator to exclude certain items from your
training set.

#### [Next: Reading Custom Events](reading-custom-events.html)
