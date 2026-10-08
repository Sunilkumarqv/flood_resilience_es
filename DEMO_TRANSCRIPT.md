# Valencia Flood Vulnerability Solution - 2-Hour Demo Transcript

**Audience:** Mixed (non-technical stakeholders, students, technical engineers)
**Duration:** 2 hours (120 minutes)
**Format:** Live hands-on walkthrough with explanations

---

## PART 1: INTRODUCTION AND CONTEXT (15 minutes)

---

### [0:00 - 0:05] Opening and Welcome

> **Presenter:**
>
> "Good morning/afternoon everyone, and welcome. Today we are going to build something real together -- a flood vulnerability analysis platform for one of the most flood-prone regions in Europe: Comunitat Valenciana, Spain.
>
> Before we touch a single line of code, let me explain *why* this matters.
>
> In October 2024, a weather event called a DANA -- which stands for 'Depresion Aislada en Niveles Altos', essentially a cold drop that forms when a pocket of cold air high in the atmosphere gets cut off from the jet stream -- hit the Valencia region. It dumped an entire year's worth of rainfall in just eight hours. Over 200 people lost their lives. Entire neighbourhoods were destroyed. The economic damage exceeded 10 billion euros.
>
> Now here is the critical question: *Could we have known which buildings, which neighbourhoods, which communities were most at risk before the event happened?*
>
> That is exactly what we are going to build today. By the end of this session, you will have:
>
> 1. A geospatial pipeline analyzing **2 million+ real building footprints** across Valencia
> 2. A **flood risk scoring system** that combines physical hazard data with social vulnerability
> 3. **AI-powered document intelligence** that can read and understand government flood mitigation plans
> 4. An **interactive dashboard** with maps and charts for decision-makers
> 5. A **conversational AI agent** that anyone -- even non-technical people -- can ask questions about flood risk in plain language
>
> And we will do all of this inside Snowflake, using a single platform."

---

### [0:05 - 0:10] Explaining Key Concepts for Everyone

> **Presenter:**
>
> "Before we dive in, let me quickly explain some terms that will come up repeatedly. I want everyone -- regardless of your background -- to follow along comfortably.
>
> **For the non-technical audience, here are the essential terms:**
>
> - **Snowflake**: Think of it as a very powerful cloud-based data warehouse. Imagine a giant spreadsheet system in the cloud that can handle billions of rows of data and run calculations on them in seconds.
>
> - **Comarca**: This is a Spanish administrative district. Think of it like a county in the US or a Landkreis in Germany. Valencia has 34 comarcas.
>
> - **PATRICOVA**: This is Spain's official flood risk zoning system for the Valencia region. It classifies areas into zones:
>   - **Zone A1**: High risk -- frequent flooding from both rivers and the coast. No new buildings allowed.
>   - **Zone A2**: Moderate-high risk -- occasional flooding. Buildings need special measures like elevated ground floors.
>   - **Zone B**: Some risk -- geological flood risk. Building allowed with conditions.
>   - **Zone C**: Low risk -- standard building regulations apply.
>
> - **H3 Hexagonal Index**: Imagine covering a map with tiny hexagons, like a honeycomb. Each hexagon covers a specific area of land. This is how we link buildings to risk zones -- it is far more efficient than checking each building one by one.
>
> - **Social Vulnerability Index (SVI)**: A score from 0 to 1 that measures how vulnerable a community is -- not because of physical flood risk, but because of social factors: poverty levels, elderly population percentage, housing quality, access to transportation. A community with many elderly residents who cannot drive will struggle to evacuate even if the flood warning comes early.
>
> - **Composite Vulnerability Score**: Our custom score (0-100) that combines three things:
>   - Physical flood risk (40% weight)
>   - Social vulnerability (30% weight)
>   - Flood zone exposure (30% weight)
>
>   A building in a high-risk flood zone, in a poor neighbourhood with many elderly residents, will score much higher than a building in the same flood zone but in a wealthy, young neighbourhood with good infrastructure.
>
> **For the technical audience**, we will be using: Snowflake Marketplace, geospatial SQL functions, H3 hexagonal indexing, Cortex AI (PARSE_DOCUMENT, Search, COMPLETE), Dynamic Tables, Streamlit in Snowflake, Cortex Agents with Cortex Analyst, and Apache Ossie semantic models. I will explain each as we encounter them."

---

### [0:10 - 0:15] Architecture Overview

> **Presenter:**
>
> "Let me show you the architecture of what we are building. Think of it as a pipeline with three main stages:
>
> **Stage 1 -- Data Collection:**
> We pull building footprints from the Snowflake Marketplace (this is like an app store for data -- someone has already collected 2.3 billion building outlines worldwide, and we just 'install' it). We also load our own flood risk data and social vulnerability data from CSV files, plus two PDF documents of Valencia's Flood Mitigation Plan.
>
> **Stage 2 -- Analysis:**
> We use geospatial functions and H3 hexagons to figure out which buildings are in which flood zones and which comarcas. We calculate a composite vulnerability score for every single building. We create summary tables and a dynamic alert system.
>
> **Stage 3 -- Intelligence and Delivery:**
> We use AI to read the policy PDFs and make them searchable. We build a Streamlit dashboard for visual exploration. We deploy a Cortex Agent that can answer natural-language questions by combining structured data queries with policy document search.
>
> Here is the flow visually:
>
> ```
> DATA IN                    ANALYSIS                    DELIVERY
> ---------                  ---------                   ---------
> Marketplace Buildings  --> H3 Spatial Joins        --> Streamlit Dashboard
> Flood Risk CSVs        --> Vulnerability Scoring   --> Dynamic Alert Table
> SVI CSVs               --> Comarca Summaries       --> Cortex AI Agent
> Policy PDFs            --> AI Document Parsing     --> Semantic Search
> ```
>
> Now, let us start building. Please open Snowsight in your browser -- that is app.snowflake.com."

