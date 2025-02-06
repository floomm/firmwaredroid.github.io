---
title: Adding your own static analyzer
author: tom
date: 2024-07-08 13:37:00 +0800
categories: [Tutorial, Static-Analysis]
tags: [getting started, static-analysis]
toc: true
---

In this tutorial, we will show you how to add a new static analyzer (python based) to FMD and scan a couple of apk 
files. For this 
tutorial we will use the `apk_scanner-worker` docker container which is already part of FMD. The `apk_scanner-worker` 
is a container including a couple of static analyzers for Android applications (.apk files). The container is 
responsible for scanning the apk files and extracting the information from the apk files. 
The extracted information is then stored in the FMD database and can be used for further analysis. 


### Overview
Adding a new static analyzer to FMD will need the following steps:
1. **Add the dependencies**: The static analyzer will run in a docker container. Thus, we need to install
the analyzer and its dependencies within the container. 
2. **Create a database model**: The database model will be responsible for storing the information extracted by the
static analyzer.
3. **Create a wrapper script**: The wrapper script will be responsible for running the static analyzer and
extracting the information from the apk files.
4. **Create a GraphQL API endpoint**: The API endpoint will be responsible for triggering the static analyzer.
5. **Test the static analyzer**: Finally, we will test the static analyzer by scanning a couple of apk files.

Next we will go through each step in detail.
   
#### Step 1: Add the dependencies
The `apk_scanner-worker` container ensures that every static analyzer runs in it own python environment 
to avoid conflicts between different versions of libraries. Thus, it is possible to install 
several python packages with conflicting dependencies. For other programming languages, it is recommended to use 
a similar approach to keep dependencies where possible separated.

