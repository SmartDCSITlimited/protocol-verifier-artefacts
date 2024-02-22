# Zip-file instructions

The release linked to this repo should contain two zip files. One contains the docker images that the protocol verifier needs, the other contains a built version of the protocol verifier. 

In order for the build to work properly, several dependencies must be met. The system should have the following;
- Docker and Docker CLI
- Containerd

To run the built-in tests and use kafka there are further dependencies;
- Git
- Python 3.11
- kafka-python3 (obtainable via pip)
- Java 17

To use this file, first extract it to your directory of choice.
```tar -xvdfz munin_1.2.1.tgz``` 
The file structure should be as follows;

```
.
├── bin
├── ConanCache.tgz
├── deploy
├── doc
├── LICENSE
├── metrics
├── models
├── README.adoc
├── test_reporting
└── tests
```

Extract and load the docker images.

```
gunzip pv_docker_images.tar.gz
docker load pv_docker_images.tar
```

Once loaded, you can remove the tarfile.

Create a docker volume called ConanCache, then extract the Conan Cache tar file to the new volume's file. Superuser permissions are required to reach this volume - rsync works for this.

```
docker volume create ConanCache
sudo rsync -av /var/lib/docker/volumes/ConanCache/ /{PATH_TO_EXTRACT_DIR}/munin_1.2.1/ConanCache/
```

You can now remove the Conan cache directory created from ConanCache.tar as well as ConanCache.tgz itself.

# Testing

To test the basic functionality of the protocol verifier, run the regression test featured in the tests directory. Protocol verifier is operated using scripts which are directory-sensitive, so ensure that scripts are run from their directory. The test scripts will generate and clean up test files which it may lose permissions to complete - grant access to the models and deploy directories to avoid this. 

```
sudo chmod 777 -R munin_1.2.1/models
sudo chmod 777 -R munin_1.2.1/deploy
cd munin_1.2.1/tests
./regression.sh
```

This test only runs against a single copy of the verifier and therefore is not performant, nor does it use kafka or test the capabilities of the full stack. In order to test the full functionality of the protocol verifier, the benchmark test in the metrics folder should be used. The test itself may return no useful data but it completing means the full stack was able to start, operate under load and stop.

```
sudo chmod 777 -R munin_1.2.1/models
sudo chmod 777 -R munin_1.2.1/deploy
cd munin_1.2.1/metrics
./run_benchmark.sh
```

# Functionality

The protocol verifier functions via mounting itself to the directory its built in and handling files that are placed in certain folders. Inputs and outputs are found in the deploy directory, which should be as follows;

```
.
├── config
│   ├── benchmarking-config.json
│   ├── configure-kafka.sh
│   ├── configure-kafka.sh.linux
│   ├── job_definitions
│   ├── log-config-pv-proc.xml
│   └── pv-config.json
├── docker-compose.kafka.yml
├── docker-compose.kafka.yml.linux
├── docker-compose.yml
├── InvariantStore
│   └── InvariantStore.db
├── JobIdStore
│   ├── 0-63
│   ├── 128-191
│   ├── 192-255
│   └── 64-127
├── logs
│   ├── reception
│   └── verifier
├── p2jInvariantStore
└── reception-processed
```

The protocol verifier requires a config file to be loaded into the config directory in .json format. This file is selected via export of an environment variable prior to launch (handled automatically in the testing scripts) and is where the application is told where its various component directories are and various operative rates. Notable important directories here include:
- IncomingDirectory, which the protocol verifier takes input job files from
- ProcessedDirectory, where the protocol verifier places files after verifying them
- JobDefinitionDirectory, where the templates for acceptable job formats are stored. Protocol verifier compares input jobs to the definitions in this folder to determine whether they are valid.

The kafka variant of protocol verifier uses kafka for file handling instead.

# Operation

In order to run the protocol verifier, first populate the job definitions directory (by default {EXTRACT_DIR}/deploy/config/job_definitions) with job definition files to test against. Add a JSON config file to the deploy/config directory and export its name as an environment variable. This zipfile comes with the benchmark testing job definitions and config still loaded, which work as a default.

```export CONFIG_FILE=benchmarking-config.json```

Next, spin up the containers via the docker compose files found in the deploy directory.

```docker compose -f docker-compose.yml up```

This will output the logs of the containers to the terminal, effectively preventing it from being used for anything else. Use the -d flag to avoid this if the terminal is still needed. If detached, check the logs using docker compose.

```docker compose -f docker-compose.yml logs```

Once running, place a job file to be tested into the incoming directory (by default deploy/incoming, which is created when the containers are run if it isn't present already). Protocol verifier will attempt to process the file and output it in the processed directory (by default deploy/processed). It will log the process in the logs directory, which splits into reception and verifier, and individual containers will output logs in docker.