---

## PART 2: PREREQUISITES AND SETUP (10 minutes)

---

### [0:15 - 0:18] Install Overture Maps from Marketplace

> **Presenter:**
>
> "The first thing we need is building data. Instead of collecting it ourselves, we will use the Snowflake Marketplace. This is one of Snowflake's most powerful features for organizations -- you can access third-party datasets without copying, moving, or paying for storage. The data stays with the provider and you query it live.
>
> Let me walk you through it:
>
> 1. In Snowsight, click **Marketplace** in the left sidebar
> 2. In the search bar, type **'Overture Maps - Buildings'**
> 3. Find the listing by **CARTO** -- this is a company that provides geospatial data
> 4. Click on it, then click **Get**
> 5. Set the database name to **OVERTURE_MAPS_BUILDINGS** -- this is important, the code expects exactly this name
> 6. Grant access to the **PUBLIC** role
> 7. Click **Get** again
>
> **What just happened?** You now have access to 2.3 billion building footprints worldwide. You did not download anything -- Snowflake created a shared database that queries the provider's data directly. This dataset comes from the Overture Maps Foundation, a collaboration between Meta, Microsoft, Amazon, and others to create an open map of the world.
>
> To verify, go to **Catalog > Explorer** and check that you see `OVERTURE_MAPS_BUILDINGS` in the list."

---

### [0:18 - 0:22] Create a Git Workspace

> **Presenter:**
>
> "Next, we need to get our project code into Snowflake. We will use a Git Workspace, which connects Snowflake directly to a GitHub repository.
>
> 1. Click **Projects** > **Workspaces** in the left sidebar
> 2. Click the **+** button > **Git Workspace**
> 3. Set the Repository URL to: `https://github.com/sfc-gh-hharkema/flood_resilience_es`
> 4. Name the workspace: **flood_resilience_es** -- again, the code references this exact name
> 5. Click **+ API Integration** and create one:
>    - Name: `FLOODS`
>    - Allowed Prefixes: `https://github.com`
>    - Allowed authentication secrets: All
>    - Click Create
> 6. Select **Public repository** (no authentication needed)
> 7. Click **Create** and wait for the sync
>
> **What is a Workspace?** For non-technical folks, think of it as a shared folder that Snowflake can access. It contains our notebook, dashboard code, data files, and AI model definitions. For technical folks, it is a first-class git integration -- Snowflake pulls the repo contents and makes them available via `snow://workspace/` URIs."

---

### [0:22 - 0:25] Open the Notebook and Connect

> **Presenter:**
>
> "Now open the notebook: navigate to `notebooks/flood_vulnerability_hol.ipynb` in the workspace.
>
> Before we can run any code, we need a compute environment:
>
> 1. Click the dropdown arrow next to **Connect**
> 2. Click **+ Create new service**
> 3. Accept the default settings and click **Create and connect**
> 4. Wait about a minute until you see a green checkmark and 'Connected'
>
> **What is happening behind the scenes?** Snowflake is spinning up a container (via Snowpark Container Services) that will execute our notebook cells. Think of it as a small virtual computer dedicated to running our code.
>
> While that starts up, let me point out the structure of our notebook. It is organized into 8 labs, each building on the previous one. We will run cells top to bottom using Shift+Enter."

---

## PART 3: LAB 1 - ENVIRONMENT SETUP (15 minutes)

---

### [0:25 - 0:28] Create Database and Warehouse

> **Presenter:**
>
> "Let us run our first cell. This creates the database, schema, and warehouse where all our work will live.
>
> ```sql
> USE ROLE ACCOUNTADMIN;
>
> CREATE DATABASE  IF NOT EXISTS FLOOD_ANALYTICS;
> CREATE SCHEMA    IF NOT EXISTS FLOOD_ANALYTICS.FLOOD;
>
> CREATE WAREHOUSE IF NOT EXISTS FLOOD_WH
>   WAREHOUSE_SIZE = 'MEDIUM'
>   AUTO_SUSPEND   = 120
>   AUTO_RESUME    = TRUE;
>
> USE DATABASE  FLOOD_ANALYTICS;
> USE SCHEMA    FLOOD;
> USE WAREHOUSE FLOOD_WH;
> ```
>
> **For non-technical audience:**
> - A **database** is like a filing cabinet -- it organizes all our data
> - A **schema** is like a drawer inside the filing cabinet -- `FLOOD` is where we will put everything
> - A **warehouse** is the computing power that runs our queries. We chose MEDIUM size. It auto-suspends after 2 minutes of inactivity to save costs, and auto-resumes when we need it.
>
> **For technical audience:**
> We are using ACCOUNTADMIN for this lab. In production, you would use a custom role with appropriate privileges. The MEDIUM warehouse gives us enough parallelism for the 2M+ row extractions coming up."

---

### [0:28 - 0:32] Verify and Extract Valencia Buildings

