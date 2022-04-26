# Aggregate variant counts for all samples
Separate `Snakemake` rules count the observations of each variant in each sample from the Illumina barcode sequencing.
This Python Jupyter notebook aggregates all of this counts, and then adds them to a codon variant table.

## Set up analysis
### Import Python modules.
Use [plotnine](https://plotnine.readthedocs.io/en/stable/) for ggplot2-like plotting.

The analysis relies heavily on the Bloom lab's [dms_variants](https://jbloomlab.github.io/dms_variants) package:


```python
import glob
import itertools
import math
import os
import warnings

import Bio.SeqIO

import dms_variants.codonvarianttable
from dms_variants.constants import CBPALETTE
import dms_variants.utils
import dms_variants.plotnine_themes

from IPython.display import display, HTML

import pandas as pd

from plotnine import *

import yaml
```

Set [plotnine](https://plotnine.readthedocs.io/en/stable/) theme to the gray-grid one defined in `dms_variants`:


```python
theme_set(dms_variants.plotnine_themes.theme_graygrid())
```

Versions of key software:


```python
print(f"Using dms_variants version {dms_variants.__version__}")
```

    Using dms_variants version 0.8.5


Ignore warnings that clutter output:


```python
warnings.simplefilter('ignore')
```

Read the configuration file:


```python
with open('config.yaml') as f:
    config = yaml.safe_load(f)
```

Make output directory if needed:


```python
os.makedirs(config['counts_dir'], exist_ok=True)
```

## Initialize codon variant table
Initialize the [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) using the wildtype gene sequence and the CSV file with the table of variants:


```python
wt_seqrecord = Bio.SeqIO.read(config['wildtype_sequence'], 'fasta')
geneseq = str(wt_seqrecord.seq)
primary_target = wt_seqrecord.name
print(f"Read sequence of {len(geneseq)} nt for {primary_target} from {config['wildtype_sequence']}")
      
print(f"Initializing CodonVariantTable from gene sequence and {config['codon_variant_table']}")
      
variants = dms_variants.codonvarianttable.CodonVariantTable(
                geneseq=geneseq,
                barcode_variant_file=config['codon_variant_table'],
                substitutions_are_codon=True,
                substitutions_col='codon_substitutions',
                primary_target=primary_target)
```

    Read sequence of 603 nt for SARS-CoV-2 from data/wildtype_sequence.fasta
    Initializing CodonVariantTable from gene sequence and results/prior_DMS_data/codon_variant_table.csv


## Read barcode counts / fates
Read data frame with list of all samples (barcode runs):


```python
print(f"Reading list of barcode runs from {config['barcode_runs']}")

barcode_runs = (pd.read_csv(config['barcode_runs'])
                .assign(sample_lib=lambda x: x['sample'] + '_' + x['library'],
                        counts_file=lambda x: config['counts_dir'] + '/' + x['sample_lib'] + '_counts.csv',
                        fates_file=lambda x: config['counts_dir'] + '/' + x['sample_lib'] + '_fates.csv',
                        )
                .drop(columns='R1')  # don't need this column, and very large
                )

assert all(map(os.path.isfile, barcode_runs['counts_file'])), 'missing some counts files'
assert all(map(os.path.isfile, barcode_runs['fates_file'])), 'missing some fates files'

display(HTML(barcode_runs.to_html(index=False)))
```

    Reading list of barcode runs from data/barcode_runs.csv



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>date</th>
      <th>experiment</th>
      <th>library</th>
      <th>antibody</th>
      <th>concentration</th>
      <th>concentration_units</th>
      <th>group</th>
      <th>selection</th>
      <th>sample</th>
      <th>frac_escape</th>
      <th>cells_sorted</th>
      <th>sample_lib</th>
      <th>counts_file</th>
      <th>fates_file</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>201106</td>
      <td>expt_68-73</td>
      <td>lib1</td>
      <td>none</td>
      <td>0</td>
      <td>ng_per_mL</td>
      <td>Vir</td>
      <td>reference</td>
      <td>expt_68-73-none-0-reference</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>expt_68-73-none-0-reference_lib1</td>
      <td>results/counts/expt_68-73-none-0-reference_lib1_counts.csv</td>
      <td>results/counts/expt_68-73-none-0-reference_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>201106</td>
      <td>expt_68-73</td>
      <td>lib2</td>
      <td>none</td>
      <td>0</td>
      <td>ng_per_mL</td>
      <td>Vir</td>
      <td>reference</td>
      <td>expt_68-73-none-0-reference</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>expt_68-73-none-0-reference_lib2</td>
      <td>results/counts/expt_68-73-none-0-reference_lib2_counts.csv</td>
      <td>results/counts/expt_68-73-none-0-reference_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>201106</td>
      <td>expt_71</td>
      <td>lib1</td>
      <td>S2X259</td>
      <td>59</td>
      <td>ng_per_mL</td>
      <td>Vir</td>
      <td>escape</td>
      <td>expt_71-S2X259-59-escape</td>
      <td>0.068</td>
      <td>700753.0</td>
      <td>expt_71-S2X259-59-escape_lib1</td>
      <td>results/counts/expt_71-S2X259-59-escape_lib1_counts.csv</td>
      <td>results/counts/expt_71-S2X259-59-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>201106</td>
      <td>expt_71</td>
      <td>lib2</td>
      <td>S2X259</td>
      <td>59</td>
      <td>ng_per_mL</td>
      <td>Vir</td>
      <td>escape</td>
      <td>expt_71-S2X259-59-escape</td>
      <td>0.064</td>
      <td>600000.0</td>
      <td>expt_71-S2X259-59-escape_lib2</td>
      <td>results/counts/expt_71-S2X259-59-escape_lib2_counts.csv</td>
      <td>results/counts/expt_71-S2X259-59-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_135-139</td>
      <td>lib1</td>
      <td>none</td>
      <td>0</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>reference</td>
      <td>expt_135-139-none-0-reference</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>expt_135-139-none-0-reference_lib1</td>
      <td>results/counts/expt_135-139-none-0-reference_lib1_counts.csv</td>
      <td>results/counts/expt_135-139-none-0-reference_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_135-139</td>
      <td>lib2</td>
      <td>none</td>
      <td>0</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>reference</td>
      <td>expt_135-139-none-0-reference</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>expt_135-139-none-0-reference_lib2</td>
      <td>results/counts/expt_135-139-none-0-reference_lib2_counts.csv</td>
      <td>results/counts/expt_135-139-none-0-reference_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_140-148</td>
      <td>lib1</td>
      <td>none</td>
      <td>0</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>reference</td>
      <td>expt_140-148-none-0-reference</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>expt_140-148-none-0-reference_lib1</td>
      <td>results/counts/expt_140-148-none-0-reference_lib1_counts.csv</td>
      <td>results/counts/expt_140-148-none-0-reference_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_140-148</td>
      <td>lib2</td>
      <td>none</td>
      <td>0</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>reference</td>
      <td>expt_140-148-none-0-reference</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>expt_140-148-none-0-reference_lib2</td>
      <td>results/counts/expt_140-148-none-0-reference_lib2_counts.csv</td>
      <td>results/counts/expt_140-148-none-0-reference_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_135</td>
      <td>lib1</td>
      <td>mAb1</td>
      <td>139</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_135-mAb1-139-escape</td>
      <td>0.077</td>
      <td>842165.0</td>
      <td>expt_135-mAb1-139-escape_lib1</td>
      <td>results/counts/expt_135-mAb1-139-escape_lib1_counts.csv</td>
      <td>results/counts/expt_135-mAb1-139-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_135</td>
      <td>lib2</td>
      <td>mAb1</td>
      <td>139</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_135-mAb1-139-escape</td>
      <td>0.079</td>
      <td>851124.0</td>
      <td>expt_135-mAb1-139-escape_lib2</td>
      <td>results/counts/expt_135-mAb1-139-escape_lib2_counts.csv</td>
      <td>results/counts/expt_135-mAb1-139-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_136</td>
      <td>lib1</td>
      <td>mAb2</td>
      <td>232</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_136-mAb2-232-escape</td>
      <td>0.057</td>
      <td>617689.0</td>
      <td>expt_136-mAb2-232-escape_lib1</td>
      <td>results/counts/expt_136-mAb2-232-escape_lib1_counts.csv</td>
      <td>results/counts/expt_136-mAb2-232-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_136</td>
      <td>lib2</td>
      <td>mAb2</td>
      <td>232</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_136-mAb2-232-escape</td>
      <td>0.055</td>
      <td>614856.0</td>
      <td>expt_136-mAb2-232-escape_lib2</td>
      <td>results/counts/expt_136-mAb2-232-escape_lib2_counts.csv</td>
      <td>results/counts/expt_136-mAb2-232-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_137</td>
      <td>lib1</td>
      <td>mAb3</td>
      <td>234</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_137-mAb3-234-escape</td>
      <td>0.110</td>
      <td>1190414.0</td>
      <td>expt_137-mAb3-234-escape_lib1</td>
      <td>results/counts/expt_137-mAb3-234-escape_lib1_counts.csv</td>
      <td>results/counts/expt_137-mAb3-234-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_137</td>
      <td>lib2</td>
      <td>mAb3</td>
      <td>234</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_137-mAb3-234-escape</td>
      <td>0.118</td>
      <td>1253385.0</td>
      <td>expt_137-mAb3-234-escape_lib2</td>
      <td>results/counts/expt_137-mAb3-234-escape_lib2_counts.csv</td>
      <td>results/counts/expt_137-mAb3-234-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_138</td>
      <td>lib1</td>
      <td>mAb4</td>
      <td>462</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_138-mAb4-462-escape</td>
      <td>0.057</td>
      <td>604056.0</td>
      <td>expt_138-mAb4-462-escape_lib1</td>
      <td>results/counts/expt_138-mAb4-462-escape_lib1_counts.csv</td>
      <td>results/counts/expt_138-mAb4-462-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_138</td>
      <td>lib2</td>
      <td>mAb4</td>
      <td>462</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_138-mAb4-462-escape</td>
      <td>0.059</td>
      <td>622698.0</td>
      <td>expt_138-mAb4-462-escape_lib2</td>
      <td>results/counts/expt_138-mAb4-462-escape_lib2_counts.csv</td>
      <td>results/counts/expt_138-mAb4-462-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_139</td>
      <td>lib1</td>
      <td>mAb5</td>
      <td>1694</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_139-mAb5-1694-escape</td>
      <td>0.206</td>
      <td>2222036.0</td>
      <td>expt_139-mAb5-1694-escape_lib1</td>
      <td>results/counts/expt_139-mAb5-1694-escape_lib1_counts.csv</td>
      <td>results/counts/expt_139-mAb5-1694-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211105</td>
      <td>expt_139</td>
      <td>lib2</td>
      <td>mAb5</td>
      <td>1694</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_139-mAb5-1694-escape</td>
      <td>0.214</td>
      <td>2245393.0</td>
      <td>expt_139-mAb5-1694-escape_lib2</td>
      <td>results/counts/expt_139-mAb5-1694-escape_lib2_counts.csv</td>
      <td>results/counts/expt_139-mAb5-1694-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_140</td>
      <td>lib1</td>
      <td>mAb6</td>
      <td>584</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_140-mAb6-584-escape</td>
      <td>0.100</td>
      <td>1002651.0</td>
      <td>expt_140-mAb6-584-escape_lib1</td>
      <td>results/counts/expt_140-mAb6-584-escape_lib1_counts.csv</td>
      <td>results/counts/expt_140-mAb6-584-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_140</td>
      <td>lib2</td>
      <td>mAb6</td>
      <td>584</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_140-mAb6-584-escape</td>
      <td>0.101</td>
      <td>1011280.0</td>
      <td>expt_140-mAb6-584-escape_lib2</td>
      <td>results/counts/expt_140-mAb6-584-escape_lib2_counts.csv</td>
      <td>results/counts/expt_140-mAb6-584-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_141</td>
      <td>lib1</td>
      <td>mAb7</td>
      <td>103</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_141-mAb7-103-escape</td>
      <td>0.171</td>
      <td>1959806.0</td>
      <td>expt_141-mAb7-103-escape_lib1</td>
      <td>results/counts/expt_141-mAb7-103-escape_lib1_counts.csv</td>
      <td>results/counts/expt_141-mAb7-103-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_141</td>
      <td>lib2</td>
      <td>mAb7</td>
      <td>103</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_141-mAb7-103-escape</td>
      <td>0.180</td>
      <td>1799047.0</td>
      <td>expt_141-mAb7-103-escape_lib2</td>
      <td>results/counts/expt_141-mAb7-103-escape_lib2_counts.csv</td>
      <td>results/counts/expt_141-mAb7-103-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_142</td>
      <td>lib1</td>
      <td>mAb8</td>
      <td>2000</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_142-mAb8-2000-escape</td>
      <td>0.142</td>
      <td>1421301.0</td>
      <td>expt_142-mAb8-2000-escape_lib1</td>
      <td>results/counts/expt_142-mAb8-2000-escape_lib1_counts.csv</td>
      <td>results/counts/expt_142-mAb8-2000-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_142</td>
      <td>lib2</td>
      <td>mAb8</td>
      <td>2000</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_142-mAb8-2000-escape</td>
      <td>0.136</td>
      <td>1374728.0</td>
      <td>expt_142-mAb8-2000-escape_lib2</td>
      <td>results/counts/expt_142-mAb8-2000-escape_lib2_counts.csv</td>
      <td>results/counts/expt_142-mAb8-2000-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_143</td>
      <td>lib1</td>
      <td>mAb9</td>
      <td>154</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_143-mAb9-154-escape</td>
      <td>0.038</td>
      <td>429649.0</td>
      <td>expt_143-mAb9-154-escape_lib1</td>
      <td>results/counts/expt_143-mAb9-154-escape_lib1_counts.csv</td>
      <td>results/counts/expt_143-mAb9-154-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_143</td>
      <td>lib2</td>
      <td>mAb9</td>
      <td>154</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_143-mAb9-154-escape</td>
      <td>0.040</td>
      <td>400943.0</td>
      <td>expt_143-mAb9-154-escape_lib2</td>
      <td>results/counts/expt_143-mAb9-154-escape_lib2_counts.csv</td>
      <td>results/counts/expt_143-mAb9-154-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_144</td>
      <td>lib1</td>
      <td>mAb10</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_144-mAb10-147-escape</td>
      <td>0.046</td>
      <td>467946.0</td>
      <td>expt_144-mAb10-147-escape_lib1</td>
      <td>results/counts/expt_144-mAb10-147-escape_lib1_counts.csv</td>
      <td>results/counts/expt_144-mAb10-147-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_144</td>
      <td>lib2</td>
      <td>mAb10</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_144-mAb10-147-escape</td>
      <td>0.044</td>
      <td>441191.0</td>
      <td>expt_144-mAb10-147-escape_lib2</td>
      <td>results/counts/expt_144-mAb10-147-escape_lib2_counts.csv</td>
      <td>results/counts/expt_144-mAb10-147-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_145</td>
      <td>lib1</td>
      <td>mAb11</td>
      <td>125</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_145-mAb11-125-escape</td>
      <td>0.054</td>
      <td>538149.0</td>
      <td>expt_145-mAb11-125-escape_lib1</td>
      <td>results/counts/expt_145-mAb11-125-escape_lib1_counts.csv</td>
      <td>results/counts/expt_145-mAb11-125-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_145</td>
      <td>lib2</td>
      <td>mAb11</td>
      <td>125</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_145-mAb11-125-escape</td>
      <td>0.044</td>
      <td>446148.0</td>
      <td>expt_145-mAb11-125-escape_lib2</td>
      <td>results/counts/expt_145-mAb11-125-escape_lib2_counts.csv</td>
      <td>results/counts/expt_145-mAb11-125-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_146</td>
      <td>lib1</td>
      <td>mAb12</td>
      <td>267</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_146-mAb12-267-escape</td>
      <td>0.053</td>
      <td>536557.0</td>
      <td>expt_146-mAb12-267-escape_lib1</td>
      <td>results/counts/expt_146-mAb12-267-escape_lib1_counts.csv</td>
      <td>results/counts/expt_146-mAb12-267-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_146</td>
      <td>lib2</td>
      <td>mAb12</td>
      <td>267</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_146-mAb12-267-escape</td>
      <td>0.053</td>
      <td>532129.0</td>
      <td>expt_146-mAb12-267-escape_lib2</td>
      <td>results/counts/expt_146-mAb12-267-escape_lib2_counts.csv</td>
      <td>results/counts/expt_146-mAb12-267-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_147</td>
      <td>lib1</td>
      <td>mAb13</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_147-mAb13-147-escape</td>
      <td>0.043</td>
      <td>430775.0</td>
      <td>expt_147-mAb13-147-escape_lib1</td>
      <td>results/counts/expt_147-mAb13-147-escape_lib1_counts.csv</td>
      <td>results/counts/expt_147-mAb13-147-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_147</td>
      <td>lib2</td>
      <td>mAb13</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_147-mAb13-147-escape</td>
      <td>0.047</td>
      <td>472865.0</td>
      <td>expt_147-mAb13-147-escape_lib2</td>
      <td>results/counts/expt_147-mAb13-147-escape_lib2_counts.csv</td>
      <td>results/counts/expt_147-mAb13-147-escape_lib2_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_148</td>
      <td>lib1</td>
      <td>mAb14</td>
      <td>256</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_148-mAb14-256-escape</td>
      <td>0.137</td>
      <td>1384538.0</td>
      <td>expt_148-mAb14-256-escape_lib1</td>
      <td>results/counts/expt_148-mAb14-256-escape_lib1_counts.csv</td>
      <td>results/counts/expt_148-mAb14-256-escape_lib1_fates.csv</td>
    </tr>
    <tr>
      <td>211108</td>
      <td>expt_148</td>
      <td>lib2</td>
      <td>mAb14</td>
      <td>256</td>
      <td>ng_per_mL</td>
      <td>Linfa_NUS</td>
      <td>escape</td>
      <td>expt_148-mAb14-256-escape</td>
      <td>0.148</td>
      <td>1506071.0</td>
      <td>expt_148-mAb14-256-escape_lib2</td>
      <td>results/counts/expt_148-mAb14-256-escape_lib2_counts.csv</td>
      <td>results/counts/expt_148-mAb14-256-escape_lib2_fates.csv</td>
    </tr>
  </tbody>
</table>


Confirm sample / library combinations unique:


```python
assert len(barcode_runs) == len(barcode_runs.groupby(['sample', 'library']))
```

Make sure the the libraries for which we have barcode runs are all in our variant table:


```python
unknown_libs = set(barcode_runs['library']) - set(variants.libraries)
if unknown_libs:
    raise ValueError(f"Libraries with barcode runs not in variant table: {unknown_libs}")
```

Now concatenate the barcode counts and fates for each sample:


```python
counts = pd.concat([pd.read_csv(f) for f in barcode_runs['counts_file']],
                   sort=False,
                   ignore_index=True)

print('First few lines of counts data frame:')
display(HTML(counts.head().to_html(index=False)))

fates = pd.concat([pd.read_csv(f) for f in barcode_runs['fates_file']],
                  sort=False,
                  ignore_index=True)

print('First few lines of fates data frame:')
display(HTML(fates.head().to_html(index=False)))
```

    First few lines of counts data frame:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>barcode</th>
      <th>count</th>
      <th>library</th>
      <th>sample</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>TTCCAAAATATTGTCA</td>
      <td>34349</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>TAGTAACAATGCGGTA</td>
      <td>11691</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>ATAAAAAGTCCATATG</td>
      <td>8080</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>TTAATTAGTATCAGGT</td>
      <td>7890</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>GCGCATAAGAAGGTAC</td>
      <td>6364</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
  </tbody>
</table>


    First few lines of fates data frame:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>fate</th>
      <th>count</th>
      <th>library</th>
      <th>sample</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>valid barcode</td>
      <td>28144729</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>invalid barcode</td>
      <td>2932535</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>low quality barcode</td>
      <td>1702149</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>failed chastity filter</td>
      <td>751519</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
    <tr>
      <td>unparseable barcode</td>
      <td>528943</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
    </tr>
  </tbody>
</table>


## Examine fates of parsed barcodes
First, we'll analyze the "fates" of the parsed barcodes.
These fates represent what happened to each Illumina read we parsed:
 - Did the barcode read fail the Illumina chastity filter?
 - Was the barcode *unparseable* (i.e., the read didn't appear to be a valid barcode based on flanking regions)?
 - Was the barcode sequence too *low quality* based on the Illumina quality scores?
 - Was the barcode parseable but *invalid* (i.e., not in our list of variant-associated barcodes in the codon variant table)?
 - Was the barcode *valid*, and so will be added to variant counts.
 
First, we just write a CSV file with all the barcode fates:


```python
fatesfile = os.path.join(config['counts_dir'], 'barcode_fates.csv')
print(f"Writing barcode fates to {fatesfile}")
fates.to_csv(fatesfile, index=False)
```

    Writing barcode fates to results/counts/barcode_fates.csv


Next, we tabulate the barcode fates in wide format:


```python
display(HTML(fates
             .pivot_table(columns='fate',
                          values='count',
                          index=['sample', 'library'])
             .applymap('{:.1e}'.format)  # scientific notation
             .to_html()
             ))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fate</th>
      <th>failed chastity filter</th>
      <th>invalid barcode</th>
      <th>low quality barcode</th>
      <th>unparseable barcode</th>
      <th>valid barcode</th>
    </tr>
    <tr>
      <th>sample</th>
      <th>library</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">expt_135-139-none-0-reference</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>6.1e+06</td>
      <td>1.4e+07</td>
      <td>1.6e+06</td>
      <td>5.9e+07</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>5.2e+06</td>
      <td>1.3e+07</td>
      <td>1.5e+06</td>
      <td>5.5e+07</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_135-mAb1-139-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>4.4e+05</td>
      <td>1.1e+06</td>
      <td>1.2e+05</td>
      <td>4.7e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>3.6e+05</td>
      <td>9.3e+05</td>
      <td>1.1e+05</td>
      <td>4.0e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_136-mAb2-232-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>2.9e+05</td>
      <td>7.2e+05</td>
      <td>7.6e+04</td>
      <td>3.1e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>2.2e+05</td>
      <td>6.3e+05</td>
      <td>7.3e+04</td>
      <td>2.8e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_137-mAb3-234-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>5.1e+05</td>
      <td>1.3e+06</td>
      <td>1.3e+05</td>
      <td>5.4e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>4.6e+05</td>
      <td>1.2e+06</td>
      <td>1.4e+05</td>
      <td>5.1e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_138-mAb4-462-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>2.8e+05</td>
      <td>7.1e+05</td>
      <td>7.5e+04</td>
      <td>3.0e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>2.3e+05</td>
      <td>6.8e+05</td>
      <td>7.5e+04</td>
      <td>2.9e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_139-mAb5-1694-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>1.1e+06</td>
      <td>2.7e+06</td>
      <td>2.9e+05</td>
      <td>1.1e+07</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.1e+06</td>
      <td>2.7e+06</td>
      <td>3.2e+05</td>
      <td>1.1e+07</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_140-148-none-0-reference</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>6.4e+06</td>
      <td>1.5e+07</td>
      <td>1.7e+06</td>
      <td>6.2e+07</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>5.4e+06</td>
      <td>1.4e+07</td>
      <td>1.6e+06</td>
      <td>5.7e+07</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_140-mAb6-584-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>1.5e+05</td>
      <td>3.8e+05</td>
      <td>3.9e+04</td>
      <td>1.6e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.0e+05</td>
      <td>2.9e+05</td>
      <td>3.3e+04</td>
      <td>1.2e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_141-mAb7-103-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>2.6e+05</td>
      <td>6.2e+05</td>
      <td>7.0e+04</td>
      <td>2.7e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>2.4e+05</td>
      <td>6.3e+05</td>
      <td>7.3e+04</td>
      <td>2.7e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_142-mAb8-2000-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>2.1e+05</td>
      <td>5.3e+05</td>
      <td>6.3e+04</td>
      <td>2.3e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>2.1e+05</td>
      <td>5.7e+05</td>
      <td>6.8e+04</td>
      <td>2.5e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_143-mAb9-154-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>7.3e+04</td>
      <td>1.8e+05</td>
      <td>2.0e+04</td>
      <td>7.5e+05</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>3.4e+04</td>
      <td>9.9e+04</td>
      <td>1.1e+04</td>
      <td>4.3e+05</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_144-mAb10-147-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>8.7e+04</td>
      <td>2.0e+05</td>
      <td>2.4e+04</td>
      <td>8.6e+05</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>5.7e+04</td>
      <td>1.5e+05</td>
      <td>1.7e+04</td>
      <td>6.4e+05</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_145-mAb11-125-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>7.0e+04</td>
      <td>1.6e+05</td>
      <td>1.8e+04</td>
      <td>7.0e+05</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>6.0e+04</td>
      <td>1.6e+05</td>
      <td>1.7e+04</td>
      <td>6.8e+05</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_146-mAb12-267-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>8.0e+04</td>
      <td>1.9e+05</td>
      <td>2.2e+04</td>
      <td>8.1e+05</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>1.7e+03</td>
      <td>4.5e+03</td>
      <td>5.0e+02</td>
      <td>1.9e+04</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_147-mAb13-147-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>5.1e+04</td>
      <td>1.2e+05</td>
      <td>1.4e+04</td>
      <td>5.1e+05</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>8.9e+04</td>
      <td>1.5e+05</td>
      <td>1.6e+04</td>
      <td>6.1e+05</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_148-mAb14-256-escape</th>
      <th>lib1</th>
      <td>0.0e+00</td>
      <td>2.7e+06</td>
      <td>1.4e+06</td>
      <td>1.6e+05</td>
      <td>4.1e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>0.0e+00</td>
      <td>2.1e+05</td>
      <td>5.3e+05</td>
      <td>6.0e+04</td>
      <td>2.2e+06</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_68-73-none-0-reference</th>
      <th>lib1</th>
      <td>7.5e+05</td>
      <td>2.9e+06</td>
      <td>1.7e+06</td>
      <td>5.3e+05</td>
      <td>2.8e+07</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>8.2e+05</td>
      <td>2.9e+06</td>
      <td>1.8e+06</td>
      <td>5.9e+05</td>
      <td>3.1e+07</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_71-S2X259-59-escape</th>
      <th>lib1</th>
      <td>6.1e+04</td>
      <td>2.2e+05</td>
      <td>1.4e+05</td>
      <td>3.6e+04</td>
      <td>2.3e+06</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>3.8e+04</td>
      <td>1.2e+05</td>
      <td>8.4e+04</td>
      <td>3.1e+04</td>
      <td>1.4e+06</td>
    </tr>
  </tbody>
</table>


Now we plot the barcode-read fates for each library / sample, showing the bars for valid barcodes in orange and the others in gray.
We see that the largest fraction of barcode reads correspond to valid barcodes, and most of the others are invalid barcodes (probably because the map to variants that aren't present in our variant table since we didn't associate all variants with barcodes). The exception to this is lib2 Titeseq_03_bin3; the PCR for this sample in the original sequencing run failed, so we followed it up with a single MiSeq lane. We did not filter out the PhiX reads from this data before parsing, so these PhiX reads will deflate the fraction of valid barcode reads as expected, but does not indicate any problems.


```python
ncol = 4
nfacets = len(fates.groupby(['sample', 'library']))

barcode_fate_plot = (
    ggplot(
        fates
        .assign(sample=lambda x: pd.Categorical(x['sample'],
                                                x['sample'].unique(),
                                                ordered=True),
                fate=lambda x: pd.Categorical(x['fate'],
                                              x['fate'].unique(),
                                              ordered=True),
                is_valid=lambda x: x['fate'] == 'valid barcode'
                ), 
        aes('fate', 'count', fill='is_valid')) +
    geom_bar(stat='identity') +
    facet_wrap('~ sample + library', ncol=ncol) +
    scale_fill_manual(CBPALETTE, guide=False) +
    theme(figure_size=(3.25 * ncol, 2 * math.ceil(nfacets / ncol)),
          axis_text_x=element_text(angle=90),
          panel_grid_major_x=element_blank()
          ) +
    scale_y_continuous(labels=dms_variants.utils.latex_sci_not,
                       name='number of reads')
    )

_ = barcode_fate_plot.draw()
```


    
![png](aggregate_variant_counts_files/aggregate_variant_counts_28_0.png)
    


## Add barcode counts to variant table
Now we use the [CodonVariantTable.add_sample_counts_df](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable.add_sample_counts_df) method to add the barcode counts to the variant table:


```python
variants.add_sample_counts_df(counts)
```

The variant table now has a `variant_count_df` attribute that gives a data frame of all the variant counts.
Here are the first few lines:


```python
display(HTML(variants.variant_count_df.head().to_html(index=False)))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>target</th>
      <th>library</th>
      <th>sample</th>
      <th>barcode</th>
      <th>count</th>
      <th>variant_call_support</th>
      <th>codon_substitutions</th>
      <th>aa_substitutions</th>
      <th>n_codon_substitutions</th>
      <th>n_aa_substitutions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>BM48-31</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>TCCAAGGAAATTGTGA</td>
      <td>150</td>
      <td>3</td>
      <td>BM48-31</td>
      <td>BM48-31</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <td>BM48-31</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>CGATGAATTCAGCTCT</td>
      <td>146</td>
      <td>12</td>
      <td>BM48-31</td>
      <td>BM48-31</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <td>BM48-31</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>CTACCAAGGATGCTAT</td>
      <td>114</td>
      <td>4</td>
      <td>BM48-31</td>
      <td>BM48-31</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <td>BM48-31</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>CCAAGAAAGGCCCCCT</td>
      <td>106</td>
      <td>2</td>
      <td>BM48-31</td>
      <td>BM48-31</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <td>BM48-31</td>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>AGCCCGGCAAAGCTCA</td>
      <td>93</td>
      <td>2</td>
      <td>BM48-31</td>
      <td>BM48-31</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>


Write the variant counts data frame to a CSV file.
It can then be used to re-initialize a [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) via its [from_variant_count_df](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable.from_variant_count_df) method:


```python
print(f"Writing variant counts to {config['variant_counts']}")
variants.variant_count_df.to_csv(config['variant_counts'], index=False, compression='gzip')
```

    Writing variant counts to results/counts/variant_counts.csv.gz


The [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) has lots of nice functions that can be used to analyze the counts it contains.
However, we do that in the next notebook so we don't have to re-run this entire (rather computationally intensive) notebook every time we want to analyze a new aspect of the counts.
