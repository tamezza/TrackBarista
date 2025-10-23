# VBFprocessEJNT.py - Detailed Code Changes

This document shows the exact code changes needed for VBFprocessEJNT.py based on patterns from processEJNT.py and schema.py.

---

## CHANGE 1: Add Imports (After line 11, before line 12)

### ADD:
```python
from functools import partial
import dask
from dask.sizeof import sizeof

# Disable problematic optimizations
dask.config.set({"awkward.optimization.enabled": False})
```

---

## CHANGE 2: Add Dask Sizeof Workaround (After imports, before line 30)

### ADD:
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

## CHANGE 3: Update bbbbaristaSchema Class (Lines 38-69)

### REPLACE:
```python
class bbbbaristaSchema(NtupleSchema):
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
    }
```

### WITH:
```python
class bbbbaristaSchema(NtupleSchema):
    # Add truth branches to be preserved as event-level variables
    truth_branches = {
        "truth_HH_m",
    }
    
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
        *truth_branches,  # Add truth branches to event_ids
    }
```

---

## CHANGE 4: Update get_required_branches Function (Line 86-100)

### REPLACE:
```python
def get_required_branches(valid_systematics, requested_systematics):
    # get minimal required branches for the analysis to avoid loading unnecessary data
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
    ]
```

### WITH:
```python
def get_required_branches(valid_systematics, requested_systematics):
    # get minimal required branches for the analysis to avoid loading unnecessary data
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
        "truth_HH_m",  # Add truth branch
    ]
```

---

## CHANGE 5: Add run_selection Function (Add NEW function before create_output)

### ADD NEW FUNCTION:
```python
def run_selection(events, cfg):
    """
    Inputs:
    - events : ak array event level variables
    - cfg : configuration file with load_config() applied
    """
    # Import processDf from the appropriate analysis module
    from bbbbarista.First_Nominal_Analysis_RNN import processDf
    # Alternative imports based on your analysis:
    # from bbbbarista.Second_Highest_pT_Analysis import processDf
    # from bbbbarista.Third_Highest_Mjj_Analysis import processDf
    
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
        
        # Add RNN parameters
        kwargs["use_rnn"] = cfg["boosted_selection"].get("use_rnn", False)
        kwargs["rnn_score_cut"] = cfg["boosted_selection"].get("rnn_score_cut", 0.5)
    
    events, evtweights, cutflows = processDf(**kwargs)
    return events, evtweights, cutflows
```

---

## CHANGE 6: Update Cutflow Saving Section (Around lines 1410-1435)

### FIND the existing cutflow saving code:
```python
if cfg["general"]["print_cutflow"]:
    for regime in ["resolved", "boosted"]:
        if cfg["general"]["analysis_strategy"] in [regime, "both"]:
            cf = out[samp][regime]
            # ... existing code ...
            with open(
                os.path.join(
                    outputDir, "cutflows", f"{fileTag}_{samp}_{regime}.json"
                ),
                "w",
            ) as f:
                f.write(json.dumps(cf, cls=NpEncoder, indent=4))
```

### ADD AFTER the JSON writing (before the line break):
```python
            # Also save cutflow as text file
            cutflow_txt_file = os.path.join(
                outputDir, "cutflows", f"{fileTag}_{samp}_{regime}.txt"
            )
            with open(cutflow_txt_file, "w") as f:
                f.write(f"Cutflow for {samp} ({regime}):\n")
                for c, n in cf.items():
                    f.write(f"{c}: {n}\n")
            
            # Print cutflow to console
            for c, n in cf.items():
                print(f"INFO: \t\t{c}: {n}")

# ADD THIS SECTION AFTER ALL REGIME CUTFLOWS
# (After the for regime loop ends, still inside the print_cutflow if block)
if "boosted_vbf" in out[samp]:
    if "cutflows" in out[samp]["boosted_vbf"]:
        print(f"INFO: boosted VBF cutflow for", samp)
        vbf_cf = out[samp]["boosted_vbf"]["cutflows"]
        for c, n in vbf_cf.items():
            print(f"INFO: \t\t{c}: {n}")
        
        # Save VBF cutflow as JSON
        with open(
            os.path.join(
                outputDir, "cutflows", f"{fileTag}_{samp}_boosted_vbf.json"
            ),
            "w",
        ) as f:
            f.write(json.dumps(vbf_cf, cls=NpEncoder, indent=4))
        
        # Save VBF cutflow as text
        vbf_cutflow_txt_file = os.path.join(
            outputDir, "cutflows", f"{fileTag}_{samp}_boosted_vbf.txt"
        )
        with open(vbf_cutflow_txt_file, "w") as f:
            f.write(f"VBF Cutflow for {samp}:\n")
            for c, n in vbf_cf.items():
                f.write(f"{c}: {n}\n")
```

