+++
date = "2017-04-17T03:57:28+02:00"
draft = false
title = "Astronomical Events"

tags = [ "Personal DWH" ]
+++

### Personal environment
Personal DWH begins with data about personal environment. Weather and day length affect our activity significantly. We already have Weather Data mart created, so now it is the time to build another one on top of astronomical events.

<div class="c_alert c_alert-success"><i class="fa fa-info-circle"></i> This new datamart is going to be a common pattern for extending our personal DWH with any new data area.</div>

### The detail layer
We start in our detail layer,  focusing on another [Observation subtype](http://lubolab.com/detail-layer-logical-data-model/), the Event. To get the information about the day length we introduce a sun events.

You can see sample Event data here:![](/images/2017/05/event.jpg)

The whole table structure is available in public repository on GitHub here [personal_dwh/tree/master/detail_layer/db/tables](https://github.com/lubomirkamensky/personal_dwh/tree/master/detail_layer/db/tables)

### Data source
As the source for event data we use [public Astronomical Applications API](http://aa.usno.navy.mil/data/docs/api.php) provided by The United States Naval Observatory (USNO).

Here is sample URL requesting all Sun and Moon events for one day on location specified by coordinates: [http://api.usno.navy.mil/rstt/oneday?date=05/12/2017&coords=48.749015N,16.983294E&tz=1](http://api.usno.navy.mil/rstt/oneday?date=05/12/2017&coords=48.749015N,16.983294E&tz=1)

### ETL environment
We use [Google Apps Script](https://developers.google.com/apps-script/) as environment for data extraction, load and transformation (ETL) and also as job scheduler.  It is used for building both detail layer in Cloud SQL and application layer in Firebase. 

### The data flow
The whole flow starts by loading the detail table [Event](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/event.tbl)

We have few views in Cloud SQL  to support the detail layer load. They define the load scope. The default scope is limited by the beginning of our DWH and end of current year. Later the begining of the load scope  moves according to already loaded data. You can see the view definitions here [personal_dwh/tree/master/detail_layer/db/views](https://github.com/lubomirkamensky/personal_dwh/tree/master/detail_layer/db/views). The detail layer load in SQL cloud is driven by the content of [Event](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/db/tables/event.tbl) table itself. 

The astronomical datamart is again defined as a view. There is not so big volume of data so all hourly, daily, monthly, yearly data is derived on top of one, hourly view [v_ daylight_ length_ hourly](https://github.com/lubomirkamensky/personal_dwh/blob/master/astronomical_mart/db/views/v_daylight_length_hourly.vw).

All hourly, daily, monthly, yearly data is materialized in the Firebase. The load is driven by  new control table [t_ loaded_ periods](https://github.com/lubomirkamensky/personal_dwh/blob/master/astronomical_mart/db/tables/t_loaded_periods.tbl). The range of data for the actual load is defined by views [v_ load_ scope](https://github.com/lubomirkamensky/personal_dwh/blob/master/astronomical_mart/db/views/v_load_scope.vw) and [v_ load_ range](https://github.com/lubomirkamensky/personal_dwh/blob/master/astronomical_mart/db/views/v_load_range.vw)

<div class="c_alert c_alert-success"><i class="fa fa-info-circle"></i> The limitation of Google Apps Script is a time constraint for an execution. It can run for a maximum period of 4-5 minutes. As we already have load control, using load scope views,  it is not a big deal to process data in small batches, manageble in given time frame. </div>

The script for loading the astro events into detail layer is here [pdwh_ detail_ event_ astro.gs](https://github.com/lubomirkamensky/personal_dwh/blob/master/detail_layer/jobs/gappscript/pdwh_detail_event_astro.js)

The script for building the access layer in Firebase is here [firebs_ meas_ astro_ daylight.gs](https://github.com/lubomirkamensky/personal_dwh/blob/master/astronomical_mart/jobs/gappscript/firebs_meas_astro_daylight.gs)

### Reporting tool
At the end we do small change in our [web app](http://lubolab.com/the-data-lab-web-app/) to include the new data.
The change takes place in one file [data-view.html](https://github.com/lubomirkamensky/personal_dwh/blob/master/web_app/public/src/data-view.html). The dropdown menus for Subject Area, Measure and Grain will be now dynamic, generated from associative arrays measureDict and grainDict.

The final result you can see here [lubolab-957a1.firebaseapp.com](https://lubolab-957a1.firebaseapp.com/). Login into your gmail account is required.
![](/images/2017/06/astro.jpg)

