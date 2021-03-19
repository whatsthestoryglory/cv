---
title: "Ontario COVID-19 Active Case Tracking"
date: 2021-03-09T17:00:00-05:00
description: Ontario COVID-19 Active Case Tracking Map
hero: /images/posts/writing-posts/Report.svg
menu:
  sidebar:
      name: Ontario COVID
      identifier: covid
      weight: 12
---

Like many people, I have been drawn to the vast amounts of data being made available by governments as they track the COVID-19 pandemic.  As I progressed through the data loading and cleaning phases of learning data science, I thought there might be a cool opportunity for a quick visualization of the data I was playing with.

{{< leaflet >}}

While doing this I learned a ton about
- Locating and collecting data from the Ontario Government
- Cleaning and associating data where key fields were not used consistently
- Handling map data and simplifying geography so the leaflet didn't end up being huge and take a long time to render
- How to use leaflet

Read more about this on [my github](https://github.com/whatsthestoryglory/OntarioCOVID)