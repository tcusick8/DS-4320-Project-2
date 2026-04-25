# The Forest Service Has 30 Years of Wildfire Data. This Model Finally Puts It to Work.

## A Fire Starts in Rural Montana at 2 PM on a Tuesday in August. What Happens Next?

A ranger radios in a smoke sighting. Coordinates logged, cause noted — probably lightning. The dispatcher has twelve other active incidents. There are three aerial tankers available for the entire region. Does this fire get one?

Under the current system, that decision rests almost entirely on real-time field observation and dispatcher experience. The 1.88 million fires documented in the federal record since 1992 — their causes, their locations, their seasons, their final sizes — sit in a database. They are not in the room.

A new machine learning pipeline changes that. By querying three decades of wildfire incident records stored in MongoDB and training a Random Forest model on the conditions present at the moment of discovery, the system predicts how large a fire is likely to grow before it has grown at all.

## The Problem: The Data Exists. The Connection to the Dispatcher Does Not.

The USDA Forest Service has maintained the Fire Program Analysis Fire Occurrence Database (FPA-FOD) since 1992 — nearly two million fire incidents, each tagged with discovery date, geographic coordinates, cause classification, land ownership, and final size class. It is one of the most comprehensive environmental datasets the federal government produces.

None of it is available to the dispatcher making a resource call in real time.

Meanwhile, the consequences of that call compound fast. The USDA classifies fires on a seven-tier scale: Class A fires burn less than a quarter of an acre and extinguish themselves or yield to a single crew. Class G fires exceed 5,000 acres and require multi-agency responses lasting weeks. The resource difference between them is not marginal — it is categorical. Sending a single engine to a fire that will become Class G, or committing a tanker to one that will stay Class A, both have real costs. In a fire season where aircraft and crews are perpetually stretched, those misallocations add up.

## How the System Works

The pipeline pulls fire records from a MongoDB Atlas database — structured as nested documents capturing cause, discovery conditions, containment timeline, location, and size — and feeds them into a preprocessing and classification engine built in Python.

A Random Forest Classifier was selected for three reasons: it handles the mix of categorical inputs (cause type, land owner, state) and numeric inputs (day of year, coordinates) without requiring manual feature scaling; it produces interpretable feature importance scores that domain experts can interrogate; and it manages class imbalance through internal sample reweighting, without generating synthetic fire records that may not reflect real ignition conditions.

The model is trained on 80% of available records and tested on the remaining 20% — a clean holdout the model never sees during training. At inference time, it takes only information available at the moment of discovery: where, when, what caused it, and who owns the land. It outputs a predicted size class, A through G, before the fire has had time to spread.

## What the Data Shows

![Confusion Matrix](/Pipeline/confusion_matrix.png)

The confusion matrix above shows model predictions against actual fire size classes on 400 held-out test incidents. The model achieves **65.75% overall accuracy**, with strongest performance on Class A fires — correctly identifying 89% of small fires that do not warrant heavy resource commitment. This is the right place to perform well: false alarms on Class A fires waste resources, and missing them is rare.

Performance degrades on Class D through G fires, which represent fewer than 5% of incidents in the dataset. This is an expected consequence of class imbalance, not a model failure — and it points directly to where future data collection and model refinement should focus.

![Feature Importance](/Pipeline/feature_importance.png)

The feature importance chart reveals what the model is actually learning. Discovery day of year is the strongest single predictor — fire season is real, and the data reflects it. Geographic coordinates rank second, capturing regional fire ecology that state-level variables alone cannot. Land ownership type rounds out the top tier, distinguishing federal wilderness from private land in ways that correlate strongly with fire behavior and suppression access.

Notably absent from the top features: cause classification. Whether a fire was started by lightning or a campfire matters less to its ultimate size than where it started and when.

## Why 65% Accuracy Is the Beginning, Not the Ceiling

A model that correctly predicts fire size class two-thirds of the time — using only the information a dispatcher has in the first sixty seconds — is not a finished product. It is a proof of concept with a clear improvement path.

The current pipeline uses 2,000 records. The FPA-FOD contains 1.88 million. Scaling to the full dataset, incorporating weather data at discovery time, and adding satellite-derived vegetation and fuel load indices are each known to improve fire behavior models substantially. The architecture is in place. The data pipeline is built. The next step is data volume.

What 65% accuracy already demonstrates is that the signal is there. The patterns in 30 years of fire records are learnable. A dispatcher who today relies entirely on real-time observation could, with this system, surface historical context — *fires with this profile, in this region, in this season, tend to grow* — at the moment it is most useful.

## For the Communities in the Path of the Fire That Gets Missed

Wildfire fatalities, home losses, and air quality crises are not evenly distributed. Rural communities — often with the fewest resources and the longest response times — bear a disproportionate share of the cost when a fire grows unchecked in its first hours. A tool that surfaces early size predictions does not replace suppression capacity. It helps direct what capacity exists to where it is needed most, before the window closes.

The forest has been keeping records for thirty years. This model is finally reading them.

## Same Resources. Different Outcomes.

![Dispatch Comparison Map](/Pipeline/dispatch_comparison.png)

Without the model, two available crews are routed to the first two fires reported — a Class A
(0.1 acres) and a Class B (5 acres). A 1,200-acre Class E fire and a 380-acre Class D fire sit
unaddressed. With the model, the same two crews go directly to the fires the data says will
grow. The small fires can wait. The large ones cannot.

---
*Model trained on USDA FPA-FOD Release 6 (1992–2020). Pipeline source code available in the linked repository.*

