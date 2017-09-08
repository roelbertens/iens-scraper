# Iens scraper

The goal of this project is to learn how to run a Python Scrapy script from Docker on Google Cloud.

### To-do

* Remove bugs in parsing some restaurants with really sparse address information. See errors in `output/scrapy_errors_20170811`
* Make sure container in google container engine can write to bigquery
* Scraping comments: include dates, grades, number of reviews, name of reviewer

### Folder structure

* In the main project folder we have all setup files (like Dockerfile, entrypoint.sh and requirements.txt)
* In the directory `data` there are some sample data sets, and the json schema required to upload to Google BigQuery
* The directory `iens_scraper` contains:
    * the `spider` folder set up by Scrapy with the crawler in `iens_spider.py`
    * Other required code (nothing necessary yet)

### About Scrapy

Use below code to call the spider named `iens`:
```
scrapy crawl iens -a placename=amsterdam -o output/iens.jsonlines -s LOG_FILE=output/scrapy.log
```
The following arguments can be set:
* `-a placename` to choose the city name for the restaurants to be scraped. Argument is passed on to the spider class
* `-o` for the location of the output file. Use file extention `.jsonlines` instead of `.json` for input Google BigQuery
* `-s LOG_FILE` to save scrapy log to file for error checking
* In `Settings.py` set `LOG_LEVEL = 'WARNING'` to only print error messages of warning or higher


### Docker

Notes:
* Docker is eigenlijk een overkill voor deze toepassing, maar wordt gebruikt om te leren omgaan met Docker
* Een simpele conda environment zou met een script scheduler goed genoeg zijn voor deze toepassing

To set-up the container:
* navigate to project folder
* build image with: `docker build -t iens_scraper .`
    - note: image won't build with 3.6.1-slim base image. Misses certain packages to set up Scrapy
    - RUN vs. CMD: RUN wordt uitgevoerd bij bouwen van de image. CMD bij het bouwen van de container
    - Note: there can only be one CMD instruction in a Dockerfile. If you list more than one CMD then only the last CMD will take effect.
* it's good practice to use ENTRYPOINT in your Dockerfile when you want to execute a shellscript when run
    - no need to use CMD in that case > set it to `CMD["--help"]`
    - Note!!: initially I build entrypoint.sh in Windows, but it has a different line seperator causing Linux to crash. Set line seperator to LF!
    - Make sure to set the permissions of `entrypoint.sh` to executable with `chmod`
* create container and bash into it to check if it was set up correctly: `docker run -it --rm --name iens_container iens_scraper bash`
    - check if folders are what you expect
    - check if scraper works with: `scrapy crawl iens -a placename=amsterdam -o output/iens.jsonlines`
    - Be sure to uncomment `ENTRYPOINT ["./entrypoint.sh"]` in the Dockerfile as otherwise this will run before you can 
    bash into the container

To spin up a container named `iens_container` after you have created the image `iens_scraper` do:
```
docker run --rm --name iens_container -v /tmp:/app/dockeroutput iens_scraper
```
Within this command `-v` does a volume mount to a local folder to store the data. Note that we don't call the volume
mount within the script as the path is system-dependent and thus isn't known in advance.

### Google storage options

Based on the following [decision tree](https://cloud.google.com/storage-options/) Google recommends us to use BigQuery.

However, we need to be sure that BigQuery [supports](https://cloud.google.com/bigquery/data-formats) the JSON format the
data is in after scraping. That seems to be possible with nested JSON (which is the case here), so let's give it a try.

### Google BigQuery


Follow quickstart command line [tutorial](https://cloud.google.com/bigquery/quickstart-command-line) to get up to speed 
on how to query BigQuery. For example use `bq ls` to list all data sets within your default project. 

To [upload a nested json](https://cloud.google.com/bigquery/loading-data#loading_nested_and_repeated_json_data) you need
a schema of the json file. A simple online editor could be used for the basis (for example [jsonschema.net]()), but we 
needed to do some manual editing on top of that to get it into the schema required by BigQuery. Also, it turns out that 
BigQuery doesn't like JSON as a list, so make sure you use `.jsonlines` as output file extension from your sraper. 
Check out the schema and sample data in the `data` folder. To upload the table do:

```
bq load --source_format=NEWLINE_DELIMITED_JSON --schema=iens_schema.json iens.iens_sample iens_sample.jsonlines
```

After uploading, the data can now be queried from the command line. For example, for the `data/iens_sample` table, the 
following query will return all restaurant names with a `Romantisch` tag:

```
bq query "SELECT info.name FROM iens.iens_sample WHERE tags CONTAINS 'Romantisch'"
```  

To clean up and avoid charges to your account, remove all tables within the `iens` dataset with `bq rm -r iens`.

**Google BigQuery from Python** 

Initially, the idea was to upload the scraper output to BigQuery from Python. However, it is not entirely clear how to 
add the table schema to the Python API, to avoid creating a new schema using [SchemaField](https://github.com/GoogleCloudPlatform/google-cloud-python/tree/master/bigquery). The following python code 
on [github](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/master/bigquery/api/load_data_from_csv.py) 
uses `job_data` to add a json schema to the job, but it seemed easier to just add the above command line option to 
`entrypoint.sh`.
 
Here is some nice [documentation](https://cloud.google.com/bigquery/create-simple-app-api#bigquery-simple-app-build-service-python)
in case you do want to work with BigQuery from Python.

### Google Cloud container registry

Follow the following [tutorial](https://cloud.google.com/container-registry/docs/pushing-and-pulling?hl=en_US) on how to 
push and pull to the Google Container Registry.

To tag and push your image to the container registry do:
```bash
docker tag iens_scraper eu.gcr.io/${PROJECT_ID}/iens_scraper:v1
gcloud docker -- push eu.gcr.io/${PROJECT_ID}/iens_scraper
```
You should now be able to see the image in the container registry.

### Google Cloud container engine

To run an application you need a container cluster from the Google Container Engine. Follow [this tutorial](https://cloud.google.com/container-engine/docs/tutorials/hello-app)
and spin up a cluster from the command line with:
```bash
gcloud container clusters create iens-cluster --num-nodes=1
```
By default Google deploys machines with 1 core and 3.75GB. 

Run the following command to deploy your application, and check that it is running with `kubectl get pods`:
```bash
kubectl run iens-deploy --image=eu.gcr.io/gdd-friday/iens_scraper_gc:v1
```
As the deployed container starts with scraping, it is not immediately clear if it is working. Therefore, we can create
another version of the container and apply a rolling update to test specific components. For example:
- Build a new image where you uncomment the scraping command in `entrypoint.sh` and where you use the dummy data 
`iens_sample.jsonlines`, to test whether the container cluster is able to upload that sample file to BigQuery at all.
>>>> currently crashes as container environment doesn't know bq command. Use google cloud sdk container as base unit? 
https://hub.docker.com/r/google/cloud-sdk/ check wat data science project guys deden



To do: give cluster access to BigQuery! (can be done by clicking in UI, but how in command line?)
https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances?hl=en_US&_ga=2.119878856.-1556116188.1504874983