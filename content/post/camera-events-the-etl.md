+++
date = "2018-01-14T03:57:28+02:00"
draft = false
title = "Camera Events: The ETL"

tags = [ "Personal DWH" ]
+++

### The ETL Script environment
We are going to access Netatmo cameras using [Netatmo Connect API](https://dev.netatmo.com/resources/technical/reference/cameras) which is basically REST API providing JSON responses. I think the best scripting environment for work with dictionaries, lists, and sets is Python. 

And there is even nice client simplifying the communication with this API in Python [netatmo-api-python](https://github.com/philippelt/netatmo-api-python) provided by [Philippe Larduinat](https://github.com/philippelt). Well, it wasn't hard to decide and I installed the latest version of 3.X [Python](https://www.python.org/downloads/).

### The Goals
We need to incrementally load camera events into database and store related snapshot photos in Google Drive. Then the identification of the snapshot files in Google Drive needs to be stored in the database as well. 

1.	Incremental load of Event Data into database
2.	Storing snapshot photos in Google Drive
3.	Storing Google photos metadata into database

### The load of Event Data into database
The first goal is accomplished in the script [netatmo_events_data.py](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/jobs/python/netatmo_events_data.py) You can find the code in the linked GitHub repository. 

Of course without the configuration files for database and the cameras. They look like:

**netatmo.json**

``` 
{
    "clientId" : "XXXXXXXXXXXXXXXXXXXXXXXXX",
    "clientSecret" : "XXXXXXXXXXXXXXXXXXXXXXXXX",
    "username" : "xxxxx.xxxx@xxxx.com",
    "password" : "xxxxxx",
    "scope" : "read_presence access_presence read_camera access_camera"
}
``` 

**mysql.json**

``` 
{
    "host" : "XX.XX.XX.XX",
    "user" : "XXX",
    "passwd" : "XXXXX",
    "db" : "pdwh_detail",
    "local_infile" : 1
}
``` 

Well, I tried to keep things as simple as possible but you know, everything tends to grow in complexity in our universe and this code is not an exception.

The first part is loading all necessary libraries and initialize the connections to the database and Netatmo API.

Database load starts with the dimensions. There is a common pattern for loading supertype and subtype tables so we define one method for loading supertype table and one for the subtype.

We use statement LOAD DATA INFILE  which reads rows from a text file into a table at a very high speed. Before using it we have to prepare the input text file. To do so we need to get needed data from Netatmo API.

For the dimensions, where the total amount of data in tables is not big, we  compare the data already loaded to tables with the data available in Netatmo API. This way we get the increment to be loaded into the particular table. Then we only need to know the last value of the surrogate key in the table as the starting point for the new generation of surrogate keys.

In Python, we heavily work with lists, dictionaries and sets and [List and Set Comprehension](https://docs.python.org/3.6/tutorial/datastructures.html#tut-listcomps) is a huge help.

Regarding the facts, the approach is similar but not the same. As I already mentioned before, there cannot be more than one event coming from the particular camera at the same time. Then we just need to find out the last timestamp of loaded events for each camera from the database. And of course the last value of the surrogate key for the observer supertype.

There is different event structure for indoor and outdoor cameras and even the structure for one type of camera varies based on the objects identified within the event. We use List Comprehension to filter the requested event data out of all events provided by Netatmo API. 

``` 
        # List Comprehension in practis
        p1 = [ [  value['time'],
                  value['type'],
                  value['person_id'],
                  [value['time'],value['snapshot']] ]
                if 'snapshot' in value.keys() and 'person_id' in value.keys() 
                else [
                      value['time'],
                      value['type'],
                      [value['time'],value['snapshot']]]
                if 'snapshot' in value.keys()
                else [value['time'],
                      value['type'],
                      value['person_id']] 
                if 'person_id' in value.keys() 
                else [value['time'],
                      value['type'],
                      # using set to get distinct event types from the whole event list
                      set([i['type'] for i in value['event_list']]),
                      [[i['time'],i['snapshot']] if 'snapshot' in i.keys() 
                                                 else [] for i in value['event_list'] ]] 
                if 'event_list' in value.keys() 
                else [value['time'],
                      value['type']] 
                for key, value in xEvents.items() if key > camera[i][1]  ]
``` 

The main filter at the very bottom takes only events having timestamp higher than events already loaded to the database. Then there are distinct cases for the event structure variations.  Outdoor events have nested list of subevents (event_list), indoor events can identify particular person (person_id), both indoor and outdoor events can contain snapshots.

So we always get event timestamp and event_type, sometimes also person_id, snapshot, additional labels describing the subevents.

Succesfully mined information about event properties is loaded into tables [observation](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation.tbl), [camera_event](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera_event.tbl), [observation_label_related](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation_label_related.tbl). 

If the snapshot information is available, the snapshot photo is stored in the local directory using the method "saveFromUrl".

``` 
# stores snapshot photos from netatmo api urls
def saveFromUrl(cameraKey,cameraDict,snapshoDict):
    if 'id' in snapshoDict[1].keys():
        url = "https://api.netatmo.com/api/getcamerapicture?image_id=" \
        + snapshoDict[1]['id'] + "&key=" + snapshoDict[1]['key']
    elif 'filename' in snapshoDict[1].keys():
        url = homeData.cameraUrls(cid=cameraKey)[0] + '/' + snapshoDict[1]['filename']
    else:
        url = ""

    snapshotPath = "./snapshots/" + xstr(cameraDict[cameraKey][2]) + "/" + xstr(snapshoDict[0]) + ".jpg"
    f = open(snapshotPath, 'wb')
    f.write(request.urlopen(url).read())
    f.close()
``` 

Each camera has a separate folder and the name of the snapshot file is the snapshot timestamp. So the combination of the folder name and the file name is always unique.

Later the snapshot photo files are processed by script [netatmo_snapshots_data](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/jobs/python/netatmo_snapshots_data.py). Let's look into it in next part.

### The load of Snapshot Data into database
This script selects from the database all events without any related snapshot and specifically search for snapshot files in the defined range of timestamps.

Based on the results, new records are created in [observation](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation.tbl) table. The snapshot is another observation subtype related  to the camera event.

At the same time, the snapshot file is renamed, the new name contains the unique Observation_Id so we don't need anymore to keep files in camera folders and we move all snapshot files to folder Google_Gate. This folder is monitored by [Google Backup and Sync](https://photos.google.com/apps) and all photos are uploaded to [Google Photos](https://photos.google.com).

The last step of the Camera Snapshot data flow is placed in the Google Drive environment.

### Linking Google Photos to Camera Events
New [Google Backup and Sync](https://photos.google.com/apps) tool create another root folder "Computers" in your Google Drive.  The underlying folder structure matches the folders selected for backup to Google Drive. The Google tool is only available for Windows and iOS.

There are two scripts. The first one [photobackupGoogleGate](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/jobs/gappscript/photobackupGoogleGate.js) only moves the snapshot files into another Google Drive directory.  

The second [sortphotos](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/jobs/gappscript/sortphotos.js) is sorting all kinds of photos coming from different sources.

There is a special case for Camera snapshots.

```
if (file.getName().substring(0,15) == "camera_snapshot") {
      var event_id = file.getName().split("_")[2].replace(".jpg","");
      
      var sqlstring = 'INSERT INTO pdwh_detail.camera_snapshot(Snapshot_Id,File_Name,File_ID) \
                       SELECT Observation_Id,?,? FROM pdwh_detail.observation \
                       WHERE Target_Table = ? AND Observation_Id = ?';
      Logger.log(sqlstring);
      var stmt = conn.prepareStatement(sqlstring);
      stmt.setString(1, file.getName());
      stmt.setString(2, fileId);
      stmt.setString(3, 'camera_snapshot');
      stmt.setString(4, event_id);
      stmt.execute();
      stmt.close();
```      

It gets Snapshot Observer_Id from the file name and File Id from Google Drive API and then stores gained information into [camera_snapshot](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera_snapshot.tbl) table.

### Glue it all together
Considering all the components included in the data flow,  starting with Netatmo API, Then local Python scripts storing data in SQL Cloud and finally the snapshot files stored in the local system just to be later transferred to Google Drive and linked with database data, one question was left hanging in the air. Why all that circus just to move data from  one Cloud to another? And especially, why to introduce any local components to the flow beginning and ending in the Cloud? Well, the answer is simple. Unlimited free storage of snapshots in the Google Photos.

It would be definitely more logical to run the ETL scripts in the Cloud, either in Google Apps Script or some small Linux virtual server.  But to achieve the advantage of  unlimited free storage the Google Backup and Sync tool has to be used and it is only available for Windows and iOS.

And as I  have my tablet "Windows server" running the load of the [weather station data](/my-personal-dwh-kickoff/) anyway, I used it for Camera events as well.

The python scripts are executed by Windows Task Scheduler every 5 minutes.  The Google Apps scripts every 15 minutes. 



