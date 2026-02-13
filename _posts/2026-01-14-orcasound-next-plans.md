Contemplating plans for [next.orcasound.net](http://next.orcasound.net/) this year. Should I create a separate zulip channel and/or standup? Or start moving features to the production app [live.orcasound.net](http://live.orcasound.net)? 

**Thesis**: accessible interpretation and storytelling layer on marine audio and environmental data that allows for an integrated read of multi-dimensional conditions, in real time or in a research context

**Potential partners / people I have chatted with:** \--

* Maritime Blue Ventures / Port of Seattle  
* UW eScience Scientific Software Engineering Center (SSEC)  
* Microsoft (Hack for Good, Microsoft Garage)  
* MarineSitu (local startup doing related work with marine mammal detection for offshore energy clients)

**Lower-effort goals (all front-end React/TS):**

* surfacing data filters in the UI (the filtering function already exists but I don't have a consolidated UI component built)  
* building a chart that counts detections of each type of animal or sound (there is already front-end Regex that scans detection comments and pulls out keywords)

**Higher effort goals (Python/FastAPI backend):**

* LLM generated summaries of current filtered data state, with RAG literature references \-- user can 'chat' by changing filters  
* natural language chat interface that can run SQL queries and pull up sample data views \-- user can change filters via chat  
* json snapshots at regular time intervals that give averages, stats, and combined data across listeners, AI audio processing models, and visual sightings (awaiting ambient-sound-analysis metrics from Valentina's team to fill this out more)

**Low-effort features that can be applied to Live already:**

* Candidates panel (right side of the map with filters and cards)  
* Map data (sightings, detections per node, audible radius)  
* Color theme using more of the brand palette  
* Expanded mobile navigation

**High-effort / high-priority – this a blocker for all other goals:**

* React state management for filtering and displaying multiple data sources \-- I am using Context API, but Paul needs to see a more robust solution using Zustand / Redux / etc

**Features awaiting orcasound/ambient-sound-analysis repo work:**

* In-browser spectrogram generation from PSD parquet files
* Candidate drawer with spectrogram (and "half height" to see underlying map view)
* Live listening drawer with spectrogram
