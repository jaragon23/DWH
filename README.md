# Deep Water Horizon Meiofauna Data Analytics Internship

This repository documents my work as a Data Analytics Intern with the Marine Ecology Lab at the University of Nevada, Reno, as part of the Nevada NSF EPSCoR Harnessing the Data Revolution for Fire Science (HDRFS) data analytics training and research internship pathway.

The internship project focused on long-term biological and environmental data from deep-sea benthic communities in the Gulf of Mexico, connected to research on ecological recovery following the Deepwater Horizon oil spill. My role centered on cleaning, standardizing, validating, and visualizing meiofauna and harpacticoid copepod abundance data so it could support scientific analysis, reporting, and eventual public repository submission.

> **Data note:** Raw and processed research datasets are not included in this public repository because they are part of ongoing unpublished research. This repo contains documentation, reproducible script templates, and project notes only.

---

## Project Context

**Project area:** Deep-sea benthic community recovery after the Deepwater Horizon oil spill
- **Institution:** University of Nevada, Reno — Marine Ecology Lab
- **Student:** Jeffery Aragon, College of Southern Nevada
- **Training background:** Python Data Analytics

The core goal was to standardize, reorganize, and validate raw meiofauna abundance data to match a published scientific formatting standard, making the data accessible to the broader scientific community through public repositories included in the United States Geological Survey (USGS) catalog.

## Mentor, Supervisor & Institution

**Mentor:** Elisa Baldrighi, Ph.D
- Research Assistant Professor, Marine Biology Lab, Department of Biology
- University of Nevada, Reno

**Professor:** Jeff Baguley
- Chief of the Marine Ecology Lab
- University of Nevada, Reno

- **Student:** Jeffery Aragon | College of Southern Nevada, Las Vegas 
---
## Project Scope

### Objective
Reorganize and standardize two raw meiofauna datasets to match the formatting of the published reference file `Bourque et al. PS2309`, enabling the data to be submitted for public scientific access.

### Onboarding
Received orientation materials and three Excel files to review before the first meeting, to get familiar with the data structure and formatting standard. The first meeting was used to define step-by-step goals and clarify scope.

### Datasets Referenced
| File | Role |
|------|------|
| `Bourque_etal_PS2309_macrofauna` | Reference/example file — target format for all output |
| `MDBC_2022_meio_coral` | Meiofauna coral dataset — required reformatting |
| `MEIOFAUNA ABUNDANCE 2022_updated` | Primary meiofauna abundance dataset |
| `Harpacticoid_family_2022_Dec.02.2024_Bang` | Harpacticoid crustacean family identification data |
| `2010_COPEPODA`, `2011`, `2014`, `2022` datasets | Multi-year harpacticoid heatmap inputs |

### Data Structure
Taxonomic abundance columns: `NEMA · COPE · NAUP · BIVAL · GPOD · ACARI · PLATY · OLIGO · SIPUN · APLAC · ECHINO`
Geographic and depth metadata: `Date · Latitude · Longitude · Depth · Fraction_upper_depth · Fraction_lower_depth`

---

## Experience & Progress Log

### Task 1 — SEC Column Transformation
The `SEC` column in `MDBC_2022_meio_coral` and `MEIOFAUNA ABUNDANCE 2022_updated` contained depth-range strings formatted as `"0-10cm"`, which needed to be split into two numeric columns to match the reference format.

**Actions taken:**
- Split `SEC` into `Fraction_upper_depth` and `Fraction_lower_depth`
- Removed `"cm"` suffixes and stripped whitespace
- Converted both columns to numeric (`errors='coerce'` to handle edge cases)
- Dropped the original `SEC` column
- Repositioned new columns to index positions 6 and 7
- Exported cleaned file to Excel

## Python Implementation

### -- SEC Column Parser & Validator -- 'DWH_Sec_Ssplit.py'
Loads the raw abundance spreadsheet, splits and cleans the depth-fraction column, repositions the new columns, exports the result, and includes a function to validate coordinate consistency between the old and new files.

