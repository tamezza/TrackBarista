# VBFprocessEJNT.py Modification Quick Reference

## 🎯 Key Differences: processEJNT.py vs VBFprocessEJNT.py

### Architecture
- **processEJNT.py**: Uses `dataset_tools.apply_to_fileset()` with direct dask compute
- **VBFprocessEJNT.py**: Uses `processor.Runner()` with `DaskExecutor` and chunked processing

### Processing Approach
- **processEJNT.py**: Single-pass with simple LocalCluster
- **VBFprocessEJNT.py**: Multi-chunk with distributed computing support (Kerberos, file caching)

---

## ✅ Essential Modifications (Must Do)

### 1. Add Dask Configuration (Top of File)
```python
import dask
dask.config.set({"awkward.optimization.enabled": False})
```

### 2. Add Truth Branches to Schema
```python
class bbbbaristaSchema(NtupleSchema):
    truth_branches = {"truth_HH_m"}
    event_ids = {
        # ... existing fields ...
        *truth_branches,  # ← Add this
    }
```

### 3. Add RNN Parameters Support
```python
# In your processing function kwargs:
kwargs["use_rnn"] = cfg["boosted_selection"].get("use_rnn", False)
kwargs["rnn_score_cut"] = cfg["boosted_selection"].get("rnn_score_cut", 0.5)
```

### 4. Add VBF Cutflow Saving
```python
# After regular cutflow saving:
if "boosted_vbf" in out[samp]["cutflows"]:
    vbf_cf = out[samp]["cutflows"]["boosted_vbf"]
    with open(f"{outputDir}/cutflows/{fileTag}_{samp}_boosted_vbf.json", "w") as f:
        f.write(json.dumps(vbf_cf, cls=NpEncoder, indent=4))
```

---

## 🔧 Important Modifications (Should Do)

### 5. Add Dask Sizeof Workaround
```python
from dask.sizeof import sizeof

@sizeof.register(dict)
def custom_sizeof_python_dict(d):
    # ... full implementation in detailed guide
```

### 6. Add Cutflow Text Files
```python
# Add alongside JSON output:
with open(f"{outputDir}/cutflows/{fileTag}_{samp}_{regime}.txt", "w") as f:
    f.write(f"Cutflow for {samp} ({regime}):\n")
    for c, n in cf.items():
        f.write(f"{c}: {n}\n")
```

### 7. Update Schema Import
```python
# Change from:
from atlas_schema.schema import NtupleSchema

# To:
from bbbbarista.schema import NtupleSchema
```

---

## 🎨 Optional Enhancements

### 8. Add PDF/Scale Branches (If Needed)
```python
class bbbbaristaSchema(NtupleSchema):
    pdf_branches = {f"generatorWeight_GEN_MUR10_MUF10_PDF303{k:03d}" for k in range(101)}
    alphas_branches = {
        "generatorWeight_GEN_MUR10_MUF10_PDF265000",
        "generatorWeight_GEN_MUR10_MUF10_PDF266000",
    }
    scale_branches = {
        "generatorWeight_GEN_MUR05_MUF05_PDF303000",
        # ... add all 7 scale variations
    }
```

---

## 📋 Code Snippets Ready to Copy

### Complete bbbbaristaSchema Class
```python
class bbbbaristaSchema(NtupleSchema):
    truth_branches = {"truth_HH_m"}
    
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
        *truth_branches,
    }
    
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

### Complete VBF Cutflow Saving
```python
# Place this in your cutflow saving section
if cfg["general"]["print_cutflow"]:
    for regime in ["resolved", "boosted"]:
        if cfg["general"]["analysis_strategy"] in [regime, "both"]:
            cf = out[samp][regime]
            
            # Save JSON
            with open(
                os.path.join(outputDir, "cutflows", f"{fileTag}_{samp}_{regime}.json"),
                "w",
            ) as f:
                f.write(json.dumps(cf, cls=NpEncoder, indent=4))
            
            # Save text
            with open(
                os.path.join(outputDir, "cutflows", f"{fileTag}_{samp}_{regime}.txt"),
                "w",
            ) as f:
                f.write(f"Cutflow for {samp} ({regime}):\n")
                for c, n in cf.items():
                    f.write(f"{c}: {n}\n")
            
            # Print to console
            for c, n in cf.items():
                print(f"INFO: \t\t{c}: {n}")
    
    # VBF cutflow (add after the regime loop)
    if "boosted_vbf" in out[samp]["cutflows"]:
        print(f"INFO: boosted VBF cutflow for", samp)
        vbf_cf = out[samp]["cutflows"]["boosted_vbf"]
        
        for c, n in vbf_cf.items():
            print(f"INFO: \t\t{c}: {n}")
        
        # Save VBF JSON
        with open(
            os.path.join(outputDir, "cutflows", f"{fileTag}_{samp}_boosted_vbf.json"),
            "w",
        ) as f:
            f.write(json.dumps(vbf_cf, cls=NpEncoder, indent=4))
        
        # Save VBF text
        with open(
            os.path.join(outputDir, "cutflows", f"{fileTag}_{samp}_boosted_vbf.txt"),
            "w",
        ) as f:
            f.write(f"VBF Cutflow for {samp}:\n")
            for c, n in vbf_cf.items():
                f.write(f"{c}: {n}\n")
