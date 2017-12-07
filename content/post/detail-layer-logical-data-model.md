+++
date = "2017-01-17T03:57:28+02:00"
draft = false
title = "Detail Layer: Logical Data Model"

tags = [ "Personal DWH" ]
+++
### No Data Changes in the Detail layer
The [first post](http://lubolab.com/my-personal-dwh-kickoff/) ended with the table weather_measuring filled with a data which describes a weather. And I mentioned that this table belongs to something called Detail layer. Now it is the time to put it in some kind of context.

Detail layer is the place where the most detailed information coming from various sources is stored. Quite probably we will use just part of it for the end user applications, maybe we will even change some data a bit for different purposes. But never directly in our detail layer. Here we keep the data as  it comes from the source.  

<p class="note">Of course it's possible to add some technical columns or to introduce mechanism for history handling. But never to apply any transformations where some information from the source is changed or even lost. </p> 

Any transformation rule is always based just on the knowledge available at that particular moment and that can change in the future.  Then it's the time to come back to the detail layer, find the original source data, and recalculate data-marts or application layers accordingly to the new rules.

### Logical Model as a Framework
Properly built detail layer is the real fortune. Here is my current idea how to logically design it for personal data warehouse. I am not going to introduce any final physical data model, that's something what will change frequently. This is more the logical framework allowing to add new entities based on specific actual needs.

My approach is very common. Any information we can get is output of some kind of observation. The Observation is then the main entity in our model. It has some subject, and both observation and it's subject can be classified in various manner and described by many details. 

### The Observation Model
The supertype Observation has 4 subtypes: Measuring, Activity, Event, Snapshot. And they have again it's subtypes like our Weather Measuring. On the supertype level the Observation can have recursive relation to itself.  Practical example of such situation can be making a snapshot and storing it. And later processing it and generating some events based on the detail analysis of the snapshot. 

![](/images/2017/01/model.png)

if you are not familiar with data modeling, try book [Data Modeling Made Simple](https://www.amazon.com/Data-Modeling-Made-Simple-Professionals/dp/0977140067/ref=s9_simh_gw_g14_i1_r?_encoding=UTF8&fpl=fresh&pf_rd_m=ATVPDKIKX0DER&pf_rd_s=&pf_rd_r=T41JFFAYD26HKYZTJ70G&pf_rd_t=36701&pf_rd_p=a6aaf593-1ba4-4f4e-bdcc-0febe090b8ed&pf_rd_i=desktop) written by [Steve Hoberman](http://stevehoberman.com/). 

And If you don't like data modeling, then just forget this post. I wrote it just that I like it a lot and in fact thinking in 3NF most of the time :)

### The Calendar Model
For those still interested in the subject, here is the model of Calendar. I believe, nothing surprising. 
![](/images/2017/01/calendar.png)

<p class="success">With both Observer and Calendar models implemented, we can start using our detail layer for building the first datamart.</p>  

