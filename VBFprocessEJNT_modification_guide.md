# Guide to Modifying VBFprocessEJNT.py

Based on analysis of `processEJNT.py` and `schema.py`, here are the key modifications needed for `VBFprocessEJNT.py`:

---

## 1. Import Changes

### Add Missing Imports (at the top of the file)
```python
from functools import partial
import dask
from dask.sizeof import sizeof

# Disable problematic optimizations
dask.config.set({"awkward.optimization.enabled": False})
```

### Add Dask Bug Workaround
```python
# A temporary fix for dask bug
@sizeof.register(dict)
def custom_sizeof_python_dict(d):
    try:
        total_size = sys.getsizeof(d)
        
        for k in d.keys():
            try:
                total_size += sizeof(k)
            except Exception:
                total_size += sys.getsizeof(k)
        
        for v in d.values():
            try:
                total_size += sizeof(v)
            except Exception:
                total_size += sys.getsizeof(v)
        
        total_size -= 2 * sys.getsizeof([])
        
        return total_size
    
    except Exception as e:
        print(f"Error calculating size of dict: {e}")
        return 1e6
```

---

## 2. Schema Class Updates

### Update the `bbbbaristaSchema` class to match `NtupleSchema` patterns:

```python
class bbbbaristaSchema(NtupleSchema):
    # Add truth branches
    truth_branches = {
        "truth_HH_m",
    }
    
    # Update event_ids to include truth branches
    event_ids = {
        "runNumber",
        "eventNumber",
        "actualInteractionsPerCrossing",
        "averageInteractionsPerCrossing",
        "GRL2016",
        "GRL2016_BjetHLT",
        "GRL2017_Triggerno17e33prim",
        "GRL2017_BjetHLT_Normal2017",
        "GRL2018_Triggerno17e33prim",
        "GRL2018_BjetHLT",
        "GRL2022",
        "GRL2023",
        "GRL2024",
        "trigger_bucket",
        *truth_branches,  # Add truth branches
    }
    
    # Update mixins to match schema.py patterns
    mixins = {
        "recojet_antikt4PFlow": "Jet",
        "recojet_antikt10UFO": "Jet",
        "truthjet_antikt10SoftDrop": "Jet",
        "met": "MissingET",
        "ph_baselineSelection_Tight_FixedCutLoose": "Photon",
        "generatorWeight": "Weight",
        "PileupWeight": "Weight",
        "trigPassed": "Pass",
        "trigger_SmallRJet_L1SF": "Weight",
        "trigger_SmallRJet_HLTSF": "Weight",
        "jvt_effSF": "Weight",
        "ftag_effSF": "Weight",
        "fjvt_effSF": "Weight",
    }
```

---

## 3. RNN Parameters Support

### Update `run_selection()` function to include RNN parameters:

```python
def run_selection(events, cfg):
    """
    Inputs:
    - events : ak array event level variables
    - cfg : configuration file with load_config() applied
    """
    analysis_strategy = cfg["general"]["analysis_strategy"]
    
    kwargs = {
        "events": events,
        "analysis_strategy": analysis_strategy,
        "unblind": cfg["general"]["unblind"],
        "doVBF": cfg["general"]["include_vbf"],
        "print_cutflow": cfg["general"]["print_cutflow"],
        "systematics": cfg["general"]["which_systs"],
    }
    
    if analysis_strategy in ["resolved", "both"]:
        kwargs["tagger"] = cfg["resolved_selection"]["tagger"]
        kwargs["WP"] = cfg["resolved_selection"]["btag_wp"]
        kwargs["min_btags"] = cfg["resolved_selection"]["min_btag"]
        kwargs["pairing"] = cfg["resolved_selection"]["which_pairing"]
        kwargs["pT_min"] = cfg["resolved_selection"]["pT_min_central"]
        kwargs["pT_min_Run3"] = cfg["resolved_selection"]["pT_min_central_Run3"]
        kwargs["eta_max"] = cfg["resolved_selection"]["eta_max_central"]
        kwargs["path_to_top_veto"] = cfg["resolved_selection"].get("path_to_top_veto")
    
    if analysis_strategy in ["boosted", "both"]:
        kwargs["boosted_tagger"] = cfg["boosted_selection"]["tagger"]
        kwargs["boosted_WP"] = cfg["boosted_selection"]["btag_wp"]
        kwargs["boosted_min_btag"] = cfg["boosted_selection"]["min_btag"]
        kwargs["boosted_min_jet_pt"] = cfg["boosted_selection"]["min_jet_pt"]
        kwargs["boosted_max_jet_pt"] = cfg["boosted_selection"]["max_jet_pt"]
        kwargs["boosted_max_jet_mass"] = cfg["boosted_selection"]["max_jet_mass"]
        kwargs["boosted_min_jet_mass"] = cfg["boosted_selection"]["min_jet_mass"]
        kwargs["boosted_max_jet_eta"] = cfg["boosted_selection"]["max_jet_eta"]
        kwargs["boosted_pT_leading_min"] = cfg["boosted_selection"]["min_leading_jet_pt"]
        kwargs["boosted_pT_subleading_min"] = cfg["boosted_selection"]["min_subleading_jet_pt"]
        kwargs["boosted_mass_leading_min"] = cfg["boosted_selection"]["min_leading_jet_m"]
        kwargs["boosted_mass_subleading_min"] = cfg["boosted_selection"]["min_subleading_jet_m"]
        kwargs["boosted_analysis_topology"] = cfg["boosted_selection"]["analysis_topology"]
        
        # ADD THESE TWO LINES FOR RNN PARAMETERS
        kwargs["use_rnn"] = cfg["boosted_selection"].get("use_rnn", False)
        kwargs["rnn_score_cut"] = cfg["boosted_selection"].get("rnn_score_cut", 0.5)
    
    events, evtweights, cutflows = processDf(**kwargs)
    return events, evtweights, cutflows
```

---

## 4. VBF Cutflow Support

### Update cutflow saving section in the main processing loop:

Add this section after the regular cutflow saving (around line 1429-1435 in your code):

```python
# Also save the VBF cutflow if it exists
if "boosted_vbf" in out[samp]["cutflows"]:
    print(f"INFO: boosted VBF cutflow for", samp)
    vbf_cf = out[samp]["cutflows"]["boosted_vbf"]
    for c, n in vbf_cf.items():
        print(f"INFO: \t\t{c}: {n}")
    with open(
        os.path.join(
            outputDir, "cutflows", f"{fileTag}_{samp}_boosted_vbf.json"
        ),
        "w",
    ) as f:
        f.write(json.dumps(vbf_cf, cls=NpEncoder, indent=4))
```

---

## 5. Analysis Import Path

### Update the import statement for the analysis module:

Change from:
```python
from bbbbarista.analysis import processDf
```

To one of these options (depending on your analysis):
```python
from bbbbarista.First_Nominal_Analysis_RNN import processDf
# OR
from bbbbarista.Second_Highest_pT_Analysis import processDf
# OR
from bbbbarista.Third_Highest_Mjj_Analysis import processDf
```

---

## 6. Schema Import Update

### Change schema import from:
```python
from atlas_schema.schema import NtupleSchema
```

To:
```python
from bbbbarista.schema import NtupleSchema
```

---

## 7. Key Differences Between VBF and Standard Processing

### VBFprocessEJNT.py uses:
- `processor.Runner` with `DaskExecutor`
- Chunked fileset processing
- Separate preprocessing and compute steps
- `ZipUploadPlugin` and `KerberosEnvPlugin` for distributed computing
- More complex file caching mechanism

### processEJNT.py uses:
- `dataset_tools.apply_to_fileset` with direct dask compute
- Single-pass processing
- Simpler local cluster setup
- Direct parquet merging

---