> **Presenter:**
>
> "Now let us verify we can access the Overture Maps data. Run this cell:
>
> ```sql
> SELECT
>     ID,
>     NAMES['primary']::STRING AS NAME,
>     SUBTYPE, CLASS, HEIGHT, NUM_FLOORS, BBOX
> FROM OVERTURE_MAPS_BUILDINGS.CARTO.BUILDING
> WHERE BBOX:xmin >= -1.53
>   AND BBOX:xmax <=  0.53
>   AND BBOX:ymin >= 37.84
>   AND BBOX:ymax <= 40.79
> LIMIT 10;
> ```
>
> **What is happening here?** We are querying 2.3 billion buildings worldwide but filtering to only those within a geographic bounding box around Valencia. The `BBOX` column contains the bounding box of each building -- its minimum and maximum latitude and longitude. We are using coordinates that cover the entire Comunitat Valenciana region.
>
> You should see 10 buildings with their names, types (residential, commercial, etc.), heights, and locations.
>
> Now let us extract ALL Valencia buildings into our own table. This is the biggest query of the lab -- it takes 3-5 minutes:
>
> ```sql
> CREATE OR REPLACE TABLE BUILDINGS_VALENCIA AS
> SELECT
>     ID,
>     NAMES['primary']::STRING                          AS NAME,
>     SUBTYPE, CLASS, HEIGHT, NUM_FLOORS, GEOMETRY, BBOX,
>     ST_X(ST_CENTROID(GEOMETRY))                       AS LONGITUDE,
>     ST_Y(ST_CENTROID(GEOMETRY))                       AS LATITUDE,
>     H3_POINT_TO_CELL_STRING(ST_CENTROID(GEOMETRY), 8) AS H3_INDEX_8,
>     H3_POINT_TO_CELL_STRING(ST_CENTROID(GEOMETRY), 6) AS H3_INDEX_6
> FROM OVERTURE_MAPS_BUILDINGS.CARTO.BUILDING
> WHERE BBOX:xmin >= -1.53
>   AND BBOX:xmax <=  0.53
>   AND BBOX:ymin >= 37.84
>   AND BBOX:ymax <= 40.79;
> ```
>
> **Let me explain the key parts while this runs:**
>
> - `ST_CENTROID(GEOMETRY)` -- every building has a polygon shape (its footprint). We calculate the center point of each building.
> - `ST_X` and `ST_Y` -- these extract the longitude and latitude from that center point.
> - `H3_POINT_TO_CELL_STRING(..., 8)` -- this is the H3 hexagonal indexing I mentioned. We convert each building's center point into a hexagon ID at resolution 8 (each hex covers about 0.74 km2, roughly 180 acres). Think of it as assigning each building to a hexagonal 'neighbourhood'.
> - `H3_POINT_TO_CELL_STRING(..., 6)` -- same thing but at resolution 6 (each hex covers about 36 km2). This is a coarser grouping we will use for heatmaps later.
>
> **Why two resolutions?** Resolution 8 gives us detailed building-to-zone matching. Resolution 6 gives us a nice visual aggregation for maps. It is like having both a detailed street map and a regional overview map.
>
> [Wait for query to complete]
>
> Let us count the result:
>
> ```sql
> SELECT COUNT(*) AS TOTAL_VALENCIA_BUILDINGS FROM BUILDINGS_VALENCIA;
> ```
>
> You should see around 2 million buildings. That is every building in the entire Comunitat Valenciana -- homes, shops, hospitals, schools, churches, fire stations -- all with their exact geographic footprints."

---

## PART 4: LAB 2 - LOADING RISK AND VULNERABILITY DATA (10 minutes)

---

### [0:40 - 0:43] Create Stage and Load CSV Files

