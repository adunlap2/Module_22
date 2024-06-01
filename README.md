# Home_Sales
![House Sales](https://github.com/adunlap2/Module_22/assets/153474345/25e2ea03-93e4-4355-94e2-80919aff0087)

# Apache Spark Home Sales Analysis

## Overview

This project involves setting up and using Apache Spark to analyze a dataset of home sales. Utilizing Google Colab and PySpark, I analyzed the average price of homes based on various criteria, such as the year built and the number of bedrooms and bathrooms. The analysis includes performance optimizations using Spark's caching mechanisms and data storage in Parquet format.

## Installation and Setup

1. **Install Spark and Java**:
   - Update the package lists.
   - Install OpenJDK 11.
   - Download and extract the specified Spark version (3.5.1).
   - Install the `findspark` library.

    ```bash
    apt-get update
    apt-get install openjdk-11-jdk-headless -qq > /dev/null
    wget -q http://www.apache.org/dist/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz
    tar xf spark-3.5.1-bin-hadoop3.tgz
    pip install -q findspark
    ```

2. **Set Environment Variables**:
   - Set the Java and Spark home directories.

    ```python
    import os
    spark_version = 'spark-3.5.1'
    os.environ['SPARK_VERSION'] = spark_version
    os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-11-openjdk-amd64"
    os.environ["SPARK_HOME"] = f"/content/{spark_version}-bin-hadoop3"
    ```

3. **Initialize Spark**:

    ```python
    import findspark
    findspark.init()
    ```

4. **Create a Spark Session**:

    ```python
    from pyspark.sql import SparkSession
    spark = SparkSession.builder.appName("SparkSQL").getOrCreate()
    ```

## Data Loading and Transformation

1. **Load Data from AWS S3**:

    ```python
    from pyspark import SparkFiles
    url = "https://2u-data-curriculum-team.s3.amazonaws.com/dataviz-classroom/v1.2/22-big-data/home_sales_revised.csv"
    spark.sparkContext.addFile(url)
    df = spark.read.option('header', 'true').csv(SparkFiles.get("home_sales_revised.csv"), inferSchema=True, sep=',', timestampFormat="mm/dd/yy")
    df.createOrReplaceTempView('home_data')
    df.show()
    ```

2. **Queries and Transformations**:

    - **Average price for four-bedroom houses sold per year**:

        ```python
        from pyspark.sql.functions import mean, round

        four_bed_per_year = df.filter(df.bedrooms == 4) \
            .groupBy("date") \
            .agg(round(mean("price"), 2).alias("avg_price"))
        four_bed_per_year.show()
        ```

    - **Average price of homes built per year with 3 bedrooms and 3 bathrooms**:

        ```python
        three_bed_three_bath_per_year = df.filter((df.bedrooms == 3) & (df.bathrooms == 3)) \
            .groupBy("date") \
            .agg(round(mean("price"), 2).alias("avg_price"))
        three_bed_three_bath_per_year.show()
        ```

    - **Average price of homes with specific conditions**:

        ```python
        avg_price_per_year = df.filter(
            (df.bedrooms == 3) &
            (df.bathrooms == 3) &
            (df.floors == 2) &
            (df.sqft_living >= 2000)
        ) \
            .groupBy("date") \
            .agg(round(mean("price"), 2).alias("avg_price"))
        avg_price_per_year.show()
        ```

    - **Average price of a home per "view" rating**:

        ```python
        from pyspark.sql.functions import avg
        import time

        start_time = time.time()

        avg_price_view = df.groupBy("view") \
            .agg(round(avg("price"), 2).alias("avg_price")) \
            .filter("avg_price >= 350000") \
            .orderBy("view", ascending=False)
        avg_price_view.show()

        print("--- %s seconds ---" % (time.time() - start_time))
        ```

## Performance Optimization

1. **Cache the DataFrame**:

    ```python
    spark.catalog.cacheTable("home_data")
    ```

2. **Run Cached Query**:

    ```python
    start_time = time.time()

    avg_price_view_cached = df.groupBy("view") \
        .agg(round(avg("price"), 2).alias("avg_price")) \
        .filter("avg_price >= 350000") \
        .orderBy("view", ascending=False)
    avg_price_view_cached.show()

    print("--- %s seconds ---" % (time.time() - start_time))
    ```

3. **Uncache the DataFrame**:

    ```python
    spark.catalog.uncacheTable("home_data")
    ```

## Parquet Format Operations

1. **Write Data in Parquet Format**:

    ```python
    df.write.partitionBy('date_built').parquet('p_sales', mode='overwrite')
    ```

2. **Read Parquet Data**:

    ```python
    parquet = spark.read.parquet('p_sales')
    parquet.createOrReplaceTempView('p_sales')
    ```

3. **Run Query on Parquet Data**:

    ```python
    start_time = time.time()

    avg_price_view_parquet = parquet.groupBy("view") \
        .agg(round(avg("price"), 2).alias("avg_price")) \
        .filter("avg_price >= 350000") \
        .orderBy("view", ascending=False)
    avg_price_view_parquet.show()

    print("--- %s seconds ---" % (time.time() - start_time))
    ```

## Conclusion

This project demonstrates how to use PySpark for big data processing, including data loading, transformations, and performance optimizations using caching and Parquet format. For further exploration, additional queries and transformations can be added to analyze other aspects of the home sales dataset.
