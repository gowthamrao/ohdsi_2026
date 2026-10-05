OHDSI Treatment Pathways Protocol:

Exploration of patients with hypertension, type 2 diabetes mellitus, and depression

**Authors:**

Patrick Ryan, PhD, Janssen Research and Development

Jon Duke, MD, Regenstrief Institute

Martijn Schuemie, PhD, Janssen Research and Development

George Hripcsak, MD, Columbia University

Nigam Shah, PhD, Stanford University

&lt;<other authors added who contribute results from their data and satisfy ICMJE guidelines&gt;>

**Date:** 30 November 2014

**Acknowledgement:** The analysis is based in part on work from the Observational Health Sciences and Informatics collaborative. OHDSI (<http://ohdsi.org>) is a multi-stakeholder, interdisciplinary collaborative to create open-source solutions that bring out the value of observational health data through large-scale analytics.

The authors declare the following disclosures: Drs. Ryan, Schuemie are employees of Janssen Research & Development.

# Background

Chronic conditions often require long-term treatment which may involve multiple alternative therapies over time. Treatment guidelines exist for many chronic diseases, but little literature is available to summarize the real-world pathways that patients experience in practice. Little is known about the specific sequence of treatments that patients follow upon initiation, particularly for a long duration. The OHDSI data network will be used to conduct a systematic analysis of three chronic diseases: hypertension, type 2 diabetes mellitus, and depression, to summarize the treatment pathways encountered by patients during the first 3 years of therapy.

# Objective

We aim to characterize the prevalence of different treatment pathways experienced by patients with 3 chronic conditions: hypertension, type 2 diabetes mellitus, and depression. We will systematically summarize the treatment pathways observed among patients with the disease who have at least 3 years of continuous observation and persistent treatment following initiation. We will stratify the results by year to evaluate temporal trends, and will further stratify by data source to determine if treatment pathways vary by population, geography, and data capture process.

# Data sources

The analyses will be performed across a network of observational healthcare databases. All databases have been transformed into the OMOP Common Data Model, version 4 or OMOP Common Data Model, version 5. The complete specification for OMOP Common Data Model, version 4 is available at: <http://omop.org/cdm>. The complete specification for OMOP Common Data Model, version 5 is available at: <https://github.com/OHDSI/CommonDataModel>. The following databases will be included in this analysis:

- Truven MarketScan Commercial Claims and Encounters (CCAE)
- Truven MarketScan Medicare Supplemental Beneficiaries (MDCR)
- Truven MarketScan Multi-state Medicaid (MDCD)
- Optum ClinFormatics (Optum)
- Clinical Practice Research Datalink (CPRD)
- Regenstrief Institute / Indiana Network for Patient Care (INPC)
- Columbia University / New York Presbyterian (NYP)
- Stanford University (STRIDE)
- &lt;<add others who agree to participate&gt;>

Truven MarketScan Commercial Claims and Encounters (CCAE)

CCAE is an administrative health claims database for active employees, early retirees, COBRA continues, and their dependents insured by employer-sponsored plans (individuals in plans or product lines with fee-for-service plans and fully capitated or partially capitated plans). As of 30November2014, CCAE contained 117m patients with patient-level observations from Jan2000 through Jul2014. Source codes used in CCAE include: conditions- ICD-9-CM; drugs: NDC, HCPCS, ICD-9-CM; procedures: CPT-4, HCPCS, ICD-9-CM; lab: LOINC.

The ETL specification for transforming CCAE into the OMOP CDM is available at: <http://omop.org/cdm>.

ACHILLES has been used to characterize the database and provide a data quality assessment. The ACHILLES summary is available internally within Janssen at: <http://hix.jnj.com/achilles/#/truven_ccae/dashboard>.

Truven MarketScan Medicare Supplemental Beneficiaries (MDCR)

MDCR is an administrative health claims database for Medicare-eligible active and retired employees and their Medicare-eligible dependents from employer-sponsored supplemental plans (predominantly fee-for-service plans). Only plans where both the Medicare-paid amounts and the employer-paid amounts were available and evident on the claims were selected for this database.. As of 30November2014, MDCR contained 9m patients with patient-level observations from Jan2000 through Jul2014. Source codes used in MDCR include: conditions- ICD-9-CM; drugs: NDC, HCPCS, ICD-9-CM; procedures: CPT-4, HCPCS, ICD-9-CM; lab: LOINC.

The ETL specification for transforming MDCR into the OMOP CDM is available at: <http://omop.org/cdm>.

ACHILLES has been used to characterize the database and provide a data quality assessment. The ACHILLES summary is available internally within Janssen at: <http://hix.jnj.com/achilles/#/truven_mdcr/dashboard>.

Truven MarketScan Multi-state Medicaid (MDCD)

MDCD is an administrative health claims database for the pooled healthcare experience of Medicaid enrollees from multiple states. As of 30November2014, MDCD contained 16m patients with patient-level observations from Jan2006 through Dec2012. Source codes used in MDCD include: conditions- ICD-9-CM; drugs: NDC, HCPCS, ICD-9-CM; procedures: CPT-4, HCPCS, ICD-9-CM; lab: LOINC.

The ETL specification for transforming MDCD into the OMOP CDM is available at: <http://omop.org/cdm>.

ACHILLES has been used to characterize the database and provide a data quality assessment. The ACHILLES summary is available internally within Janssen at: <http://hix.jnj.com/achilles/#/truven_mdcd/dashboard>.

Optum ClinFormatics (Optum)

Optum is an administrative health claims database for members of United Healthcare, who enrolled in commercial plans (including ASO, 36.31M), Medicaid (prior to July 2010, 1.25M) and Legacy Medicare Choice (prior to January 2006, 0.36M) with both medical and prescription drug coverage. As of 30November2014, Optum contained 38m patients with patient-level observations from Oct2005 through Dec2013. Source codes used in Optum include: conditions- ICD-9-CM; drugs: NDC, HCPCS, ICD-9-CM; procedures: CPT-4, HCPCS, ICD-9-CM; lab: LOINC.

The ETL specification for transforming Optum into the OMOP CDM is available at: <http://omop.org/cdm>.

ACHILLES has been used to characterize the database and provide a data quality assessment. The ACHILLES summary is available internally within Janssen at: <http://hix.jnj.com/achilles/#/optum/dashboard>.

Clinical Practice Research Datalink (CPRD)

CPRD is an anonymized longitudinal electronic health records from primary care practices in UK. Patient management system with many aspects of patient care covered, including diagnoses, prescriptions, signs and symptoms, procedures, labs, lifestyle factors, clinical and administrative/social data. As of 30November2014, CPRD contained 11m patients with patient-level observations from Jan1988 through Nov2013. Source codes used in CPRD include: conditions- Read; drugs: Multilex; procedures: OPCS.

The ETL specification for transforming CPRD into the OMOP CDM is available at: <http://omop.org/cdm>.

ACHILLES has been used to characterize the database and provide a data quality assessment. The ACHILLES summary is available internally within Janssen at: <http://hix.jnj.com/achilles/#/cprd/dashboard>.

_Database X_

_Database X description_

The ETL specification for transforming _Database X_ into the OMOP CDM is available at: _ETL_specification_URL_

ACHILLES has been used to characterize the database and provide a data quality assessment. The ACHILLES summary is available at: _URL to ACHILLES_.

# Population

The general approach to defining the population of interest of patients with a chronic disease who have persistent exposure for at least 3 years following treatment initiation is illustrated in Figure 1:

Figure : Treatment pathway population criteria

## Hypertension

The analysis will be performed in a cohort of patients who satisfy the following criteria:

Index rule defining the index date:

- First exposure to an antihypertensive treatment with at least 365 days of observation prior to exposure and at least 1095 days of observation after the exposure

Inclusion rules based on the index date:

- At least one condition occurrence of hypertension between all days before index date and all days after index date
- At most zero condition occurrences of pregnancy between all days before index date and all days after index date
- At most zero drug exposures of antihypertensive treatment between all days before index date and 1 day before index date
- At last one drug exposure of antihypertensive treatment between 121 days after index date and 240 days after index date
- At last one drug exposure of antihypertensive treatment between 241 days after index date and 360 days after index date
- At last one drug exposure of antihypertensive treatment between 361 days after index date and 480 days after index date
- At last one drug exposure of antihypertensive treatment between 481 days after index date and 600 days after index date
- At last one drug exposure of antihypertensive treatment between 601 days after index date and 720 days after index date
- At last one drug exposure of antihypertensive treatment between 721 days after index date and 840 days after index date
- At last one drug exposure of antihypertensive treatment between 841 days after index date and 960 days after index date
- At last one drug exposure of antihypertensive treatment between 961 days after index date and 1080 days after index date

## Type 2 diabetes mellitus

The analysis will be performed in a cohort of patients who satisfy the following criteria:

Index rule defining the index date:

- First exposure to an antihyperglycemia treatment with at least 365 days of observation prior to exposure and at least 1095 days of observation after the exposure

Inclusion rules based on the index date:

- At least one condition occurrence of Type 2 diabetes mellitus between all days before index date and all days after index date
- At most zero condition occurrences of pregnancy between all days before index date and all days after index date
- At most zero condition occurrences of Type 1 diabetes mellitus between all days before index date and all days after index date
- At most zero drug exposures of antihyperglycemia treatment between all days before index date and 1 day before index date
- At last one drug exposure of antihyperglycemia treatment between 121 days after index date and 240 days after index date
- At last one drug exposure of antihyperglycemia treatment between 241 days after index date and 360 days after index date
- At last one drug exposure of antihyperglycemia treatment between 361 days after index date and 480 days after index date
- At last one drug exposure of antihyperglycemia treatment between 481 days after index date and 600 days after index date
- At last one drug exposure of antihyperglycemia treatment between 601 days after index date and 720 days after index date
- At last one drug exposure of antihyperglycemia treatment between 721 days after index date and 840 days after index date
- At last one drug exposure of antihyperglycemia treatment between 841 days after index date and 960 days after index date
- At last one drug exposure of antihyperglycemia treatment between 961 days after index date and 1080 days after index date

## Depression

The analysis will be performed in a cohort of patients who satisfy the following criteria:

Index rule defining the index date:

- First exposure to an antidepressant treatment with at least 365 days of observation prior to exposure and at least 1095 days of observation after the exposure

Inclusion rules based on the index date:

- At least one condition occurrence of Type 2 diabetes mellitus between all days before index date and all days after index date
- At most zero condition occurrences of pregnancy between all days before index date and all days after index date
- At most zero condition occurrences of bipolar disorder or schizophrenia between all days before index date and all days after index date
- At most zero drug exposures of antidepressant treatment between all days before index date and 1 day before index date
- At last one drug exposure of antidepressant treatment between 121 days after index date and 240 days after index date
- At last one drug exposure of antidepressant treatment between 241 days after index date and 360 days after index date
- At last one drug exposure of antidepressant treatment between 361 days after index date and 480 days after index date
- At last one drug exposure of antidepressant treatment between 481 days after index date and 600 days after index date
- At last one drug exposure of antidepressant treatment between 601 days after index date and 720 days after index date
- At last one drug exposure of antidepressant treatment between 721 days after index date and 840 days after index date
- At last one drug exposure of antidepressant treatment between 841 days after index date and 960 days after index date
- At last one drug exposure of antidepressant treatment between 961 days after index date and 1080 days after index date

Concepts and associated descendant source codes for each concept used to define the population are provided in Appendix 1: Population concepts.


# Methods

For each disease, we will create a cohort meeting the population inclusion criteria, as detailed above. For each patient in the qualifying cohorts, we will identify the sequence of treatments exposed during the 3-year prior after first exposure. Treatments will be summarized as the RxNorm ingredient level, and the DRUG_ERA table will be used to identify exposures. The sequence will be determined by ordering the dates of first exposure to each qualifying ingredient for the disease. All patients will have at least one treatment (since to qualify for the cohort, they had at least one first exposure). If a person was maintained on the same treatment for the entire 3-year period, meaning all drug eras during the 3-year window were for the same ingredient and no other ingredient for the same disease was observed during that same interval, then the person would be classified as having a 1-drug treatment sequence. A person who switched between treatments only once during their interval, would result in a 2-drug treatment sequence. We summarize all sequence combinations that are less than 20 drugs. Note, this definition of treatment pathway does not characterize patients who may switch back to a treatment used previously (that is, an ingredient will only be listed once within a sequence, since we only use the first exposure to each ingredient). Also, this treatment pathway will not distinguish between switching vs. augmentation behavior, as the sequence of ingredients does not summarize whether the ingredients are being used concomitantly.

Once each patient's treatment sequence is constructed, we will count the number of unique persons with the same treatment sequence. The counts will be produced within each source for each disease. We will also stratify the counts by the index year of the first exposure.

Only summary statistics, no patient-level data, will be provided by distributed data partners for this analysis. The analysis code provides each data partner the option to suppress any summary statistics below a minimum cell count number (the default is to remove all counts < 5).

The table shells for the four results tables of summary statistics are provided below with illustrative results generated during the initial feasibility assessment.

These summary results will then be used to create tabular and graphical summaries of the evidence across the OHDSI data network to characterize the prevalence of treatment pathways by source and year. As a descriptive summary analysis with no pre-specified hypotheses, no statistical tests will be performed. All results will be placed in the public domain and a summary report will be generated and submitted for publication upon completion.

Table : Number of persons, by source and disease

| **SOURCE** | **Disease** | **num_persons** |
| ---------- | ----------- | --------------- |
| CCAE       | Depression  | 114437          |
| MDCD       | Depression  | 7177            |
| MDCR       | Depression  | 22980           |
| OPTUM      | Depression  | 38524           |
| CCAE       | HTN         | 466730          |
| MDCD       | HTN         | 16131           |
| MDCR       | HTN         | 128617          |
| OPTUM      | HTN         | 122283          |
| CCAE       | T2DM        | 118782          |
| MDCD       | T2DM        | 4641            |
| MDCR       | T2DM        | 43903           |
| OPTUM      | T2DM        | 28547           |

Table : Number of persons, by source and disease by year

| **SOURCE** | **Disease** | **index_year** | **num_persons** |
| ---------- | ----------- | -------------- | --------------- |
| CCAE       | Depression  | 2000           | 2               |
| CCAE       | Depression  | 2001           | 4360            |
| CCAE       | Depression  | 2002           | 7053            |
| CCAE       | Depression  | 2003           | 9122            |
| CCAE       | Depression  | 2004           | 9007            |
| CCAE       | Depression  | 2005           | 11267           |
| CCAE       | Depression  | 2006           | 12314           |
| CCAE       | Depression  | 2007           | 12996           |
| CCAE       | Depression  | 2008           | 11782           |
| CCAE       | Depression  | 2009           | 14095           |
| CCAE       | Depression  | 2010           | 13640           |
| CCAE       | Depression  | 2011           | 8799            |

Table : Treatment pathway summary, by source and disease

| **SOURCE** | **Disease** | **d1_concept_id** | **d2_concept_id** | **d1_concept_name** | **d2_concept_name** | **num_persons** |
| ---------- | ----------- | ----------------- | ----------------- | ------------------- | ------------------- | --------------- |
| CCAE       | Depression  | 739138            | NULL              | Sertraline          | NULL                | 8302            |
| CCAE       | Depression  | 715939            | NULL              | Escitalopram        | NULL                | 6400            |
| CCAE       | Depression  | 797617            | NULL              | Citalopram          | NULL                | 5693            |
| CCAE       | Depression  | 755695            | NULL              | Fluoxetine          | NULL                | 4991            |
| CCAE       | Depression  | 750982            | NULL              | Bupropion           | NULL                | 3993            |
| CCAE       | Depression  | 743670            | NULL              | venlafaxine         | NULL                | 3287            |
| CCAE       | Depression  | 722031            | NULL              | Paroxetine          | NULL                | 2818            |
| CCAE       | Depression  | 715939            | 797617            | Escitalopram        | Citalopram          | 1976            |
| CCAE       | Depression  | 715939            | 750982            | Escitalopram        | Bupropion           | 1478            |
| CCAE       | Depression  | 715259            | NULL              | duloxetine          | NULL                | 1477            |
| CCAE       | Depression  | 739138            | 750982            | Sertraline          | Bupropion           | 1377            |
| CCAE       | Depression  | 797617            | 750982            | Citalopram          | Bupropion           | 1099            |

Table : Treatment pathway summary, by source and disease, by year

| **SOURCE** | **Disease** | **index_year** | **d1_concept_id** | **d2_concept_id** | **d1_concept_name** | **d2_concept_name** | **num_persons** |
| ---------- | ----------- | -------------- | ----------------- | ----------------- | ------------------- | ------------------- | --------------- |
| CCAE       | Depression  | 2001           | 739138            | NULL              | Sertraline          | NULL                | 232             |
| CCAE       | Depression  | 2001           | 722031            | NULL              | Paroxetine          | NULL                | 160             |
| CCAE       | Depression  | 2001           | 797617            | NULL              | Citalopram          | NULL                | 152             |
| CCAE       | Depression  | 2001           | 755695            | NULL              | Fluoxetine          | NULL                | 138             |
| CCAE       | Depression  | 2001           | 743670            | NULL              | venlafaxine         | NULL                | 99              |
| CCAE       | Depression  | 2001           | 750982            | NULL              | Bupropion           | NULL                | 77              |
| CCAE       | Depression  | 2001           | 739138            | 750982            | Sertraline          | Bupropion           | 52              |
| CCAE       | Depression  | 2001           | 722031            | 750982            | Paroxetine          | Bupropion           | 47              |
| CCAE       | Depression  | 2001           | 755695            | 750982            | Fluoxetine          | Bupropion           | 47              |
| CCAE       | Depression  | 2001           | 797617            | 715939            | Citalopram          | Escitalopram        | 47              |
