# Subject night day table

Tabulates, night by night, how much faster each subject in an EarthRanger subject group moves at night than by day. The night/day ratio compares movement speed in darkness with speed in daylight: 1 is balanced, above 1 the subject moves faster at night, below 1 it moves faster by day.

Full documentation: [docs/index.html](docs/index.html) (User Guide, Technical Guide and Troubleshooting).

## What it does

| Step | What it does |
|---|---|
| **Set Workflow Details** | Name and description shown on the dashboard. |
| **Connect to EarthRanger** | The EarthRanger site to read from. |
| **Set Time Range** | The period to analyse. |
| **Set Groupers** | How the table is split: per subject, per time period or per spatial region. Leave empty for one view. |
| **Subject Group** | Fetches the location fixes of the subject group picked from the drop-down, for the time range. |
| **Process Relocations** | Cleans the fixes and drops known bad coordinates (0,0 / 1,1 / 180,90). |
| **Classify Is Night** | Flags each fix as night when the sun is below the horizon at its location and time. |
| **Persist Relocations** | Saves the classified fixes as `relocations.geoparquet`. |
| **Trajectory Segment Filter** | Joins the fixes into track segments, dropping segments outside the Trajectory Segment Filter limits. |
| **Persist Trajectories** | Saves the track segments as `trajectories.geoparquet`. |
| **Map Trajectory Columns** | Renames the subject and day/night columns (e.g. `subject_name`, `is_night`) and drops unused patrol columns. |
| **Sort Night Day Values** | Orders the segments by their day/night flag. |
| **Extract Date from Segment Start** | Adds a `segment_start_date` column: the date each segment starts on (kept in the data; the ratio dates each night itself). |
| **Split Groups** | One table of segments per group from Set Groupers. |
| **Summarize Night Day Df** | Computes the night/day ratio for each subject on each solar day (local noon to local noon, so one whole night), dated by that night's local midnight (2 decimal places). |
| **Exclude null values from night_day_ratio** | Keeps only rows with a ratio above 0, dropping solar days that couldn't be scored. |
| **Persist Night Day Df** | Saves each group's nightly ratios as a CSV file. |
| **Map Night Day DF** | Renames the columns to Date, Subject Name and Night Day Ratio for the table. |
| **Draw Night Day Table** | Draws the nightly ratios as a table, one row per subject per night. |
| **Persist Night Day Table as HTML** | Saves each table as an HTML file. |
| **Create Table Widget Single View** | Wraps each table as a **Night Day Ratio Table** widget. |
| **Merge Widget Views** | Combines the per-group tables into one widget, so the dashboard filter switches between them. |
| **Gather Dashboard** | Collects the table widget into the dashboard. |

## What it produces

| Output | Description |
|---|---|
| `relocations.geoparquet` | The cleaned location fixes, each with an `is_night` flag. |
| `trajectories.geoparquet` | The track segments built from those fixes. |
| One CSV per group | The night/day ratio for each subject on each solar day, with columns `subject_name`, `solar_date` and `night_day_ratio`. Days that couldn't be scored are left out. |
| One HTML table per group | As shown on the dashboard. |
| Dashboard | A **Night Day Ratio Table**; with groupers set, the dashboard filter switches between groups. |

## Requirements

- Access to an **EarthRanger** data source, set up as a connection

---

## 1. Load the Workflow

In the workflow runner, add a new template using this repository's URL. Once added, select
**Subject night day table** from the available templates list to load it.

---

## 2. Configure the Workflow

You'll be prompted to fill in the following parameters before running. Fields marked *Advanced* are collapsed
and can be left at their defaults.

### Set Workflow Details

| Field | Description | Default |
|---|---|---|
| Workflow Name *(required)* | A name for this run. | — |
| Workflow Description | Optional notes about this run. | — |

### Connect to EarthRanger

| Field | Description | Default |
|---|---|---|
| Data source *(required)* | Select one of your configured EarthRanger data sources. Choose it first: the Subject Group drop-down lists this data source's groups. | — |

### Set Time Range

| Field | Description | Default |
|---|---|---|
| Since *(required)* | The start of the period to analyse. | — |
| Until *(required)* | The end of the period to analyse. | — |
| Timezone | Timezone the times are given in. | — |

### Set Groupers

| Field | Description | Default |
|---|---|---|
| Group by | Optional. Add a grouper to get one table per group: **Category** (only *Subject Name*, one view per subject), **Temporal** (e.g. one per month) or **Spatial** (one per spatial region). Leave empty for a single table with a row for every subject and night. | — |

### Subject Group

| Field | Description | Default |
|---|---|---|
| Subject Group Name *(required)* | The EarthRanger subject group to analyse, picked from a drop-down of the chosen data source's subject groups. ⚠️ A group with mixed subtypes could give unexpected results. | — |

### Trajectory Segment Filter

| Field | Description | Default |
|---|---|---|
| Trajectory Segment Filter | Limits on segment length, duration and speed; segments outside them are dropped. Advanced. | length 0.001 m – 100 km, time 1 s – 48 h, speed 0.01 – 500 km/h |

---

## 3. Results

The workflow writes its files to the results folder the runner gives it (`ECOSCOPE_WORKFLOWS_RESULTS`) and
shows its dashboard when it finishes. A ratio of 1 means equal speed by night and day, above 1 faster at night, below 1 faster by day. Each row is one night: a solar day runs from local noon to local noon and is dated by the local midnight in it, so the night of 3–4 May is listed as 4 May. Days without more than 1 hour of observed data in both daylight and darkness, or with no movement by night or by day, are left out.

---

## Developing

The run form is set in `spec.yaml`:

- `rjsf-overrides` turns **Subject Group Name** into a drop-down: the `EarthRangerEnumResolver` transform fills it with the subject groups of the data source picked under **Connect to EarthRanger** (`depends_on: properties.er_connection.properties.data_source.properties.name`).
- `rjsf-overrides` `$defs` limits the **Category** grouper to *Subject Name* (`ValueGrouper.properties.index_name.oneOf`), the only subject column the views are split on.
- The steps are not in task groups, so each form override is keyed by its step id (e.g. `subjectgroup_observations.properties.subject_group_name`).
- **Summarize Night Day Df** uses `summarize_night_day_ratio` from `ecoscope-workflows-ext-custom` (0.1.5), which scores each solar day separately (night speed ÷ day speed) instead of slicing segments by UTC date, which would cut nights near longitude 0 in half. **Exclude null values from night_day_ratio** uses `filter_rows` from `ecoscope-workflows-ext-ste` (0.0.24) to drop the days it returns no ratio for.
- The **Spatial** grouper's *Spatial Regions* is still a text field. To make it a drop-down of the site's spatial feature groups, add an `EarthRangerEnumResolver` transform with `type: spatial_feature_group` on `groupers.properties.groupers`.

```bash
# recompile spec.yaml (first time: --install; after that: --update, which bumps VERSION.yaml)
./dev/recompile.sh --update

# run the test cases in test-cases.yaml
pixi run --manifest-path pixi.toml -e default bash dev/pytest-cli.sh subject_night_day_table --all
```

Documentation: [docs/index.html](docs/index.html).
