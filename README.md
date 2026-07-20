# CMS-L1-TrackFindingTracklet

This repository documents a workflow for running the **CMS Level-1 Tracklet emulation** using **CMSSW**.

Official repository:

https://github.com/cms-sw/cmssw/tree/master/L1Trigger/TrackFindingTracklet

This guide follows the setup and execution procedure described in the official repository, with additional notes for local development.

---

# 1. Initial Workspace Setup (First-time Only)

Run these steps only once when creating a new CMSSW workspace.

```bash
# Load CMSSW environment
source /cvmfs/cms.cern.ch/cmsset_default.sh

# Create a workspace
mkdir -p ~/L1TK_Workspace
cd ~/L1TK_Workspace

# Create the CMSSW release
cmsrel CMSSW_15_1_0_pre4

# Enter the release area
cd CMSSW_15_1_0_pre4/src
cmsenv
```

---

# 2. Repository Setup

Fetch the Tracklet development branch from the CMS L1TK repository.

```bash
# Initialize the CMSSW Git environment
git cms-init

# Checkout the development topic
git cms-checkout-topic -u cms-L1TK:fw_synch_250903

# Add the TrackFindingTracklet package
git cms-addpkg L1Trigger/TrackFindingTracklet
```

---

# 3. Daily Setup

Run these commands every time you open a new terminal.

A valid CMS proxy is required to access remote datasets via XRootD.

```bash
source /cvmfs/cms.cern.ch/cmsset_default.sh

cd ~/L1TK_Workspace/CMSSW_15_1_0_pre4/src
cmsenv

voms-proxy-init -voms cms -valid 192:0

cd L1Trigger/TrackFindingTracklet/test/
```

---

# 4. Configuration

## Geometry

Set the detector geometry in your configuration file (e.g. `L1TrackNtupleMaker_cfg.py`):

```python
GEOMETRY = "D110"
```

Use the corresponding Phase2Spring24 D110 input samples.

---

## Tracking Algorithm

The default algorithm is

```python
L1TRKALGO = 'HYBRID_NEWKF'
```

### Prompt tracking

```python
L1TRKALGO = 'HYBRID_NEWKF'
```

### Displaced tracking

```python
L1TRKALGO = 'HYBRID_DISPLACED'
```

You can switch automatically using

```bash
sed -i "s/L1TRKALGO = 'HYBRID_NEWKF'/L1TRKALGO = 'HYBRID_DISPLACED'/g" \
L1Trigger/TrackFindingTracklet/test/L1TrackNtupleMaker_cfg.py
```

or edit the configuration manually.

---

## Hardware Truncation Monitoring (Optional)

To study firmware data rates and truncation, edit

```
L1Trigger/TrackFindingTracklet/interface/Settings.h
```

and set

```cpp
writeMonitorData_ = true;
```

---

# 5. Compilation

Recompile after modifying any `.cc`, `.h`, or Python configuration files.

```bash
cd ~/L1TK_Workspace/CMSSW_15_1_0_pre4/src

scram b -j 8
```

---

# 6. Running the Emulation

Move to the test directory.

```bash
cd L1Trigger/TrackFindingTracklet/test/
```

Run the standard ntuple production.

```bash
cmsRun L1TrackNtupleMaker_cfg.py
```

This produces

```
L1TrkNtuple.root
```

---

## Alternative Configuration

Another useful configuration is

```bash
cmsRun HybridTracksNewKF_cfg.py
```

This script runs

- DTC
- Prompt Tracklet
- KF interface
- New KF emulator
- Analyzer for each processing stage

It is a lightweight version of `L1TrackNtupleMaker_cfg.py` and is useful for debugging because each reconstruction stage can be inspected independently.

---

# 7. Output ROOT File

The output filename is defined in

```
L1TrackNtupleMaker_cfg.py
```

Example:

```python
process.TFileService = cms.Service(
    "TFileService",
    fileName = cms.string("L1TrkNtuple.root"),
    closeFileFast = cms.untracked.bool(True)
)
```

If unchanged, the default output is typically

```
TTbar_PU200_D76.root
```

---

# 8. Performance Analysis

Generate standard performance plots using

```bash
csh makeHists.csh L1TrkNtuple.root
```

This executes the ROOT macros

- `L1TrackNtuplePlot.C`
- `L1TrackQualityPlot.C`

---

## L1TrackNtuplePlot.C

Output directory

```
TrkPlots/
```

Generates standard tracking performance plots including

- Tracking efficiency vs. $p_T$
- Tracking efficiency vs. $\eta$
- Fake rate
- Duplicate rate
- Kalman filter $\chi^2$ distributions

---

## L1TrackQualityPlot.C

Output directory

```
MVA_plots/
```

Evaluates the Boosted Decision Tree (BDT) used by the Track Quality module.

Generated plots include

- MVA score distributions
- ROC curves

These are used to determine the optimal FPGA hard-cut threshold.

---

# 9. Debugging

Several debugging and warning switches are available in

```
src/L1Trigger/TrackFindingTracklet/interface/Settings.h
```

(around line 914).

Changing selected boolean variables from

```cpp
false
```

to

```cpp
true
```

enables additional debug output.

The generated information can be found in

```
src/L1Trigger/TrackFindingTracklet/test/L1Trigger/
```

including directories such as

- `LUTs/`
- `MEMPrints/`

Large amounts of information are also printed to the terminal, so saving the log is recommended.

Example:

```bash
cmsRun L1TrackNtupleMaker_cfg.py |& tee run.log
```

---

# 10. Future Work

The various `Analyzer*.cc` modules have not yet been fully investigated.

Based on the current understanding, they compare firmware and software outputs while analyzing hardware-oriented objects such as structured **TTStub collections** produced by upstream modules.
