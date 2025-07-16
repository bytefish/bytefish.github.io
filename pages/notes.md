title: notes
date: 2025-06-28 10:30
author: Philipp Wagner
template: page
summary: Notes, Ideas and Links

## Table of contents ##

[TOC]

## 2025-07-16: Taking the Repositories to Private ##

I have put most of my repositories into private mode. Why? Because I don’t want to feed LLM training sets anymore and see my tutorials and code copies verbatim without proper attribution.

If you need access to any of the repositories, I am more than happy to share the code with you. But it’s going to take you at least a mail or an issue in this page issue tracker, to ask for it.

## 2025-07-15: Disappearing and Resurrecting the Page ##

Last week, after 22 years of running this page, I thought: It’s enough! Google is de-ranking the page. LLMs are stealing the content, and on top “AI startups” crawling the page to death. 

So I have deleted my domains, and then canceled the contract with my host of 22 years. Most of my GitHub repositories have been set into Private mode.

The lesson here: Never rely on the mental well-being of a single open source developer for anything, you are running in production. These developers could disappear from one moment to the next.

While on vacation I re-registered my domain and moved it to GitHub Pages and am going to move to Cloudflare as a Domain Registrar, so the costs of this page are reduced.

## 2025-06-29: Importing Routing Data for pgRouting on Windows ##

`pgRouting` is a library for adding Routing capabilities to PostgreSQL. However the import using `osm2pgrouting` does not 
support importing `*.pbf` files out of the box. So we need to convert them to an `osm` file first. This can be done using 
`osmosis`.

`osmosis` can be downloaded from:

