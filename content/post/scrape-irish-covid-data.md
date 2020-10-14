---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Scrape of Irish Covid Data"
subtitle: ""
summary: "The Health Protection Surveillance Centre (HPSC) and the Health Service Executive (HSE) provide daily updates of COVID on a datahub. This post details scraping that data and publishing to glcoud. 
"
authors: []
tags: [covid,datasette,gcloud, github actions]
categories: []
date: 2020-10-13T15:41:08+01:00
lastmod: 2020-10-13T15:41:08+01:00
featured: false
draft: false
---

The COVID-19 Data Hub as a number of [data sets](https://covid19ireland-geohive.hub.arcgis.com/search) available to download. After a bit of research one of the datasets contains cummulative counts of the COVID by Irish county. 

I wrote a [blog post]({{< relref "deploy-data-gcloud.md" >}}) about using using [Datasette](https://github.com/simonw/datasette) to publich data on gcloud. This post extends this to a target use case.. 

This is running on python with the follow libraries installed. 

```bash
pip3 install datasette sqlite-utils geojson-to-sqlite
```

The data is in geojson format. Each line includes all the geo data - this could be seperated out as it doesn't change. I played around with the [API](https://covid19ireland-geohive.hub.arcgis.com/datasets/d9be85b30d7748b5b7c09450b8aede63_0) but wasn't able to get the correct parameters for pulling the data without the geo info. 

So for now I grab the data, with geo data and then strip it out.  it.(:hourglass: There are probably better ways to achieve this)

```bash
curl https://opendata.arcgis.com/datasets/d9be85b30d7748b5b7c09450b8aede63_0.geojson \
    | geojson-to-sqlite covid-ireland.db dailycases_geo - --pk OBJECTID

sqlite3 covid-ireland.db <<'END_SQL'
.timeout 2000
CREATE TABLE dailycases AS SELECT OBJECTID, ORIGID,CountyName,PopulationCensus16,TimeStamp,
ConfirmedCovidCases,PopulationProportionCovidCases FROM dailycases_geo;
DROP TABLE IF EXISTS dailycases_geo;
END_SQL
```
Similar to previous [blog post]({{< relref "deploy-data-gcloud.md" >}}) I created a github action - that automatically runs using cron at 01:00 daily. 

```yaml
name: Fetch latest Irish COVID data at 1am

on:
  schedule:
    - cron: "0 1 * * *"
    
jobs:
  # This workflow contains a single job called "scheduled-fetch-data"
```

:exclamation: [Crontab Guro](https://crontab.guru/) has nice interface for working out cron expressions.

Thi [github folder](https://github.com/boboburo/data/tree/master/covid-ireland) contains script and meta data file and associated [workflow](https://github.com/boboburo/data/blob/master/.github/workflows/scheduled-covid.yml). 

This is up and running the data is [available here](https://covid-irl-sxsrkkt7yq-ew.a.run.app/covid-ireland/dailycases) 

(:exclamation: Sometimes the data link times out on first click with a *Service Unavailable* message. I think the pod is slow spinning up.  :hourglass: I need to add a custom domain)


> After I wrote this I found another source for [Irish Covid Data](https://covid19datahub.io/articles/iso/IRL.html)