```python
import pandas as pd
import numpy as np

# 1. Load spreadsheet
file_in = r"MEIOFAUNA ABUNDANCE 2022_updated.xlsx"
df = pd.read_excel(file_in)

# 2. Split the SEC column (e.g. "0-10cm") into two numeric depth columns
df[['Fraction_upper_depth', 'Fraction_lower_depth']] = df['SEC'].str.split('-', expand=True)

# 3. Clean "cm" suffix and whitespace
df['Fraction_lower_depth'] = df['Fraction_lower_depth'].astype(str).str.replace('cm', '', regex=False).str.strip()
df['Fraction_upper_depth'] = df['Fraction_upper_depth'].astype(str).str.replace('cm', '', regex=False).str.strip()

# 4. Convert to numeric (coerce errors to NaN to prevent crashes)
df['Fraction_upper_depth'] = pd.to_numeric(df['Fraction_upper_depth'], errors='coerce')
df['Fraction_lower_depth'] = pd.to_numeric(df['Fraction_lower_depth'], errors='coerce')

# 5. Drop original SEC column and reposition new ones at index 6 & 7
df = df.drop(columns=['SEC'])
upper_col = df.pop('Fraction_upper_depth')
lower_col = df.pop('Fraction_lower_depth')
df.insert(6, 'Fraction_upper_depth', upper_col)
df.insert(7, 'Fraction_lower_depth', lower_col)

# 6. Export
df.to_excel('dwh_jt1.xlsx', index=False)


def compare_excel_files(old_path, new_path, diff_path):
    """Compares coordinate columns between original and processed files."""
    df_old = pd.read_excel(old_path)
    df_new = pd.read_excel(new_path)

    df_old_check = df_old.drop(columns=['SEC'], errors='ignore')
    df_new_check = df_new.drop(columns=['Fraction_upper_depth', 'Fraction_lower_depth'], errors='ignore')

    common_cols = ['Latitude', 'Longitude', 'Depth']

    missing = [c for c in common_cols if c not in df_old_check.columns or c not in df_new_check.columns]
    if missing:
        print(f"Error: Missing columns: {missing}")
        return None

    diff = df_old_check[common_cols].compare(df_new_check[common_cols], align_axis=1, result_names=('Old', 'New'))

    if diff.empty:
        print("SUCCESS: Coordinate columns are identical.")
    else:
        print(f"WARNING: {len(diff)} rows differ.")
        diff.to_excel(diff_path)

    return diff
```
### Task 2 — Coordinate Validation
Verified consistency of geographic coordinate data (`Latitude`, `Longitude`, `Depth`) across the working files against the reference spreadsheet `Bourque_etal_PS2309`. Uploaded shared links and reported status to supervisor. No critical discrepancies found.

## Python Implementation
```python
# This checks if the new split columns actually resemble the old SEC column
print("\n--- Verifying Split Logic ---")
df_old_ver = pd.read_excel(file_in)
df_new_ver = pd.read_excel(file_out)

# Reconstruct SEC from new file to see if it matches old file
reconstructed_sec = df_new_ver['Fraction_upper_depth'].astype(str) + '-' + df_new_ver['Fraction_lower_depth'].astype(str) + "cm"
# Note: This reconstruction is approximate because we stripped whitespace/cm,
# but looking at a sample helps verify you didn't lose data.
print("Sample Check (Old vs New Split):")
check_df = pd.DataFrame({
    'Original_SEC': df_old_ver['SEC'].head(),
    'New_Upper': df_new_ver['Fraction_upper_depth'].head(),
    'New_Lower': df_new_ver['Fraction_lower_depth'].head()
})
print(check_df)

# Execute Comparison
compare_excel_files(file_in, file_out, file_diff)
### -- Validation Quantification 'Check_Diff.py"
```
### Task 3 — Harpacticoid Copepod Heatmap Visualizations
Following positive feedback on earlier meiofauna and taxa maps, Prof. Baguley requested heatmaps of harpacticoid copepod family abundances across four sampling years: **2010, 2011, 2014, and 2022**. The 2022 heatmap was completed first as a proof of concept.

**Data structure:**
- Rows (Y axis): Harpacticoid family names (e.g. Ameiriidae, Tisbidae, Miraciidae)
- Columns (X axis): Station IDs
- Values: Raw abundance counts per family per station
- Layer: `0–3cm` sediment fraction only (single layer, simplified)

**Approach:**
- Loaded the `Harpacticoid_family_2022.xlsx` file with Pandas
- Set the `Station` column as the DataFrame index
- Generated a heatmap using Seaborn with the `YlOrRd` colormap (yellow → orange → red, low → high abundance)
- Exported the output as a `.png` file

**Output:** A publication-ready heatmap image showing which families dominate at which stations across the Gulf of Mexico sampling area. Supervisor (Elisa) confirmed the approach and sent the 2010 dataset to continue the multi-year series.

**Years completed:** 2010 ✅ · 2011 ✅ · 2014 ✅ · 2022 ✅