---

## CHANGE 7: Update Schema Import (Line 17)

### REPLACE:
```python
from atlas_schema.schema import NtupleSchema
```

### WITH:
```python
from bbbbarista.schema import NtupleSchema
```

---

## CHANGE 8: Optional - Add PDF/Scale Branches Support

### If you need PDF replicas and systematic variations, ADD to bbbbaristaSchema class:

```python
class bbbbaristaSchema(NtupleSchema):
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
        "generatorWeight_GEN_MUR05_MUF05_PDF303000",  # μ_R=0.5, μ_F=0.5
        "generatorWeight_GEN_MUR05_MUF10_PDF303000",  # μ_R=0.5, μ_F=1.0
        "generatorWeight_GEN_MUR10_MUF05_PDF303000",  # μ_R=1.0, μ_F=0.5
        "generatorWeight_GEN_MUR10_MUF10_PDF303000",  # μ_R=1.0, μ_F=1.0 (central)
        "generatorWeight_GEN_MUR20_MUF10_PDF303000",  # μ_R=2.0, μ_F=1.0
        "generatorWeight_GEN_MUR10_MUF20_PDF303000",  # μ_R=1.0, μ_F=2.0
        "generatorWeight_GEN_MUR20_MUF20_PDF303000",  # μ_R=2.0, μ_F=2.0
    }
    
    # Add truth branches
    truth_branches = {
        "truth_HH_m",
    }
    
    # Update event_ids to include PDF, scale, and truth branches if needed
    event_ids = {
        "runNumber",
        "eventNumber",
        # ... rest of event_ids ...
        *truth_branches,
        # *pdf_branches,  # Uncomment if needed
        # *alphas_branches,  # Uncomment if needed
        # *scale_branches,  # Uncomment if needed
    }
```

---

## CHANGE 9: Update create_output Function Call

### If you modified create_output to use run_selection, ensure it's called properly:

```python
# In the main processing section where create_output is called:
proc = create_output(cfg, get_weight_systematics())

# Make sure create_output internally calls run_selection:
def create_output(cfg, weight_systematics):
    def process_events(events):
        sel_events, _, cutflows = run_selection(events, cfg)
        # ... rest of processing
        return output
    return process_events
```

---

## Summary of Changes

| Change # | Description | Lines Affected | Priority |
|----------|-------------|----------------|----------|
| 1 | Add new imports | After line 11 | High |
| 2 | Add dask sizeof workaround | Before line 30 | High |
| 3 | Update bbbbaristaSchema class | Lines 38-69 | High |
| 4 | Update get_required_branches | Lines 86-100 | High |
| 5 | Add run_selection function | New function | High |
| 6 | Update cutflow saving | Lines 1410-1435 | Medium |
| 7 | Update schema import | Line 17 | High |
| 8 | Add PDF/scale branches | Optional | Low |
| 9 | Update create_output | Function level | Medium |

---

## Testing Checklist

- [ ] Code compiles without syntax errors
- [ ] Dask optimization disabled successfully
- [ ] Truth branches load correctly
- [ ] RNN parameters passed to analysis
- [ ] Regular cutflows save (resolved/boosted)
- [ ] VBF cutflows save correctly
- [ ] Both JSON and text cutflows created
- [ ] Parquet files merge successfully
- [ ] Memory usage is acceptable
- [ ] No crashes on problematic collections

---

## Configuration File Updates

Ensure your YAML config includes:

```yaml
boosted_selection:
  # ... existing parameters ...
  use_rnn: true  # or false
  rnn_score_cut: 0.5  # adjust as needed

general:
  include_vbf: true  # if using VBF analysis
```

---

## Common Issues and Solutions

### Issue 1: Import errors with schema
**Solution**: Ensure `from bbbbarista.schema import NtupleSchema` path is correct

### Issue 2: Missing RNN parameters
**Solution**: Add default values in run_selection: `.get("use_rnn", False)`

### Issue 3: VBF cutflows not saving
**Solution**: Check that processDf returns cutflows dict with "boosted_vbf" key

### Issue 4: Memory issues
**Solution**: Ensure dask sizeof workaround is properly registered

### Issue 5: Truth branches not found
**Solution**: Verify ROOT files contain "truth_HH_m" branch