> **Presenter:**
>
> "We have the buildings. Now we need two more datasets: flood risk scores and social vulnerability scores. These are stored as CSV files in our workspace.
>
> First, we create a 'stage' -- think of it as a landing zone where Snowflake can pick up files:
>
> ```sql
> CREATE OR REPLACE STAGE FLOOD_DATA_STAGE
>   DIRECTORY = (ENABLE = TRUE)
>   ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
>
> CREATE OR REPLACE FILE FORMAT CSV_FORMAT
>   TYPE                         = 'CSV'
>   FIELD_OPTIONALLY_ENCLOSED_BY = '\"'
>   PARSE_HEADER                 = TRUE
>   NULL_IF                      = ('', 'NULL', 'None', 'NA', '-999')
>   EMPTY_FIELD_AS_NULL          = TRUE;
> ```
>
> Now we copy files from our workspace into the stage, then load them into proper tables. Run this cell:
>
> ```sql
> COPY FILES INTO @FLOOD_DATA_STAGE/comarca/
> FROM 'snow://workspace/USER$.PUBLIC.\"flood_resilience_es\"/versions/live/'
> FILES=('data/comarca_centroids/Valencia_Comarca_Centroids.csv');
>
> COPY FILES INTO @FLOOD_DATA_STAGE/risk/
> FROM 'snow://workspace/...'
> FILES=('data/flood_risk/Flood_Risk_Valencia.csv');
>
> COPY FILES INTO @FLOOD_DATA_STAGE/svi/
> FROM 'snow://workspace/...'
> FILES=('data/social_vulnerability/SVI_Valencia.csv');
> ```
>
> Then we create and populate three tables:
>
> 1. **COMARCA_CENTROIDS** -- 34 rows, one per comarca, with the center latitude/longitude of each district
> 2. **FLOOD_RISK_VALENCIA** -- ~80 rows, one per risk zone, with risk scores for fluvial (river), coastal, and DANA flooding, plus expected annual losses in euros
> 3. **SVI_VALENCIA** -- ~80 rows, matching zones, with social vulnerability scores: socioeconomic status, demographics, housing quality, elderly percentage, immigrant percentage
>
> **Important note:** This data is synthetic -- it is modeled after real PATRICOVA zones and INE (Spain's national statistics institute) indicators, but the actual numbers are generated for this lab. In a real deployment, you would use the actual PATRICOVA and INE datasets."

---

### [0:43 - 0:50] Derive PATRICOVA Flood Zones

> **Presenter:**
>
> "Now we derive the official PATRICOVA flood zone classification. Run this cell:
>
> ```sql
> CREATE OR REPLACE TABLE FLOOD_ZONES AS
> SELECT
>     ZONE_CODE, COMARCA, COMARCA_CODE,
>     CASE
>         WHEN FLUVIAL_RISK_RATING IN ('Very High','Relatively High')
>              AND COASTAL_RISK_RATING IN ('Very High','Relatively High') THEN 'A1'
>         WHEN FLUVIAL_RISK_RATING IN ('Very High','Relatively High') THEN 'A2'
>         WHEN FLUVIAL_RISK_RATING = 'Relatively Moderate' THEN 'B'
>         ELSE 'C'
>     END AS FLOOD_ZONE,
>     ...
> FROM FLOOD_RISK_VALENCIA;
> ```
>
> **What is this doing?** It is translating the raw risk ratings into the official PATRICOVA zones:
>
> | Condition | Zone | Meaning |
> |---|---|---|
> | High fluvial AND high coastal risk | A1 | Most dangerous. Both rivers and sea threaten the area. No new construction allowed. |
> | High fluvial risk only | A2 | Dangerous. River flooding likely. New buildings need elevated ground floors. |
> | Moderate risk | B | Some risk. Construction allowed with conditions. |
> | Everything else | C | Low risk. Standard regulations. |
>
> The result tells us how many zones fall into each category. In a real disaster scenario, zones A1 and A2 are where emergency responders should focus first."

---

## PART 5: LAB 3 - GEOSPATIAL ANALYSIS (20 minutes)

---

### [0:50 - 0:55] H3 Comarca Mapping

> **Presenter:**
>
> "This is where the magic happens. We need to connect our 2 million buildings to the 34 comarcas and their flood risk zones. The challenge is: the building data does not tell us which comarca a building is in. We need to figure that out spatially.
>
> Here is our approach: we use H3 hexagons as a bridge. Every building already has an H3 index (from Lab 1). Now we map each H3 hexagon to its nearest comarca using the Haversine formula -- the same formula pilots use to calculate distances between two points on the Earth's curved surface.
>
> ```sql
> CREATE OR REPLACE TABLE H3_COMARCA_MAP AS
> WITH h3_cells AS (SELECT DISTINCT H3_INDEX_6 FROM BUILDINGS_VALENCIA),
> h3_with_centroid AS (
>     SELECT h.H3_INDEX_6,
>         ST_Y(H3_CELL_TO_POINT(h.H3_INDEX_6)) AS H3_LAT,
>         ST_X(H3_CELL_TO_POINT(h.H3_INDEX_6)) AS H3_LON
>     FROM h3_cells h
> )
> SELECT hc.H3_INDEX_6, cc.COMARCA_CODE, cc.COMARCA, cc.PROVINCE
> FROM h3_with_centroid hc
> CROSS JOIN COMARCA_CENTROIDS cc
> QUALIFY ROW_NUMBER() OVER (
>     PARTITION BY hc.H3_INDEX_6
>     ORDER BY HAVERSINE(hc.H3_LAT, hc.H3_LON, cc.LATITUDE, cc.LONGITUDE)
> ) = 1;
> ```
>
> **Step by step for the non-technical audience:**
> 1. We get every unique hexagon that contains at least one building
> 2. We find the center point of each hexagon
> 3. For each hexagon, we calculate the distance to ALL 34 comarca centers
> 4. We keep only the closest comarca -- that is what `QUALIFY ROW_NUMBER() ... = 1` does
>
> **For the technical audience:** This is a nearest-neighbour join using CROSS JOIN + QUALIFY with HAVERSINE distance. The ROW_NUMBER window function partitions by hexagon and orders by distance, keeping rank 1. This is far more efficient than doing point-in-polygon tests for 2M+ buildings."

---

### [0:55 - 1:05] Building Flood Risk Table (The Core Table)

> **Presenter:**
>
> "Now we build the most important table in the entire project: **BUILDING_FLOOD_RISK**. This assigns every single building a composite vulnerability score.
>
> This is a complex query, so let me walk through the logic:
>
> **Step 1:** We prepare the zone data. Each comarca has multiple risk zones. We number them so we can distribute buildings across zones realistically -- we do not want every building in a comarca to have the same score.
>
> **Step 2:** We join each building to its comarca using the H3 map we just created.
>
> **Step 3:** We assign each building to a specific zone within its comarca using a hash function. This is a clever trick:
>
> ```sql
> ABS(HASH(bc.ID)) % zn.ZONE_CNT = zn.ZONE_IDX
> ```
>
> **In plain language:** we take the building's unique ID, run it through a hash function (which produces a seemingly random number), and use modulo arithmetic to assign it to one of the zones in its comarca. This ensures buildings are spread across zones in a deterministic but distributed way.
>
> **Step 4:** We calculate the composite vulnerability score:
>
> ```sql
> ROUND(
>     COALESCE(zn.RISK_SCORE, 0) * 0.40
>   + COALESCE(zn.SVI_OVERALL, 0) * 100 * 0.30
>   + CASE zn.FLOOD_ZONE
>       WHEN 'A1' THEN 100
>       WHEN 'A2' THEN 80
>       WHEN 'B'  THEN 40
>       ELSE 10
>     END * 0.30
> , 2) AS COMPOSITE_VULNERABILITY_SCORE
> ```
>
> **Breaking this down for everyone:**
> - **40% from flood risk score** -- how physically dangerous is the area?
> - **30% from social vulnerability** -- how able is the community to cope?
> - **30% from flood zone classification** -- is this building in A1 (score 100), A2 (80), B (40), or C (10)?
>
> A building scoring 80+ is in serious danger -- high physical risk, vulnerable community, and sitting in a severe flood zone.
>
> Run the cell now. This takes 2-4 minutes because we are processing 2M+ buildings.
>
> [Wait for completion]
>
> ```sql
> SELECT COUNT(*) AS TOTAL_ROWS FROM BUILDING_FLOOD_RISK;
> ```
>
> There it is -- over 2 million buildings, each with a complete risk profile."

---

### [1:05 - 1:10] Summary Tables

> **Presenter:**
>
> "Now let us create two summary views of this data.
>
> **Comarca Summary (PARISH_FLOOD_SUMMARY):**
>
> ```sql
> CREATE OR REPLACE TABLE PARISH_FLOOD_SUMMARY AS
> SELECT
>     PARISH,
>     COUNT(*) AS TOTAL_BUILDINGS,
>     COUNT(CASE WHEN IN_SPECIAL_FLOOD_HAZARD_AREA THEN 1 END) AS BUILDINGS_IN_SFHA,
>     ROUND(COUNT(CASE WHEN IN_SPECIAL_FLOOD_HAZARD_AREA THEN 1 END)*100.0/COUNT(*),1) AS PCT_IN_SFHA,
>     ROUND(AVG(COMPOSITE_VULNERABILITY_SCORE),2) AS AVG_COMPOSITE_SCORE,
>     ...
> FROM BUILDING_FLOOD_RISK GROUP BY PARISH ORDER BY AVG_COMPOSITE_SCORE DESC;
> ```
>
> This gives us one row per comarca with: total buildings, how many are in high-risk flood zones, the percentage in flood zones, average vulnerability score, and total expected annual loss. This is the executive view -- what a government official or emergency manager would look at first.
>
> **H3 Heatmap (H3_FLOOD_RISK_MAP):**
>
> ```sql
> CREATE OR REPLACE TABLE H3_FLOOD_RISK_MAP AS
> SELECT
>     H3_INDEX_6,
>     COUNT(*) AS BUILDING_COUNT,
>     ROUND(AVG(COMPOSITE_VULNERABILITY_SCORE),2) AS AVG_VULNERABILITY,
>     ...
> FROM BUILDING_FLOOD_RISK GROUP BY H3_INDEX_6 HAVING COUNT(*) >= 10;
> ```
>
> This aggregates vulnerability data at the hexagonal level for map visualization. Each hexagon shows the average vulnerability of the buildings within it. We filter out hexagons with fewer than 10 buildings to avoid noise.
>
> Look at the top 10 results -- these are the most vulnerable hexagonal areas in all of Valencia."

---

## PART 6: LAB 4 - DYNAMIC TABLES (8 minutes)

---

### [1:10 - 1:18] Automated Risk Alerts

> **Presenter:**
>
> "In a real emergency management system, you do not want someone to manually re-run queries every hour. You want the data to update itself. That is what Snowflake Dynamic Tables do.
>
> ```sql
> CREATE OR REPLACE DYNAMIC TABLE FLOOD_RISK_ALERTS
>   TARGET_LAG = '1 hour'
>   WAREHOUSE = FLOOD_WH
> AS
> SELECT
>     PARISH, COMARCA_CODE,
>     COUNT(*) AS BUILDINGS_AT_RISK,
>     ROUND(AVG(COMPOSITE_VULNERABILITY_SCORE), 2) AS AVG_VULNERABILITY_SCORE,
>     ...
>     CASE
>         WHEN AVG(COMPOSITE_VULNERABILITY_SCORE) >= 70 THEN 'CRITICAL'
>         WHEN AVG(COMPOSITE_VULNERABILITY_SCORE) >= 50 THEN 'HIGH'
>         WHEN AVG(COMPOSITE_VULNERABILITY_SCORE) >= 30 THEN 'MODERATE'
>         ELSE 'LOW'
>     END AS RISK_LEVEL
> FROM BUILDING_FLOOD_RISK
> WHERE IN_SPECIAL_FLOOD_HAZARD_AREA = TRUE
> GROUP BY PARISH, COMARCA_CODE;
> ```
>
> **What is a Dynamic Table?** For non-technical folks: imagine a spreadsheet that watches other spreadsheets and automatically recalculates whenever the source data changes. That is exactly what this is, but at scale.
>
> **The `TARGET_LAG = '1 hour'` part** means Snowflake guarantees this table will never be more than 1 hour behind the source data. If someone updates the flood risk scores at 2:00 PM, by 3:00 PM this alert table will reflect the changes automatically.
>
> **The CASE statement** classifies each comarca into alert levels:
> - **CRITICAL** (score >= 70): Immediate action needed
> - **HIGH** (50-69): Elevated concern
> - **MODERATE** (30-49): Monitoring required
> - **LOW** (< 30): Standard operations
>
> In a real deployment, you could connect this to an email/SMS alert system. When a comarca moves from MODERATE to CRITICAL, an emergency manager gets a notification.
>
> Run the query and look at the results -- you will see the distribution of comarcas across alert levels."

---

## PART 7: LAB 5 - AI DOCUMENT INTELLIGENCE (15 minutes)

---

### [1:18 - 1:23] Upload and Parse Policy PDFs

> **Presenter:**
>
> "So far, everything we have done has been with structured data -- rows and columns. But in the real world, critical information lives in documents. Valencia has a Flood Mitigation Plan -- a policy document with strategies, project descriptions, historical event analysis, and EU funding information.
>
> We have two PDF files:
> - `Valencia_Flood_Mitigation_Plan_2024_Intro.pdf` -- background and historical context
> - `Valencia_Flood_Mitigation_Plan_2024_Strategies.pdf` -- mitigation strategies and projects
>
> Let us upload and parse them:
>
> ```sql
> CREATE OR REPLACE STAGE FLOOD_POLICY_DOCS
>   DIRECTORY = (ENABLE = TRUE)
>   ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
>
> COPY FILES INTO @FLOOD_POLICY_DOCS/
> FROM 'snow://workspace/...'
> FILES=('data/policy_docs/Valencia_Flood_Mitigation_Plan_2024_Intro.pdf',
>        'data/policy_docs/Valencia_Flood_Mitigation_Plan_2024_Strategies.pdf');
> ```
>
> Now here is the AI part -- we use **Cortex PARSE_DOCUMENT** to extract text from these PDFs:
>
> ```sql
> CREATE OR REPLACE TABLE PARSED_POLICY_DOCS AS
> SELECT
>     RELATIVE_PATH AS FILE_NAME,
>     SNOWFLAKE.CORTEX.PARSE_DOCUMENT(@FLOOD_POLICY_DOCS, RELATIVE_PATH) AS PARSED_CONTENT,
>     PARSED_CONTENT:content::STRING AS FULL_TEXT
> FROM DIRECTORY(@FLOOD_POLICY_DOCS)
> WHERE RELATIVE_PATH LIKE '%.pdf';
> ```
>
> **What just happened?** Snowflake's built-in AI read both PDFs and extracted all the text. No external tools, no Python libraries, no OCR setup -- one SQL function call.
>
> Then we chunk the text into smaller pieces for searching:
>
> ```sql
> CREATE OR REPLACE TABLE POLICY_DOC_CHUNKS AS
> SELECT FILE_NAME, chunk.INDEX AS CHUNK_INDEX,
>     TRIM(chunk.VALUE::STRING) AS CHUNK_TEXT
> FROM PARSED_POLICY_DOCS,
>     LATERAL FLATTEN(INPUT => SPLIT(FULL_TEXT, '\\n\\n')) AS chunk
> WHERE LENGTH(TRIM(chunk.VALUE::STRING)) > 80;
> ```
>
> **Why chunk?** AI search works better with smaller, focused text segments than with entire documents. Each chunk is roughly a paragraph."

---

### [1:23 - 1:28] Cortex Search Service

> **Presenter:**
>
> "Now we create a semantic search service over these chunks:
>
> ```sql
> CREATE OR REPLACE CORTEX SEARCH SERVICE FLOOD_POLICY_SEARCH
>   ON CHUNK_TEXT
>   ATTRIBUTES FILE_NAME, CHUNK_INDEX
>   WAREHOUSE = FLOOD_WH
>   TARGET_LAG = '1 day'
> AS (SELECT CHUNK_TEXT, FILE_NAME, CHUNK_INDEX FROM POLICY_DOC_CHUNKS);
> ```
>
> **What is Cortex Search?** For non-technical audience: imagine Google search, but just for your documents. You type a question in plain English, and it finds the most relevant paragraphs. It does not just look for exact word matches -- it understands the *meaning* of your question.
>
> For example, if you search for 'river flooding projects,' it will also find paragraphs that mention 'fluvial mitigation infrastructure' because it understands these mean the same thing.
>
> Let us test it:
>
> ```sql
> SELECT PARSE_JSON(SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
>     'FLOOD_ANALYTICS.FLOOD.FLOOD_POLICY_SEARCH',
>     '{\"query\": \"What projects are planned for the Xuquer river?\",
>       \"columns\": [\"CHUNK_TEXT\",\"FILE_NAME\"], \"limit\": 3}'
> )) AS SEARCH_RESULTS;
> ```
>
> Look at the results -- it found relevant paragraphs about the Xuquer river from the policy documents."

---

### [1:28 - 1:33] AI Executive Summary

> **Presenter:**
>
> "Now let us see Cortex AI generate an executive summary. This combines our structured comarca data with the power of a large language model:
>
> ```sql
> SELECT SNOWFLAKE.CORTEX.COMPLETE('llama3.1-70b', CONCAT(
>     'You are a senior flood risk analyst for Valencia Emergency Management. ',
>     'Based on the comarca-level flood risk data below, write a 3-paragraph executive summary.\\n\\n',
>     'COMARCA RISK DATA (top 15):\\n',
>     (SELECT LISTAGG(PARISH||': composite='||AVG_COMPOSITE_SCORE||...))
> )) AS EXECUTIVE_SUMMARY;
> ```
>
> **What is happening here?**
> 1. We take the top 15 most vulnerable comarcas from our summary table
> 2. We format them as text and feed them to the Llama 3.1 70B language model (a powerful open-source AI model running inside Snowflake)
> 3. The AI writes a professional executive summary interpreting the data
>
> **Key point for everyone:** The data never leaves Snowflake. The AI model runs inside the Snowflake infrastructure. This is critical for organizations with data sovereignty requirements -- your sensitive data is not sent to any external API.
>
> Read the output -- it should identify the highest-risk comarcas, explain the relationship between flood risk and social vulnerability, and recommend priority areas for intervention."

---

## PART 8: LAB 6 - DEPLOYMENT (12 minutes)

---

### [1:33 - 1:38] Deploy the Streamlit Dashboard

> **Presenter:**
>
> "Now we put a user-friendly face on all this data. We deploy a Streamlit dashboard -- an interactive web application -- directly inside Snowflake.
>
> ```sql
> CREATE OR REPLACE STAGE FLOOD_ANALYTICS.FLOOD.STREAMLIT_STAGE
>   DIRECTORY = (ENABLE = TRUE)
>   ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
>
> COPY FILES INTO @FLOOD_ANALYTICS.FLOOD.STREAMLIT_STAGE/
> FROM 'snow://workspace/...'
> FILES=('streamlit/flood_dashboard.py', 'streamlit/environment.yml',
>        'streamlit/.streamlit/config.toml');
>
> CREATE OR REPLACE STREAMLIT FLOOD_ANALYTICS.FLOOD.FLOOD_VULNERABILITY_DASHBOARD
>   ROOT_LOCATION = '@FLOOD_ANALYTICS.FLOOD.STREAMLIT_STAGE/streamlit'
>   MAIN_FILE = 'flood_dashboard.py'
>   QUERY_WAREHOUSE = FLOOD_WH
>   TITLE = 'Valencia Flood Vulnerability Dashboard';
> ```
>
> **What is Streamlit in Snowflake?** It is a Python framework for building data apps. The code runs entirely inside Snowflake -- no separate web server needed, no deployment pipeline, no infrastructure to manage. You write Python, and Snowflake serves the web application.
>
> Let me open the dashboard. Go to **Projects > Streamlit** and click on 'Valencia Flood Vulnerability Dashboard'.
>
> **Tab 1 -- Comarca Overview:**
> - Top KPI cards showing total comarcas, buildings, buildings in flood zones, and region-wide annual loss
> - Bar chart of the top 15 comarcas by percentage of buildings in flood zones
> - Scatter plot showing the relationship between risk score (x-axis) and social vulnerability (y-axis), with bubble size representing building count and colour representing the composite score
> - Full data table below
>
> **Tab 2 -- Building Explorer:**
> - Filter by comarca, minimum vulnerability score, and number of buildings to display
> - An interactive map showing actual building footprints colour-coded by vulnerability (green = low, red = high)
> - You can hover over any building to see its name, type, flood zone, and vulnerability score
> - Histogram showing the distribution of vulnerability scores
>
> **Tab 3 -- H3 Heatmap:**
> - A hexagonal heatmap of the entire Valencia region
> - Each hexagon is colour-coded by average vulnerability
> - A donut chart showing the distribution of hex cells across risk levels
> - Table of the top 10 highest-risk hexagons
>
> **Tab 4 -- AI Insights:**
> - A text input where anyone can ask questions in plain language
> - The AI reads your question, combines it with the actual comarca data, and generates an answer
> - Try asking: 'Which comarca should we prioritise for emergency preparedness?'"

---

### [1:38 - 1:45] Deploy the Cortex Agent

> **Presenter:**
>
> "The dashboard is great for visual exploration. But what if a government official wants to ask: 'What does the mitigation plan say about funding for the Xuquer river AND which comarcas along the Xuquer are most at risk?'
>
> That question needs *both* structured data (the comarca risk scores) AND unstructured data (the policy documents). This is where the Cortex Agent comes in.
>
> ```sql
> CREATE OR REPLACE AGENT FLOOD_ANALYTICS.FLOOD.FLOOD_RISK_AGENT
> FROM SPECIFICATION $$
> {
>   \"instructions\": {
>     \"orchestration\": \"You are a Valencia flood risk analyst. You have access to two tools:
>       (1) query_flood_data for structured analysis...
>       (2) search_policy_docs for finding information from the Flood Mitigation Plan...\"
>   },
>   \"tools\": [
>     {\"tool_spec\": {\"type\": \"cortex_analyst_text_to_sql\", \"name\": \"query_flood_data\", ...}},
>     {\"tool_spec\": {\"type\": \"cortex_search\", \"name\": \"search_policy_docs\", ...}}
>   ]
> }
> $$;
> ```
>
> **For everyone -- what is a Cortex Agent?** Think of it as a smart assistant that has two abilities:
>
> 1. **query_flood_data** (Cortex Analyst): It can translate your plain-language question into SQL and run it against our 2M+ building database. 'How many hospitals are in flood zones?' becomes a SQL query automatically.
>
> 2. **search_policy_docs** (Cortex Search): It can search the flood mitigation plan documents for relevant information.
>
> The agent decides which tool to use -- or both -- based on your question. It is like having a flood risk analyst on your team who has memorized both the database and the policy documents.
>
> Let us test it. Go to **AI & ML > Agent Studio** in Snowsight.
>
> Try these questions:
>
> | Question | What the agent does |
> |---|---|
> | 'Which 5 comarcas have the highest flood vulnerability?' | Queries structured data only |
> | 'What does the plan say about the barranco del Poyo project?' | Searches policy documents only |
> | 'Which comarcas are most at risk and what EU funds are available?' | Uses BOTH tools -- queries risk data AND searches policy documents |
> | 'What happened during the 2024 DANA?' | Searches policy documents for historical information |
>
> Notice how the agent seamlessly combines numbers from the database with context from the policy documents. This is the power of bringing structured and unstructured data together."

---

## PART 9: LAB 7 - SEMANTIC MODELS AND APACHE OSSIE (12 minutes)

---

### [1:45 - 1:52] From YAML to Semantic View to Ossie

> **Presenter:**
>
> "For our final technical lab, let us talk about semantic models and why they matter.
>
> **For non-technical audience:** A semantic model is like a dictionary for your database. It tells AI systems: 'This column called PARISH actually means comarca name. This number called SVI_OVERALL is a social vulnerability score from 0 to 1 where higher means more vulnerable.' Without this, an AI just sees column names and numbers with no context.
>
> **For technical audience:** The semantic model (`flood_risk_model.yaml`) defines two entities with their dimensions, measures, default aggregations, and verified queries. It is what powers Cortex Analyst's text-to-SQL capability.
>
> Now we are going to do two transformations:
>
> **Step 1: Convert the YAML to a Snowflake Semantic View:**
>
> ```sql
> SET yaml_spec = (
>   SELECT $1
>   FROM @FLOOD_DATA_STAGE/semantic_model/flood_risk_model.yaml
>        (FILE_FORMAT => 'YAML_FF')
> );
>
> CALL SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML(
>   'FLOOD_ANALYTICS.FLOOD',
>   $yaml_spec
> );
> ```
>
> A Semantic View is a native Snowflake object. It stores the same metadata as the YAML file but as a first-class database object that can be managed, versioned, and governed like any other Snowflake object.
>
> **Step 2: Export to Apache Ossie:**
>
> ```sql
> COPY INTO @FLOOD_DATA_STAGE/ossie/flood_ossie.yaml
> FROM (
>   SELECT SYSTEM$READ_OSSIE_YAML_FROM_SEMANTIC_VIEW(
>     'FLOOD_ANALYTICS.FLOOD.FLOOD_RISK_MODEL'
>   )
> )
> ...
> ```
>
> **What is Apache Ossie?** It is an open, vendor-neutral format for semantic models. Think of it this way:
>
> - A Cortex Analyst YAML works only inside Snowflake
> - A Snowflake Semantic View is a Snowflake-native object
> - An Apache Ossie document works with *any* compatible tool -- Snowflake, Tableau, Power BI, dbt, or any other platform that supports the Ossie standard
>
> This is important because in real organizations, data teams use many tools. If you define your metrics and business logic in Ossie format, every tool sees the same definitions. 'Total Expected Annual Loss' means exactly the same thing whether you are looking at it in Snowflake, Tableau, or a Python script.
>
> Let us verify the export:
>
> ```sql
> LIST @FLOOD_DATA_STAGE/ossie/;
> SELECT $1 FROM @FLOOD_DATA_STAGE/ossie/flood_ossie.yaml;
> ```
>
> The output is a YAML document in Apache Ossie format, with richer documentation including `ai_context` fields that explain domain-specific terms like DANA_RISK_SCORE, COMARCA_CODE, and why the column called PARISH actually holds comarca names."

---

## PART 10: RECAP AND Q&A (8 minutes)

---

### [1:52 - 1:57] Summary of What We Built

> **Presenter:**
>
> "Let us step back and look at what we accomplished in under 2 hours:
>
> | Lab | What We Built | Snowflake Feature | Why It Matters |
> |---|---|---|---|
> | 1 | Extracted 2M+ buildings from a global dataset | Marketplace + Geospatial + H3 | Real building footprints, not synthetic data |
> | 2 | Loaded flood risk and social vulnerability data | Stages + COPY INTO | Combined physical and social factors |
> | 3 | Scored every building's vulnerability | H3 Indexing + SQL Analytics | Actionable, building-level intelligence |
> | 4 | Created auto-refreshing alert system | Dynamic Tables | Always up-to-date, no manual work |
> | 5 | AI-parsed government policy documents | Cortex PARSE_DOCUMENT + Search | Bridged structured data and unstructured documents |
> | 6 | Deployed dashboard + AI agent | Streamlit + Cortex Agent | Accessible to technical and non-technical users |
> | 7 | Exported as open semantic model | Semantic Views + Apache Ossie | Portable, vendor-neutral metadata |
>
> **For the non-technical audience -- the big takeaway:**
> We built a system where anyone can ask 'Which neighbourhood should we evacuate first?' and get an answer backed by 2 million buildings of real data, flood risk science, social vulnerability research, and government policy documents. No SQL knowledge required. No data science degree needed. Just ask the question.
>
> **For students -- the learning takeaway:**
> You saw how a modern cloud data platform integrates: geospatial analysis, AI/ML, application development, and semantic modeling -- all in one environment. These are the skills the industry is hiring for right now.
>
> **For the technical team -- the architecture takeaway:**
> Everything runs inside Snowflake's security perimeter. Data never leaves the platform. The AI models run natively. The dashboard is governed by Snowflake's RBAC. The Dynamic Table ensures freshness. The Semantic View ensures consistent business logic. This is what a production-grade analytics platform looks like."

---

### [1:57 - 2:00] Questions and Next Steps

> **Presenter:**
>
> "Before we open for questions, a few things you might explore next:
>
> 1. **Try the Cortex Agent** with your own questions -- it handles surprisingly complex queries that combine data and documents
>
> 2. **Explore the Building Explorer tab** -- zoom into a specific comarca you are interested in and look at individual building footprints
>
> 3. **Read `docs/understanding-apache-ossie.md`** in the workspace -- it is a comprehensive guide to semantic models written for beginners
>
> 4. **Cleanup** -- when you are done, Lab 8 has optional cleanup commands to remove everything we created today. Just uncomment the DROP statements and run them.
>
> I am happy to take questions now. Whether you are wondering about the PATRICOVA flood zone system, how H3 indexing works, how to set up a Cortex Agent in your own project, or how Snowflake's AI features compare to other platforms -- fire away."

---

## APPENDIX: QUICK REFERENCE

### Key URLs to Have Open
- Snowsight: `https://app.snowflake.com`
- Marketplace listing: Search "Overture Maps - Buildings" by CARTO
- GitHub repo: `https://github.com/sfc-gh-hharkema/flood_resilience_es`

### Troubleshooting Common Issues

| Issue | Solution |
|---|---|
| "OVERTURE_MAPS_BUILDINGS does not exist" | Go to Marketplace, search for Overture Maps Buildings by CARTO, click Get |
| Notebook cells fail with "no active warehouse" | Run `USE WAREHOUSE FLOOD_WH;` |
| Building extraction takes > 10 minutes | Normal on SMALL warehouse; upgrade to MEDIUM |
| Cortex COMPLETE returns error | Check that your Snowflake region supports Cortex AI and llama3.1-70b |
| Streamlit dashboard shows no data | Ensure Labs 1-3 completed successfully; check that PARISH_FLOOD_SUMMARY has rows |
| Agent not found in Agent Studio | Ensure the CREATE AGENT statement succeeded; refresh the page |

### Key Tables Created

| Table | Rows | Description |
|---|---|---|
| BUILDINGS_VALENCIA | ~2M | Raw building footprints with H3 indices |
| COMARCA_CENTROIDS | 34 | Center points of each comarca |
| FLOOD_RISK_VALENCIA | ~80 | Zone-level flood risk scores |
| SVI_VALENCIA | ~80 | Zone-level social vulnerability |
| FLOOD_ZONES | ~80 | PATRICOVA classifications (A1/A2/B/C) |
| H3_COMARCA_MAP | varies | H3 hexagons mapped to comarcas |
| BUILDING_FLOOD_RISK | ~2M | Core table: every building with composite score |
| PARISH_FLOOD_SUMMARY | 34 | Comarca-level aggregated statistics |
| H3_FLOOD_RISK_MAP | varies | Hexagonal heatmap data |
| FLOOD_RISK_ALERTS | 34 | Dynamic table with CRITICAL/HIGH/MODERATE/LOW |
| PARSED_POLICY_DOCS | 2 | Full text of parsed PDFs |
| POLICY_DOC_CHUNKS | varies | Paragraph-level text chunks for search |

### Composite Vulnerability Score Formula

```
Score = (Flood Risk Score x 0.40)
      + (Social Vulnerability Index x 100 x 0.30)
      + (Flood Zone Exposure x 0.30)

Where Flood Zone Exposure:
  A1 (Frequent Fluvial + Coastal) = 100
  A2 (Occasional Flooding)        = 80
  B  (Geomorphic Risk)            = 40
  C  (Low Risk)                   = 10
```
