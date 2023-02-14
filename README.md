# Gaspereau Exploratory Data Analysis and Data Cleaning Prior to Import Into DM Apps

### Some Useful SQL Queries for Checking Import
* uncomment clauses as required to check results

```
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

```
