---
name: ohdsi-cohorts
description: Core directives and operationalization standards for clinical cohorts in OHDSI. Read whenever defining, extracting, or designing a clinical cohort (e.g. exposure or outcome), or working with TCO (Target, Comparator, Outcome) mapping, Circe, ATLAS, or OMOP cohort generation.
---

> **Author & Attribution**: Based on OHDSI cohort principles and [`schuemie/AgentPlayGround`](https://github.com/schuemie/AgentPlayGround) by **Dr. Martijn Schuemie**.

# Core Directives

- **NEVER** use the term "cohort" to mean a general population sample or static list of patient IDs.
- **ALWAYS** define a cohort strictly as a set of persons who satisfy one or more inclusion criteria for a duration of time.
- When instructed to operationalize a cohort, you must assume the target output relies on the OHDSI Circe JSON standard (compilable to SQL via CirceR / Capr) unless explicitly instructed otherwise.

# Term Definition

We define a cohort as a set of persons who satisfy one or more inclusion criteria for a duration of time. 

**Strict Logical Consequences:**

- One person **MAY** belong to multiple cohorts simultaneously.
- One person **MAY** belong to the same cohort for multiple different time periods (multiple distinct episodes).
- One person **MAY NOT** belong to the same cohort multiple times during the exact same period of time (episodes within a single cohort cannot overlap; era collapse merges adjacent or overlapping spans).
- A cohort **MAY** have zero or more members.

# Cohort Operationalization (Circe Standard)

OHDSI Circe (implemented in ATLAS, WebAPI, and Capr R package) offers the standard specification to operationalize a cohort definition. These definitions are expressed as JSON and converted to SQL to instantiate the cohort table (`cohort_definition_id`, `subject_id`, `cohort_start_date`, `cohort_end_date`) within an OMOP Common Data Model database.

### Key Elements of a Circe Cohort Definition:
1. **Cohort Entry Event (Primary Criteria)**: The initial clinical event (condition occurrence, drug exposure, procedure, observation, visit) that qualifies a person for entry into the cohort. This acts as the temporal anchor.
2. **Index Date**: The start date (`cohort_start_date`) of the qualifying cohort entry event. All temporal inclusion rules and baseline covariates are calculated relative to this date (e.g. `[-365, 0]` days).
3. **Inclusion Rules (Attrition Criteria)**: Additional clinical criteria that must be satisfied relative to the index date (e.g. requires at least 365 days of continuous observation prior to index in `observation_period`, no prior history of the outcome, age restrictions, or concurrent medications).
4. **Cohort Exit Strategy**: The rule establishing when the person no longer satisfies cohort membership, determining `cohort_end_date`:
   - *Fixed duration exit* (e.g. 30 days after index date).
   - *End of continuous observation* (end of `observation_period`).
   - *Drug exposure era exit* (end of persistent drug era with allowed persistence window/gap).
   - *Censoring events* (exit immediately upon occurrence of a conflicting clinical event, e.g. pregnancy or alternative diagnosis).
5. **Era Collapse**: Rules for merging adjacent or overlapping episodes into single continuous cohort eras.
