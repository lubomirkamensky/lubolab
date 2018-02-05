+++
date = "2018-01-13T03:57:28+02:00"
draft = false
title = "Camera Events: Data Model"

tags = [ "Personal DWH", "Netatmo" ]
+++

### The Idea
I have few [smart Netatmo cameras](https://www.netatmo.com/en-US/product/security/), both indoor and outdoor. And it is quite logical to integrate them into my personal Datawarehouse. There is even [nice open API](https://dev.netatmo.com/resources/technical/reference/cameras) supporting such intention.

The goal is to load event data into [detail layer](/detail-layer-logical-data-model/) and store the snapshot photos in [Google photos](https://photos.google.com) using the [Google Backup and Sync](https://photos.google.com/apps). Then, of course, somehow link the data with the photos.

### The Data Model Concept
This time we are going to face bit more complicated interaction with the source system. So it is worth spending some time on robust Data Model design. Our Observation model utilizes the approach of supertype and subtypes.  In fact, it offers something like Inheritance in object-oriented programming.

<div class="c_alert c_alert-success"><i class="fa fa-info-circle"></i> The supertype entity contains the common properties for many related objects, while the properties specific for the particular object take place in the subtype entity. Properties can be both attributes or relations to other entities. Supertype shares the primary key (PK) with all its subtypes</div>

To keep things simple we will always use surrogate keys as PK. Integers generated outside of database in ETL scripts. Never using any internal autoincrement database feature. The surrogate key is generated for the supertype entity and then provided to all its subtypes. By saying so, it is clear that we can never load data into subtype table without having loaded records in its supertype.

<div class="c_alert c_alert-success"><i class="fa fa-info-circle"></i> So the supertype is the entrance point for the data coming from the source system. To simplify the communication with the source system we store here also all needed source keys. Then we don't need any external mapping tables when connecting data in our data warehouse with the source system data.</div>

### The Dimensions
Let's start with the dimensions. We have supertype table [sensor](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/sensor.tbl). It contains the surrogate key "Sensor_Id" which is also the PK, then "Sensor_Source_Cd" identifying the sensor in the source, then "Source_Name" identifying the source system and finally "Target_Table" which specifies the subtype table, in our case, it is the table [camera](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera.tbl).
That is the typical pattern for dimension supertype. Similarly we have supertype [entity](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/entity.tbl) and its subtype [person](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/person.tbl). 

Then we have two small lookup tables defined directly in our data warehouse, [event_type](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/event_type.tbl) and [observation_label](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation_label.tbl), which are not linked to any source and which keys are hardcoded in table DDL. Such tables, of course, don't need any supertype.

### The Fact tables
Now, let's move to the fact tables. We store data about the events in table [camera_event](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera_event.tbl) and we can attach snapshot photos to the event using table [camera_snapshot](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera_snapshot.tbl). 

Both tables are subtypes of one supertype table [observation](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation.tbl)

There is no source key generated for events in the source system. But we know the logical primary key. It is the combination of the Camera identification and the event timestamp. There cannot be more than one event coming from the particular camera at the same time.

Supertype [observation](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation.tbl) contains "Observation_Id" the surrogate key. Then "Observation_Timestamp" the source timestamp specifying the event or snapshot creation time. And also "Sensor_Id" The Sensor surrogate key, in our case primary key of the camera in the personal data warehouse. Using "Related_Observation_Id" we can relate a camera snapshot to the camera event. The field "Target_Table" specifies target table (camera_event or camera_snapshot).

Subtype [camera_event](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera_event.tbl) extends its supertype providing more specific details about the event. Classify it using "Event_type". It can store the person identification in "Person_Id" for person events generated by a camera with face recognition. Creation time is available in the supertype as [The Unix timestamp](https://www.unixtimestamp.com/)  The same value, just  transformed into Date time field is "Event_Start_Dt".

Subtype [camera_snapshot](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/camera_snapshot.tbl) stores snapshot photo file name in "File_Name" and Google Drive File Id in "File_ID".

The last table to mention  is [observation_label_related](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/observation_label_related.tbl).  It allows us to assign several labels to one event. The outdoor camera generates events where multiple objects are identified, like a car, person, animal. In such case, we use labeling to add all needed information to one event. 