#### Heatmap Output — 2010
Harpacticoid Family Abundance by Station (2010)

- **Ameiriidae** is the dominant family overall, with peak abundances at stations **LBNL4** (~300+) and **LBNL9** (~330+)
- **Miraciidae** shows a notable spike at **LBNL9** (~110) and a secondary presence at station **2.21**
- **Ectinosomatidae** shows a moderate peak at station **D050S**
- **Tisbidae** is sparsely distributed but present across several stations
- Most families show near-zero abundance across most stations, consistent with the patchy distribution typical of deep-sea benthic communities
- The **LBNL station cluster** (LBNL1, LBNL4, LBNL7, LBNL9) shows the highest overall activity — worth flagging for comparison against post-spill years (2011, 2014, 2022)

#### Heatmap Output — 2011
![Harpacticoid Family Abundance by Station (2011)]

- **Ameiridae** remains dominant, now with peak abundances at **D044S**, **NF006MOD**, **D034S**, and **LBNL4** — a broader geographic spread compared to 2010
- **Ectinosomatidae** shows a notable spike at **NF012** (~160), largely absent in 2010 — a potential indicator of post-spill community shifts
- **Canthocamptidae** shows a strong presence at **LBNL4**, suggesting this station remained a hotspot one year post-spill
- **Cletopsyllidae** shows moderate activity at **LBNL1**, a new appearance not prominent in 2010
- Overall family diversity appears broader in 2011, with more families detectable across stations

#### Heatmap Output — 2014
![Harpacticoid Family Abundance by Station (2014)]

- Peak abundances are notably lower across the board (~160 max vs ~330 in 2010) — a potential signal of longer-term community suppression following the spill
- **Cletopsyllidae** emerges as the dominant peak at **NF006MOD** (~170), a family that was minor in prior years
- **Canthocamptidae** shows consistent distribution across multiple stations, becoming one of the more broadly distributed families by 2014
- **Ameiridae** remains present but at reduced peak levels compared to 2010 and 2011
- **Miraciidae** shows a moderate presence at **LBNL7**, continuing its appearance from prior years
- **NF006MOD** appears as a new hotspot, replacing the LBNL cluster that dominated in 2010

#### Heatmap Output — 2022
![Harpacticoid Family Abundance by Station (2022)]

- **Ameiridae** again dominates with peaks at **D031S** (~150) and **2.21**, consistent with its role as the most resilient family across all years
- Family diversity is the highest across all four years — 21+ families showing detectable presence, suggesting longer-term community recovery
- **Ectinosomatidae** shows a broad, low-level distribution across many stations — wider spread than any prior year
- **Canthocamptidae** and **Miraciidae** maintain consistent presence, indicating these families stabilized as part of the recovering community
- The **LBNL station cluster** shows reduced dominance compared to 2010, with abundance more evenly spread across the full station range
- 
### -- Multi-Year Harpacticoid Heatmap Generator -- `Heatmap_Harpacticoid.py` 
Produces a station-by-family abundance heatmap from any year's harpacticoid dataset. The same script is reused across the 2010, 2011, 2014, and 2022 datasets by swapping the input file path.

```python
# Import libraries
import pandas as pd
import numpy as np

# Import visualization libraries
import matplotlib.pyplot as plt
import seaborn as sns

# Load harpacticoid family abundance data
# Rows = Harpacticoid families, Columns = Station IDs
file_in = r"Harpacticoid family_2022.xlsx"
df = pd.read_excel(file_in)

# Set family names as the index (Y axis)
df.set_index('Station', inplace=True)

# Generate heatmap
plt.figure(figsize=(15, 8))

# The original color theme was difficult to clearly display abundance intensity so I modified the heatmap color scheme after recieving this feedback 
sns.heatmap(df, cmap="YlOrRd", annot=False, linewidths=.5)

# Set the titles of the chart, Y and X labels accordingly 
plt.title('Harpacticoid Family Abundance by Station (2022 Data Samples)')
plt.ylabel('Harpacticoid Family')
plt.xlabel('Station ID')

# Export headtmap
file_out = r"Heatmap_of_Harpacticoid_family_2022.png"
plt.savefig(file_out, bbox_inches='tight', dpi=150)
plt.show()
print(f"Heatmap saved to {file_out}")
```
> **Note:** To generate heatmaps for 2010, 2011, or 2014, update `file_in` and the title string. The 2010 dataset (`2010_COPEPODA.xlsx`) contains 33 stations and 21 harpacticoid families — same column structure, ready to run.
<img width="1215" height="759" alt="image" src="https://github.com/user-attachments/assets/cc271700-7740-4532-9225-906c6a0ca10a" />