```

---

## 🚀 Implementation Order

1. **First**: Add dask config and sizeof workaround (stability fixes)
2. **Second**: Update schema class (data structure changes)
3. **Third**: Add RNN parameters (feature support)
4. **Fourth**: Update cutflow saving (output improvements)
5. **Last**: Optional PDF/scale branches (only if needed)

---

## ⚠️ Common Pitfalls

| Issue | Cause | Solution |
|-------|-------|----------|
| Memory crash | No dask sizeof fix | Add `@sizeof.register(dict)` workaround |
| Missing truth data | Not in event_ids | Add `*truth_branches` to event_ids |
| VBF cutflow missing | Wrong dict structure | Check `out[samp]["cutflows"]["boosted_vbf"]` exists |
| RNN not working | Parameters not passed | Add to kwargs with `.get()` defaults |
| Schema import error | Wrong path | Use `from bbbbarista.schema import ...` |

---

## 🧪 Testing Commands

```bash
# Test with minimal config
python VBFprocessEJNT.py --config configs/test_config.yaml

# Check output structure
ls -lh output_dir/cutflows/
ls -lh output_dir/dataframes/

# Verify cutflows exist
cat output_dir/cutflows/*_boosted_vbf.txt
```

---

## 📊 Expected Output Structure

```
output_dir/
├── cutflows/
│   ├── tag_sample_resolved.json
│   ├── tag_sample_resolved.txt
│   ├── tag_sample_boosted.json
│   ├── tag_sample_boosted.txt
│   ├── tag_sample_boosted_vbf.json  ← New!
│   └── tag_sample_boosted_vbf.txt   ← New!
├── dataframes/
│   └── tag_sample.parquet
└── performance_reports/
    └── dask-report-compute.html
```

---

## 🔍 Verification Checklist

```python
# Run these checks after modification:

# 1. Import check
try:
    from bbbbarista.schema import NtupleSchema
    print("✓ Schema import works")
except ImportError as e:
    print(f"✗ Schema import failed: {e}")

# 2. Schema attributes check
schema = bbbbaristaSchema({})
assert hasattr(schema, 'truth_branches'), "Missing truth_branches"
assert "truth_HH_m" in schema.truth_branches, "Missing truth_HH_m"
print("✓ Schema has truth branches")

# 3. Config check
cfg = load_config("your_config.yaml")
assert "use_rnn" in cfg["boosted_selection"], "Missing RNN config"
assert "rnn_score_cut" in cfg["boosted_selection"], "Missing RNN cut"
print("✓ Config has RNN parameters")

# 4. Dask config check
import dask
assert not dask.config.get("awkward.optimization.enabled"), "Optimization still enabled"
print("✓ Dask optimization disabled")
```

---

## 📚 Related Files to Update

1. **Your analysis module** (e.g., `First_Nominal_Analysis_RNN.py`)
   - Ensure it returns VBF cutflows in the right format
   - Check it accepts `use_rnn` and `rnn_score_cut` parameters

2. **Your config YAML file**
   ```yaml
   boosted_selection:
     use_rnn: true
     rnn_score_cut: 0.5
   
   general:
     include_vbf: true
   ```

3. **create_fileset.py** (if used)
   - Ensure it handles truth branches correctly

---

## 💡 Key Insights from schema.py

### Collections to Remove (from schema.py experience):
```python
problematic_collections = [
    "passRelativeDeltaRToVRJetCutUFO",
    "beamSpotWeight",
    "GRL2015"
]
# These cause crashes and should be removed
```

### Debug Pattern for New Branches:
```python
# Add to schema._build_collections() for debugging
pdf_found = [k for k in branch_forms.keys() if 'PDF303' in k]
print(f"DEBUG: Found {len(pdf_found)} PDF branches")
```

---

## 🎓 Understanding the Differences

### Why VBFprocessEJNT is More Complex:

1. **Chunked Processing**: Splits fileset into chunks (nominal_only, nominal_plus_sys)
2. **Distributed Computing**: Supports Chicago cluster with Kerberos + XCache
3. **File Caching**: Can pre-cache files for faster access
4. **Plugin System**: Uses worker plugins for environment setup
5. **Step Sizes**: Dynamic step_size based on analysis strategy

### Why processEJNT is Simpler:

1. **Single Pass**: No chunk splitting
2. **Local Only**: LocalCluster with simple configuration
3. **Direct Processing**: `apply_to_fileset` handles everything
4. **Fewer Dependencies**: No plugins or caching needed

---

## 🔗 Quick Links to Documentation Sections

- Full modification guide: `VBFprocessEJNT_modification_guide.md`
- Detailed line-by-line changes: `VBFprocessEJNT_detailed_changes.md`
- This quick reference: `VBFprocessEJNT_quick_reference.md`

---

**Last Updated**: Based on processEJNT.py and schema.py analysis
**Compatibility**: VBFprocessEJNT.py using processor.Runner architecture
