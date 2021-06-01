# General Description
This repository contains two Python scripts used for triaging compromised systems with Assemblyline.
1. The "Pusher" (`al_incident_submitter.py`): pushes files from the compromised system to an Assemblyline
instance for analysis.
2. The "Puller" (`al_incident_analyzer.py`): pulls the submissions from the
Assemblyline instance and reports on if the submissions are safe/unsafe.
3. The "Downloader" (`al_incident_downloader.py`): downloads files submitted to Assemblyline that are under a certain 
score threshold, matching the folder structure of the files as they were submitted.
   

# How do I use this thing?
## General Process
The "Pusher" needs to run from the compromised machine, which needs network access to the Assemblyline instance
which it will be sending files to.

The "Puller" needs to run from a machine that has network access to the Assemblyline instance
which you sent files to via the "Pusher". It is considered good practice to run the "Puller" from a machine that 
isn't compromised.

The "Downloader" needs to run on a machine where you want all clean files downloaded to. This machine needs network 
access to the Assemblyline instance that you sent files to via the "Pusher".

## Prequisites
For the machine(s) running the "Pusher", the "Puller", and the "Downloader":
- You will need at least Python 3
  - Download here: https://www.python.org/downloads/
- If on Linux, you will need to install the following packages via APT/YUM: `libffi-dev`, `libssl-dev`
  - By command line: 
    - (APT) `sudo apt-get install libffi-dev libssl-dev`
    - (YUM) `sudo yum install libffi-dev libssl-dev`
  - By browser, download .deb files here: https://packages.debian.org/jessie/libffi-dev, https://packages.debian.org/jessie/libssl-dev
- You will need the `assemblyline-incident-manager` PIP module and its dependencies installed
  - `pip install assemblyline-incident-manager`
- For the offline installation of these packages and libraries, see the Offline Installation section