#### Cross-Year Summary
| Year | Approx. Max Abundance | Dominant Family | Notable Pattern |
|------|------------------------|------------------|------------------|
| 2010 | ~330 | Ameiridae | LBNL cluster hotspot; pre/early spill baseline |
| 2011 | ~290 | Ameiridae | Broader spread; Ectinosomatidae spike emerges |
| 2014 | ~170 | Cletopsyllidae / Canthocamptidae | Reduced peak abundance; NF006MOD becomes hotspot |
| 2022 | ~155 | Ameiridae | Highest family diversity; possible recovery pattern |

> Together, the four heatmaps provide a visual timeline of how the harpacticoid copepod community in the Gulf of Mexico responded to and recovered from the 2010 Deepwater Horizon oil spill.

### Task 4 — Bibliographic Research & Literature Review
Assigned to conduct a systematic literature search to build a scientific reference base supporting the lab's ongoing research, across three core topic areas relevant to the DWH project and meiofauna ecology.

Developed a bibliographic search with Python using the following keywords – nematodes, copepods, meiofauna deep-sea diversity, oil spill, DWH. 
All the available literature was retrieved, and it will be used for a review paper on global patterns in dee-sea meiofauna diversity.

**Research strategy developed:**
- Boolean search logic (`AND` / `OR`) to refine queries and reduce noise
- Citation mapping — starting from high-quality anchor papers and tracing references outward
- Google dorking (advanced search syntax) to surface specific document types and sources
- AI-assisted abstract screening tools to efficiently filter large result sets
- A structured logging framework to track exhausted queries and avoid redundant searches

**Topic areas researched:**
| Topic Area | Query Focus | Sources Found |
|------------|-------------|----------------|
| Biodiversity Patterns | Meiofauna deep-sea biodiversity, global distribution | 15 |
| Nematodes & Harpacticoids | Deep-sea impact on nematodes and harpacticoid copepods | 14 |
| Marine Oil Spill / DWH | Deepwater Horizon oil spill ecology and impacts | 15 |
| **Total** | | **44** |

Sources span 1992–2025, with key works from journals including *PNAS*, *Nature*, *Frontiers in Marine Science*, *Science of the Total Environment*, and *Annual Reviews of Marine Science*.

**Notable sources identified:**
- Murawski et al. (2023) — overview of living marine resource vulnerability to the DWH spill
- Ingels et al. (2023) — deep-sea meiofauna connectivity and biodiversity
- Vanreusel et al. (2023) — meiofauna biogeography paradigms and challenges
- Bonk et al. (2025) — harpacticoid assemblages in deep-sea trenches (most recent)
- Fisher et al. (2016) — direct assessment of DWH impact on deep-sea ecosystems

**Status at reporting:** Research framework established, initial passes complete across all three topic areas, with work ongoing. Supervisor (Elisa) invited for a progress review meeting.