To install new dependencies in the `apk_scanner-worker` container, you need to modify the `Dockerfile` located in
`docker/base/`(see [Dockerfile_apk_scanner](https://github.com/FirmwareDroid/FirmwareDroid/blob/main/docker/base/Dockerfile_apk_scanner)). 
The `Dockerfile_apk_scanner` is responsible for building the docker image of the 
`apk_scanner-worker` container. The `apk_scanner-worker` container is based on the `firmwaredroid-base` image 
(see [Dockerfile_BASE](https://github.com/FirmwareDroid/FirmwareDroid/blob/main/Dockerfile_BASE)) and
one of the latest `openjdk:XX-jdk-slim-bullseye` images. Therefore, the python and Java runtimes are 
already installed in the container.

There are three ways to install new dependencies:
1. **Using pip**: You can install new python packages using pip. For example, to install the `requests` package, you 
can add the required packages to the `requirements_apk_scanner.txt` file. This package will be installed during the
build process of the docker image.

2. **Using apt**: You can install new packages using apt. For example, to install the `curl` package, you can add the
following line to the `Dockerfile_apk_scanner` file:
   ```
   RUN apt-get update && apt-get install -y curl
   ```
   
3. **Using the setup_apk_scanner.py script**: The script is located in the `docker/base/` folder. 
The script installs a number of python requirements from the `requirements` folder. The key difference to the 
installation with pip is, that for every requirement file a new python environment is created to keep the dependencies
separated. 
To install a new package, you need to create a new `requirements_YOUR_ANALYZER.txt` file in the `requirements` 
folder. Then, you need to reference the new file in the `setup_apk_scanner.py` script by adjusting the following line
with the name of your new requirement file:
   ```
    PYTHON_SCANNERS = ["androguard",
                   "androwarn",
                   "apkid",
                   "apkleaks",
                   "exodus",
                   "qark",
                   "quark_engine",
                   "virustotal",
                   "manifest_parser",
                   "YOUR_ANALYZER"]
    ```
The script will automatically concatenate the string `requirments_` with the name of the analyzer and the string `.txt`.
During build time, the script will create a new python environment for the new analyzer and install the packages 
specified in the `requirements_YOUR_ANALYZER.txt` file.

After you have added the dependencies, you need to rebuild the `apk_scanner-worker` container. We recommend to
clean the docker images before rebuilding the container. You can do this by running the following command:
```
docker container prune -f && docker image prune -f && docker builder prune -f
```
Then, you can rebuild all the containers by running the following command:
```
./docker/build_docker_images.sh
```


#### Step 2: Create a database model
After you have installed the dependencies, you need to create a database model to store the information extracted by the
static analyzer. The database model is a python class that inherits from mongoengine's `Document` class. The class 
defines the structure of the document that will be stored in the database and all database models are located in the
`model` folder (see ["/source/model"](https://github.com/FirmwareDroid/FirmwareDroid/tree/main/source/model)). 
For Apk scanners, the new model should inherit from the `ApkScannerReport` class 
(see [ApkScannerReport](https://github.com/FirmwareDroid/FirmwareDroid/blob/main/source/model/ApkScannerReport.py)).

Example of the database model for `ApkScannerReport` class:
```
class ApkScannerReport(Document):
    meta = {'allow_inheritance': True}
    report_date = DateTimeField(required=True, default=datetime.datetime.now)
    android_app_id_reference = LazyReferenceField(AndroidApp, reverse_delete_rule=CASCADE, required=True)
    scanner_version = StringField(required=True)
    scanner_name = StringField(required=True)
```

The new model should inherit from the `ApkScannerReport` class and define the fields that are required to store the
information extracted by the static analyzer. We create a new class `YourAnalyzerReport.py` that inherits from the 
`ApkScannerReport` class and define the fields that are required to store the information extracted by the static 
analyzer in the `model` folder.

```
from mongoengine import StringField, DictField
class YourAnalyzerReport(ApkScannerReport):
    # Define the fields that are required to store the information extracted by the static analyzer
    some_static_result = StringField(required=True)
    some_dynamic_result = DictField(required=True)
```

Depending on the information extracted by the static analyzer, you can define different static or dynamic schema fields.
Let's assume your scanner stores the scanning results as JSON file. In this case, you can use the `DictField` to store the
complete JSON file in the database. In case the scanner is update at a later point, you don't need to update the database
model as the json file can be stored as is. If you have a static result or some additional information, 
for instance, a string, you can use the `StringField` to store the result in the database.

After creating the database model, you need to register the new model in the `__init__.py` file located in the
`model` folder. The `__init__.py` file is responsible for importing all the database models and making them available
to the rest of the application. Add the following line to the `__init__.py` file to import the new model
`YourAnalyzerReport`:
```
...
from .YourAnalyzerReport import YourAnalyzerReport
```

As we have now created the database model, we need are going to add the new model to the AndroidApp document so that 
the results can later be accessed via the AndroidApp document. The AndroidApp document is located in the `model` folder
under `AndroidApp.py`. The AndroidApp document is responsible for storing the information about the Android apps (apk) 
and we will just add a new reference field to the `YourAnalyzerReport` model. 

```
...
youranalyzer_report_reference = LazyReferenceField('YourAnalyzerReport', reverse_delete_rule=DO_NOTHING)
...
```

Consequently, the AndroidApp document is now linked to the `YourAnalyzerReport` model 
and the results can be accessed via the AndroidApp document.



#### Step 3: Create a wrapper script
After you have created the database model, you need to create a wrapper script that will be responsible for running the
static analyzer and extracting the information from the apk files. Static analyzers are usually command-line tools or
libraries that can be used in python scripts. The wrapper script for static analyzers should be located in the 
`source/static_analysis` folder 
(see ["source/static_analysis"](https://github.com/FirmwareDroid/FirmwareDroid/tree/main/source/static_analysis))). 

Create a new directory with the name of your analyzer and a new python wrapper script `YourAnalyzer_wrapper.py`. We
will use this script later to access the static analyzer from the GraphQL API. 

An example that can be used as a template for the wrapper script is available in the `source/static_analysis/Example` 
folder. The example 
(see [Example_wrapper.py](https://github.com/FirmwareDroid/FirmwareDroid/tree/main/source/static_analysis/Example/Example_wrapper.py)) 
shows how a wrapper script can be implemented to run static analyzer from python. In the following, we will go through
the key components of the wrapper script:

- **class YourAnalyzerJob(ScanJob)**: The wrapper script should contain a class that inherits from the `ScanJob` 
class and implements the `start_scan` method. The `start_scan` method is responsible to start the correct 
python interpreter within the docker container and run the static analyzer on the Android apps. To boost performance, 
it uses the `start_python_interpreter` function, which starts multiple instances of the scanner to analyse a 
list of Android apps on multiple processors.
  - Adjust the class with the name of your analyzer, for example, `class YourAnalyzerJob(ScanJob)`. 
    - Adjust the `MODULE_NAME` and `INTERPRETER_PATH` variables with the name of your analyzer and the path to the 
      python interpreter.
  - Adjust the `worker_function` argument with the function that will be executed on multiple cores.
- **Implement the `process_android_app` method**: The method should invoke your static analyzer and extract the
result either as a string or as a file.
- **Implement the `store_result` method**: Take the result from the `process_android_app` method and store it in the
database. The method should create a new instance of the `YourAnalyzerReport` model and save the extracted information.


#### Step 4: Create a GraphQL API endpoint
After you have created the wrapper script, you need to add a reference to the wrapper script for the GraphQL API. 
The path of the wrapper scripts needs to be added to the `ScannerModules` enum in the 
`source/api/v2/schema/AndroidAppSchema.py)` file. The enum looks like this and you can just append to the end of the
enum with the name of your analyzer and the path to the wrapper script:
```
class ScannerModules(Enum):
    ANDROGUARD = {"AndroGuardScanJob": "static_analysis.AndroGuard.androguard_wrapper"}
    ANDROWARN = {"AndrowarnScanJob": "static_analysis.Androwarn.androwarn_wrapper"}
    APKID = {"APKiDScanJob": "static_analysis.APKiD.apkid_wrapper"}
    APKLEAKS = {"APKLeaksScanJob": "static_analysis.APKLeaks.apkleaks_wrapper"}
    EXODUS = {"ExodusScanJob": "static_analysis.Exodus.exodus_wrapper"}
    QUARKENGINE = {"QuarkEngineScanJob": "static_analysis.QuarkEngine.quark_engine_wrapper"}
    QARK = {"QarkScanJob": "static_analysis.Qark.qark_wrapper"}
    SUPER = {"SuperAndroidAnalyzerScanJob": "static_analysis.SuperAndroidAnalyzer.super_android_analyzer_wrapper"}
    MORF = {"MORFScanJob": "static_analysis.MORF.morf_wrapper"}
    VIRUSTOTAL = {"VirusTotalScanJob": "static_analysis.Virustotal.virus_total_wrapper"}
    MANIFEST = {"ManifestParserScanJob": "static_analysis.ManifestParser.android_manifest_parser"}
    MOBSF = {"MobSFScanJob": "static_analysis.MobSFScan.mobsfscan_wrapper"}
    YOUR_ANALYZER = {"YourAnalyzerScanJob": "static_analysis.YourAnalyzer.your_analyzer_wrapper"}
```

Adding the reference to the `ScannerModules` enum will make the new static analyzer available in the GraphQL API under
the `CreateApkScanJob` mutation. The `CreateApkScanJob` mutation is responsible for triggering the static analyzer and
scanning the apk files. The mutation is located in the `source/api/v2/mutations/AndroidAppSchema.py` file and you don't
need to adjust anything else in this file to make the scanner available in the GraphQL API.

We can now scan Android apps but the GraphQL API does not have an endpoint to retrieve the results of the scan job. 
Thus, we need to create a new resolver query in the `source/api/v2/schema/YourAnalyzerSchema.py` file. 
The resolver should be responsible for retrieving the results of the scan job from the database and look 
similar to this example:
```
import graphene
from graphene_mongo import MongoengineObjectType
from graphql_jwt.decorators import superuser_required
from api.v2.types.GenericFilter import get_filtered_queryset, generate_filter
from model.YourAnalyzerReport import YourAnalyzerReport

ModelFilter = generate_filter(YourAnalyzerReport)


class YourAnalyzerReportType(MongoengineObjectType):
    class Meta:
        model = YourAnalyzerReport


class YourAnalyzerReportQuery(graphene.ObjectType):
    your_analyzer_report_list = graphene.List(YourAnalyzerReportType,
                                      object_id_list=graphene.List(graphene.String),
                                      field_filter=graphene.Argument(ModelFilter),
                                      name="your_analyzer_report_list"
                                      )

    @superuser_required
    def resolve_your_analyzer_report_list(self, info, object_id_list=None, field_filter=None):
        return get_filtered_queryset(YourAnalyzerReport, object_id_list, field_filter)
```

Add then your new resolver query to the `Query` class in the `source/api/v2/schema/FirmwareDroidRootSchema` file:
```
class Query(ApplicationSettingQuery,
            StoreSettingsQuery,
            ...
            YourAnalyzerReportQuery,
            ...)
```
This will then expose the new resolver query in the GraphQL API and you can access the results of the scan job via the
GraphQL API.

#### Step 5: Testing the static analyzer
If you have implemented all the steps above, you can now test the static analyzer by scanning a couple of apk files.
First we start the containers by running the following command:
```
docker compose up
```
We then navigate to the GraphQL API at [https://fmd.localhost/graphql/](https://fmd.localhost/graphql/) and run the
`createApkScanJob` mutation to start the scan job. The mutation should look similar to this one:
```
mutation createScanJob {
  createApkScanJob(
    moduleName: "YOUR_ANALYZER"
    objectIdList: ["SOME_ANDROID_APP_ID", "SOME_ANDROID_APP_ID", "SOME_ANDROID_APP_ID"]
    queueName: "default-python"
  ) {
    jobIdList
  }
  _debug {
    exceptions {
      stack
      message
      excType
    }
  }
}
```
Replace `YOUR_ANALYZER` with the name of your analyzer and `SOME_ANDROID_APP_ID` with the object ids of the Android apps
you want to scan. The mutation will start the scan job and the results will be stored in the database. You can access
the results via the GraphQL API or directly in the database.

To retrieve the results via the GraphQL API, you can run a query similar to this one:
```
query getAPKScannerReport {
  your_scanner_report_list(
    fieldFilter: {android_app_id_reference: "SOME_ANDROID_APP_ID"}) {
    scannerVersion
    scannerName
    results
    reportDate
    id
  }
}
```

### Conclusion

In this tutorial, we have shown you how to add a new static analyzer to FMD and scan a couple of apk files. We have
gone through the key steps of adding a new static analyzer, including adding the dependencies, creating a database model,
creating a wrapper script, creating a GraphQL API endpoint, and testing the static analyzer. We hope this tutorial has
helped you to get started with adding your own static analyzer to FMD. If you have any questions or need further
assistance, please feel free to reach out to us.

