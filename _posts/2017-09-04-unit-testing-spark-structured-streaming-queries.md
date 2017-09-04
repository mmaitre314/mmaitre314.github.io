---
layout: post
title: Unit-Testing Spark Structured Streaming queries
comments: true
---

Spark's [Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) offers a powerful platform to process high-volume data streams with low latency.
In Azure we use it to analyze data coming from [Event Hubs](https://github.com/hdinsight/spark-eventhubs) and [Kafka](https://docs.microsoft.com/en-us/azure/hdinsight/hdinsight-apache-kafka-spark-structured-streaming) for instance.

As projects mature and data processing becomes more complex, unit-tests become useful to prevent regressions. This requires mocking the inputs and outputs of the 
Spark queries to isolate them from the network and remove the need for an external Spark cluster. This blog goes through the various pieces needed to make that work.

The full code is available on [GitHub](https://github.com/mmaitre314/SparkStructuredStreamingDemo).
It is written in Java, using [IntelliJ](https://www.jetbrains.com/idea/) as IDE, [Maven](http://search.maven.org/) as package manager, and [JUnit](http://junit.org/) as test framework.

Let's test a simple [stream enrichment](http://blog.madhukaraphatak.com/introduction-to-spark-structured-streaming-part-6/) query. It takes a stream of events as input
and adds human-friendly names to the events by joining with a reference table.

{% highlight Java %}
public static Dataset<Row> setupProcessing(SparkSession spark, Dataset<Row> stream, Dataset<Row> reference) {
  return stream.join(reference, "Id");
}
{% endhighlight %}

Besides running on remote clusters, Spark also supports running on the local machine. Let's use that to create a Spark session in the tests.

{% highlight Java %}
class ProcessingTest {

    private static SparkSession spark;

	@BeforeAll
	public static void setUpClass() throws Exception {
		spark = SparkSession.builder()
			.appName("SparkStructuredStreamingDemo")
			.master("local[2]")
			.getOrCreate();
	}
}
{% endhighlight %}

The unit test follows the standard Arrange/Act/Assert pattern, which here requires creating two mock inputs (one streaming table, one reference static table) and one mock output.

{% highlight Java %}
@Test
void testSetupProcessing() {
    Dataset<Row> stream = createStreamingDataFrame();
    Dataset<Row> reference = createStaticDataFrame();

    stream = Main.setupProcessing(spark, stream, reference);
    List<Row> result = processData(stream);

    assertEquals(3, result.size());
    assertEquals(RowFactory.create(2, 20, "Name2"), result.get(0));
}
{% endhighlight %}

[MemorySink](https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/memory.scala) comes in handy to mock the output, collecting all the output data 
into an `Output` table in memory that is then queried to obtain a `List<Row>`.

{% highlight Java %}
private static List<Row> processData(Dataset<Row> stream) {
    stream.writeStream()
        .format("memory")
        .queryName("Output")
        .outputMode(OutputMode.Append())
        .start()
        .processAllAvailable();

    return spark.sql("select * from Output").collectAsList();
}
{% endhighlight %}

The input static table can be mocked in a couple of different ways. The first one is to read the data from an external CSV file.

{% highlight Java %}
Dataset<Row> reference = spark.read()
    .schema(new StructType()
        .add("Id", DataTypes.IntegerType)
        .add("Name", DataTypes.StringType))
    .csv("data\\input\\reference.csv");
{% endhighlight %}

The second one is to use an in-memory `List<Row>` with values part of the test code itself and to wrap this list in a `DataFrame` (a.k.a. `Dataset<Row>`).

{% highlight Java %}
Dataset<Row> reference = spark.createDataFrame(
    Arrays.asList(
        RowFactory.create(1, "Name1"),
        RowFactory.create(2, "Name2"),
        RowFactory.create(3, "Name3"),
        RowFactory.create(4, "Name4")
    ),
    new StructType()
        .add("Id", DataTypes.IntegerType)
        .add("Name", DataTypes.StringType));
{% endhighlight %}

Likewise for the input streaming table. The first option is to read the data from a folder containing CSV files.

{% highlight Java %}
Dataset<Row> stream = spark.readStream()
    .schema(new StructType()
        .add("Id", DataTypes.IntegerType)
        .add("Count", DataTypes.IntegerType))
    .csv("data\\input\\stream");
{% endhighlight %}

The second one is to use an in-memory `List<String>` that is wrapped in a [MemoryStream](https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/memory.scala),
and converted to `DataFrame`. Ideally this would be a `List<Row>` but setting up an `Encoder` did not look trivial, so the code relies instead on a list of CSV strings that are parsed & cast
using a SQL `SELECT` statement.

{% highlight Java %}
MemoryStream<String> input = new MemoryStream<String>(42, spark.sqlContext(), Encoders.STRING());
input.addData(JavaConversions.asScalaBuffer(Arrays.asList(
    "2,20",
    "3,30",
    "1,10")).toSeq());
Dataset<Row> stream = input.toDF().selectExpr(
    "cast(split(value,'[,]')[0] as int) as Id",
    "cast(split(value,'[,]')[1] as int) as Count");
{% endhighlight %}

After that, it is just a matter of running the test and getting it to green!

<img src="{{ site.url }}/images/2017-09-04-spark-unit-test.png" alt="Test Screenshot" style="width: 200px;"/>

