# ROR-MRIQC
Integration of MRIQC into the research information system Helse Vest

MRIQC computes quality related scores from a T1-weighted image series in Nifti format. In order to use this as a workflow step in the Research Information System we have to solve the following issues:

 - convert the DICOM files in input into a nifti file (dcm2niix)
 - create a BIDS directory structure and copy the .nii and .json files into the correct location
 - run mriqc
 - copy the output JSON file to the right location
 - convert the resulting scores to a REDCap notation adding event name

## Setup

We start with the docker container for MRIQC:

```bash
docker run -it nipreps/mriqc:latest --version
```

We install dcm2niix.

```bash
apt update; apt install dcm2niix
```

We store a general 'work.sh' script in the /app/ directory inside the container and specify that script as the entrypoint for the container. It should be sufficient to call the container with argument "/data" (mount point for the input directory). The input directory was created using the ror application.

```bash
#!/usr/bin/env bash

# We will be called like this:
# input_dicom_dir="/Users/hauke/src/mriqc/data/"
# output_dir="/Users/hauke/src/mriqc/output"
# docker run --rm -it --entrypoint /usr/bin/dcm2niix -v $input_dicom_dir:/data:ro -v $output_dir:/output mriqc:21.0.0rc2 -v 2 -o /output /data

input="${1}"
output="${2}"
tmp="/tmp/"

# if we have a single argument we need to look inside /data and find out input and output ourselves
if [ "$#" -eq 1 ]; then
  input="${1}/input"
  output="${1}/output"
  if [ ! -d "${output}" ]; then
    mkdir -p "${output}"
  fi
fi
echo "run with input: ${input} and output: ${output}"

# create the nii and json files
/usr/bin/dcm2niix -v 2 -o "${output}" "${input}"

# convert to BIDS directory structure
patientID=`find ${input} -type f | xargs -I'{}' dcmdump +P PatientID {} | sort | uniq | cut -d'[' -f2 | cut -d']' -f1 | sed '/^[[:space:]]*$/d' | head -1`
event=`find ${input} -type f | xargs -I'{}' dcmdump +P ReferringPhysicianName {} | sort | uniq | cut -d'[' -f2 | cut -d']' -f1 | sed '/^[[:space:]]*$/d' | head -1 | sed 's/EventName://g'`
mkdir "${output}/sub-${patientID}"
mkdir -p "${output}/sub-${patientID}/ses-1/anat/"
for u in `ls ${output}/*.json`; do
   nii="${u%%.json}.nii"
   mv "$u" "${output}/sub-${patientID}/ses-1/anat/sub-${patientID}_ses-1_T1w.json"
   mv "$nii" "${output}/sub-${patientID}/ses-1/anat/sub-${patientID}_ses-1_T1w.nii"
done
echo "{\"Name\": \"Single patient analysis for ${patientID} event ${event}\", \"BIDSVersion\": \"1.0.2\"}" > "${output}/dataset_description.json"

# and call mriqc
report_dir="${output}/report"
mkdir -p "${report_dir}"
echo "/opt/conda/bin/mriqc --verbose --no-sub -w /tmp/ ${output} \"${report_dir}\" participant --participant_label \"sub-${patientID}\""
/opt/conda/bin/mriqc --verbose --no-sub -w /tmp/ ${output} "${report_dir}" participant --participant_label "sub-${patientID}"

cat "${report_dir}/sub-${patientID}/ses-1/anat/sub-${patientID}_ses-1_T1w.json"
cp "${report_dir}/sub-${patientID}/ses-1/anat/sub-${patientID}_ses-1_T1w.json" "${output}/output.json"

# some of the values we want to import (call a conversion script)
/app/convertToREDCap.py "${patientID}" "${event}" "${output}/output.json" "${output}/output.json"
```

The referenced "convertToREDCap.py" script creates in a simple loop individual variables tupel to link to a study in REDCap.

```python
#!/usr/bin/env python3

import sys
import json

patientID=sys.argv[1]
event=sys.argv[2]
input=sys.argv[3]
output=sys.argv[4]

print("PatientID: %s, event: %s input file: %s" % (patientID, event, input))

data = {}
with open(input, 'r') as f:
    config = json.load(f)
for key, value in config.items():
    print("key: %s, value: %s" % (key, value))
    data[key] = { 'record_id': patientID, 'event_name': "%s_arm_1" % (event), 'field_name': key, 'value': value }

print(json.dumps(data))
with open(output, 'w') as f:
    json.dump(data, f)
```
