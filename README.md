# Brain Tumor Distribution Analysis 
An interactive Power BI dashboard that explores 10,000 brain tumor patient records across tumor type, grade, brain region, age group, tumor size and diagnosis year. Two slicers let the viewer slice the story by **tumor grade** (image chiclet slicer) and by **histology confirmation status** (Confirmed / Unconfirmed button slicer).

The aim is to turn raw clinical records into answers a hospital, neuro-oncology unit or health planner can act on: where the caseload sits, who is affected, how severe it is, and where diagnostic and resource gaps exist.



## Dataset

- **Source file:** `Brain_Tumor_Data.csv`
- **Size:** 10,000 patient records, 12 columns
- **Period covered:** 2015 to 2025 (11 diagnosis years)
- **Fields:** Patient ID, Age, Gender, Tumor Type, Tumor Grade, Affected Brain Region, Tumor Size (cm), Symptoms, Laterality, Histology Confirmation, Treatment Type, Date of Diagnosis
- **Data quality:** no missing values in any column

Tumor types in the data are Glioma, Meningioma, Medulloblastoma, Ependymoma and Pituitary Tumor. Grades run from Grade 1 to Grade 4, and brain regions include Frontal, Temporal, Parietal, Occipital, Cerebellum, Brainstem, Pituitary and Multiple.

---
## Business Problem

Brain tumor care is resource heavy. Different grades need different specialists, equipment and follow-up pathways, and decisions made on incomplete diagnoses carry real clinical risk. Decision makers need to know:

- How many patients are coming in each year, and is intake stable?
- Which tumor types and brain regions dominate each grade?
- Which age groups carry the biggest burden?
- How does severity (tumor size) change as grade increases?
- How many cases are still waiting on histology confirmation?

This dashboard answers those questions in one place, with the grade and confirmation slicers keeping every visual in sync.

---
## Tools Used

- **Power BI Desktop** for modelling, DAX and report design
- **DAX** for the calculated column and KPI measures
- **Chiclet Slicer** (custom visual) for the image-based grade selector
- **Button slicer** for Confirmed / Unconfirmed filtering
- **GitHub** for documentation

---

## Process

**1. Data import and inspection**
The CSV was loaded into Power BI and profiled for data types, blanks and category consistency. Date of Diagnosis was set to a date type so a year hierarchy could drive the trend line, and Tumor Size (cm) was set as a numeric field.

**2. Age grouping**
A calculated column, `Age Group`, bins patients into 0-10, 11-20, 21-30, 31-40, 41-50, 51-60 and 60+. This turns a continuous age field into a readable ranking of the most affected groups.

**3. Image lookup for the grade slicer**
A small `Tumor Images` table holds one row per grade with an `Icon URL`. The Chiclet Slicer reads this table, so each grade is shown as a brain scan icon instead of plain text.

**4. Measures table**
All KPI logic lives in a dedicated measures table so the model stays tidy and every measure is easy to find.

**5. Visual build**
Visuals were added in this order: KPI cards, tumor type donut, brain region bar chart, age group column chart, annual diagnosis line chart, and finally the MRI image panel that reacts to the grade selection.

**6. Interactivity**
The chiclet slicer (grade) and the button slicer (confirmation status) were wired to every visual so a single click reshapes the whole page.

**7. Design**
A dark, monochrome theme was used so the white data marks and the MRI image stand out, with consistent fonts and card styling across the page.

---

## Data Model

The model has two data tables plus a measures table:

- **Brain Tumor Data**, the main fact table holding the 12 source columns plus the calculated `Age Group` column
- **Tumor Images**, a lookup table with the `Icon URL` used by the chiclet slicer
- **MEASURES**, a table that holds only the DAX measures

---

## Measures Created


**Avg Intake py**
The average number of patients diagnosed per year for the current selection. It is the total patient count divided by the number of distinct diagnosis years, so it responds to the grade and confirmation slicers.

