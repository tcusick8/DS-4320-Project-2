# Wildfire Prediction Pipeline
**DS 4320 - Project 2**

This notebook builds a dual-target prediction pipeline using 30 years of US wildfire records (1992–2020) stored in MongoDB.

**Targets:**
1. `FIRE_SIZE_CLASS` — how large a fire is likely to grow (A through G)
2. `STAT_CAUSE_DESCR` — what probably caused the fire

**Model:** Random Forest Classifier for both targets, with SMOTE oversampling for the cause prediction task where rare classes are heavily underrepresented.


```python
import logging

# ── Logging Configuration ─────────────────────────────────────────────
# logs to both console and a file for submission
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("pipeline.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)
logger.info("Logging initialized.")
```


```python
!pip install pymongo fiona geopandas --quiet
logger.info("Some libraries installed.")
```

    [2K   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m1.8/1.8 MB[0m [31m26.1 MB/s[0m eta [36m0:00:00[0m
    [2K   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m331.1/331.1 kB[0m [31m9.8 MB/s[0m eta [36m0:00:00[0m
    [?25h


```python
# ── Imports ───────────────────────────────────────────────────────────
import os
import logging
import warnings
import fiona
import pymongo
import numpy as np
import pandas as pd
import seaborn as sns
import geopandas as gpd
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches

from tqdm import tqdm
from dotenv import load_dotenv
from pymongo import MongoClient

from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import LabelEncoder, OrdinalEncoder, OneHotEncoder
from sklearn.metrics import (
    classification_report, confusion_matrix,
    ConfusionMatrixDisplay, accuracy_score
)
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from imblearn.over_sampling import SMOTE

# ── Logging Configuration ─────────────────────────────────────────────
# logs to both console and a persistent log file
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("pipeline.log"),  # persists logs for submission
        logging.StreamHandler()               # prints to notebook output
    ]
)
logger = logging.getLogger(__name__)

# ── Global Settings ───────────────────────────────────────────────────
warnings.filterwarnings('ignore')

# reproducibility seed used across all stochastic operations
RANDOM_STATE = 42

# Matplotlib style — serif fonts, clean axes
plt.rcParams.update({
    'font.family': 'serif',
    'axes.spines.top': False,
    'axes.spines.right': False,
    'figure.dpi': 150,
})

logger.info("Libraries loaded and configuration set.")
```


```python
# Download the USFS Fire Occurrence Database (FPA-FOD) zip archive from the official
# Forest Service Research Data Archive and save it locally as FPA_FOD_20221014.zip.
try:
    logger.info("Downloading USFS Fire Occurrence Database...")
    !wget -O FPA_FOD_20221014.zip "https://www.fs.usda.gov/rds/archive/products/RDS-2013-0009.6/RDS-2013-0009.6_Data_Format3_GPKG.zip"
    logger.info("Download complete.")

    # Extract the zip contents into the Data/ directory; the .gpkg file will land
    # at Data/Data/FPA_FOD_20221014.gpkg due to the archive's internal folder structure.
    logger.info("Extracting archive to Data/...")
    !unzip FPA_FOD_20221014.zip -d Data/
    logger.info("Extraction complete. File available at Data/Data/FPA_FOD_20221014.gpkg")

except Exception as e:
    # Log the error with full details and re-raise so the notebook surfaces the failure
    # rather than silently continuing with missing data.
    logger.error(f"Failed to download or extract data: {e}")
    raise
```

    --2026-04-25 15:47:57--  https://www.fs.usda.gov/rds/archive/products/RDS-2013-0009.6/RDS-2013-0009.6_Data_Format3_GPKG.zip
    Resolving www.fs.usda.gov (www.fs.usda.gov)... 52.235.253.124, 2001:489a:3604::7c0
    Connecting to www.fs.usda.gov (www.fs.usda.gov)|52.235.253.124|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 221225738 (211M) [application/x-zip-compressed]
    Saving to: ‘FPA_FOD_20221014.zip’
    
    FPA_FOD_20221014.zi 100%[===================>] 210.98M  7.08MB/s    in 41s     
    
    2026-04-25 15:48:38 (5.12 MB/s) - ‘FPA_FOD_20221014.zip’ saved [221225738/221225738]
    
    Archive:  FPA_FOD_20221014.zip
       creating: Data/Data/
      inflating: Data/Data/FPA_FOD_20221014.gpkg  
      inflating: Data/Data/_variable_descriptions.csv  



```python
# ── Load Raw Data ─────────────────────────────────────────────────────
# reads the Fires layer from the .gpkg file into a GeoDataFrame.
# the .gpkg contains multiple layers — Fires is the one with incident records.

FOLDER_PATH = "/content/Data/Data/FPA_FOD_20221014.gpkg"

try:
    logger.info(f"Loading Fires layer from {FOLDER_PATH}...")
    fires = gpd.read_file(FOLDER_PATH, layer="Fires")
    logger.info(f"Loaded {len(fires):,} fire records with {len(fires.columns)} columns.")

except FileNotFoundError:
    logger.error(f"File not found at {FOLDER_PATH}. Make sure the download and extraction ran successfully.")
    raise

except Exception as e:
    logger.error(f"Failed to load data: {e}")
    raise
```


```python
# ── Column Selection & Slimming ───────────────────────────────────────
# retains only the columns needed for analysis and MongoDB storage.
# date columns are cast to string since MongoDB handles them better
# as ISO strings than as pandas Timestamp objects.

#FOLDER_PATH = "/content/Data/Data/FPA_FOD_20221014.gpkg"

#logger.info(f"Loading Fires layer from {FOLDER_PATH}...")
#fires = gpd.read_file(FOLDER_PATH, layer="Fires")
#logger.info(f"Loaded {len(fires):,} fire records with {len(fires.columns)} columns.")

cols_to_keep = [
    'FOD_ID',                    # unique fire identifier
    'FIRE_YEAR',                 # year fire was discovered
    'DISCOVERY_DATE',            # full discovery date
    'DISCOVERY_DOY',             # discovery day of year (1-366)
    'NWCG_CAUSE_CLASSIFICATION', # broad cause: Human or Natural
    'NWCG_GENERAL_CAUSE',        # more specific cause category
    'CONT_DATE',                 # containment date
    'CONT_DOY',                  # containment day of year
    'FIRE_SIZE',                 # size in acres
    'FIRE_SIZE_CLASS',           # size class A-G (target variable)
    'LATITUDE',                  # fire origin latitude
    'LONGITUDE',                 # fire origin longitude
    'STATE',                     # US state abbreviation
    'COUNTY',                    # county name
    'OWNER_DESCR'                # land owner/agency description
]

try:
    logger.info("Slimming dataset to selected columns...")
    fires_slim = fires[cols_to_keep].copy()

    # cast dates to string for clean MongoDB storage
    fires_slim['DISCOVERY_DATE'] = fires_slim['DISCOVERY_DATE'].astype(str)
    fires_slim['CONT_DATE'] = fires_slim['CONT_DATE'].astype(str)

    logger.info(f"Slim shape: {fires_slim.shape}")
    logger.info(f"Estimated in-memory size: {fires_slim.memory_usage(deep=True).sum() / 1e6:.1f} MB")

except KeyError as e:
    logger.error(f"Expected column not found in dataset: {e}")
    raise

except Exception as e:
    logger.error(f"Failed during column selection: {e}")
    raise
```


```python
def shape_document(row):
    """
    Converts a single row from the fires DataFrame into a nested
    MongoDB document. Groups related fields into sub-documents
    (cause, discovery, containment, size, location) for cleaner
    schema design and easier querying in Atlas.

    Args:
        row (pd.Series): a single row from fires_slim

    Returns:
        dict: a nested MongoDB-ready document
    """
    try:
        return {
            "fod_id":    row.get("FOD_ID"),
            "fire_year": row.get("FIRE_YEAR"),
            "cause": {
                "classification": row.get("NWCG_CAUSE_CLASSIFICATION"),
                "general_cause":  row.get("NWCG_GENERAL_CAUSE")
            },
            "discovery": {
                "date":        row.get("DISCOVERY_DATE"),
                "day_of_year": row.get("DISCOVERY_DOY")
            },
            "containment": {
                "date":        row.get("CONT_DATE"),
                "day_of_year": row.get("CONT_DOY")
            },
            "size": {
                "acres":      row.get("FIRE_SIZE"),
                "size_class": row.get("FIRE_SIZE_CLASS")
            },
            "location": {
                "latitude":  row.get("LATITUDE"),
                "longitude": row.get("LONGITUDE"),
                "state":     row.get("STATE"),
                "county":    row.get("COUNTY"),
                "owner":     row.get("OWNER_DESCR")
            }
        }

    except Exception as e:
        logger.error(f"Failed to shape document for FOD_ID {row.get('FOD_ID', 'UNKNOWN')}: {e}")
        raise


# ── Shape All Documents ───────────────────────────────────────────────
try:
    logger.info("Shaping DataFrame rows into MongoDB documents...")
    fires_docs = fires_slim.apply(shape_document, axis=1).tolist()
    logger.info(f"Shaped {len(fires_docs):,} documents successfully.")
    logger.info(f"Sample document: {fires_docs[0]}")

except Exception as e:
    logger.error(f"Failed during document shaping: {e}")
    raise
```


```python
# ── Connect to MongoDB Atlas ──────────────────────────────────────────
#MONGO_URI is stored in .env file as:
#MONGO_URI=mongodb+srv://username:password@cluster0.xxx.mongodb.net/?appName=Cluster0
# never hardcode credentials — load from environment instead.

try:
    load_dotenv()  # loads variables from .env file
    logger.info("Environment variables loaded from .env.")

    MONGO_URI = os.getenv("MONGO_URI")  # reads URI from .env instead of hardcoding

    if not MONGO_URI:
        raise ValueError("MONGO_URI not found in environment. Check your .env file.")

    client = pymongo.MongoClient(MONGO_URI)
    db = client["ds4320"]
    collection = db["hw10"]

    # ping Atlas to confirm connection before proceeding
    client.admin.command("ping")
    logger.info("Successfully connected to MongoDB Atlas.")
    logger.info(f"Database: {db.name} | Collection: {collection.name}")

except ValueError as e:
    logger.error(f"Missing environment variable: {e}")
    raise

except pymongo.errors.ConnectionFailure as e:
    logger.error(f"MongoDB connection failed: {e}")
    raise

except Exception as e:
    logger.error(f"Unexpected error during MongoDB connection: {e}")
    raise
```


```python
def check_storage_mb():
    """
    Queries Atlas dbStats to return current storage usage in MB.
    Useful for monitoring against the 512MB M0 free tier cap.

    Returns:
        float: storage used in MB
    """
    try:
        stats = db.command("dbStats")
        return stats["storageSize"] / (1024 * 1024)
    except Exception as e:
        logger.error(f"Failed to retrieve storage stats: {e}")
        raise

# ── Insert Documents into Atlas ───────────────────────────────────────
# drops the collection first to avoid duplicates on re-runs,
# then inserts up to 300k documents to stay under the 512MB M0 cap.

try:
    logger.info("Dropping existing collection to ensure clean insert...")
    collection.drop()
    logger.info("Collection cleared and ready.")

    storage_before = check_storage_mb()
    logger.info(f"Storage before insert: {storage_before:.1f} MB / 512 MB")

    #meets the 1000 documents requirement
    docs_to_insert = fires_docs[:2000]

    logger.info(f"Inserting {len(docs_to_insert):,} documents into Atlas...")

    collection.insert_many(docs_to_insert)

    final_count = collection.count_documents({})
    final_storage = check_storage_mb()

    logger.info(f"Insert complete. Records in collection: {final_count:,}")
    logger.info(f"Storage after insert: {final_storage:.1f} MB / 512 MB")

except pymongo.errors.BulkWriteError as e:
    logger.error(f"Bulk write error during insert: {e.details}")
    raise

except Exception as e:
    logger.error(f"Failed during document insertion: {e}")
    raise
```


```python
# ── Query MongoDB → DataFrame ─────────────────────────────────────────
# aggregation pipeline flattens the nested MongoDB document structure
# back into a tabular DataFrame for analysis. metadata sub-document
# is excluded here since it contains supplementary fields not needed
# for modeling.

pipeline = [
    {
        "$project": {
            "_id": 0,
            # core fields
            "fod_id":        "$fod_id",
            "fire_year":     "$fire_year",
            # cause sub-document
            "cause_class":   "$cause.classification",
            "general_cause": "$cause.general_cause",
            # discovery sub-document
            "discovery_doy": "$discovery.day_of_year",
            "discovery_date": "$discovery.date",
            # containment sub-document
            "cont_date":     "$containment.date",
            "cont_doy":      "$containment.day_of_year",
            # size sub-document
            "size_acres":    "$size.acres",
            "size_class":    "$size.size_class",
            # location sub-document
            "latitude":      "$location.latitude",
            "longitude":     "$location.longitude",
            "state":         "$location.state",
            "county":        "$location.county",
            "owner":         "$location.owner"
        }
    }
]

try:
    logger.info("Running aggregation pipeline to pull data from Atlas...")
    cursor = collection.aggregate(pipeline)
    df = pd.DataFrame(list(cursor))
    logger.info(f"Records loaded: {len(df):,}")
    logger.info(f"Columns: {df.columns.tolist()}")

except pymongo.errors.OperationFailure as e:
    logger.error(f"MongoDB aggregation failed: {e}")
    raise

except Exception as e:
    logger.error(f"Failed to load data from Atlas: {e}")
    raise

finally:
    # close connection after data is loaded regardless of success or failure
    client.close()
    logger.info("MongoDB connection closed.")

print(f"Records loaded: {len(df):,}")
print(f"Columns: {df.columns.tolist()}")
df.head()
```

    Records loaded: 2,000
    Columns: ['fod_id', 'fire_year', 'cause_class', 'general_cause', 'discovery_doy', 'discovery_date', 'cont_date', 'cont_doy', 'size_acres', 'size_class', 'latitude', 'longitude', 'state', 'county', 'owner']






  <div id="df-87860d4f-e03c-4b20-8638-ed2120dea745" class="colab-df-container">
    <div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fod_id</th>
      <th>fire_year</th>
      <th>cause_class</th>
      <th>general_cause</th>
      <th>discovery_doy</th>
      <th>discovery_date</th>
      <th>cont_date</th>
      <th>cont_doy</th>
      <th>size_acres</th>
      <th>size_class</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>state</th>
      <th>county</th>
      <th>owner</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>2005</td>
      <td>Human</td>
      <td>Power generation/transmission/distribution</td>
      <td>33</td>
      <td>2/2/2005</td>
      <td>2/2/2005</td>
      <td>33.0</td>
      <td>0.10</td>
      <td>A</td>
      <td>40.036944</td>
      <td>-121.005833</td>
      <td>CA</td>
      <td>63</td>
      <td>USFS</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>2004</td>
      <td>Natural</td>
      <td>Natural</td>
      <td>133</td>
      <td>5/12/2004</td>
      <td>5/12/2004</td>
      <td>133.0</td>
      <td>0.25</td>
      <td>A</td>
      <td>38.933056</td>
      <td>-120.404444</td>
      <td>CA</td>
      <td>61</td>
      <td>USFS</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>2004</td>
      <td>Human</td>
      <td>Debris and open burning</td>
      <td>152</td>
      <td>5/31/2004</td>
      <td>5/31/2004</td>
      <td>152.0</td>
      <td>0.10</td>
      <td>A</td>
      <td>38.984167</td>
      <td>-120.735556</td>
      <td>CA</td>
      <td>17</td>
      <td>STATE OR PRIVATE</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>2004</td>
      <td>Natural</td>
      <td>Natural</td>
      <td>180</td>
      <td>6/28/2004</td>
      <td>7/3/2004</td>
      <td>185.0</td>
      <td>0.10</td>
      <td>A</td>
      <td>38.559167</td>
      <td>-119.913333</td>
      <td>CA</td>
      <td>3</td>
      <td>USFS</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>2004</td>
      <td>Natural</td>
      <td>Natural</td>
      <td>180</td>
      <td>6/28/2004</td>
      <td>7/3/2004</td>
      <td>185.0</td>
      <td>0.10</td>
      <td>A</td>
      <td>38.559167</td>
      <td>-119.933056</td>
      <td>CA</td>
      <td>3</td>
      <td>USFS</td>
    </tr>
  </tbody>
</table>
</div>
    <div class="colab-df-buttons">

  <div class="colab-df-container">
    <button class="colab-df-convert" onclick="convertToInteractive('df-87860d4f-e03c-4b20-8638-ed2120dea745')"
            title="Convert this dataframe to an interactive table."
            style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 -960 960 960">
    <path d="M120-120v-720h720v720H120Zm60-500h600v-160H180v160Zm220 220h160v-160H400v160Zm0 220h160v-160H400v160ZM180-400h160v-160H180v160Zm440 0h160v-160H620v160ZM180-180h160v-160H180v160Zm440 0h160v-160H620v160Z"/>
  </svg>
    </button>

  <style>
    .colab-df-container {
      display:flex;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    .colab-df-buttons div {
      margin-bottom: 4px;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

    <script>
      const buttonEl =
        document.querySelector('#df-87860d4f-e03c-4b20-8638-ed2120dea745 button.colab-df-convert');
      buttonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';

      async function convertToInteractive(key) {
        const element = document.querySelector('#df-87860d4f-e03c-4b20-8638-ed2120dea745');
        const dataTable =
          await google.colab.kernel.invokeFunction('convertToInteractive',
                                                    [key], {});
        if (!dataTable) return;

        const docLinkHtml = 'Like what you see? Visit the ' +
          '<a target="_blank" href=https://colab.research.google.com/notebooks/data_table.ipynb>data table notebook</a>'
          + ' to learn more about interactive tables.';
        element.innerHTML = '';
        dataTable['output_type'] = 'display_data';
        await google.colab.output.renderOutput(dataTable, element);
        const docLink = document.createElement('div');
        docLink.innerHTML = docLinkHtml;
        element.appendChild(docLink);
      }
    </script>
  </div>


    </div>
  </div>





```python
def prepare_features(df):
    """
    Drops leakage and high-cardinality columns, handles nulls,
    and separates features from target.

    Drops:
        - size_acres: directly determines size_class (leakage)
        - county: too high cardinality, low signal

    Returns:
        X (DataFrame): feature matrix
        y (Series): target labels (size_class)
    """
    try:
        logger.info("Starting feature preparation...")

        # drop leakage column and low-signal column
        drop_cols = ["size_acres", "county"]
        df_model = df.drop(columns=drop_cols)
        logger.info(f"Dropped columns: {drop_cols}")

        # drop rows where target is null
        df_model = df_model.dropna(subset=["size_class"])
        logger.info(f"Rows after dropping null targets: {len(df_model):,}")

        # separate features and target
        X = df_model.drop(columns=["size_class"])
        y = df_model["size_class"]

        logger.info(f"Feature matrix shape: {X.shape}")
        logger.info(f"Target distribution:\n{y.value_counts()}")

        return X, y

    except Exception as e:
        logger.error(f"Error in prepare_features: {e}")
        raise

X, y = prepare_features(df)
```

## Analysis Rationale

**Why Random Forest?**
Random Forest was chosen over logistic regression or a single decision tree for three reasons: it handles mixed feature types (categorical + numeric) naturally, it is robust to outliers in continuous features like `discovery_doy` and `fire_year`, and its built-in `feature_importances_` provides direct interpretability for a domain audience.

**Why `class_weight="balanced"` instead of SMOTE?**
SMOTE operates on a dense numeric feature matrix, but our data contains categorical columns that must be one-hot encoded first. Applying SMOTE after OHE produces synthetic samples that mix encoded categories in ways that may not correspond to real observations. `class_weight="balanced"` achieves the same reweighting effect without generating potentially invalid synthetic records.

**Why drop `size_acres`?**
`FIRE_SIZE_CLASS` is defined directly from `size_acres` via a fixed acreage threshold table (NWCG standard). Including it would be target leakage — the model would memorize the binning rule rather than learn any predictive signal.

**Why drop `county`?**
County has thousands of unique values across 30 years of records. One-hot encoding it would produce a massive, sparse feature matrix where most counties appear only a handful of times — adding noise, not signal. `STATE` captures the geographic signal at a more stable granularity.

**Why stratified train/test split?**
Fire size classes are heavily imbalanced (class A fires vastly outnumber class G). A random split risks putting rare classes almost entirely in train or test. `stratify=y` ensures every class is proportionally represented in both sets.


```python
def build_pipeline(X):
    """
    Builds a sklearn Pipeline with:
        - OneHotEncoder for categorical columns
        - passthrough for numeric columns
        - RandomForestClassifier as the model
    SMOTE is applied separately before fitting due to pipeline constraints.

    Args:
        X (DataFrame): feature matrix, used to infer column types

    Returns:
        pipeline (Pipeline): unfitted sklearn pipeline
    """
    try:
        logger.info("Building preprocessing and model pipeline...")

        # identify categorical vs numeric columns
        cat_cols = X.select_dtypes(include=["object", "category"]).columns.tolist()
        num_cols = X.select_dtypes(include=["number"]).columns.tolist()
        logger.info(f"Categorical columns: {cat_cols}")
        logger.info(f"Numeric columns: {num_cols}")

        # preprocessing: OHE for categoricals, passthrough for numerics
        preprocessor = ColumnTransformer(transformers=[
            ("cat", OneHotEncoder(handle_unknown="ignore", sparse_output=False), cat_cols),
            ("num", "passthrough", num_cols)
        ])

        # full pipeline: preprocess → random forest
        pipeline = Pipeline(steps=[
            ("preprocessor", preprocessor),
            ("classifier", RandomForestClassifier(
                n_estimators=100,
                random_state=RANDOM_STATE,
                n_jobs=-1,        # use all available cores
                class_weight="balanced"  # handles class imbalance without SMOTE
            ))
        ])

        logger.info("Pipeline built successfully.")
        return pipeline

    except Exception as e:
        logger.error(f"Error in build_pipeline: {e}")
        raise

pipeline = build_pipeline(X)
```


```python
def train_model(pipeline, X, y):
    """
    Splits data into train/test sets and fits the pipeline.
    Uses stratified split to preserve class distribution.

    Args:
        pipeline: unfitted sklearn Pipeline
        X (DataFrame): feature matrix
        y (Series): target labels

    Returns:
        pipeline: fitted pipeline
        X_test, y_test: held-out test set for evaluation
    """
    try:
        logger.info("Splitting data into train/test sets...")

        X_train, X_test, y_train, y_test = train_test_split(
            X, y,
            test_size=0.2,
            random_state=RANDOM_STATE,
            stratify=y  # preserve class balance in both splits
        )
        logger.info(f"Train size: {len(X_train):,} | Test size: {len(X_test):,}")

        logger.info("Fitting pipeline...")
        pipeline.fit(X_train, y_train)
        logger.info("Model training complete.")

        return pipeline, X_test, y_test

    except Exception as e:
        logger.error(f"Error in train_model: {e}")
        raise

pipeline, X_test, y_test = train_model(pipeline, X, y)
```

## Visualization Rationale

**Confusion Matrix**
A confusion matrix was chosen over a simple accuracy score because fire size classes are ordinal (A < B < C … < G). A scalar accuracy metric treats all errors equally, but misclassifying a class-A fire (< 0.25 acres) as class-G (> 5000 acres) is far more consequential than a one-class miss. The matrix makes the error distribution visible — we can see whether the model's mistakes are adjacent-class (acceptable) or cross-range (alarming). The `Oranges` colormap was chosen to reinforce the fire theme without sacrificing readability.

**Feature Importance Bar Chart**
The horizontal bar chart of the top 15 Mean Decrease in Impurity (MDI) scores answers a practical question for fire managers: *which variables matter most?* MDI is the native importance measure of Random Forest and requires no additional computation. Limiting to 15 features keeps the chart readable — beyond that, importances are small enough to be indistinguishable. This chart also serves as a sanity check: if `fire_year` or `discovery_doy` dominate, it suggests temporal rather than geographic patterns are driving fire size, which is meaningful domain knowledge.


```python
def evaluate_model(pipeline, X_test, y_test):
    """
    Evaluates the fitted model on the held-out test set.
    Prints accuracy, classification report, and renders
    a confusion matrix.

    Args:
        pipeline: fitted sklearn Pipeline
        X_test (DataFrame): test features
        y_test (Series): true test labels
    """
    try:
        logger.info("Evaluating model on test set...")

        # strip OHE/passthrough prefixes from feature names for clean labels
        y_pred = pipeline.predict(X_test)
        acc = accuracy_score(y_test, y_pred)
        logger.info(f"Test accuracy: {acc:.4f}")
        print(f"\nAccuracy: {acc:.4f}")
        print("\nClassification Report:")
        print(classification_report(y_test, y_pred))

        fig, ax = plt.subplots(figsize=(8, 6))
        ConfusionMatrixDisplay.from_predictions(
            y_test, y_pred,
            ax=ax,
            colorbar=False,
            cmap="Oranges",
            values_format="d",        # show integers, not scientific notation
            normalize=None
        )
        ax.set_title("Fire Size Class — Confusion Matrix", fontsize=13, fontweight="bold")
        ax.set_xlabel("Predicted Class", fontsize=11)
        ax.set_ylabel("True Class", fontsize=11)
        # publication caption below the figure
        fig.text(
            0.5, -0.02,
            f"Random Forest Classifier  |  Test set n={len(y_test):,}  |  Accuracy={acc:.3f}",
            ha="center", fontsize=9, color="#555555"
        )
        plt.tight_layout()
        plt.savefig("confusion_matrix.png", dpi=150, bbox_inches="tight")
        plt.show()

        logger.info("Confusion matrix saved to confusion_matrix.png")

    except Exception as e:
        logger.error(f"Error in evaluate_model: {e}")
        raise

evaluate_model(pipeline, X_test, y_test)
```

    
    Accuracy: 0.6575
    
    Classification Report:
                  precision    recall  f1-score   support
    
               A       0.73      0.89      0.80       261
               B       0.39      0.25      0.31       106
               C       0.25      0.13      0.17        23
               D       0.00      0.00      0.00         4
               E       0.00      0.00      0.00         3
               F       0.00      0.00      0.00         2
               G       0.00      0.00      0.00         1
    
        accuracy                           0.66       400
       macro avg       0.20      0.18      0.18       400
    weighted avg       0.59      0.66      0.62       400
    



    
![png](File_1_Fire_Pipeline_files/File_1_Fire_Pipeline_16_1.png)
    



```python
def plot_feature_importance(pipeline, X):
    """
    Extracts and plots feature importances from the trained
    RandomForestClassifier after preprocessing.
    Top 15 features shown for readability.

    Args:
        pipeline: fitted sklearn Pipeline
        X (DataFrame): feature matrix (used to recover feature names)
    """
    try:
        logger.info("Extracting feature importances...")

        # recover feature names after OHE expansion

        feature_names = pipeline.named_steps["preprocessor"].get_feature_names_out()
        importances = pipeline.named_steps["classifier"].feature_importances_

        importance_df = pd.DataFrame({
            "feature": feature_names,
            "importance": importances
        }).sort_values("importance", ascending=False).head(15)

        # strip sklearn prefixes (e.g. "cat__STATE_CA" → "STATE_CA")
        importance_df["feature"] = importance_df["feature"].str.replace(
            r"^(cat__|num__)", "", regex=True
        )

        # plot
        fig, ax = plt.subplots(figsize=(9, 5))
        sns.barplot(data=importance_df, x="importance", y="feature", ax=ax, color="#C0392B")
        ax.set_title(
            "Top 15 Feature Importances — Fire Size Class Prediction",
            fontsize=13, fontweight="bold"
        )
        ax.set_xlabel("Mean Decrease in Impurity (MDI)", fontsize=11)
        ax.set_ylabel("")
        ax.bar_label(ax.containers[0], fmt="%.4f", padding=3, fontsize=8, color="#333333")
        fig.text(
            0.5, -0.02,
            "Importances are normalized MDI scores from the fitted RandomForestClassifier.",
            ha="center", fontsize=9, color="#555555"
        )
        plt.tight_layout()
        plt.savefig("feature_importance.png", dpi=150, bbox_inches="tight")
        plt.show()


        logger.info("Feature importance plot saved to feature_importance.png")

    except Exception as e:
        logger.error(f"Error in plot_feature_importance: {e}")
        raise

plot_feature_importance(pipeline, X)
```


    
![png](File_1_Fire_Pipeline_files/File_1_Fire_Pipeline_17_0.png)
    