([https://docs.google.com/spreadsheets/d/1eob1o2CBKbQBOx-_njTvUdF7DON1_miD/edit?gid=1003607963#gid=1003607963:~:text=%3Ciframe%20src%3D%22https%3A//docs.google.com/spreadsheets/d/e/2PACX%2D1vT8p7iAAUzqnxQ2LzE8mbAmpVgyKXuIQqRDab4kQTe3dy5WzhLgYmyZKtly0UJ2_w/pubhtml%3Fwidget%3Dtrue%26amp%3Bheaders%3Dfalse%22%3E%3C/iframe%3E])

### Task 5 — Harpacticoid Family Table Reformatting
Reformatted the `Harpacticoid_family_2022` sheet to reorganize harpacticoid crustacean family abundance data station-by-station, matching the provided example format.

**Progress:**
- Completed abundance entries for all families across all stations
- Identified a formatting inconsistency: rows 24–28 (Adult, Copepodite, Unid. Family, broken/missing, Origin. Total) were being included but were not part of the final presentation format
- Manual fix was possible but not scalable — flagged for a scripted solution
- Reported status to supervisor ahead of scheduled meeting

---
### Example Setup
```bash
python -m venv .venv
source .venv/bin/activate  # macOS/Linux
pip install -r requirements.txt
```
Run SEC transformation:
```bash
python scripts/dwh_sec_split.py \
  --input "data/MEIOFAUNA ABUNDANCE 2022_updated.xlsx" \
  --output "outputs/dwh_jt1.xlsx" \
  --reference "data/MEIOFAUNA ABUNDANCE 2022_updated.xlsx" \
  --diff-output "outputs/coordinate_diff.xlsx"
```
Run heatmap generation:
```bash
python scripts/heatmap_harpacticoid.py \
  --input "data/Harpacticoid_family_2022.xlsx" \
  --output "outputs/Heatmap_of_Harpacticoid_family_2022.png" \
  --title "Harpacticoid Family Abundance by Station (2022 Data Samples)"
```
---
## Dataset Summary Output

`df.describe()` on the cleaned `MEIOFAUNA ABUNDANCE 2022` dataset:

| Stat | Latitude | Longitude | Depth (m) | NEMA | N m⁻² |
|------|----------|-----------|-----------|------|--------|
| count | 123 | 123 | 123 | 123 | 123 |
| mean | 28.67° | -88.43° | 1465 m | 433 | 227,772 |
| std | 0.20 | 0.28 | 262 m | 271 | 130,829 |
| min | 27.83° | -89.49° | 735 m | 77 | 37,039 |
| max | 28.99° | -87.76° | 2389 m | 1698 | 726,053 |

**123 records** across stations in the Gulf of Mexico, spanning depths of **735m to 2,389m**. Nematodes (NEMA) were the dominant taxonomic group, with a mean abundance of 433 per sample and peak densities reaching **726,053 individuals per m²**.
---
## Repository Structure
```text
//DWH-Internship-UNR/
//├── README.md                              # This file — full project documentation
//├── LICENSE
//├── .gitignore
//├── requirements.txt
//├── scripts/
//│   ├── dwh_sec_split.py                   # SEC column parser, cleaner & validator
//│   └── heatmap_harpacticoid.py            # Multi-year harpacticoid heatmap generator
//├── data/
//│   ├── README.md
//│   └── Bibliographic_Research_search_results.xlsx   # Literature review — 44 sources across 3 topics
//│   └── (other data files not included — proprietary research data)
//└── outputs/
//    ├── README.md
//    └── (processed Excel files and heatmap .png exports)
```
> **Note:** Raw and processed data files are not included in this repository as they are part of ongoing unpublished scientific research at UNR.
---
## Key Takeaways
- Gained hands-on experience working with real scientific research data in an academic lab setting
- Applied Python data-wrangling skills (Pandas) to solve practical formatting problems that would be tedious and error-prone to do manually
- Learned the importance of data standardization for scientific reproducibility and public data sharing
- Identified the limitations of manual fixes at scale and began developing scripted solutions — a direct application of the programming mindset to research workflows
- Collaborated remotely with a university research team, communicating progress and blockers clearly in writing
- Practiced scientific visualization using seaborn and matplotlib
- This internship bridged my networking background with data analytics in a meaningful, real-world context — reinforcing that the transition I'm making is the right one
- The data matrices, cleaned up and organized, will be publicly available on the USGS repository for the project after NOAA and USGS quality check
---
## Related Resources
- [USGS Data Catalog]([https://www.usgs.gov/products/data])
- [USGS Data Release Products]([https://www.sciencebase.gov/catalog/item/677e95a4d34e760b392c48d2]) — Public repository where finalized data will be published 
- [UNR Marine Ecology Lab](https://www.unr.edu) — University of Nevada, Reno
- [NSF EPSCoR HDRFS](https://hdrfs.epscorspo.nevada.edu) — Nevada National Science Foundation - Established Program to Stimulate Competitive Research - Harnessing The Data Revolution For Fire Science Data Analytics Training Program
---
## Status
Work in progress. This repository is intended as a cleaned, public-facing portfolio and documentation version of the internship project — not as a release of unpublished lab data.

## Special thanks
- Data Analytics Coordinator (UNLV), Christopher Mendoza
- Data Analytics Coordinator (UNR), Ilaria Vinci
- Associate Research Scientist (DRI), Meghan Collins
- Department Chair – Computing and Information Technology (CSN), Dr. Naser Heravi
- NCLab Manager of Administration (Reno, NV), AJ Moore
- NCLab Coaches (Reno, NV): Heidi Seaton & Armando Serrato
- NCLab Director of Sales (Reno, NV), Heather Godwin
- Collegue and Mentee (CSN), Iryna Rozenstein
- Truckee Meadows Community College (TMCC)
---

*Jeffery Aragon | College of Southern Nevada | Data Analytics Intern, UNR Marine Biology Lab 2026*