```DAX
Avg Intake py = 
VAR _StartYear =
    YEAR ( MIN ( 'Brain Tumor Data'[Date of Diagnosis] ) )
VAR _EndYear =
    YEAR ( MAX ( 'Brain Tumor Data'[Date of Diagnosis] ) )
VAR _NumYears = _EndYear - _StartYear + 1

VAR _TotalPatients =
    CALCULATE ( COUNTROWS ( 'Brain Tumor Data' ) )

RETURN
    DIVIDE ( _TotalPatients, _NumYears )

**vs Avg Intake py**
A formatted text version of the yearly average that sits under the Total Patients KPI and carries a directional arrow, giving a quick sense of the typical annual caseload for the selection.

VAR _CurrentIntake =
    [Avg Intake py]
VAR _OverallIntake =
    CALCULATE (
        [Avg Intake py],
        ALL ( 'Brain Tumor Data' )
    )
VAR _Diff = _CurrentIntake - _OverallIntake

RETURN
    SWITCH (
        TRUE(),
        _Diff > 0, UNICHAR(9650) & " " & FORMAT(_CurrentIntake, "0.0"),
        _Diff < 0, UNICHAR(9660) & " " & FORMAT(_CurrentIntake, "0.0"),
        FORMAT(_CurrentIntake, "0.0")
    )


**MABR (Most Affected Brain Region)**
VAR TotalPatients = COUNTROWS('Brain Tumor Data')
VAR RegionCounts =
    SUMMARIZE(
        'Brain Tumor Data',
        'Brain Tumor Data'[Affected Brain Region],
        "Cnt", COUNTROWS('Brain Tumor Data')
    )
VAR TopRegion = TOPN(1, RegionCounts, [Cnt], DESC)
VAR RegionName = MAXX(TopRegion, 'Brain Tumor Data'[Affected Brain Region])
VAR RegionCount = MAXX(TopRegion, [Cnt])

RETURN
RegionName & " (" & FORMAT(DIVIDE(RegionCount, TotalPatients), "0%") & ")"


**Most Affected Age Group**
Returns the age group with the most patients in the current selection
VAR AgeGroupCounts =
    SUMMARIZE(
        'Brain Tumor Data',
        'Brain Tumor Data'[Age Group],
        "PatientCount", DISTINCTCOUNT('Brain Tumor Data'[Patient ID])
    )
VAR TopGroup =
    TOPN(1, AgeGroupCounts, [PatientCount], DESC)
RETURN
MAXX(TopGroup, 'Brain Tumor Data'[Age Group])


**Age Group (calculated column)**

```DAX
VAR AgeValue = 'Brain Tumor Data'[Age]
RETURN
SWITCH(
    TRUE(),
    AgeValue >= 0 && AgeValue <= 10, "0-10",
    AgeValue >= 11 && AgeValue <= 20, "11-20",
    AgeValue >= 21 && AgeValue <= 30, "21-30",
    AgeValue >= 31 && AgeValue <= 40, "31-40",
    AgeValue >= 41 && AgeValue <= 50, "41-50",
    AgeValue >= 51 && AgeValue <= 60, "51-60",
    AgeValue > 60, "60+",
    "Unknown"
)
```

**Total Patients, Average Age and Average Tumor Size**
Built from standard aggregations: a row count of patients, the average of `Age`, and the average of `Tumor Size (cm)`

---
## Dashboard Design

- **Header and KPI strip:** Total Patients, Average Age and Average Tumor Size, each with a sparkline and a supporting line (average yearly intake, most affected age group, most affected brain region)
- **Left column:** tumor type distribution donut and a ranked bar chart of top affected brain regions
- **Centre:** an MRI image that changes with the selected grade
- **Right column:** annual patients diagnosed trend line and a ranked column chart of the top affected age groups
- **Left rail:** chiclet slicer to choose Grade 1, 2, 3 or 4
- **Top right:** Confirmed / Unconfirmed button slicer

---

## Dashboard Views Explained

Both views below use the **Confirmed** button slicer, with a different grade picked in the chiclet slicer on the left. Everything on the page, including the MRI image, responds to those two selections.

### View 1: Grade 1, Confirmed

![Grade 1 confirmed cases](/grade1-confirmed.png)

**What the selection does:** choosing the Grade 1 icon and Confirmed filters the page to 2,428 histology-confirmed Grade 1 patients.

**KPI cards**
- **Total Patients: 2,428.** The supporting line shows an average intake of about 220.7 patients per year, and the sparkline shows how that intake moved over time.
- **Average Age: 51.15.** The most affected age group is 41-50.
- **Average Tumor Size: 1.99 cm.** The most affected brain region (MABR) is the Pituitary at 24%. This is the smallest tumor size of any grade, which fits the low-grade profile.

**Tumor Type Distribution.** Meningioma makes up 68.45% of cases, Pituitary Tumor 23.72% and Glioma 7.83%. Grade 1 is dominated by slow-growing, usually benign tumor types.

**Top Affected Brain Region.** The Pituitary leads with 576 patients, followed by Occipital (403), Parietal (382), Temporal (379), Multiple (345) and Frontal (343). The Pituitary lead comes from pituitary tumors, which appear only in Grade 1.

**Top Affected Age Group.** The 41-50 group has 718 patients, 51-60 has 708 and 60+ has 600. These three groups account for about 83% of Grade 1 patients. Younger groups are small, with 206 in 31-40 and 178 in 21-30.

**Annual Patients Diagnosed.** Diagnoses peak in 2016 at about 250 patients, settle near 212 around 2018 and 2019, climb to a second peak of about 240 in 2022, then ease back toward 212.

**Takeaway.** Grade 1 is a middle-aged, small-tumor, high-volume group. It is the largest confirmed group and a natural candidate for planned, routine pathways such as surgery lists and structured follow-up.

### View 2: Grade 2, Confirmed

![Grade 2 confirmed cases](/grade2-confirmed.png)

**What the selection does:** choosing the Grade 2 icon and Confirmed filters the page to 680 histology-confirmed Grade 2 patients, and the MRI panel switches to the Grade 2 scan.

**KPI cards**
- **Total Patients: 680.** The average intake drops to about 61.8 patients per year, roughly a quarter of Grade 1.
- **Average Age: 42.35.** Patients are almost nine years younger than in Grade 1, and the most affected group is still 41-50.
- **Average Tumor Size: 4.01 cm.** This is double the Grade 1 size. The most affected brain region is now the Brainstem at 24%.

**Tumor Type Distribution.** Ependymoma leads at 45.15%, followed by Glioma at 28.97% and Meningioma at 25.88%. The type mix has shifted from benign-leaning tumors to ones that need more specialist handling.

**Top Affected Brain Region.** The Brainstem has 160 patients and the Cerebellum 147, together about 45% of cases. Temporal (92), Frontal (87), Occipital (85), Parietal (77) and Multiple (32) follow. The disease moves toward deeper, harder-to-operate regions.

**Top Affected Age Group.** The 41-50 group leads with 166 patients, then 60+ (121), 21-30 (112), 31-40 (103), 51-60 (96) and 11-20 (76). The spread across all ages is much wider than in Grade 1 and includes children and young adults.

**Annual Patients Diagnosed.** Diagnoses are lowest in 2016 at about 44 patients, peak in 2019 at about 78, dip to about 53 in 2023 and recover to about 63 by 2025. The swings are proportionally larger than in Grade 1 because the numbers are smaller.

**Takeaway.** Grade 2 is a smaller but more demanding group. Patients are younger, tumors are larger and sit in deeper regions, and the age range is wide, so services need to cover paediatric through older adult care.

### Comparing the two views

- **Severity:** tumor size doubles from 1.99 cm to 4.01 cm.
- **Age:** average age falls from 51.15 to 42.35, and the age spread widens.
- **Type:** meningioma gives way to ependymoma and glioma.
- **Region:** the Pituitary gives way to the Brainstem and Cerebellum.
- **Volume:** yearly intake falls from about 221 to about 62 patients.

### Using the Unconfirmed button

Switching the button slicer to Unconfirmed on the same grade selections shows the cases still waiting for histology confirmation. Across the dataset, 1,588 Grade 1 and 441 Grade 2 patients are unconfirmed, roughly four in ten patients in those grades, compared with about one in ten in Grades 3 and 4. This makes the Unconfirmed view a practical worklist for pathology follow-up.

---

## Business Questions

1. How many patients does each grade bring in per year, and is that intake stable enough to plan staffing and theatre capacity around?
2. Which tumor types drive each grade, and do they need different specialist teams?
3. Which brain regions carry the highest caseload, and does that match where neurosurgery and imaging capacity is concentrated?
4. Which age groups should awareness, screening and referral programmes prioritise?
5. How much does tumor size grow from one grade to the next, and what does that imply for the urgency of intervention?
6. Why do patients get younger as grade increases, and what does that mean for paediatric and young adult services?
7. What share of cases is still waiting on histology confirmation, and does that gap differ by grade?
8. Were the peaks and dips in annual diagnoses driven by real incidence changes or by access, diagnostic capacity or reporting changes?

---

## Overall Findings and Insights

These findings come from the whole dashboard with no slicer selected, covering all 10,000 patients across every grade and both confirmation statuses.

### Volume and trend
- The dataset holds **10,000 patients** diagnosed between 2015 and 2025, an average of about **909 diagnoses per year**.
- Yearly diagnoses stay within a narrow band of roughly 855 to 980, so intake is steady with a modest swing rather than a sharp rise or fall.
- Diagnoses peaked in 2022 at about 980 patients, then declined to about 857 by 2025, close to where the period began.

### Who is affected
- The average patient age is **44.87**.
- The **60+ age group is the largest** with 2,372 patients, followed by 41-50 with 2,100 and 51-60 with 1,855.
- Patients aged 41 and above make up about 63% of all cases, and patients over 60 alone account for about 24%.
- Patients aged 20 and under number 1,324, or 13% of the total, so paediatric and young adult cases are a meaningful share of the caseload.

### Tumor mix
- **Glioma is the most common type at 40.58%**, followed by **Meningioma at 30.55%**. Together they account for more than 70% of all cases.
- Medulloblastoma (about 10%), Ependymoma (9.61%) and Pituitary Tumor (9.25%) split the remainder almost evenly.
- Tumor type is closely tied to age. Medulloblastoma averages about 10 years old and Ependymoma about 30, while Glioma averages 50 and Meningioma 55.

### Brain regions
- The caseload is spread across the main lobes. Occipital leads with about 1.7K patients, with Temporal, Parietal and Frontal at about 1.6K each and the Cerebellum at about 1.5K.
- Pituitary (0.9K), Multiple (0.6K) and Brainstem (0.5K) carry far fewer cases.
- The most affected region, Occipital, accounts for only 17% of cases, so no single region dominates.

### Severity
- The average tumor size is **4.38 cm**, which is fairly large.
- Size depends heavily on type. Pituitary tumors (2.0 cm) and Meningiomas (2.2 cm) are small, while Glioma (5.9 cm) and Medulloblastoma (6.5 cm) are large.
- Nearly half of all patients, **48.6%**, are Grade 3 or Grade 4. Grade 4 alone is 40.8%, Grade 1 is 40.2% and Grade 2 is 11.2%.

### Diagnosis and treatment
- About **25.3% of patients (2,527) have no histology confirmation**, so one in four cases is still unverified.
- Surgery is the most common treatment at about 60%, followed by radiation at 18%, observation at 12% and chemotherapy at 10%.
- Female patients make up about 57% of the dataset.

### What this means for decisions
1. Nearly half of patients are high grade, so surgical, radiotherapy and chemotherapy capacity matters more than the average tumor size alone suggests.
2. Glioma and meningioma are over 70% of the load and sit in different age bands, so they need different follow-up pathways.
3. Medulloblastoma and ependymoma patients are young and present with large, high-grade tumors, which justifies a dedicated paediatric track.
4. Because cases are spread across the lobes, imaging and neurosurgery capacity cannot be built around one region.
5. One in four cases lacks histology confirmation, which is a diagnostic quality gap worth closing.
6. Intake is stable, so staffing can be planned on a steady base with flexibility for small yearly swings.

---


## Recommendations

1. **Plan capacity by grade, not by total volume.** Use the average yearly intake for each grade (about 220 for Grade 1 and 62 for Grade 2 in confirmed cases) to size outpatient follow-up, imaging slots and surgical lists separately, and review them yearly because intake swings.

2. **Build specialist pathways around the dominant tumor type and region.** Pituitary and meningioma cases suggest tight links between neurosurgery and endocrinology for Grade 1. Brainstem and cerebellum ependymoma in Grade 2 calls for specialist neurosurgical and paediatric-capable oncology expertise.

3. **Protect capacity for high-grade care.** Nearly half of all patients are Grade 3 or 4, so radiotherapy, chemotherapy and surgical capacity should be planned for that share rather than for the average case.

4. **Target awareness and early detection at ages 41-50 and over 60.** These groups carry the largest burden. Primary care referral guidelines for persistent headaches, vision problems and neurological symptoms should be sharpened for these age bands.

5. **Strengthen young adult and paediatric readiness.** Grade 2 spans every age group and about 13% of all patients are aged 20 or under, so services cannot be designed only around middle-aged patients.

6. **Catch tumors earlier.** Because size roughly doubles from Grade 1 to Grade 2, faster referral and imaging turnaround would move more patients into the low-grade, small-tumor group where outcomes are more favourable.

7. **Close the histology confirmation gap.** Set a target turnaround for pathology confirmation, prioritise unconfirmed Grade 1 and Grade 2 cases for review, and audit whether treatment is being started ahead of confirmation.

8. **Investigate the yearly peaks and dips before acting on them.** Check whether they reflect referral patterns, imaging availability, screening drives or reporting changes, so staffing is not built around noise.

---

## Limitations

- The grade views above are filtered to **Confirmed** cases only, so unconfirmed patients are excluded from those figures.
- Only the Grade 1 and Grade 2 views are documented here. Grade 3 and Grade 4 are available through the same slicer.
- Annual figures are counts by diagnosis year and are not adjusted for population size or referral volume.
- Treatment, symptoms, gender and laterality are in the model but not yet featured in the dashboard visuals.
- This is an analytical portfolio project, not a clinical decision tool.
