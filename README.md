# Stock Pattern Stream
### Analysis service that finds past charts similar to the current stock price chart in real time

Forecasting the flow of financial markets by looking at the price and volume of stocks is called "technical analysis."
The stock's stock price pattern is sometimes repeated, and many people who trade stocks analyze these patterns to predict their stock price.
As shown in the following figure, if the current stock price chart is very similar to a point in time in the past, the trend at that point in time may help predict the current price trend.

<p align="center">
  <img src="img/introduce.png" alt="introduce" width="70%">
</p>

This repositories *" Wouldn't it be convenient if there is a program that finds in real time the section most similar to the current chart in the stock's past stock chart?"*
Is the implementation of Spark Streaming.


## 1. Architecture

### 1.1. Find real-time past patterns

In this process, window sliding is applied to the stock's past stock price chart to create n-sized slides.

Spark Streaming Job receives the "last n salary" of the item, 
Calculate the Pearson correlation coefficient with the pre-generated slides to find the closest slide.

<p align="center">
  <img src="img/job_input_output.png" alt="job_input_output" width="60%">
</p>

The above pattern search is performed in parallel for one item. 
If there are multiple items to be analyzed, this task is performed sequentially for each item.

### 1.2. Full configuration
<p align="center">
  <img src="img/architecture.png" alt="architecture" width="85%">
</p>

-<b>Kafka Producer</b>
  -Producer process that collects item distribution data in real time through Kiwoom OpenAPI and sends it to Kafka
  -The latest n distributions of the items to be analyzed are transmitted.
-<b>Kafka</b>
  -Distribution data storage queue
-<b>Find real-time past patterns</b>
  -Scope of implementation of this repository
  -Real-time analysis of Kafka data input with Spark Streaming Job
  -Save analysis results in FS
-<b>ELK Stack</b>
  -Visualize real-time analysis results using Logstash, Elasticsearch, and Kibana
  -Logstash: Enter saved analysis results into Elasticsearch
  -Kibana: Visualize real-time analysis results on the dashboard


### 1.3. Analysis specifications

-<b>Spark Mode</b>
  -Standalone (Operation without a separate Hadoop cluster)
  -(Hadoop cluster mode will be considered later)
-<b>Streaming Perfomance</b>
  -Streaming analysis of 10 minutes within 1 minute (batchInterval)
  -The default batchInterval is set to 5 minutes
-<b>Analysis data length</b>
  -window slide length n: 59 minutes
  -Past salary data length: 629 salary
  
### 1.4. Input/Output data structure
Currently, the daily salary data is being replaced, and it will be changed to the salary later

|input data| -> |output data|
|:---:|:---:|:---:|
|Past data by item</br>Real-time data by item</br>Item code->Item name mapping table|(*spark streaming job*)|Real-time analysis result|

-(Input) <b>Past salary data by event</b>
  -Input method: Enter the file path with ```--hist-path [file path]`'' among spark-submit arguments
  -Data format: json
    ```
    // {hour1: salary price, hour2: salary price, ..., time 629:salary price, symb:item code}
    {"20160201": 5630,"20160202": 5633,...,"symb":"A123456"},{...},...
    ```
-(Input) <b>Real-time distribution data for each event</b>
  -Input method: kafka consuming (message's value)
  -Data format: json
    ```
    // {hour1: minute price, hour2: minute price, ..., time59: minute price, symb:item code}
    {"20160201": 5630,"20160202": 5633,...,"symb":"A123456"},{...},...
    ```
-(Input) <b>Item code->Item name mapping table</b>
  -Input method: Enter a file path with ```--symb2name-path [file path]`'' among spark-submit arguments
  -Data format: json
    ```
    // [{"symb":item code,"name":item name}, ...]
    [{"symb":"A001720","name":"Shinyoung Securities"},...]
    ```
-(Output) <b>Real-time analysis result</b>
  -Saving method: Save as a file in the value path of ```--output-dir [directory path]`'' among spark-submit arguments
  -Data format: csv
    ```
    // item code, item name, real-time installment time, real-time installment price, past installment time, past installment price, version, correlation number
    A097950, CJ CheilJedang, 20190705,294000.0,20190402,326000.0,20190930,0.949744
    A097950, CJ CheilJedang, 20190708,289000.0,20190403,326000.0,20190930,0.949744
    A097950, CJ CheilJedang, 20190709,284000.0,20190404,325000.0,20190930,0.949744
    ...(Omitted)...
    ```
  -Data visualization example (CJ CheilJedang, similarity 94%)
    -Now: Real-time distribution price according to real-time distribution time
    -Then: Past installment price according to past installment time
    <p align="left">
      <img src="img/result_plot.png" alt="result_plot" width="50%">
    </p>


 
### 1.5. Main source code

-<b>StockPatternStream.scala</b>
  -Main singleton. arguments parsing and analytic function calls
-<b>Preprocessor.scala</b>
  -DataFrame pre-processing
-<b>PatternFinder.scala</b>
  -Correlation calculation using Spark ML
-<b>StreamingManager.scala</b>
  -Spark Streaming Definition
-<b>CommonUtils.scala</b>
  -Utility function collection

## 2. How to use

### 2.1. Prerequisites
|name|version|
|:---|:---|
|Scala|2.11.12|
|SBT|1.3.10|
|JDK|1.8.0|
|Apache Spark|2.4.5|
|spark-streaming-kafka|0.10.0|

-Spark works as standalone

### 2.2. Usage

create fat jar
```
sbt assembly
```

spark submit
```
spark-submit jarfile --hist-path [path1] --symb2name-path [path2] --output-dir [path3]
        --kafka-bootstrap-server [addr] --kafka-group-id [id] --kafka-topic [topic] --batch-interval [seconds]

arguments
 --hist-path: Past salary data path for each event
 --symb2name-path: item code-> item name mapping table path
 --output-dir: path to save analysis results
 --kafka-bootstrap-server: Kafka bootstrap address (localhost:9092)
 --kafka-group-id: Consumer group ID
 --kafka-topic: Topic name
 --batch-interval: Spark Streaming batch interval (default: 300)
```



## 3. Future tasks

-Integration of Spark Streaming and Elasticsearch and removal of Logstash step
-Daily salary data -> Change salary data (replacement of salary with current salary data)
-Single mode -> Cluster switching performance experiment
