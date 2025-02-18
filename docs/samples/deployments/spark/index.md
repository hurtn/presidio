# Anonymize PII using Presidio on Spark

You can leverage presidio to perform data anonymization as part of spark notebooks.

The following samples cover uses in [Azure Databricks](https://docs.microsoft.com/en-us/azure/databricks/) or [Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-overview) and simple text files hosted on [Azure Blob Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/). However, it can easily be changed to fit any other scenario which requires PII analysis or anonymization as part of spark jobs.

**Note** that this code works for either:
Databricks runtime 8.1 (Spark 3.1.1) and the libraries described [here](https://docs.microsoft.com/en-us/azure/databricks/release-notes/runtime/8.1)

Synapse Analytics (Spark 3.2) and the libraries described [here](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-32-runtime)

## The basics of working with Presidio in Spark - single column analysis and anonymization

A typical use case of Presidio in Spark is transforming a text column in a data frame, by anonymizing its content. The following code sample, a part of [transform presidio notebook](./notebooks/01_transform_presidio.py), is the basis of the e2e sample which uses Azure Databricks or Synapse Analytics as the Spark environment.

```python
anonymized_column = "value" # name of column to anonymize
analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

# broadcast the engines to the cluster nodes
broadcasted_analyzer = sc.broadcast(analyzer)
broadcasted_anonymizer = sc.broadcast(anonymizer)

# define a pandas UDF function and a series function over it.
def anonymize_text(text: str) -> str:
    analyzer = broadcasted_analyzer.value
    anonymizer = broadcasted_anonymizer.value
    analyzer_results = analyzer.analyze(text=text, language="en")
    anonymized_results = anonymizer.anonymize(
        text=text,
        analyzer_results=analyzer_results,
        operators={
            "DEFAULT": OperatorConfig("replace", {"new_value": "<ANONYMIZED>"})
        },
    )
    return anonymized_results.text


def anonymize_series(s: pd.Series) -> pd.Series:
    return s.apply(anonymize_text)


# define a the function as pandas UDF
anonymize = pandas_udf(anonymize_series, returnType=StringType())

# apply the udf
anonymized_df = input_df.withColumn(
    anonymized_column, anonymize(col(anonymized_column))
)

```

## Presidio in Spark - known column(s) with PII (overriding analysis) and anonymization
In this example we supply the list of columns to anonymise the entire column as opposed to only words which contain PII data. For this reason there is no need to call the analyzer for PII detection
```python
columnstoanonymize = ['first_name','last_name','email','city']
anonymizer = AnonymizerEngine()

# define a pandas UDF function and a series function over it.
def anonymize_text(text: str) -> str:
    #no need for analysis as we have specified the columns to anonymize, therefore overriding the analyzer results 
    #analyzer_results = analyzer.analyze(text=text, language="en")
 
    #call the anonymize function with dummy analyzer_results param
    if text:
        anonymized_results = anonymizer.anonymize(
            text=text,
            analyzer_results=[RecognizerResult('DEFAULT', 0, len(text), 0.85)],
            operators={
                "DEFAULT": OperatorConfig("mask", {"masking_char": "*", "chars_to_mask": 4, "from_end": True})
            },
        )
        return anonymized_results.text
    else:
        return text

def anonymize_series(s: pd.Series) -> pd.Series:
    return s.apply(anonymize_text)


# define a the function as pandas UDF
anonymize = pandas_udf(anonymize_series, returnType=StringType())

#apply the udf

for col_name in df.columns:
    for columntoanonymize in columnstoanonymize:
        
        if col_name == columntoanonymize:
            df = df.withColumn(
                col_name, anonymize(col(col_name)))

display(df)
```


## Presidio in Spark - multiple column batch analysis and anonymization
The same logic as above for anonymisation however in this example we use the batch analysis technique to determine which columns contain PII data using a sample of the dataframe
```python
# take a sample for detection/analysis
detectionsample = 10

# define the categories of data you want to anonymize
pii_categories = 'PERSON EMAIL_ADDRESS LOCATION'

# limit the rows in the dataframe for sampling purposes
dfsample= df.limit(detectionsample).toPandas()

# DataFrame to dict
df_dict = dfsample.to_dict(orient="list")

# initialise the analyzer engine and analyze the sample for PII
analyzer = AnalyzerEngine()
batch_analyzer = BatchAnalyzerEngine(analyzer_engine=analyzer)

analyzer_results = batch_analyzer.analyze_dict(df_dict, language="en")
analyzer_results = list(analyzer_results)

columnstoanonymize = []

for analyzerresult in analyzer_results:
    for recognizerresult in analyzerresult.recognizer_results:
        for result in recognizerresult:
            if result and isinstance(result,RecognizerResult):

                if str(result.entity_type) in pii_categories:
                    if result.score>0.8 and analyzerresult.key not in columnstoanonymize:
                      columnstoanonymize.append(analyzerresult.key)
                    break

anonymizer = AnonymizerEngine()

# define a pandas UDF function and a series function over it.
def anonymize_text(text: str) -> str:
    #no need for analysis as we have already predetermined the columns to anonymize, therefore overriding the analyzer results 
    #analyzer_results = analyzer.analyze(text=text, language="en")
 
    #call the anonymize function with dummy analyzer_results param
    if text:
        anonymized_results = anonymizer.anonymize(
            text=text,
            analyzer_results=[RecognizerResult('DEFAULT', 0, len(text), 0.85)],
            operators={
                "DEFAULT": OperatorConfig("mask", {"masking_char": "*", "chars_to_mask": 4, "from_end": True})
            },
        )
        return anonymized_results.text
    else:
        return text

def anonymize_series(s: pd.Series) -> pd.Series:
    return s.apply(anonymize_text)


# define a the function as pandas UDF
anonymize = pandas_udf(anonymize_series, returnType=StringType())

#apply the udf

for col_name in df.columns:
    for columntoanonymize in columnstoanonymize:
        
        if col_name == columntoanonymize:
            df = df.withColumn(
                col_name, anonymize(col(col_name)))

display(df)
```

## Presidio in Spark - custom anonmization pattern
```python
@F.udf(returnType=StringType())
def anonymizeText(text):
    # Check if text null or empty
    if not text or len(text) <= 0:
        return text

    # Get Presidio Anonymizer and Analzer
    anonymizer = AnonymizerMagic.get()

    def get_placeholder(operator: str, item: str)-> str:

        # Create hash
        item_hash = hashlib.sha1(item.encode("UTF-8")).hexdigest()
        chars_hash = ''.join([i for i in item_hash if not i.isdigit()])
        lower_hash = chars_hash.lower()+ chars_hash.lower()+ chars_hash.lower()
        upper_hash = chars_hash.upper()+chars_hash.upper()+chars_hash.upper()

        hashtable = str.maketrans("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ", lower_hash[:26]+upper_hash[:26])

        return item.translate(hashtable)


    # Anonymize Text
    try:
        anonymizer_result = anonymizer.anonymize(
            text=text,
            analyzer_results=[RecognizerResult('DEFAULT', 0, len(text), 0.85)],
            operators={
                "DEFAULT": OperatorConfig("custom", {"lambda": lambda x: get_placeholder("DEFAULT", x)})
            },
        )
        return anonymizer_result.text
    except:
        return "Exception"


columnstoanonymize = ['first_name','last_name','email','city']
for col_name in df.columns:
    for columntoanonymize in columnstoanonymize:
       
        if col_name == columntoanonymize:
            df = df.withColumn(
                col_name, anonymizeText(F.col(col_name)))

display(df)
```

## Synapse
### Pre-requisites

If you do not have an instance of Azure Synapse, either use [the following guide](https://learn.microsoft.com/en-us/azure/synapse-analytics/quickstart-deployment-template-workspaces) to provision a workspace or script below.

### Deploy Infrastructure

Provision the Azure Synapse workspace by running the following script.

``` bash
export RESOURCE_GROUP=[resource group name]
export STORAGE_ACCOUNT_NAME=[storage account name]
export STORAGE_CONTAINER_NAME=[blob container name]
export DATABRICKS_WORKSPACE_NAME=[databricks workspace name]
export DATABRICKS_SKU=[basic/standard/premium]
export LOCATION=[location]

# Create the resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Use ARM template to build the resources and get back the workspace URL

```

### Configure Synpase workspace packages only for Data Exfiltration (DEP) Enabled workspaces

The following script will download the required packages to run Presidio on Synapse. Where a workspace has been Data Exfiltration Protection (DEP) enabled it will not allow Synapse to connect to external data sources or common Python public repositories like Python Package Index (PyPI).
This script must be run in an Ubuntu 18.04+ environment with network connectivity to the workspace. 
For windows users, the easiest method is to install the Ubuntu terminal environment app through the Microsoft Store, # otherwise utilise an Ubuntu VM in Azure

``` bash
sh ./scripts/configure_synapse.sh
```

 ### Create a Synapse Spark pool

Follow [this guide](https://learn.microsoft.com/en-us/azure/synapse-analytics/quickstart-create-apache-spark-pool-portal#create-new-apache-spark-pool) to create a Spark pool but ensure to select Spark 3.2 in the additional settings tab. For the purposes of this demonstration choose a small cluster with 3 nodes and autoscaling disabled.  

#### Configure permissions to the storage account

Ensure that the user running the notebook has been assiged Storage Blob Data Reader to the storage account. 

#### Add the workspace packages to the Spark pool

For workspaces without DEP enabled [follow this guide](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/synapse-spark-encryption-decryption-and-data-masking/ba-p/3615094) to install the required packages.

For DEP enabled workspaces add all of the workspace packages uploaded by the script above to the spark pool.  


#### Upload presidio notebooks

coming soon

## Running the sample

### Configure Presidio transformation notebook

Open the provided 01_transform_presidio notebook and attach it to the cluster preisidio_cluster.
Run the first code-cell and note the following parameters on the top end of the notebook (notebook widgets) and set them accordingly

* Input File Format - text (selected).
* Input path - a folder on the container where input files are found.
* Output Folder - a folder on the container where output files will be written to.
* Column to Anonymize - value (selected).

### Run the notebook

Upload a text file to the blob storage input folder, using any preferd method ([Azure Portal](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal), [Azure Storage Explorer](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-storage-explorer), [Azure CLI](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-cli)).

```bash
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME  --container $STORAGE_CONTAINER_NAME --file ./[file name] --name input/[file name]
```

Run the notebook cells, the output should be csv files which contain two columns, the original file name, and the anonymized content of that file.

## Databricks
### Pre-requisites

If you do not have an instance of Azure Databricks, follow through with the following steps to provision and setup the required infrastrucutre.

If you do have a Databricks workspace and a cluster you wish to configure to run Presidio, jump over to the [Configure an existing cluster](#Configure-an-existing-cluster) section.

### Deploy Infrastructure

Provision the Azure resources by running the following script.

``` bash
export RESOURCE_GROUP=[resource group name]
export STORAGE_ACCOUNT_NAME=[storage account name]
export STORAGE_CONTAINER_NAME=[blob container name]
export DATABRICKS_WORKSPACE_NAME=[databricks workspace name]
export DATABRICKS_SKU=[basic/standard/premium]
export LOCATION=[location]

# Create the resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Use ARM template to build the resources and get back the workspace URL
deployment_response=$(az deployment group create -g $RESOURCE_GROUP --template-file ./docs/samples/deployments/spark/arm-template/databricks.json  --parameters location=$LOCATION workspaceName=$DATABRICKS_WORKSPACE_NAME storageAccountName=$STORAGE_ACCOUNT_NAME containerName=$STORAGE_CONTAINER_NAME)

export DATABRICKS_WORKSPACE_URL=$(echo $deployment_response | jq -r ".properties.outputs.workspaceUrl.value")
export DATABRICKS_WORKSPACE_ID=$(echo $deployment_response | jq -r ".properties.outputs.workspaceId.value")

```

### Setup Databricks

The following script will setup a new cluster in the databricks workspace and prepare it to run presidio anonymization jobs.
Once finished, the script will output an access key which you can use when working with databricks cli.

``` bash

sh ./scripts/configure_databricks.sh

```

### Configure an existing cluster

Only follow through with the steps in this section if you have an existing databricks workspace and clsuter you wish to configure to run presidio. If you've followed through with the "Deploy Infrastructure" and "Setup Databricks" sections you do not have to run the script in this section.

#### Set up secret scope and secrets for storage account

Add an Azure Storage account key to secret scope.

``` bash
STORAGE_PRIMARY_KEY=[Primary key of storage account]

databricks secrets create-scope --scope storage_scope --initial-manage-principal users
databricks secrets put --scope storage_scope --key storage_account_access_key --string-value "$STORAGE_PRIMARY_KEY"

```

#### Upload or update cluster init scripts

Presidio libraries are loaded to the cluster on init.
Upload the cluster setup script or add its content to the existing cluster's init script.

```bash
databricks fs cp "./setup/startup.sh" "dbfs:/FileStore/dependencies/startup.sh"

```

Setup the cluster to run the [init script](https://docs.microsoft.com/en-us/azure/databricks/clusters/configure#init-scripts).

#### Upload presidio notebooks

```bash
databricks workspace import_dir "./notebooks" "/notebooks" --overwrite

```

#### Update cluster environment

Add the following [environment variables](https://docs.microsoft.com/en-us/azure/databricks/clusters/configure#environment-variables) to your databricks cluster:

```bash
"STORAGE_MOUNT_NAME": "/mnt/files"
"STORAGE_CONTAINER_NAME": [Blob container name]
"STORAGE_ACCOUNT_NAME": [Storage account name]

```

#### Mount the storage container

Run the notebook 00_setup to mount the storage account to databricks.

## Running the sample

### Configure Presidio transformation notebook

From Databricks workspace, under notebooks folder, open the provided 01_transform_presidio notebook and attach it to the cluster preisidio_cluster.
Run the first code-cell and note the following parameters on the top end of the notebook (notebook widgets) and set them accordingly

* Input File Format - text (selected).
* Input path - a folder on the container where input files are found.
* Output Folder - a folder on the container where output files will be written to.
* Column to Anonymize - value (selected).

### Run the notebook

Upload a text file to the blob storage input folder, using any preferd method ([Azure Portal](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal), [Azure Storage Explorer](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-storage-explorer), [Azure CLI](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-cli)).

```bash
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME  --container $STORAGE_CONTAINER_NAME --file ./[file name] --name input/[file name]
```

Run the notebook cells, the output should be csv files which contain two columns, the original file name, and the anonymized content of that file.
