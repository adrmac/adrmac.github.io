---
title: "React user interface - temperature / humidity anomaly monitor"
date: 2026-03-10
---

This week I am launching a temperature/humidity data monitoring interface built with TypeScript / React / Next.js: [https://esp32ui.vercel.app](https://esp32ui.vercel.app). 

The app displays live data from the ESP32 sensor in my home office, mediated by a cloud-hosted Python API and PosgreSQL database. It includes a filter set for viewing the data over different time periods, aggregation buckets, and other adjustments. A user-controlled threshold setting defines the list of noteworthy Events. Each Event includes badges showing the severity of anomalies across three key metrics, and an interpretive description.

___
<img width="1705" height="942" alt="index1" src="https://github.com/user-attachments/assets/5cec38ab-9264-4e1d-a257-db6f82a2d334" />

_Home screen_
____

#### Technical summary
- I added a new `/timeseries` endpoint to the [serverless Python API deployed on Cloud Run](https://adrmac.github.io/2026/03/06/serverless-deployment-with-cloud-run.html) for fetching raw data and aggregating averages by time bucket using SQL.
- I am using React Query by Tanstack to fetch and cache data requests in the browser.
- I am iterating on the interface using the Codex plugin in VS Code to ‘vibe-code’ in TypeScript with the Devias Pro MUI UI Kit template and Apex Charts.

___
#### Approach
This is an exploratory prototype. My focus is on charting out one data channel, and making it legible and useful. Future iterations could add other channels like sound, air quality, or motion. For now, temperature / humidity is a good starting point that aligns with new city and state legislation to track Building Energy Performance (Seattle BEPS / Washington State CBPS) for multifamily buildings like our historic 120-year old co-op. 

**User story:**
_"As a building occupant or manager, I need to interpret current temperature/humidity readings in the context of recent time periods and the local climate, so I can better understand environmental conditions in my space."_

The interface needs to: 
- display the data
- determine anomalies and normal state by calculating baselines and thresholds
- provide visual and textual assistance for interpreting the data patterns

___
<img width="868" height="503" alt="baseline" src="https://github.com/user-attachments/assets/8c9bc4da-59ad-4b6e-bc31-15338a84c51b" />

_30-day hour-of-day baselines with +/-5% threshold bands_
___


#### Design
In preparing this prototype for testing, I’ve thought a lot about showing and explaining baselines, thresholds, and anomalies in an intuitive way. 

I looked at the relationship between temperature and relative humidity – they are roughly inverse metrics, because warmer air can hold more moisture. It's important to understand the distinction between relative and absolute humidity:
- Relative humidity (RH) is the percentage toward full moisture saturation, relative to the air temperature, and represents the humidity we feel. 
- Absolute humidity (AH) is g / m3 of moisture in the air, and can be derived from knowing temperature and RH. 

The interlocking dynamics of those three metrics can be hard to visually parse. 

The first problem is that temperature and absolute humidity operate independently, making it difficult to see which is having a stronger influence on the relative humidity. A temperature spike drives RH down, while a moisture spike drives it up, and vice versa. At any given moment the two forces could be opposing, resonating, or in stasis.

The second problem is that each time series has different units -- degrees F, % saturation, and g/m3, so it's necessary to make decisions about how they scale when charted next to each other.

To get a visual handle on this, I plotted a dashed-line curve for what the "expected" RH would be at a given hour of day based on temperature, given average baseline moisture. This yields a roughly inverse curve to temperature, that I can use to vertically scale RH % units against degrees F. 

To scale the moisture curve in g/m3, I then roughly matched the peaks to the RH curve, since the RH min/max events are typically aligned with AH min/max. 
___

<img width="922" height="409" alt="main-chart" src="https://github.com/user-attachments/assets/e384c422-23a9-4f41-8617-3472a104946e" />

_Combined last 7 days with 30-day threshold bands_
___

In the above chart, the most obvious pattern is the spike in both absolute moisture (blue) and relative humidity (orange) over the weekend. Temperature also dropped slightly, pushing the felt humidity a little higher. 

Was this due to our occupancy patterns over the weekend? One way to compare is by looking at outdoor weather data: 

___
<img width="932" height="545" alt="image" src="https://github.com/user-attachments/assets/97d9c04a-776f-438d-a29a-6f72e5a78f83" />

_Outdoor moisture from both the NOAA station at Boeing Field, and a localized prediction model, vs. indoor moisture from the ESP32 sensor_
___

Given this context, exterior weather probably had a stronger impact on the indoor relative humidity than we did. The exterior climate had actually been moist for several days prior, but our relative humidity wasn't as elevated because our air was warm. Then a slight temperature dip over the weekend pushed RH above threshold. 

Did we feel it? Did we have a window open? I don't remember, I was taking a nap. Or I was in the zone writing a blog post. Or I was picking up the kids. I'm not necessarily in tune with my surroundings all the time, maybe an app like this helps. Regardless, if I'm getting an alert on my phone about a threshold breach, I want to see an explanation of what's happening so I know what to make of it. 

___
<img width="759" height="758" alt="card-detail" src="https://github.com/user-attachments/assets/198ad203-3425-4a9e-8d75-9b1806c7707d" />

_A 3-day period of unusually high relative humidity driven by a combination of below-average temperature and high moisture from the outside weather_
___

### Next steps

There are many ways to go from here. Some immediate thoughts:

**Improving performance**

The initial page load is sluggish, and there are too many queries to the database that should be cached. On the backend, there has already been a pass to try to optimize the SQL query which currently takes around 4s. We could try to shorten this by precomputing aggregates and storing them in additional Postgres tables.

I don't think "cold start" from my serverless API would be a problem because my ESP32 sensor is posting to it every 2 seconds, so it should never be spinning down to zero. I should verify this in the GCP dashboard.

In a future iteration, I could incorporate seasonal cycles into the baseline and give context about how much current readings differ from the annual average. This would require querying over a year of data, so most likely I would want to do it periodically on the server-side, not "on the fly" based on a client request. 

**Improving UX**

While I felt like the pieces were in place for a functional MVP, there are a few fast-follow improvements I'd like to make. 

_Filters_

The filters are awkwardly stuffed into the heading line on the dashboard. I like having them available in one click, rather than hidden in a modal, but in their current configuration I find them distracting from the visual hierarchy which should focus on the heading line and current metrics. They are also working double duty as both controls and as indicators of the current state, e.g. 'last 30 days', which users might miss if they are not parsing that level of information density. 

A cleaner solution would be to put the filters in a side drawer on the left, where they can be accessed all at once without opening a modal or even a selection dropdown. The left-side drawer real estate is currently taken by the vertical navigation bar, which isn't needed and could easily move to a horizontal top nav or hamburger menu. 

_Event details_

I'd like to rework the text of the detail drawers to include more comprehensive metrics on each event. What's needed is really a matrix of data points. Across temperature, relative humidity, and absolute moisture, I want to see:
- observed average
- baseline average
- average deviation
- observed maximum
- maximum deviation
- \# of threshold breaches
- % of time within thresholds
- data completeness / confidence

If I later have an LLM agent summarizing the event or comparing it to other snapshots, I want to give it useful details to work with.

_Data completeness_

There should be a separate dashboard panel that shows whether there have been any dropouts from the data stream, or any slowdowns in the pace of data from an expected 2-sec cadence. This could affect the interpretation of the event if there is some correlation between data dropouts and unusual readings, so makes sense to include in the same vertical stack as other time series on the main page. 

_Mobile_

The mobile experience is going to be important for this app from a user perspective. People want to be able to check on things wherever they are. As the designer, I want to be able to show it off as a demo when I'm out networking. The Devias Pro Kit comes in handly here because it ships with mobile-ready component patterns, but I still need to do a pass through it before I can fully user test or publish a link.