In general:
- You will need the URL of an Assemblyline instance that you have an account on. 
  - Want to create your own Assemblyline instance? [HOW-TO](https://cybercentrecanada.github.io/assemblyline4_docs/docs/installation.html)
- You will need two API keys generated by Assemblyline, ideally one with read access and another with write access. 
The Write-only key will be used for the "Pusher", and the Read-only key will be used for the "Puller" and the "Downloader".
  - It is considered best practice to not use an API key that has both Read-Write access on the compromised system, so 
  we *highly* recommend using two keys.
    
### Offline Installation
You will need to run the following code from a machine that has Internet access and then transfer it to the machine
that does not have Internet access.
#### Linux:
```
mkdir offline_packages
sudo su
apt-get install --download-only python3 python3-pip libffi-dev libssl-dev --reinstall -y
mv /var/cache/apt/archives/*.deb ~/offline_packages
python3 -m pip download pip assemblyline-incident-manager -d ~/offline_packages
exit
tar -czvf offline_packages.tar.gz ~/offline_packages/
Copy this file over using SCP, FTP or some other method
```

On the machine that is offline, do the following:
```
tar -xzvf offline_packages.tar.gz
cd offline_packages
sudo apt-get install ./*.deb -y
for x in `ls *.whl`;  do python3 -m pip install $x; done
```

#### Windows
- Download and install the most recent Python .msi installer from https://www.python.org/downloads/release. 
- Upgrade PIP: `python -m pip install --upgrade pip`
- Install the required PIP packages: `python3 -m pip download assemblyline-incident-manager`

## Run the thing!
### Pusher
On the compromised machine...

To get a sense of the options available to you:
```
python3 al_incident_submitter.py --help
Usage: al_incident_submitter.py [OPTIONS] COMMAND [ARGS]...

  Example: python al_incident_submitter.py --url="https://<domain-of-Assemblyline-
  instance>" --username="<user-name>"
  --apikey="/path/to/file/containing/apikey"
  --classification="<classification>" --service_selection="<service-
  name>,<service-name>" --path="/path/to/compromised/directory"
  --incident_num=123

Options:
  --url TEXT                The target URL that hosts Assemblyline.
                            [required]

  --username TEXT           Your Assemblyline account username.  [required]
  --apikey PATH             A path to a file that contains only your
                            Assemblyline account API key. NOTE that this API
                            key requires write access.  [required]

  --ttl INTEGER             The amount of time that you want your Assemblyline
                            submissions to live on the Assemblyline system (in
                            days).

  --classification TEXT     The classification level for each file submitted
                            to Assemblyline.  [required]

  --service_selection TEXT  A comma-separated list (no spaces!) of service
                            names to send files to. If not provided, all
                            services will be selected.

  -t, --is_test             A flag that indicates that you're running a test.
  --path PATH               The directory path containing files that you want
                            to submit to Assemblyline.  [required]

  -f, --fresh               We do not care about previous runs and resuming
                            those.

  --incident_num TEXT       The incident number for each file to be associated
                            with.  [required]

  --resubmit-dynamic        All files that score higher than 500 will be
                            resubmitted for dynamic analysis.

  --alert                   Generate alerts for this submission.
  --threads INTEGER         Number of threads that will ingest files to
                            Assemblyline.

  --dedup_hashes            Only submit files with unique hashes. If you want
                            100% file coverage in a given path, do not use
                            this flag

  --priority INTEGER        Provide a priority number which will cause the
                            ingestion to go to a specific priority queue.

  --do_not_verify_ssl       Verify SSL when creating and using the
                            Assemblyline Client.

  --help                    Show this message and exit.
```

After a successful run you should get some logs, followed by "All done!"

You can check that these files were ingested successfully by browsing to the Submissions page of the
Assemblyline instance that you're using.

### Puller
On the non-compromised machine...

To get a sense of the options available to you:
```
python al_incident_analyzer.py --help
Usage: al_incident_analyzer.py [OPTIONS] COMMAND [ARGS]...

  Example: python al_incident_analyzer.py --url="https://<domain-of-
  Assemblyline-instance>" --username="<user-name>"
  --apikey="/path/to/file/containing/apikey" --incident_num=123

Options:
  --url TEXT           The target URL that hosts Assemblyline.  [required]
  -u, --username TEXT  Your Assemblyline account username.  [required]
  --apikey PATH        A path to a file that contains only your Assemblyline
                       account API key. NOTE that this API key requires write
                       access.  [required]

  --min_score INTEGER  The minimum score for files that we want to query from
                       Assemblyline.

  --incident_num TEXT  The incident number for each file to be associated
                       with.  [required]

  -t, --is_test        A flag that indicates that you're running a test.
  --help               Show this message and exit.
```

After a successful run, you should get some logs, followed by "All done!"

Now check the `report.csv` file that was created by the "Puller". This file will contain what files 
are safe/unsafe.

Act accordingly with this wealth of knowledge at your disposal.

### Downloader
On the machine where you want the "safe" files downloaded to...

To get a sense of the options available to you:

```
python al_incident_downloader.py --help
Usage: al_incident_downloader.py [OPTIONS] COMMAND [ARGS]...

  Example: python al_incident_downloader.py --url="https://<domain-of-
  Assemblyline-instance>" --username="<user-name>"
  --apikey="/path/to/file/containing/apikey" --incident_num=123
  --min_score=100 --download_path=/path/to/where/you/want/downloads
  --upload_path=/path/from/where/files/were/uploaded/from

Options:
  --url TEXT                    The target URL that hosts Assemblyline.
                                [required]

  -u, --username TEXT           Your Assemblyline account username.
                                [required]

  --apikey PATH                 A path to a file that contains only your
                                Assemblyline account API key. NOTE that this
                                API key requires read access.  [required]

  --min_score INTEGER           The minimum score for files that we want to
                                query from Assemblyline.  [required]

  --incident_num TEXT           The incident number that each file is
                                associated with.  [required]

  --download_path PATH          The path to the folder that we will download
                                files to.  [required]

  --upload_path PATH            The base path from which the files were
                                ingested from on the compromised system.
                                [required]

  -t, --is_test                 A flag that indicates that you're running a
                                test.

  --num_of_downloaders INTEGER  The number of threads that will be created to
                                facilitate downloading the files.

  --do_not_verify_ssl           Verify SSL when creating and using the
                                Assemblyline Client.

  --help                        Show this message and exit.
```

If you check the download path you supplied, you should have all files downloaded there.