## 8. Collections Handling (from schema.py)

### Important pattern from schema.py for handling collections:

```python
# These collections should be removed to avoid crashes
if "passRelativeDeltaRToVRJetCutUFO" in collections:
    collections.remove("passRelativeDeltaRToVRJetCutUFO")

if "beamSpotWeight" in collections:
    collections.remove("beamSpotWeight")

if "GRL2015" in collections:
    collections.remove("GRL2015")
```

---

## 9. PDF and Systematic Branches (from schema.py)

### If you need PDF replica support, add these to your schema:

```python
# Generate PDF replica branch names (PDF303000 to PDF303100)
pdf_branches = {
    f"generatorWeight_GEN_MUR10_MUF10_PDF303{k:03d}" 
    for k in range(0, 101)  # 000 to 100 (replicas only)
}

# Add α_s variation branches
alphas_branches = {
    "generatorWeight_GEN_MUR10_MUF10_PDF265000",  # α_s = 0.117
    "generatorWeight_GEN_MUR10_MUF10_PDF266000",  # α_s = 0.119
}

# Add scale variation branches
scale_branches = {
    "generatorWeight_GEN_MUR05_MUF05_PDF303000",
    "generatorWeight_GEN_MUR05_MUF10_PDF303000",
    "generatorWeight_GEN_MUR10_MUF05_PDF303000",
    "generatorWeight_GEN_MUR10_MUF10_PDF303000",
    "generatorWeight_GEN_MUR20_MUF10_PDF303000",
    "generatorWeight_GEN_MUR10_MUF20_PDF303000",
    "generatorWeight_GEN_MUR20_MUF20_PDF303000",
}
```

---

## 10. Required Branches Update

### Add truth branches to `get_required_branches()` function:

```python
def get_required_branches(valid_systematics, requested_systematics):
    branches = [
        "runNumber",
        "eventNumber",
        "actualInteractionsPerCrossing",
        "averageInteractionsPerCrossing",
        "GRL2016",
        "GRL2016_BjetHLT",
        "GRL2017_Triggerno17e33prim",
        "GRL2017_BjetHLT_Normal2017",
        "GRL2018_Triggerno17e33prim",
        "GRL2018_BjetHLT",
        "GRL2022",
        "GRL2023",
        "GRL2024",
        "truth_HH_m",  # ADD THIS LINE
    ]
    # ... rest of the function
```

---

## 11. Cutflow Text File Addition

### Add text file output for cutflows (in addition to JSON):

```python
# Write cutflow to a text file
cutflow_txt_file = os.path.join(
    outputDir, "cutflows", f"{fileTag}_{samp}_{regime}.txt"
)
with open(cutflow_txt_file, "w") as f:
    f.write(f"Cutflow for {samp} ({regime}):\n")
    for c, n in cf.items():
        f.write(f"{c}: {n}\n")
```

---

## Summary of Critical Changes

1. **Add dask optimization disabling** at the top
2. **Add dask sizeof workaround** for dictionary handling
3. **Update schema class** to include truth branches in event_ids
4. **Add RNN parameter support** in run_selection()
5. **Add VBF cutflow saving** in the output section
6. **Update analysis import path** to use specific analysis module
7. **Consider adding cutflow text file output** alongside JSON

---

## Testing Recommendations

After making changes:
1. Test with a small subset of files first
2. Verify all cutflows are being saved correctly (regular + VBF)
3. Check that truth branches are accessible in the output
4. Ensure RNN parameters are being passed correctly
5. Monitor memory usage with the dask workaround
6. Verify parquet files are merging correctly

---

## Notes

- The main structural difference is that VBFprocessEJNT.py uses a more complex chunked processing approach suitable for large-scale distributed computing
- processEJNT.py uses a simpler approach with `dataset_tools.apply_to_fileset`
- Both can coexist, but you should align the schema and analysis function calls
- The schema.py file shows important patterns for handling collections and avoiding crashes
