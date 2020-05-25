# **CacheCheck**

CacheCheck can effectively detect cache-related bugs in Spark applications.
See paper [CacheCheck](http://www.tcse.cn/~wsdou/papers/2020-issta-cachecheck.pdf) to learn more details.

## **1. Run CacheCheck**
#### 1.1 Build CacheCheck
Enter the main directory, and build it by 
```bash
mvn package -DskipTests
``` 
A runnable jar file "core-1.0-SNAPSHOT.jar" is generated under `core/target/`.
#### 1.2 Instrument Spark
The trace collection code (i.e., the modification to Spark) is located in `instrument/`. Specially, the trace collection code starts with the comment "`// Start trace collection in CacheCheck`", and ends with the comment "`// End trace collection in CacheCheck`". 
First, you need to replace `$SPARK_HOME/core/src/main/scala/org/apache/spark/rdd/RDD.scala` with `cachecheck/core/instrument/RDD.scala` and `$SPARK_HOME/core/src/main/scala/org/apache/spark/SparkContext.scala` with `cachecheck/core/instrument/SparkContext.scala`.  

We use Spark-2.4.3 in our experiment. If you want to run on other versions, you'd better to manually add the instrumented code in the comment blocks, since directly replacing files may be incompatiable.

Then, you can build a Spark with the trace collection by running the command in `$SPARK_HOME/`: 
```bash
mvn package -DskipTests
```
#### 1.3 Collect Traces
While the application runs on Spark, the instrumented code can collect traces and store them in `$SPARK_HOME/trace/`.
In our experiment, we use Spark's build-in examples and six word count examples to drive Spark running. 
Taking SparkPi as the example, it can run by the command 
```bash
$SPARK_HOME/bin/run-example SparkPi
```
We provide six word count examples in directory `wordcount`. You can add the whole directory in `SPARK_HOME/examples/src/main/scala/org/apache/spark/examples`, and then compile example module, and run the similar command, such as
```bash
$SPARK_HOME/bin/run-example wordcount.MissingPersist
```  
In our paper, we mainly ran examples in GraphX, MLLib, and Spark SQL. They can also be run by similar command, such as 
```bash
$SPARK_HOME/bin/run-example graphx.ConnectedComponentsExample
```  
Considering there are too many examples to run, we provide some one-click tools for easy configuration and execution. See details in [Code Structure](#code-structure).  
#### 1.4 Perform Detection
The detection is performed by 
```bash
java -jar cachecheck/core/target/core-1.0-SNAPSHOT.jar $TraceDir $AppName [-d]
```
`$TraceDir` is the directory that stores traces, i.e., `$SPARK_HOME/trace/`. `$AppName` is the name of the application, which is usually set in the application code. It is also the file name of the intermediate files got in step 3. For SparkPi, its application name is `Spark Pi`. `-d` is an option about debug mode. In default, CacheCheck delete all intermediate files after detection. If you want keep them, add `-d` please.  
After the detection, a bug report, named `$AppName.report`, is generated  in `$SPARK_HOME/trace/`.
If you add `-d`, the bug report will be `$AppName.report`. It has more information for inspecting the bug.

## **2. Code Structure**
CacheCheck mainly has two modules, i.e., `core` and `tools`. `core` module realizes the algorithms and the approach introduced in our paper. `tools` module provides three tools, i.e., `ExampleRunner`, `CachecheckRunner`, and `Deduplicator` for easy and automatic detection. After [Build Cachecheck](#1-build-cachecheck), three runnable jars are generated under `cachecheck/tools/traget/`. They are `tools-examplerunner.jar`, `tools-cachecheckrunner.jar`, and `tools-deduplicator.jar`.  
#### 2.1 ExampleRunner
ExampleRunner can automatically run Spark's build-in examples. It requires a configuration file, which is a xml file just like `example-list-all.xml` in `cachecheck/tools/resource`. In this file, you can denote which examples to run.  
The execution command is 
```bash
java -jar cachecheck/tools/target/tools-examplerunner.jar $ExampleList $SparkDir
```
`$ExampleList` is the path of the configuration file. `$SparkDir` is the base directory of Spark, e.g., `$SPARK_HOME`.  
#### 2.2 CacheCheckRunner
CacheCheckRunner can automatically analyze all the traces under the same directory and get the bug reports.  
The execution command is 
```bash
java -jar cachecheck/tools/target/tools-cachecheckrunner.jar  $TraceDir
```
`$TraceDir` is the diretocry that traces locate in, e.g., `$SPARK_HOME/trace`.  
#### 2.3 Deduplicator
Deduplicator can collect all the bug reports under the same directory, make deduplication and generate a summary bug report. The command is 
```bash
java -jar cachecheck/tools/target/tools-deduplicator.jar  $ReportDir
``` 
`$ReportDir` is the directory storing bug reports. When finishing, Deduplicator will generate a `summary.report` file under the same directory.
***

## **3. Detected Unknown bugs**
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js">
    window.onload=function(){
        console.log("begin!");
        $.ajax({
            type:"get",
            url:"./statistics",
            data:{},
            async: false,
            success:function (data) {
                //console.log(data);
                var rows = data.split("\n");
                //console.log("data num:" + rows.length);
                for(var i=0;i < rows.length;i++){
                    var x=document.getElementById('data').insertRow();
                    var row = rows[i];
                    var values = row.split("\t");
                    if(values.length > 1) {
                        var cell=x.insertCell();
                        cell.innerHTML = i + 1;
                        for(var j=0;j < values.length;j++){
                            //console.log("value num:" + values.length);
                            var value = values[j];
                            cell=x.insertCell();
                            if(j == 1 && value.indexOf("http") > -1) {
                                //console.log("http index value:" + value.indexOf("http"));
                                    var link, issueName;
                                    if (value.indexOf(",") > 0) {
                                        link = value.split(",")[0];
                                        issueName = value.split(",")[1];
                                    } else {
                                        link = value;
                                        issueName = value.substr(value.lastIndexOf("/")+1);
                                    }
                                    //console.log("link:" + link + " issueName:"+ issueName);
                                    //cell.innerHTML = $("<a href=\"" + link + "\">" + issueName + "</a>").html();
                                    var htmlString = "<a href=\"" + link + "\">" + issueName + "</a>";
                                    console.log("htmlString:"+htmlString);
                                    cell.innerHTML = htmlString;
                            } else {
                                cell.innerHTML=values[j];}
                        }
                    }
                }
           }
        });
        console.log("end!");
    }
</script>
<table id="data">
    <tr>
        <th>ID</th>
        <th>Project</th>
        <th>Issue ID</th>
        <th>Bug Type</th>
        <th>Location</th>
        <th>Related RDD variable</th>
        <th>Status</th>
        <th>Fixed</th>
    </tr>
</table>