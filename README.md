# Exploratory Data Analysis and Data Cleaning of Gaspereau Data

Exploratory data analysis was performed to explore 63 features over 3 datasets prior to integration into the larger database. Data were visualised in context to in order to understand the data, and verify the internal consistency of the datasets. The example visualisation below shows the total length counts over time for the Alosa species (gaspereau):

<img width="800" src="https://user-images.githubusercontent.com/94803263/219615457-2d84ff48-a958-44aa-ad59-c5406f1c485c.png">

In this analysis, 3 distinct but interelated dataframes are used:
* Sample Data: **df_SD**
  * This is logbook data from trapping locations. In cases when a sample was taken for lab detailing or length frequencies, there is a corresponding logbook entry from which this sample was taken. For simplicity, in the database, these logbook entries are referred to as samples (although in some cases, logbook entries may not have had a sample taken). From now on, these entries are referred to simply as 'Samples.' These Samples are linked to Fish Details and Length Frequencies.
* Fish Details: **df_FD**
  * This data is obtained from fish detailing labs. Includes detailed information, such as sex, age, species (alewife or blueback), length, and weight.
* Length Frequencies: **df_LF**
  * This data focusses only on length distributions of a sample of fish.

## [Summary of Initial Findings and Potential Solutions](https://github.com/KevinCarr42/Gaspereau/blob/master/1%20Cleaning%20and%20Creating%20Tables.ipynb)

### Sample Data
<img src="https://user-images.githubusercontent.com/94803263/227007087-d05e8511-871c-42b9-83a2-6f5d74e6f0aa.png" width="600" />

### Fish Details
<img src="https://user-images.githubusercontent.com/94803263/227006823-65252c2b-8a2c-412e-a32b-8ec36c92f4a2.png" width="600" />

### Length Frequencies
<img src="https://user-images.githubusercontent.com/94803263/227007125-014db7dd-1748-48b9-91eb-6be189d7c99e.png" width="600" />



# Appendix

### Useful SQL Queries for Checking Import
notes:  uncomment clauses as required to check results

