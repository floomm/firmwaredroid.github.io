---
title: Exploring the API
author: tom
date: 2024-04-02 13:37:00 +0800
categories: [Tutorial, API]
tags: [getting started, api, tutorial]
toc: true
---

All the examples in this tutorial use the graphql API available under 
[https://fmd.localhost/graphql](https://fmd.localhost/graphql), 
which allows to explore the API documentation and to build graphQL queries and mutations. Please note
that the API is only available when the application is running. The API is subject to change and queries in this
example might not work in future versions of FMD. However, the general structure of the API should remain the same.


### Importing Android Firmware

After setting up the application, you might want to start exploring some Android firmware. By default,
FMD creates a folder `blob_storage`, where all the data (databases and blobs) will be stored. To import Android firmware
you need to copy the Android firmware archives (.zip, .tar) into the import folder:
`blob_storage/00_file_storage/<random_id>/firmware_import`

1. Copy your firmware into the import folder:
   ```
   `blob_storage/00_file_storage/<random_id>/firmware_import`
   ```
   
2. Navigate to the graphql API (https://fmd.localhost/graphql) and start the mutation job: `createFirmwareExtractorJob`.
   This will trigger the `extractor-worker-high-1` container to import the firmware from the `/firmware_import` folder.
   ```
   # First, log-in in case you haven't already
   query MyQuery {
     tokenAuth(password: "XXXXX", username: "XXXX") {
       token
       payload
     }
   }
   
   # Second, start the import.
   mutation StartImport {
     createFirmwareExtractorJob(createFuzzyHashes: false, 
     queueName: "high-python") {
       jobId
     }
   }
   ```
   Importing takes several minutes. Might be a good moment to get a coffee.

3. You can monitor the status of the `createFirmwareExtractorJob` on https://fmd.localhost/django-rq or
   alternative you can connect directly to the database to see if it was successfully imported.
   To connect to the database you need a MongoDb-client (e.g., Studio 3T). You will find the connection credentials
   for mongodb in the `.env` file:
   ```
   cat .env
   ...
   MONGODB_USERNAME=XXXX
   MONGODB_PASSWORD=XXXX
   ...
   ```
   Connect to 127.0.0.1:27017 using SCRAM-SHA-256 authentication. You find all successfully imported firmware samples
   in the collection `android_firmware`.

4. If the firmware was successfully imported you should have the collections `android_firmware` and `android_app`
   in the database, where you can find already some meta-data. Moreover, you can find the extracted
   apps within the blob storage:
   ```
   APKs: `blob_storage/00_file_storage/<random-id>/android_app_store/<firmware-hash>/<partition-name>/`
   Firmware: `blob_storage/00_file_storage/<random-id>/firmware_store/<android-version>/<firmware-hash>/`
   Firmware (failed): `blob_storage/00_file_storage/<random-id>/firmware_import_failed/`
   ```
   If for some reason the importer wasn't able to extract the firmware, the firmware will be moved to the
   `firmware_import_failed` directory in the blob storage and you need to check the docker logs why it failed.
   
   You can also use the graphql API to fetch all available firmware data with the following example queries
   in case the import was successful:
   ```
   # Gets a list of firmware object-ids
   query GetAndroidFirmwareIds {
     android_firmware_id_list
   }
   ```

   Take the resulting firmware object-ids from the query above (`GetAndroidFirmwareIds`) and use them for the
   following query `GetAndroidFirmwareIds`
   to fetch some firmware meta-data. (Replace XXXX with the firmware id you want to fetch in the query below)
   ```
   query GetAndroidFirmwareIds {
     android_firmware_list(objectIdList: ["XXXXX"]) {
       absoluteStorePath
       fileSizeBytes
       filename
       has_file_index
       has_fuzzy_hash_index
       id
       indexedDate
       md5
       originalFilename
       osVendor
       relativeStorePath
       sha1
       sha256
       tag
       versionDetected
     }
   }
   ```

   You can fetch meta-data about the Android apps with the following two queries. (Replace XXXXX with the firmware id)
   ```
   # Fetch Android app objects by firmware id.
   query GetAndroidApps {
     android_app_list(
     objectIdList: ["XXXXX"], 
     documentType: "AndroidFirmware") {
       sha256
       sha1
       relativeStorePath
       relativeFirmwarePath
       pk
       packagename
       md5
       indexedDate
       id
       filename
       fileSizeBytes
       absoluteStorePath
     }
   }
   ```

   ```
   # Fetch just the Android app object-ids by the firmware-id
   query GetAndroidAppIds {
     android_app_id_list_by_firmware(
       objectIdList: ["XXXXX"])
   }
   ```


### Static Analysis on Android apps

Currently, FMD does not have a user interface for all features. To scan we use the graphql API. We are working
on the FMD user-interface. However, since FMD is a research project our focus is currently mainly on enhancing the
backend and not the frontend.

To scan Android apps, you will need to have the object-ids of the Android apps you want to scan. You can get the
object-ids directly from the database (collection: `android_app`) or via the graphql API.
1. To do it via graphql, navigate to https://fmd.localhost/graphql and run the following query to fetch the object-ids.
   (Replace XXXXXX with the firmware id.)
   ```
   query GetAndroidAppIds {
     android_app_id_list_by_firmware(objectIdList: ["XXXXXX"])
   }
   ```
   As result, you will get a list of all the available Android app object-ids from the specified firmware. We can now
   scan these Android apps with one of the static-analysers.

2. In this example, we use AndroGuard. However, you could use any of the following static-analysers as well. Please,
   note that not every scanner was built for mass scanning and some of them might be slow in scanning speed.
   ```
     ANDROGUARD
     ANDROWARN
     APKID
     APKLEAKS
     EXODUS
     QUARKENGINE
     QARK
     SUPER 
   ```
   We start a scan job with the `createApkScanJob` mutation on https://fmd.localhost/graphql. We use the mutation as
   follows. (Replace XXXX with the Android app object-ids you want to scan. Set the moduleName option to one from the
   above list)
   ```
   mutation MyMutation {
     createApkScanJob(
       moduleName: "ANDROGUARD"
       objectIdList: ["XXXXX", "XXXXX", "XXXXX", "XXXXX"]
       queueName: "default-python"
     ) {
       jobId
     }
   }
   ```
3. After executing the mutation, the docker container named `apk_scanner-worker-1` should start scanning
   the Android apps. You can monitor the status of jobs on https://fmd.localhost/django-rq or
   take a look at the logs in docker with: `docker-compose logs -f apk_scanner-worker-1`

4. The results of the scan job can either be viewed over the graphql API or directly on the database
   (collection `apk_scanner_report` in the db). After scanning, a reference to the scan result
   is stored in the Android app document. For instance,
   when scanning with AndroGuard, apps will have a reference called `androguard_report_reference` holding a object-id
   reference to the scanner report document (e.g., `androguard_report`).

   If we want to fetch the results, we used the object-ids for the `androguard_report` from the android app object.
   (Replace XXXXX with the androguard_report object-ids)
   ```
   query GetAndroGuardReports {
     androguard_report_list(
       objectIdList: ["XXXXX", "XXXXX", "XXXXX"]) {
       Cls
       androidVersionCode
       androidVersionName
       appName
       effectiveTargetVersion
       isAndroidtv
       id
       isLeanback
       isMultidex
       isSignedV1
       isSignedV2
       isSignedV3
       isValidApk
       isWearable
       mainActivity
       manifestXml
       maxSdkVersion
       minSdkVersion
       packagename
       permissionDetails
       permissionsDeclaredDetails
       reportDate
       scannerName
       scannerVersion
       targetSdkVersion
     }
   }
   ```
   As a result you will get the scanning result from AndroGuard.
