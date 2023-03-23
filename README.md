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

## Summary of Initial Findings and Potential Solutions
Detailed workbooks where initial fingings were calculated:
* [Summary of Findings for Discussion at Stakeholder Meeting](https://github.com/KevinCarr42/Gaspereau/blob/meeting/6.%20Data%20Cleaning%20-%20Gaspereau%20Logbook%20Data%20(Samples).ipynb)
* [Data Cleaning and Table Creation Based on Follow-up From Meeting](https://github.com/KevinCarr42/Gaspereau/blob/master/1%20Cleaning%20and%20Creating%20Tables.ipynb)

### Sample Data
* Summary of recommendations:
<img src="https://user-images.githubusercontent.com/94803263/227007087-d05e8511-871c-42b9-83a2-6f5d74e6f0aa.png" width="600" />

### Fish Details
* FL_STD outliers appear to be scaled by 1/10x (likely inputted as cm, not mm):
<img src="https://user-images.githubusercontent.com/94803263/227188268-d959f440-fefc-4ec3-a0d8-d2259ea6a490.png" width="600" />
<div>
  <img src="https://user-images.githubusercontent.com/94803263/227187204-d3ad9c5b-ef96-4783-a2e2-3957d53982cb.png" width="300" />
  <img src="https://user-images.githubusercontent.com/94803263/227187223-5e996613-3062-4b4a-8ef2-950a3d6fcd60.png" width="300" />
</div>

* WEIGHT outlier significantly outside of the distribution:
<img src="https://user-images.githubusercontent.com/94803263/227192920-a4ca7c29-c575-4024-babf-97f66a0f657b.png" width="600" />

* GONAD_WEIGHT outlier significantly outside of the distribution:
<img src="https://user-images.githubusercontent.com/94803263/227193040-87cade21-8a2b-46e9-8ba7-ee7c49d84f58.png" width="600" />


* Summary of recommendations:
<img src="https://user-images.githubusercontent.com/94803263/227006823-65252c2b-8a2c-412e-a32b-8ec36c92f4a2.png" width="600" />

### Length Frequencies
* Summary of recommendations:
<img src="https://user-images.githubusercontent.com/94803263/227007125-014db7dd-1748-48b9-91eb-6be189d7c99e.png" width="600" />



# Appendix

### Import Script into dm_apps

```py

import csv
from datetime import datetime, time
import os
from alive_progress import alive_bar

# # UNCOMMENT THIS BLOCK TO RUN FROM PYCHARM SCRIPT CONFIGURATIONS
# import django
# os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'dm_apps.settings')  # set environment variable to run script
# django.setup()

from django.db import models
from django.conf import settings
from django.utils import timezone
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from herring import models

##########################
# FILE PATHS
##########################


rootdir = os.path.join(settings.BASE_DIR, 'herring', 'temp')
sample_file = 'gaspereau_sample_data.csv'  # sample data to import
invalid_sample_file = 'rejected_sample_data.csv'  # new file to write to
fishdetail_file = 'gaspereau_fish_details.csv'  # fishdetail data to import
invalid_fishdetail_file = 'rejected_fish_details.csv'  # new file to write to
length_frequency_file = 'gaspereau_LF_grouped.csv'  # length frequency data to import
invalid_length_frequency_file = 'rejected_length_frequencies.csv'  # new file to write to
site_file = 'gaspereau_sites.csv'  # site data to import
trap_supervisors_file = 'gaspereau_trap_supervisors.csv'  # new trap supervisors to import


##########################
# GENERAL HELPER FUNCTIONS
##########################


def clean(row):
    for key in row:
        # remove any trailing spaces
        row[key] = row[key].strip()
        # if there is a null value of any kind, replace with NoneObject
        if row[key] in ['', 'NA']:
            row[key] = None
    return row


def count_csv_rows(csv_full_path):
    with open(csv_full_path, 'r', encoding='utf-8-sig') as f:
        return sum(1 for row in csv.reader(f)) - 1


def validate_row(row, required_fields_list):
    # check if all required data is available
    for i in required_fields_list:
        if not row[i]:
            return False
    return True


def validate_kwargs(kwargs):
    if not kwargs:  # if something went wrong, kwargs should return None (and be invalid)
        return False
    return True


def get_site_from_site_number(site_number):
    """
    Usage:
    kwargs['site'] = get_site_from_site_number(row[site_column])
    number >90 are replacement numeric names used for matching old_id/sample_id
    """
    if not str(site_number).isnumeric():  # None or non-numeric
        return None
    elif float(site_number) == 90:
        site = '1A'
    elif float(site_number) == 91:
        site = '1B'
    elif float(site_number) == 92 or float(site_number) == 93:
        site = 'John Eric MacFarlane'
    elif float(site_number) == 94:
        site = 'JA Coady'
    elif 0 < float(site_number) < 90:
        site = site_number
    else:
        return None

    # next, use qs to match row['SITE1'] to a Site FK
    qs = models.Site.objects.filter(site_number__iexact=site)

    return qs.first()


def delete_empty_ghost_samples():
    """
    some ghost samples were created for fish details or length frequencies that were rejected
    this function will delete all ghost samples that are not linked to any fish details or length frequencies
    """
    qs = models.Sample.objects\
        .filter(is_ghost=True)\
        .filter(length_frequencies__isnull=True)\
        .filter(fish_details__isnull=True)

    number_to_delete = qs.count()
    qs.delete()
    print(number_to_delete, 'Ghost Samples deleted.')


##########################
# SAMPLES
##########################


def get_sample_kwargs_from_row(row):
    kwargs = dict(
        type=3,  # trap
        no_nets=row['no_nets'],
        old_id=row['id'],
        creation_date=timezone.now(),
        remarks=row['remarks']
    )

    # check for ambiguous samples, ambiguous matches, and ghost samples and flag for follow-up
    if row['FLAG_AMBIGUOUS_SAMPLE'] or row['FLAG_AMBIGUOUS_MATCH']:
        kwargs['is_ambiguous'] = True
    if row['FLAG_GHOST_SAMPLE']:
        kwargs['is_ghost'] = True

    # import estimate for total fish preserved
    if total_fish_preserved := row['total_fish_preserved']:
        kwargs['total_fish_preserved'] = int(float(total_fish_preserved))  # cast through float to avoid invalid literal

    # import estimate for total fish measured
    if total_fish_measured := row['total_fish_measured']:
        kwargs['total_fish_measured'] = int(float(total_fish_measured))  # cast through float to avoid invalid literal

    # strings to be converted to numeric if not None, else left blank
    if row['hours_fished']:
        if str(row['hours_fished']).strip().lower() == 'maximum':
            kwargs['hours_fished'] = 9999
        else:
            kwargs['hours_fished'] = float(row['hours_fished'])
    if row['catch_lbs']:
        kwargs['catch_weight_lbs'] = float(row['catch_lbs'])
    if row['wt_lbs']:
        kwargs['sample_weight_lbs'] = float(row['wt_lbs'])

    # sample date (and time) and period
    naive_date = datetime.strptime(row['DATETIME'], "%Y-%m-%d").date()
    if row['AM_PM_PERIOD'] == 'AM':  # 6am Atlantic Daylight time
        kwargs['sample_date'] = timezone.make_aware(datetime.combine(naive_date, time(9, 0)), timezone=timezone.utc)
        kwargs['period'] = 1
    elif row['AM_PM_PERIOD'] == 'PM':  # 6pm Atlantic Daylight time
        kwargs['sample_date'] = timezone.make_aware(datetime.combine(naive_date, time(21, 0)), timezone=timezone.utc)
        kwargs['period'] = 2
    elif row['AM_PM_PERIOD'] == 'AD':  # All Day: use 12 Noon Atlantic Daylight time
        kwargs['sample_date'] = timezone.make_aware(datetime.combine(naive_date, time(15, 0)), timezone=timezone.utc)
        kwargs['period'] = 3
    else:  # Unknown: 0 O'clock UTC (consistent with sample import form)
        kwargs['sample_date'] = timezone.make_aware(datetime.combine(naive_date, time(0, 0)), timezone=timezone.utc)

    # sites
    if site := get_site_from_site_number(row['SITE1']):
        kwargs['site'] = site  # shouldn't return None for Samples (except for ghost samples)
    else:
        kwargs['site'] = None  # otherwise if we want to reject this, it could return None

    # check trap_supervisor, and populate where possible
    if row['NAME']:  # leave null trap_supervisor null
        qs = models.TrapSupervisor.objects.filter(
            first_name__iexact=row['NAME'].rsplit(maxsplit=1)[0],
            last_name__iexact=row['NAME'].rsplit(maxsplit=1)[1]
        )
        if qs.exists():  # if there is a match, link it
            kwargs['trap_supervisor'] = qs.first()
        else:  # non null trap_supervisor, but not in trap_supervisor table
            kwargs['trap_supervisor'] = None  # if we want to reject this, it could return None

    return kwargs


def process_samples(directory, input_file, invalid_file):
    # full paths to files
    sample_path = os.path.join(directory, input_file)
    invalid_path = os.path.join(directory, invalid_file)

    # count rows for alive bar
    row_count = count_csv_rows(sample_path)

    # kwargs that are the same for all samples
    kwargs_all = {
        'catch': models.Catch.objects.filter(aphia_id__iexact=125715).first(),
        'gear': models.Gear.objects.filter(gear_code__iexact='81').first(),
        'district': models.District.objects.filter(district_id=2).first(),
        'created_by': User.objects.get(email='Kevin.Carr@dfo-mpo.gc.ca'),
        'last_modified_by': User.objects.get(email='Kevin.Carr@dfo-mpo.gc.ca'),
    }

    # multi-item with statements are treated as nested
    with open(invalid_path, 'w') as wf, \
            open(sample_path, 'r', encoding='utf-8-sig') as f, \
            alive_bar(row_count, force_tty=True) as bar:

        writer = csv.writer(wf, delimiter=',', lineterminator='\n')
        csv_reader = csv.DictReader(f)

        # header row for rejected/invalid entries
        writer.writerow(list(csv_reader.fieldnames) + ['DESCRIPTION OF PROBLEM'])

        for r in csv_reader:

            bar()  # progress bar

            r = clean(r)  # clean spaces and convert null to None

            # check if the row is valid
            required_fields = ['DATETIME']
            if not validate_row(r, required_fields):
                writer.writerow([r[key] for key in r] + ['missing or ambiguous DATETIME, record is not valid'])
                continue  # if the data isn't valid, go to the next row

            # get all of the data from the row into a dictionary to create a new object
            kwargs = dict(**get_sample_kwargs_from_row(r), **kwargs_all)

            if validate_kwargs(kwargs):  # check if the kwargs are valid
                models.Sample.objects.create(**kwargs)  # create a new object
            else:
                writer.writerow([r[key] for key in r] + ['kwargs rejected, record not valid'])
                continue  # if the data isn't valid, go to the next row


##########################
# FISH DETAILS
##########################


def get_fishdetail_kwargs_from_row(row):
    if row['FLAG_MISNUMBERED_FISH_DETAILS']:
        """There is no way to import these without either guessing or violating constraints"""
        return None

    kwargs = dict(
        fish_number=row['FISH_NO'],
        fish_length=row['fish_length'],
        fish_weight=row['WEIGHT'],
        gonad_weight=row['GONAD_WEIGHT'],
        scale_age_1=row['AGE_1'],
        scale_fsp_1=row['FSP_1'],
        scale_age_2=row['AGE_2'],
        scale_fsp_2=row['FSP_2'],
        remarks=row['remarks']
        # NOTE: sample matched in main import function in order to check for valid match (or error output)
    )

    # samplers
    samplers = {
        'JM': 2884,  # John Mallory
        'LF': 878,  # Larry Forsyth
        'Jmac': 2027  # Jenna MacEachern
    }
    kwargs['scale_reader_1_id'] = samplers.get(row['Ager_1'])
    kwargs['scale_reader_2_id'] = samplers.get(row['Ager_2'])

    # condition (fresh / frozen)
    if row['CONDITION'] == 'Fresh':
        kwargs['condition'] = 1
    elif row['CONDITION'] == 'Frozen':
        kwargs['condition'] = 2

    if maturity := row['MATURITY']:
        if str(maturity) in ['1', '2', '3', '4', '5', '6', '7', '8', '9', '99']:
            kwargs['maturity_id'] = maturity  # None if not one of these values

    # check species and populate where possible
    if row['SPECIES'] == 'A':  # Alewife
        kwargs['species'] = 158669
    elif row['SPECIES'] == 'B':  # Blueback
        kwargs['species'] = 158667

    # sex
    sexes = {'F': 2, 'M': 1, 'H': 3}
    if row['SEX'] in sexes:
        kwargs['sex_id'] = sexes[row['SEX']]
    else:
        kwargs['sex_id'] = 9  # unknown

    return kwargs


def process_fishdetail(directory, input_file, invalid_file):
    # full paths to files
    fishdetail_path = os.path.join(directory, input_file)
    invalid_path = os.path.join(directory, invalid_file)

    # count rows for alive bar
    row_count = count_csv_rows(fishdetail_path)

    # multi-item with statements are treated as nested
    with open(invalid_path, 'w') as wf, \
            open(fishdetail_path, 'r', encoding='utf-8-sig') as f, \
            alive_bar(row_count, force_tty=True) as bar:

        writer = csv.writer(wf, delimiter=',', lineterminator='\n')
        csv_reader = csv.DictReader(f)

        # header row for rejected/invalid entries
        writer.writerow(list(csv_reader.fieldnames) + ['DESCRIPTION OF PROBLEM'])

        for r in csv_reader:

            bar()  # progress bar

            r = clean(r)  # clean spaces and convert null to None

            # get all of the data from the row into a dictionary to create a new object
            kwargs = get_fishdetail_kwargs_from_row(r)

            # confirm valid kwargs
            if not validate_kwargs(kwargs):
                writer.writerow([r[key] for key in r] + ['kwargs rejected, record not valid'])
                continue  # if the data isn't valid, go to the next row

            # validate fish details: check for a matching sample
            qs = models.Sample.objects.filter(old_id__iexact=r['id'])
            if qs.exists():  # if there is a match, link it
                kwargs['sample'] = qs.first()
            else:  # otherwise create a ghost sample
                writer.writerow([r[key] for key in r] + ['Unmatched fish details.'])

            # create new object
            models.FishDetail.objects.create(**kwargs)  # create a new object


##########################
# LENGTH FREQUENCY
##########################


def process_length_frequency(directory, input_file, invalid_file):
    # full paths to files
    length_freq_path = os.path.join(directory, input_file)
    invalid_path = os.path.join(directory, invalid_file)

    # count rows for alive bar
    row_count = count_csv_rows(length_freq_path)

    with open(invalid_path, 'w') as wf, \
            open(length_freq_path, 'r', encoding='utf-8-sig') as f, \
            alive_bar(row_count, force_tty=True) as bar:

        writer = csv.writer(wf, delimiter=',', lineterminator='\n')
        csv_reader = csv.DictReader(f)

        # header row for rejected/invalid entries
        writer.writerow(list(csv_reader.fieldnames) + ['DESCRIPTION OF PROBLEM'])

        for r in csv_reader:

            bar()  # progress bar

            r = clean(r)  # clean spaces and convert null to None

            # counts and lengths
            kwargs = {
                'count': r['count'],
                'length_bin_id': r['length_bin_id']
            }

            # check for a matching sample
            qs = models.Sample.objects.filter(old_id__iexact=r['sample_id'])
            if qs.exists():  # if there is a match, link it
                kwargs['sample'] = qs.first()
            else:  # otherwise create ghost sample
                writer.writerow([r[key] for key in r] + ['Unmatched length frequencies.'])

            models.LengthFrequency.objects.create(**kwargs)


##########################
# DEPENDENCIES
##########################


# SITE IMPORT
def create_sites(directory, input_file):
    # full paths to files
    sites_path = os.path.join(directory, input_file)

    # count rows for alive bar
    row_count = count_csv_rows(sites_path)

    with open(sites_path, 'r', encoding='utf-8-sig') as f, alive_bar(row_count, force_tty=True) as bar:

        csv_reader = csv.DictReader(f)

        for r in csv_reader:

            bar()

            kwargs = {
                'site_number': r['site'],
                'name': r['name']
            }

            # leave lat and long blank if either is non-numeric
            if str(r['latitude_n']).isnumeric() and str(r['longitude_w']).isnumeric():
                kwargs['latitude_n']: r['latitude_n']
                kwargs['longitude_w']: r['longitude_w']

            if r['license_number']:
                kwargs['license_number']: r['license_number']

            if r['zone'] == 'upper':
                kwargs['location'] = 1
            elif r['zone'] == 'lower':
                kwargs['location'] = 2

            if r['flag_for_followup']:
                kwargs['flag_for_followup'] = True

            models.Site.objects.create(**kwargs)


# ADD TRAP SUPERVISORS
def add_trap_supervisors(directory, input_file):
    # full paths to files
    trap_supervisors_path = os.path.join(directory, input_file)

    # count rows for alive bar
    row_count = count_csv_rows(trap_supervisors_path)

    with open(trap_supervisors_path, 'r', encoding='utf-8-sig') as f, alive_bar(row_count, force_tty=True) as bar:
        csv_reader = csv.DictReader(f)

        for r in csv_reader:
            bar()

            kwargs = {
                'first_name': r['first_name'],
                'last_name': r['last_name'],
                'notes': r['notes']
            }
            models.TrapSupervisor.objects.create(**kwargs)


################################
# BACKUP PLAN - DELETE SAMPLES
################################

def delete_gaspereau():
    """
    This is our backup plan. If import goes wrong, delete all samples created by Kevin. Cascades.
    This could also be done for TrapSupervisors and Sites, but those are minimal.
    """
    kevin = User.objects.get(email='Kevin.Carr@dfo-mpo.gc.ca')
    qs = models.Sample.objects.filter(created_by=kevin)
    number_to_delete = qs.count()
    qs.delete()
    print('\nGASPEREAU SAMPLES DELETED: ', number_to_delete, '\n')


################################
# PROCESS AND IMPORT EVERYTHING
################################


def process_files(do_dependencies=False,
                  do_samples=False,
                  do_fishdetail=False,
                  do_length_freq=False,
                  do_delete_empty_ghost_samples=False):

    if do_dependencies:
        print('\n\nPROCESSING DEPENDENCIES')
        print('PROCESSING SITES')
        create_sites(rootdir, site_file)
        print('PROCESSING TRAP SUPERVISORS')
        add_trap_supervisors(rootdir, trap_supervisors_file)
        print('STEP COMPLETE')

    if do_samples:
        print('\n\nPROCESSING SAMPLE DATA')
        process_samples(rootdir, sample_file, invalid_sample_file)
        print('STEP COMPLETE')

    if do_fishdetail:
        print('\n\nPROCESSING FISH DETAIL DATA')
        process_fishdetail(rootdir, fishdetail_file, invalid_fishdetail_file)
        print('STEP COMPLETE')

    if do_length_freq:
        print('\n\nPROCESSING LENGTH FREQUENCY DATA')
        process_length_frequency(rootdir, length_frequency_file, invalid_length_frequency_file)
        print('STEP COMPLETE')

    if do_delete_empty_ghost_samples:
        print('\n\nDELETING EMPTY GHOST SAMPLES')
        delete_empty_ghost_samples()
        print('STEP COMPLETE')

    print('\n\nIMPORT COMPLETE')


if __name__ == '__main__':
    process_files(True, True, True, True, True)

```

### Merge Samples Script

```py

from django.db import transaction
from herring import models


@transaction.atomic
def merge_samples(primary_pk, ghost_pk, keep_old=False):
    # get the two samples
    primary_sample = models.Sample.objects.get(pk=primary_pk)
    ghost_sample = models.Sample.objects.get(pk=ghost_pk)

    # update remarks to include a note about being merged and the concatenation of both remarks
    remarks_primary, old_id_primary = getattr(primary_sample, 'remarks'), getattr(primary_sample, 'old_id')
    remarks_ghost, old_id_ghost = getattr(ghost_sample, 'remarks'), getattr(ghost_sample, 'old_id')
    merged_primary, merged_ghost = False, False
    if remarks_primary:
        if remarks_primary[:13] == 'MERGED SAMPLE':
            old_id_primary, end_notes = remarks_primary[14:].split('.', 1)
            remarks_primary = end_notes.strip()
            merged_primary = True
    if remarks_ghost:
        if remarks_ghost[:13] == 'MERGED SAMPLE':
            old_id_ghost, end_notes = remarks_ghost[14:].split('.', 1)
            remarks_ghost = end_notes.strip()
            merged_ghost = True
    remarks_combined = f'MERGED SAMPLE: {old_id_primary.strip()} and {old_id_ghost.strip()}.'
    remarks_combined += f'\n\n{remarks_primary}' if merged_primary \
        else f'\n\n{old_id_primary} REMARKS: {remarks_primary}' if remarks_primary \
        else f'\n\n{old_id_primary} REMARKS: none'
    remarks_combined += ';' if remarks_combined[-1] != '.' else None
    remarks_combined += f'\n\n{remarks_ghost}' if merged_ghost \
        else f'\n\n{old_id_ghost} REMARKS: {remarks_ghost}' if remarks_ghost \
        else f'\n\n{old_id_ghost} REMARKS: none'
    setattr(primary_sample, 'remarks', remarks_combined)

    # transfer all fish_details and length_frequencies
    for field in ['fish_details', 'length_frequency_objects']:
        for obj in getattr(ghost_sample, field).all():
            setattr(obj, 'sample', primary_sample)
            obj.save()

    # Try to fill all missing values in primary object by values of duplicates
    blank_fields = set([field.attname for field in primary_sample._meta.local_fields
                        if getattr(primary_sample, field.attname) in [None, '']])
    for field_name in blank_fields:
        val = getattr(ghost_sample, field_name)
        if val not in [None, '']:
            setattr(primary_sample, field_name, val)

    # set is_ambiguous to false
    """the sample may remain a ghost sample if two ghosts were merged, but it is no longer ambiguous"""
    setattr(primary_sample, 'is_ambiguous', False)

    if not keep_old:
        ghost_sample.delete()
    primary_sample.save()

```

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
