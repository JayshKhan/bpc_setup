# BPC Setup

## Setup

**On Local PC (online) only:**

```bash
mkdir -p ~/ws_bpc/src
cd ~/ws_bpc/src
git clone [https://github.com/opencv/bpc.git](https://github.com/opencv/bpc.git)
```

## Build

**On Local PC (online) only:**

### Setting up the baseline solution

```bash
### Pull Baseline Solution Code ###
cd bpc/
wget [https://storage.googleapis.com/akasha-public/IPBC/baseline_solution/v1/models.zip](https://storage.googleapis.com/akasha-public/IPBC/baseline_solution/v1/models.zip)
unzip models.zip
rm models.zip
git clone [https://github.com/CIRP-Lab/bpc_baseline](https://github.com/CIRP-Lab/bpc_baseline)
```

### Build the ibpc_pose_estimator

```bash
cd ~/ws_bpc/src/bpc
docker buildx build -t ibpc:pose_estimator \
    --file./Dockerfile.estimator \
    --build-arg="MODEL_DIR=models" \
  .
```

### Build the ibpc_tester

```bash
cd ~/ws_bpc/src/bpc
docker buildx build -t ibpc:tester \
    --file./Dockerfile.tester \
  .
```

## Run

**On Local PC (online):**

### Start the Zenoh router

```bash
docker run --init --rm --net host eclipse/zenoh:1.2.1 --no-multicast-scouting
```

### Run the pose estimator

We use [rocker](https://github.com/osrf/rocker) to add GPU support to Docker containers. To install rocker, run `pip install rocker` on the host machine.

```bash
rocker --nvidia --cuda --network=host ibpc:pose_estimator
```

### Run the tester

> Note: Substitute the `<PATH_TO_DATASET>` with the directory that contains the [ipd](https://huggingface.co/datasets/bop-benchmark/ipd/tree/main) dataset. Similarly, substitute `<PATH_TO_OUTPUT_DIR>` with the directory that should contain the results from the pose estimator. By default, the results will be saved as a `submission.csv` file but this filename can be updated by setting the `OUTPUT_FILENAME` environment variable.

```bash
docker run --network=host -e BOP_PATH=/opt/ros/underlay/install/datasets -e SPLIT_TYPE=val -v<PATH_TO_DATASET>:/opt/ros/underlay/install/datasets -v<PATH_TO_OUTPUT_DIR>:/submission -it ibpc:tester
```

## Exporting Images for Offline Use (Workstation)

**On Local PC (online):**

1. **Save the Docker images to a tar archive:**

```bash
docker save ibpc:tester > ibpc_tester.tar
docker save ibpc:pose_estimator > ibpc_pose_estimator.tar
docker save eclipse/zenoh:1.2.1 > eclipse_zenoh.tar
```

2. **Transfer the tar archives to the Workstation (offline):**

Use any method to transfer these files (e.g., USB drive, network share, cloud storage).

**On Workstation (offline):**

3. **Load the images:**

```bash
docker load < ibpc_tester.tar
docker load < ibpc_pose_estimator.tar
docker load < eclipse_zenoh.tar
```

4. **Run the pose estimator (offline):**

```bash
rocker --nvidia --cuda --network=host ibpc:pose_estimator
```

5. **Run the tester (offline):**

```bash
docker run --network=host -e BOP_PATH=/opt/ros/underlay/install/datasets -e SPLIT_TYPE=val -v<PATH_TO_DATASET>:/opt/ros/underlay/install/datasets -v<PATH_TO_OUTPUT_DIR>:/submission -it ibpc:tester
```
