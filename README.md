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