```SQL
  ------ CHECK WHETHER IMPORTS WORKED CORRECTLY

  ---- check import counts vs calculations from the jupyter notebook
  SELECT COUNT(DISTINCT herring_sample.id) FROM herring_sample
  --JOIN herring_lengthfrequency ON herring_lengthfrequency.sample_id = herring_sample.id
  --JOIN herring_fishdetail ON herring_fishdetail.sample_id = herring_sample.id
  --WHERE is_ghost = TRUE
  --WHERE is_ambiguous = TRUE AND is_ghost = FALSE
  WHERE herring_sample.catch_id = 3

  ---- unmatched samples count
  SELECT COUNT(DISTINCT id) FROM herring_sample 
  WHERE 
    catch_id = 3 
    AND id NOT IN (
      SELECT DISTINCT sample_id FROM herring_fishdetail 
    )
    AND id NOT IN (
      SELECT DISTINCT sample_id FROM herring_lengthfrequency 
    )

  ---- Ager IDs
  SELECT auth_user.username, au.username FROM herring_sample 
  JOIN herring_fishdetail ON herring_sample.id = herring_fishdetail.sample_id 
  JOIN auth_user ON herring_fishdetail.scale_reader_1_id = auth_user.id
  JOIN auth_user AS au ON herring_fishdetail.scale_reader_2_id = au.id
  GROUP BY auth_user.username, au.username

  SELECT COUNT(username), username FROM herring_sample 
  JOIN herring_fishdetail ON herring_sample.id = herring_fishdetail.sample_id 
  --JOIN auth_user ON herring_fishdetail.scale_reader_1_id = auth_user.id
  JOIN auth_user ON herring_fishdetail.scale_reader_2_id = auth_user.id
  WHERE username = 'jmac@dfo-mpo.gc.ca'
  --WHERE username != 'jmac@dfo-mpo.gc.ca'

  ---- Kevin user for all gaspereau imports
  SELECT COUNT(*) FROM herring_sample
  JOIN auth_user ON auth_user.id = herring_sample.created_by_id
  WHERE email = 'Kevin.Carr@dfo-mpo.gc.ca'

  ---- herring districts
  SELECT DISTINCT herring_district.district_id FROM herring_sample
  JOIN herring_district ON herring_sample.district_id = herring_district.id 
  WHERE catch_id = 2

  ---- all gaspereau are from district 2
  SELECT DISTINCT herring_district.district_id FROM herring_sample
  JOIN herring_district ON herring_sample.district_id = herring_district.id
  WHERE herring_sample.catch_id = 3

  ---- another variant
  SELECT COUNT(herring_district.district_id), herring_district.district_id FROM herring_sample
  JOIN herring_district ON herring_sample.district_id = herring_district.id
  WHERE herring_sample.catch_id = 3
  GROUP BY herring_district.district_id

  ---- check bool flags
  SELECT DISTINCT is_ambiguous FROM herring_sample
  WHERE catch_id = 2

  SELECT DISTINCT is_ambiguous FROM herring_sample
  WHERE catch_id = 3

  SELECT DISTINCT is_ghost FROM herring_sample
  WHERE catch_id = 2

  SELECT DISTINCT is_ghost FROM herring_sample
  WHERE catch_id = 3

  ---- what about species is None? 
  SELECT DISTINCT herring_fishdetail.species FROM herring_sample
  JOIN herring_fishdetail ON herring_sample.id = herring_fishdetail.sample_id
  WHERE herring_sample.catch_id = 2

  ---- every sample that is missing a site is tagged as a ghost sample for followup
  SELECT DISTINCT id, is_ghost FROM herring_sample
  WHERE herring_sample.site_id ISNULL AND herring_sample.catch_id = 3

  ---- check ambiguous samples 
  SELECT id, remarks FROM herring_sample 
  WHERE id IN (SELECT DISTINCT sample_id FROM herring_fishdetail) 
  AND is_ambiguous = TRUE
  --AND remarks NOT LIKE '%AMBIGUI%'

  SELECT id, remarks FROM herring_sample 
  WHERE id IN (SELECT DISTINCT sample_id FROM herring_lengthfrequency)
  AND is_ambiguous = TRUE
  --AND remarks NOT LIKE '%AMBIGUI%'

  SELECT id, remarks FROM herring_sample 
  WHERE is_ambiguous = TRUE AND is_ghost = FALSE
    AND id IN (SELECT DISTINCT sample_id FROM herring_fishdetail) 
    AND id IN (SELECT DISTINCT sample_id FROM herring_lengthfrequency)

  SELECT id, remarks FROM herring_sample 
  WHERE is_ambiguous = TRUE AND is_ghost = TRUE
  --AND remarks NOT LIKE '%AMBIGUI%'

  SELECT id, remarks FROM herring_sample 
  WHERE is_ambiguous = TRUE AND is_ghost = FALSE
  --AND remarks NOT LIKE '%AMBIGUI%'

  ---- check notes for null fishdetail.species -- only ambiguous samples are noted
  SELECT sample_id, herring_sample.remarks, herring_fishdetail.remarks AS fishdetail_remarks, site_number, scale_age_1, scale_fsp_1  FROM herring_sample
  JOIN herring_fishdetail ON herring_sample.id = herring_fishdetail.sample_id 
  JOIN herring_site ON herring_sample.site_id = herring_site.id 
  WHERE catch_id = 3 AND species ISNULL
  GROUP BY herring_sample.id
    
  ---- check how many fish details are gaspereau -- 36397 (0 after delete_gaspereau())
  SELECT COUNT(*) FROM herring_fishdetail 
  WHERE SPECIES IN (158669, 158667)
    
  ---- check how many length frequencies are gaspereau -- 11127 (0 after delete_gaspereau())
  SELECT COUNT(*) FROM herring_lengthfrequency
  JOIN herring_sample ON herring_sample.id = herring_lengthfrequency.sample_id
  WHERE herring_sample.catch_id = 3

  ---- check results from progress report

  ---- fish counts - correct for 2019

  SELECT COUNT(*) FROM herring_fishdetail
  JOIN herring_sample ON herring_fishdetail.sample_id = herring_sample.id
  WHERE STRFTIME("%Y", sample_date) = '2019' 
    AND catch_id = 3

  SELECT COUNT(*) FROM herring_fishdetail
  JOIN herring_sample ON herring_fishdetail.sample_id = herring_sample.id
  WHERE lab_processing_complete 
    AND STRFTIME("%Y", sample_date) = '2019' 
    AND catch_id = 3

  SELECT COUNT(*) FROM herring_fishdetail
  JOIN herring_sample ON herring_fishdetail.sample_id = herring_sample.id
  WHERE scale_1_processed_date NOTNULL 
    AND STRFTIME("%Y", sample_date) = '2019' 
    AND catch_id = 3

  SELECT COUNT(*) FROM herring_fishdetail
  JOIN herring_sample ON herring_fishdetail.sample_id = herring_sample.id
  WHERE scale_2_processed_date NOTNULL 
    AND STRFTIME("%Y", sample_date) = '2019' 
    AND catch_id = 3

  ---- sample counts - correct for 2019

  SELECT COUNT(*) FROM herring_sample
  WHERE STRFTIME("%Y", sample_date) = '2019'
    AND catch_id = 3

  SELECT COUNT(*) FROM herring_sample
  WHERE STRFTIME("%Y", sample_date) = '2019' 
    AND lab_processing_complete
    AND catch_id = 3

  SELECT SUM(matches) FROM (
    SELECT MIN(scale_1_processed_date NOT NULL) AS matches FROM herring_sample
    JOIN herring_fishdetail ON herring_fishdetail.sample_id = herring_sample.id
    WHERE STRFTIME("%Y", sample_date) = '2019'
      AND catch_id = 3
    GROUP BY sample_id
  )

  SELECT SUM(matches) FROM (
    SELECT MIN(scale_2_processed_date NOT NULL) AS matches FROM herring_sample
    JOIN herring_fishdetail ON herring_fishdetail.sample_id = herring_sample.id
    WHERE STRFTIME("%Y", sample_date) = '2019'
      AND catch_id = 3
    GROUP BY sample_id
  )

```
