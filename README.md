# pyspark
to poc, locally without cluster, a rest api that allows interaction with apache spark and delta lake for read/write

Goal: to poc, locally without cluster, a rest api that allows interaction with apache spark and delta lake for read/write

Installation: ps. on my mac, 
    1. java 8 (java 14 doesn't work so i had to install java 8 with brew cask)
    2. miniconda 3 (installed before)
    3. create conda env 'spark' and install pyspark 3 (required by delta lake) and also jupyter

    To start interactive pyspark shell with delta lake: 
        1. . activate spark 
        2. set 4 env variables: 
            ** even with --packages, it only works for 'pyspark' not 'spark-submit'
            ** when running pyspark (shell) with --package, it actually downloads the jar and put into ~/.ivy2/jars/io.delta_delta-core_2.12-0.8.0.jar

            export SPARK_HOME=/opt/miniconda3/envs/spark/lib/python3.7/site-packages/pyspark
            export PACKAGES="io.delta:delta-core_2.12:0.8.0"
            export PYSPARK_SUBMIT_ARGS="--packages ${PACKAGES} pyspark-shell"
            export PYSPARK_PYTHON=python3

        3. run cmd: 
        pyspark --packages io.delta:delta-core_2.12:0.8.0 \
            --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
            --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog" \
            --master local[2]
        
        4. in interactive shell, you can then create spark session then context

        import pyspark

        spark = pyspark.sql.SparkSession.builder.appName("MyApp") \
            .config("spark.jars.packages", "io.delta:delta-core_2.12:0.8.0") \
            .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
            .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
            .getOrCreate()

        sc = spark.sparkContext

        data = spark.range(0, 5)
        data.write.format("delta").save("/tmp/delta-table")

        df = spark.read.format("delta").load("/tmp/delta-table")
        df.show()

    To start interactive jupyter with pyspark & delta lake
        1. . activate spark 
        2. set 4 env variables above and two more below    
            export PYSPARK_DRIVER_PYTHON="jupyter"
            export PYSPARK_DRIVER_PYTHON_OPTS="notebook"

        3. run cmd: ** it will start jupyter notebook
        pyspark --packages io.delta:delta-core_2.12:0.8.0 \
            --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
            --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog" \
            --master local[2]        

        4. create a python notebook and can reuse code above

    To start flask app with pyspark & delta lake
        1. . activate spark 
        2. set 3 env variables
            export SPARK_HOME=/opt/miniconda3/envs/spark/lib/python3.7/site-packages/pyspark
            export PACKAGES="io.delta:delta-core_2.12:0.8.0"
            export PYSPARK_PYTHON=python3

        3. code in ~/Documents/source/spark/rest.py

            from flask import Flask
            import pyspark

            spark = pyspark.sql.SparkSession.builder.appName("MyApp") \
                .config("spark.jars.packages", "io.delta:delta-core_2.12:0.8.0") \
                .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
                .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
                .getOrCreate()

            app = Flask(__name__)

            @app.route('/read')
            def deltalakeread():
                df = spark.read.format("delta").load("/tmp/delta-table")
                return df.toPandas().to_json()

            if __name__ == '__main__':
                app.run()

        4. run cmd: ** this will require full path to jar downloaded when running pyspark, you can download manually
                    ** also the --jars has to be the first arg before python main file 'rest.py'

        spark-submit --jars ~/.ivy2/jars/io.delta_delta-core_2.12-0.8.0.jar ~/Documents/source/spark/rest.py \
            --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
            --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog" \
            --master local 

        5. bring up browser and type in http://localhost:5000/read to see the data created before


References:
    1. delta lake: https://docs.delta.io/latest/quick-start.html
    2. jupyter with delta lake: https://github.com/pyMixin/DeltaLake/blob/master/Delta%20Lake%20on%20Jupyter%20Notebooks.ipynb
    3. flask: https://realpython.com/flask-by-example-part-1-project-setup/ 