* [https://github.com/openstreetmap/osmosis/releases](https://github.com/openstreetmap/osmosis/releases)

We can then convert the data to `osm` using:

```
.\bin\osmosis.bat --read-pbf muenster-regbez-latest.osm.pbf --write-xml file=muenster-regbez-latest.osm
```

This takes something around 45 seconds and takes up 20x the size of the original file:

```
Jun 29, 2025 11:25:05 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Osmosis Version 0.49.2
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
Jun 29, 2025 11:25:05 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Preparing pipeline.
Jun 29, 2025 11:25:05 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Launching pipeline execution.
Jun 29, 2025 11:25:05 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Pipeline executing, waiting for completion.
Jun 29, 2025 11:25:50 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Pipeline complete.
Jun 29, 2025 11:25:50 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Total execution time: 45971 milliseconds.
```

We can then use `osm2pgrouting.exe` (available in the `bin` folder of your PostgreSQL installation.

`osm2pgrouting` creates all tables required and imports the data required for routing:

```
./osm2pgrouting.exe --file "C:\Users\philipp\Downloads\muenster-regbez-latest.osm" \
    --host localhost \
    --port 5432 \
    --dbname flinkjam \
    --username postgis \
    --password postgis \ 
    --clean \
    --conf ".\conf\mapconfig_for_cars.xml"
```

The `mapconfig_for_cars.xml` is in the `osm2pgrouting` directory and contains the features to be implemented:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <tag_name name="highway" id="1">
    <tag_value name="motorway"          id="101" priority="1.0" maxspeed="130" />
    <tag_value name="motorway_link"     id="102" priority="1.0" maxspeed="130" />
    <tag_value name="motorway_junction" id="103" priority="1.0" maxspeed="130" />
    <tag_value name="trunk"             id="104" priority="1.05" maxspeed="110" />
    <tag_value name="trunk_link"        id="105" priority="1.05" maxspeed="110" />    
    <tag_value name="primary"           id="106" priority="1.15" maxspeed="90" />
    <tag_value name="primary_link"      id="107" priority="1.15" maxspeed="90" />    
    <tag_value name="secondary"         id="108" priority="1.5" maxspeed="90" />
    <tag_value name="secondary_link"    id="109" priority="1.5" maxspeed="90"/>  
    <tag_value name="tertiary"          id="110" priority="1.75" maxspeed="90" />
    <tag_value name="tertiary_link"     id="111" priority="1.75" maxspeed="90" />  
    <tag_value name="residential"       id="112" priority="2.5" maxspeed="50" />
    <tag_value name="living_street"     id="113" priority="3" maxspeed="20" />
    <tag_value name="service"           id="114" priority="2.5" maxspeed="50" />
    <tag_value name="unclassified"      id="117" priority="3" maxspeed="90"/>
    <tag_value name="road"              id="100" priority="5" maxspeed="50" />
  </tag_name> 
</configuration>
```



## 2025-06-28: Going from a Project Idea to an Implementation ##

In these notes page you can see, how "flink-jam" went from a small project idea 
to an implementation. This is how most of my projects are built and how I work 
professionally.

Gather sample data. Model the problem in code. Start to build a prototype. Validate 
your plan. Concentrate on the 95% and the missing 5% probably need to be manually 
fixed. Then extract common logic. Write tutorials on it. Get the team on board.

You have to understand, that my approach to products, projects and programming is 
to first create lots and lots of chaos, then iterate on the chaos until things 
become less and less chaotic and then at a point they start to fall into their 
place.

This approach is often at odds with larger teams and it's at odds with agile 
development, that requires you to estimate all your tasks ahead. Thing is: I 
often don't know how to model the problem *yet*, thus I cannot estimate it 
reliably.

At the moment I am trying to solve this issue by getting team members on 
board early on and participate in the process. But it's often taxing for 
younger developers, because I move from *A* to *C* to *B* to *E* and 
back to *A* to understand how to build *D*.

## 2025-06-28: Implementing the Apache Flink Pipeline in flink-jam ##

A four-day business trip to Leipzig comes to an end. While a lot of meetings could be 
done virtual these days, I think meeting in person is very important to 
*know each other*.

While being away, one of the girls got sick, the car has to be repaired, leaving 
my wife stranded. All the usual drama, that occurs during business trips.

Yesterday I had some time to work on flink-jam and implemented the Java application 
using Apache Flink. It’s still a little chaotic, it has zero tests, but it 
works well.

## 2025-06-22: OpenStreetMap Data in flink-jam ##

I have moved flink-jam into a proper repository:

* [https://github.com/bytefish/flink-jam/](https://github.com/bytefish/flink-jam/)

The create table script now gets executed when the Docker container boots.

Yesterday I've got the SQL side working, so today I am going to start with the Java implementation. After 
a year of C\# without any Java, I need to re-learn a few things. Type-safety I am taking for granted with 
C\# now needs to take type-erasure into account. 😓

The Apache Flink API has changed a lot since 2016. What a surprise! So I need to research some stuff, and 
before you ask me why I am not using an LLM? Because they deliver results based on very outdated APIs, 
being unable to even make it compile.

It's already 18:00, and the children need attention. So it won't be finished today. 

But I think the repository has seen some progress, and it's something to build upon. 👏

## 2025-06-21: Starting with flink-jam ##

So... Another week went by, but I finally had some time to look into my project idea. 

First of all I gave it the catchy name `flink-jam`. In the past week I have read some articles on 
my smartphone on how to import OSM data and query it. Today I was allowed to sit in front of the 
screen, and I was able to download and import the OSM data. 

It wasn't as smooth as I had expected. The initial queries had been super slow and importing the OSM 
data initially led to problems with the Docker volumes. A GiST index was missing and the queries on 
my data have been extremly slow. 

But after some playing around with the query planner and indices, I got it to work just fine.

The next round is going to be with Java.

## 2025-06-14: Formulation a Project Idea ##

I think I have come up with an idea to play around with. 🥹 I’ll call it “Traffic Congestion Detection” for lack of a cool name.

The idea is to first import OSM data to Postgres. This should be painless, I’ll download the data from Geofabrik and import it to Postgres. I am going to use the Postgres EDB installer to automatically add the Postgis extension. 

Then in Postgres I’ll extract the Road Segments and assign them speed limits. I guess you need to use some kind of Road Segment to group traffic of multiple vehicles. In another table I am going to store Traffic Light Positions, because we expect congestion at Traffic Lights, right?

A small Python (not sure yet?) script is used to generate various traffic patterns within positions in a specific area in Germany. This will also give me the possibility to load test the application.

I could then use Apache Flink to detect various pattern in traffic like “Light Congestion”, “Severe Congestion”, “Traffic Jam”. The interesting thing here is, that we could use the CEP with different windows and see how it works.

In a later installment I am going to revive my “OpenSky Flight Tracker” project to also visualize traffic (or at least draw circles around alarms). I have to see, how complicated coloring road segments are. But the possibilities with this experiment are endless!

However, the week ahead is again filled with family things to do. So there is no leisure time left for my “very important side projects”. 

## 2025-06-14: Keep on writing  ##

So… After the initial high of free-form writing, things slowed down, while I am working full-time. The weekend is going to be filled with gardening, keeping the children busy and sleeping.

There’s not much programming to be expected. And I didn’t come up with a 1-day side project idea yet, which could be done within some hours. It’s just not feasible to start larger projects right now, because there’s no time left.

Anyways, let’s keep on writing!

## 2025-06-09: How Ideas for Projects Come to Life  ##

I am wondering, what project I could do next, so let's see my thought process.

Something I always wanted to get my hands on again is Apache Flink and its CEP Engine. From time to time I browse through my article and wonder... how could I revive it?

The Apache Flink example is in the following repository:

* [https://github.com/bytefish/FlinkExperiments](https://github.com/bytefish/FlinkExperiments)

Last year I have played around with NATS and JetStream to understand how their JetStream engine works and how a distributed system could be built... without the hellhole Kafka is. 

There's a GitHub repository I used to experiment with NATS in .NET here:

* [https://github.com/bytefish/NatsExperiments](https://github.com/bytefish/NatsExperiments)

The GitHub Repository comes with a Docker Compose file, so getting a NATS Cluster up and running *should* be possible relatively quick. 

The Synadia folks, that maintain the NATS ecosystem, also provide a NATS Connector at:

* [https://github.com/synadia-io/flink-connector-nats](https://github.com/synadia-io/flink-connector-nats)

So it shouldn't require a lot of infrastructure code to connect NATS an Apache Flink. We can find a few examples in the Synadia repository with sample code to get some inspiration from.

But where do we get the data to stream from?

See I have previously written an article for analyzing the German weather data to detect heat strikes, where there was severe heat in Germany for consecutive days:

* [https://github.com/bytefish/DwdAnalysis](https://github.com/bytefish/DwdAnalysis)

It would be cool to reuse it and see if I can reproduce the findings with Apache Flink. But the example repository is based on SQL Server, and SQL Server has a super complicated license. 

In order to have an example, that people could download and get it up and running, I would need to rewrite this to PostgreSQL. This would also require to port the parsing to Java, because I don't want .NET and Java mixed. 

The simplest approach for parsing the CSV is to use JTinyCsvParser:

* [https://github.com/bytefish/JTinyCsvParser](https://github.com/bytefish/JTinyCsvParser)

While at it, it's probably useful to update it to Java 24, too? I could then bulk import everything into PostgreSQL using something like this:

* [https://www.bytefish.de/blog/bulk_updates_postgres.html](https://www.bytefish.de/blog/bulk_updates_postgres.html)

Or I could dogfood and use PgBulkInsert, which would open yet another front, where I could find things to do, instead of concentrating on Apache Flink. But if I use Postgres without PgBulkInsert, then how do I use Composite Types?

All in all, this idea is canceled for now, because it's not something to write up within a day. It's an overly ambitious project, that takes weeks to finish, and only if I get a few hours on the weekends.


## 2025-06-08: Shifting Priorities and Programming ##

It’s Pentecost, so Germany has a long weekend. Back when I didn’t have a family, these days would have been filled with working myself into things. And most importantly: I would have used the time to program something for *myself*, instead of *writing code for a company*.

But ever since having a wife and children things surprisingly changed. Of course priorities shift, and instead of learning all about, say “Graph Databases”, my free time is spent with family things. I am waiting for weeks to find some quiet hours to try ideas accumulated inside.

By now I do most of my writing and reading on a smartphone. The latest article on Google Zanzibar is no different, the code has been written in the Notes app on the phone. Auto-correct is the enemy!

Once I could hide inside my office, I boot a computer (thanks to modern hard drives it doesn’t take a lot of time!). I am copying the code, correct all the syntax errors and draft an article in parallel.

I wonder how other people handle it?

## 2025-06-07: How it all started... 22 years ago  ##

This page started in 2003 as a way to share my poems with the world. It was a time full of beautiful self-pity, and not knowing what to do with my life… basically the whole range of first-world problems.

Please never use the Wayback Machine!

To the next 20 years to come!🍻

## 2025-06-07: The Microsoft Copilot Distortion Bubble ##

The Microsoft Build 2025 conference was all about AI and how it improves productivity. So it’s interesting to see Microsoft dogfooding Copilot in their `dotnet/aspnetcore` and `dotnet/runtime` repositories:

* [https://github.com/dotnet/runtime/pulls](https://github.com/dotnet/runtime/pulls)

It’s hilarious to see these world class engineers begging their Copilot to “Please do not forget to add the files to the project!” and “Do not make up APIs!”. Yes, they label it an experiment, but I am sure it’s mandated use.

And why label it an experiment? The very same engineers just went on stage and told us, how it improves their workflow! Why am I not witnessing any use of Copilot in other PRs? What am I missing here?

Of course there’s a chance I am getting the whole thing wrong and Microsoft engineers believe in this? Or am I witnessing a cult? Whatever it is, outside the distortion bubble this whole thing looks pretty bad.

## 2025-06-07: Fulltext Search Engine using SQLite and FTS5 Tables ##

I've started a new job in 2024, so my free-time budget for writing blog articles took a cut.

One of the projects I didn't write about is a Full-Text Search Engine using SQLite FTS5 and Blazor WASM:

* [https://github.com/bytefish/SqliteFulltextSearch](https://github.com/bytefish/SqliteFulltextSearch)

## 2025-06-07: Another "Google Zanzibar" Experiment with .NET ##

Relationship-based Access Control is an interesting approach to Authorization, with "Google Zanzibar" as an 
implementation described by Google. It's kind of a nemesis for me, because I have already tried to:

* Build something with OpenFGA, which is a Google Zanzibar implementation (Failed)
* Build a ANTLR4 Parser for the Google Zanzibar Configuration Language (Success, but not useful)
* Implement the Check and Expand API described in the Google Zanzibar paper (Maybe Success)

So of course I am now exploring it again and try to implement the Check API, Expand API and ListObjects API:

* [https://github.com/bytefish/GoogleZanzibarExperiments](https://github.com/bytefish/GoogleZanzibarExperiments)

I don't know yet, if this is going to be successful or where it is going.

## 2025-06-07: Notes or "Lowering the Bar for Writing" ##

This "notes page" is an idea, to make it easier for me to *write*, without the tedious work of long-form blog articles. Maybe notes on this page are something akin to a small tweet.

At the moment this is a single page with technical and non-technical content being mixed wildly. I am not sure, if it’s a good idea. We will see.

However it lowers the bar for writing and publishing by a lot and I am sure I’ll find a way to turn this into a more coherent piece.

As of now, I really like this format! 🚀 
