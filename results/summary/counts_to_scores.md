# Analyze counts and compute escape scores
This Python Jupyter notebook analyzes the variant counts and looks at mutation coverage and jackpotting.
It then computes an "escape scores" for each variant after grouping by barcode or substitutions as specified in the configuration.

## Set up analysis

This notebook primarily makes use of the Bloom lab's [dms_variants](https://jbloomlab.github.io/dms_variants) package, and uses [plotnine](https://github.com/has2k1/plotnine) for ggplot2-like plotting syntax:


```python
import collections
import math
import os
import warnings

import Bio.SeqIO

import dms_variants.codonvarianttable
from dms_variants.constants import CBPALETTE
import dms_variants.plotnine_themes

from IPython.display import display, HTML

import matplotlib.pyplot as plt

import numpy

import pandas as pd

from plotnine import *

import seaborn

import yaml
```

Set [plotnine](https://github.com/has2k1/plotnine) theme to the gray-grid one defined in [dms_variants](https://jbloomlab.github.io/dms_variants):


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

Create output directory:


```python
os.makedirs(config['escape_scores_dir'], exist_ok=True)
```

Read information about the samples:


```python
samples_df = pd.read_csv(config['barcode_runs'])
```

## Initialize codon-variant table
Initialize [CodonVariantTable](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html#dms_variants.codonvarianttable.CodonVariantTable) from wildtype gene sequence and the variant counts CSV file.
We will then use the plotting functions of this variant table to analyze the counts per sample:


```python
wt_seqrecord = Bio.SeqIO.read(config['wildtype_sequence'], 'fasta')
geneseq = str(wt_seqrecord.seq)
primary_target = wt_seqrecord.name
print(f"Read sequence of {len(geneseq)} nt for {primary_target} from {config['wildtype_sequence']}")
      
print(f"Initializing CodonVariantTable from gene sequence and {config['variant_counts']}")
      
variants = dms_variants.codonvarianttable.CodonVariantTable.from_variant_count_df(
                geneseq=geneseq,
                variant_count_df_file=config['variant_counts'],
                primary_target=primary_target)
      
print('Done initializing CodonVariantTable.')
```

    Read sequence of 603 nt for SARS-CoV-2 from data/wildtype_sequence.fasta
    Initializing CodonVariantTable from gene sequence and results/counts/variant_counts.csv.gz
    Done initializing CodonVariantTable.


## Sequencing counts per sample
Average counts per variant for each sample.
Note that these are the **sequencing** counts, in some cases they may outstrip the actual number of sorted cells:


```python
p = variants.plotAvgCountsPerVariant(libraries=variants.libraries,
                                     by_target=False,
                                     orientation='v')
p = p + theme(panel_grid_major_x=element_blank())  # no vertical grid lines
_ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_19_0.png)
    


And the numerical values plotted above:


```python
display(HTML(
 variants.avgCountsPerVariant(libraries=variants.libraries,
                               by_target=False)
 .pivot_table(index='sample',
              columns='library',
              values='avg_counts_per_variant')
 .round(1)
 .to_html()
 ))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>library</th>
      <th>lib1</th>
      <th>lib2</th>
    </tr>
    <tr>
      <th>sample</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>expt_68-73-none-0-reference</th>
      <td>282.4</td>
      <td>318.7</td>
    </tr>
    <tr>
      <th>expt_71-S2X259-59-escape</th>
      <td>23.2</td>
      <td>14.3</td>
    </tr>
    <tr>
      <th>expt_130-none-0-reference</th>
      <td>263.7</td>
      <td>271.6</td>
    </tr>
    <tr>
      <th>expt_130-S2K146-63-escape</th>
      <td>22.2</td>
      <td>17.9</td>
    </tr>
    <tr>
      <th>expt_135-139-none-0-reference</th>
      <td>594.8</td>
      <td>561.7</td>
    </tr>
    <tr>
      <th>expt_140-148-none-0-reference</th>
      <td>622.1</td>
      <td>586.3</td>
    </tr>
    <tr>
      <th>expt_135-mAb1-139-escape</th>
      <td>47.4</td>
      <td>40.8</td>
    </tr>
    <tr>
      <th>expt_136-mAb2-232-escape</th>
      <td>31.2</td>
      <td>28.4</td>
    </tr>
    <tr>
      <th>expt_137-mAb3-234-escape</th>
      <td>54.5</td>
      <td>52.2</td>
    </tr>
    <tr>
      <th>expt_138-mAb4-462-escape</th>
      <td>30.3</td>
      <td>29.9</td>
    </tr>
    <tr>
      <th>expt_139-mAb5-1694-escape</th>
      <td>113.3</td>
      <td>115.4</td>
    </tr>
    <tr>
      <th>expt_140-mAb6-584-escape</th>
      <td>16.0</td>
      <td>12.8</td>
    </tr>
    <tr>
      <th>expt_141-mAb7-103-escape</th>
      <td>27.4</td>
      <td>27.5</td>
    </tr>
    <tr>
      <th>expt_142-mAb8-2000-escape</th>
      <td>23.3</td>
      <td>25.3</td>
    </tr>
    <tr>
      <th>expt_143-mAb9-154-escape</th>
      <td>7.6</td>
      <td>4.4</td>
    </tr>
    <tr>
      <th>expt_144-mAb10-147-escape</th>
      <td>8.6</td>
      <td>6.5</td>
    </tr>
    <tr>
      <th>expt_145-mAb11-125-escape</th>
      <td>7.0</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>expt_146-mAb12-267-escape</th>
      <td>8.1</td>
      <td>0.2</td>
    </tr>
    <tr>
      <th>expt_147-mAb13-147-escape</th>
      <td>5.1</td>
      <td>6.2</td>
    </tr>
    <tr>
      <th>expt_148-mAb14-256-escape</th>
      <td>41.3</td>
      <td>23.0</td>
    </tr>
  </tbody>
</table>


## Mutations per variant
Average number of mutations per gene among all variants of the primary target, separately for each date:


```python
# this plotting is very slow when lots of samples, so for now plots are commented out

#for date, date_df in samples_df.groupby('date', sort=False):
#    p = variants.plotNumCodonMutsByType(variant_type='all',
#                                        orientation='v',
#                                        libraries=variants.libraries,
#                                        samples=date_df['sample'].unique().tolist(),
#                                        widthscale=2)
#    p = p + theme(panel_grid_major_x=element_blank())  # no vertical grid lines
#    fig = p.draw()
#    display(fig)
#    plt.close(fig)
```

Now similar plots but showing mutation frequency across the gene:


```python
# this plotting is very slow when lots of samples, so for now code commented out

# for date, date_df in samples_df.groupby('date', sort=False):
#    p = variants.plotMutFreqs(variant_type='all',
#                              mut_type='codon',
#                              orientation='v',
#                              libraries=variants.libraries,
#                              samples=date_df['sample'].unique().tolist(),
#                              widthscale=1.5)
#    fig = p.draw()
#    display(fig)
#    plt.close(fig)
```

## Jackpotting and mutation coverage in pre-selection libraries
We look at the distribution of counts in the "reference" (pre-selection) libraries to see if they seem jackpotted (a few variants at very high frequency):


```python
pre_samples_df = samples_df.query('selection == "reference"')
```

Distribution of mutations along the gene for the pre-selection samples; big spikes may indicate jackpotting:


```python
# this plotting is very slow when lots of samples, so for now code commented out

#p = variants.plotMutFreqs(variant_type='all',
#                          mut_type='codon',
#                          orientation='v',
#                          libraries=variants.libraries,
#                          samples=pre_samples_df['sample'].unique().tolist(),
#                          widthscale=1.5)
#_ = p.draw()
```

How many mutations are observed frequently in pre-selection libraries?
Note that the libraries have been pre-selected for ACE2 binding, so we expect stop variants to mostly be missing.
Make the plot both for all variants and just single-mutant variants:


```python
# this plotting is very slow when lots of samples, so for now code commented out

#for variant_type in ['all', 'single']:
#    p = variants.plotCumulMutCoverage(
#                          variant_type=variant_type,
#                          mut_type='aa',
#                          orientation='v',
#                          libraries=variants.libraries,
#                          samples=pre_samples_df['sample'].unique().tolist(),
#                          widthscale=1.8,
#                          heightscale=1.2)
#    _ = p.draw()
```

Now make a plot showing what number and fraction of counts are for each variant in each pre-selection sample / library.
If some variants constitute a very high fraction, then that indicates extensive bottlenecking:


```python
for ystat in ['frac_counts', 'count']:
    p = variants.plotCountsPerVariant(ystat=ystat,
                                      libraries=variants.libraries,
                                      samples=pre_samples_df['sample'].unique().tolist(),
                                      orientation='v',
                                      widthscale=1.75,
                                      )
    _ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_33_0.png)
    



    
![png](counts_to_scores_files/counts_to_scores_33_1.png)
    


Now make the same plot breaking down by variant class, which enables determination of which types of variants are at high and low frequencies.
For this plot (unlike one above not classified by category) we only show variants of the primary target (not the homologs), and also group synonymous with wildtype in order to reduce number of plotted categories to make more interpretable:


```python
for ystat in ['frac_counts', 'count']:
    p = variants.plotCountsPerVariant(ystat=ystat,
                                      libraries=variants.libraries,
                                      samples=pre_samples_df['sample'].unique().tolist(),
                                      orientation='v',
                                      widthscale=1.75,
                                      by_variant_class=True,
                                      classifyVariants_kwargs={'syn_as_wt': True},
                                      primary_target_only=True,
                                      )
    _ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_35_0.png)
    



    
![png](counts_to_scores_files/counts_to_scores_35_1.png)
    


We also directly look to see what is the variant in each reference library / sample with the highest fraction counts.
Knowing if the highest frequency variant is shared helps determine **where** in the experiment the jackpotting happened:


```python
frac_counts_per_variant = (
        variants.add_frac_counts(variants.variant_count_df)
        .query(f"sample in {pre_samples_df['sample'].unique().tolist()}")
        )

display(HTML(
    frac_counts_per_variant
    .sort_values('frac_counts', ascending=False)
    .groupby(['library', 'sample'])
    .head(n=1)
    .sort_values(['library', 'sample'])
    .set_index(['library', 'sample'])
    [['frac_counts', 'target', 'barcode', 'aa_substitutions', 'codon_substitutions']]
    .round(4)
    .to_html()
    ))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>frac_counts</th>
      <th>target</th>
      <th>barcode</th>
      <th>aa_substitutions</th>
      <th>codon_substitutions</th>
    </tr>
    <tr>
      <th>library</th>
      <th>sample</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4" valign="top">lib1</th>
      <th>expt_68-73-none-0-reference</th>
      <td>0.0012</td>
      <td>SARS-CoV-2</td>
      <td>TTCCAAAATATTGTCA</td>
      <td>D59N F156S</td>
      <td>GAT59AAC TTT156TCG</td>
    </tr>
    <tr>
      <th>expt_130-none-0-reference</th>
      <td>0.0034</td>
      <td>SARS-CoV-2</td>
      <td>TTCCAAAATATTGTCA</td>
      <td>D59N F156S</td>
      <td>GAT59AAC TTT156TCG</td>
    </tr>
    <tr>
      <th>expt_135-139-none-0-reference</th>
      <td>0.0006</td>
      <td>SARS-CoV-2</td>
      <td>TTCCAAAATATTGTCA</td>
      <td>D59N F156S</td>
      <td>GAT59AAC TTT156TCG</td>
    </tr>
    <tr>
      <th>expt_140-148-none-0-reference</th>
      <td>0.0006</td>
      <td>SARS-CoV-2</td>
      <td>TTCCAAAATATTGTCA</td>
      <td>D59N F156S</td>
      <td>GAT59AAC TTT156TCG</td>
    </tr>
    <tr>
      <th rowspan="4" valign="top">lib2</th>
      <th>expt_68-73-none-0-reference</th>
      <td>0.0003</td>
      <td>SARS-CoV-2</td>
      <td>GGCAAGCGTCCAACTA</td>
      <td>F156W Q176A</td>
      <td>GCT105GCC TTT156TGG CAG176GCC</td>
    </tr>
    <tr>
      <th>expt_130-none-0-reference</th>
      <td>0.0008</td>
      <td>SARS-CoV-2</td>
      <td>CAGTACAAAAGTATAA</td>
      <td>K87N V173Y</td>
      <td>CTA60CTG AAG87AAC GTC173TAC</td>
    </tr>
    <tr>
      <th>expt_135-139-none-0-reference</th>
      <td>0.0003</td>
      <td>SARS-CoV-2</td>
      <td>CAATATTTCCGCAATT</td>
      <td>K56E</td>
      <td>AAG56GAG</td>
    </tr>
    <tr>
      <th>expt_140-148-none-0-reference</th>
      <td>0.0003</td>
      <td>SARS-CoV-2</td>
      <td>CAATATTTCCGCAATT</td>
      <td>K56E</td>
      <td>AAG56GAG</td>
    </tr>
  </tbody>
</table>


To further where the jackpotting relative to the generation of the reference samples, we plot the correlation among the fraction of counts for the different reference samples.
If the fractions are highly correlated, that indicates that the jackpotting occurred in some upstream step common to the reference samples:


```python
# this code makes a full matrix of scatter plots, but is REALLY SLOW. So for now,
# it is commented out in favor of code that just makes correlation matrix.
#for lib, lib_df in frac_counts_per_variant.groupby('library'):
#    wide_lib_df = lib_df.pivot_table(index=['target', 'barcode'],
#                                     columns='sample',
#                                     values='frac_counts')
#    g = seaborn.pairplot(wide_lib_df, corner=True, plot_kws={'alpha': 0.5}, diag_kind='kde')
#    _ = g.fig.suptitle(lib, size=18)
#    plt.show()
```

## Examine counts for wildtype variants
The type of score we use to quantify escape depends on how well represented wildtype is in the selected libraries.
If wildtype is still well represented, we can use a more conventional functional score that gives differential selection relative to wildtype.
If wildtype is not well represented, then we need an alternative score that does not involve normalizing frequencies to wildtype.

First get average fraction of counts per variant for each variant class:


```python
counts_by_class = (
    variants.variant_count_df
    .pipe(variants.add_frac_counts)
    .pipe(variants.classifyVariants,
          primary_target=variants.primary_target,
          non_primary_target_class='homolog',
          class_as_categorical=True)
    .groupby(['library', 'sample', 'variant_class'])
    .aggregate(avg_frac_counts=pd.NamedAgg('frac_counts', 'mean'))
    .reset_index()
    .merge(samples_df[['sample', 'library', 'date', 'antibody', 'concentration', 'selection']],
           on=['sample', 'library'], validate='many_to_one')
    )

display(HTML(counts_by_class.head().to_html(index=False)))
```


<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>library</th>
      <th>sample</th>
      <th>variant_class</th>
      <th>avg_frac_counts</th>
      <th>date</th>
      <th>antibody</th>
      <th>concentration</th>
      <th>selection</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>wildtype</td>
      <td>0.000026</td>
      <td>201106</td>
      <td>none</td>
      <td>0</td>
      <td>reference</td>
    </tr>
    <tr>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>synonymous</td>
      <td>0.000025</td>
      <td>201106</td>
      <td>none</td>
      <td>0</td>
      <td>reference</td>
    </tr>
    <tr>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>1 nonsynonymous</td>
      <td>0.000020</td>
      <td>201106</td>
      <td>none</td>
      <td>0</td>
      <td>reference</td>
    </tr>
    <tr>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>&gt;1 nonsynonymous</td>
      <td>0.000008</td>
      <td>201106</td>
      <td>none</td>
      <td>0</td>
      <td>reference</td>
    </tr>
    <tr>
      <td>lib1</td>
      <td>expt_68-73-none-0-reference</td>
      <td>stop</td>
      <td>0.000002</td>
      <td>201106</td>
      <td>none</td>
      <td>0</td>
      <td>reference</td>
    </tr>
  </tbody>
</table>


Plot average fraction of all counts per variant for each variant class.
If the values for wildtype are low for the non-reference samples (such as more similar to stop the nonsynonymous), then normalizing by wildtype in calculating scores will probably not work well as wildtype is too depleted:


```python
min_frac = 1e-7  # plot values < this as this

p = (ggplot(counts_by_class
            .assign(avg_frac_counts=lambda x: numpy.clip(x['avg_frac_counts'], min_frac, None))
            ) +
     aes('avg_frac_counts', 'sample', color='selection') +
     geom_point(size=2) +
     scale_color_manual(values=CBPALETTE[1:]) +
     facet_grid('library ~ variant_class') +
     scale_x_log10() +
     theme(axis_text_x=element_text(angle=90),
           figure_size=(2.5 * counts_by_class['variant_class'].nunique(),
                        0.2 * counts_by_class['library'].nunique() * 
                        counts_by_class['sample'].nunique())
           ) +
     geom_vline(xintercept=min_frac, linetype='dotted', color=CBPALETTE[3])
     )

_ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_43_0.png)
    


## Compute escape scores
We use the escape score metric, which does **not** involve normalizing to wildtype and so isn't strongly affected by low wildtype counts.
We compute the scores using the method [dms_variants.codonvarianttable.CodonVariantTable.escape_scores](https://jbloomlab.github.io/dms_variants/dms_variants.codonvarianttable.html?highlight=escape_scores#dms_variants.codonvarianttable.CodonVariantTable.escape_scores).

First, define what samples to compare for each calculation, matching each post-selection (escape) to the pre-selection (reference) sample on the same date:


```python
score_sample_df = (
    samples_df
    .query('selection == "escape"')
    .rename(columns={'sample': 'post_sample',
                     'cells_sorted': 'pre_cells_sorted'})
    .merge(samples_df
           .query('selection == "reference"')
           [['sample', 'library', 'date', 'cells_sorted']]
           .rename(columns={'sample': 'pre_sample',
                            'cells_sorted': 'post_cells_sorted'}),
           how='left', on=['date', 'library'], validate='many_to_one',
           )
    .assign(name=lambda x: x['antibody'] + '_' + x['concentration'].astype(str))
    # add dates to names where needed to make unique
    .assign(n_libs=lambda x: x.groupby(['name', 'date'])['pre_sample'].transform('count'))
    .sort_values(['name', 'date', 'n_libs'], ascending=False)
    .assign(i_name=lambda x: x.groupby(['library', 'name'], sort=False)['pre_sample'].cumcount(),
            name=lambda x: x.apply(lambda r: r['name'] + '_' + str(r['date']) if r['i_name'] > 0 else r['name'],
                                   axis=1),
            )
    .sort_values(['antibody', 'concentration', 'library', 'i_name'])
    # get columns of interest
    [['name', 'library', 'antibody', 'concentration', 'concentration_units', 'date',
      'pre_sample', 'post_sample', 'frac_escape', 'pre_cells_sorted', 'post_cells_sorted']]
    )

assert len(score_sample_df.groupby(['name', 'library'])) == len(score_sample_df)

print(f"Writing samples used to compute escape scores to {config['escape_score_samples']}\n")
score_sample_df.to_csv(config['escape_score_samples'], index=False)

display(HTML(score_sample_df.to_html(index=False)))
```

    Writing samples used to compute escape scores to results/escape_scores/samples.csv
    



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>name</th>
      <th>library</th>
      <th>antibody</th>
      <th>concentration</th>
      <th>concentration_units</th>
      <th>date</th>
      <th>pre_sample</th>
      <th>post_sample</th>
      <th>frac_escape</th>
      <th>pre_cells_sorted</th>
      <th>post_cells_sorted</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>S2K146_63</td>
      <td>lib1</td>
      <td>S2K146</td>
      <td>63</td>
      <td>ng_per_mL</td>
      <td>210526</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>0.071</td>
      <td>816032.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>lib2</td>
      <td>S2K146</td>
      <td>63</td>
      <td>ng_per_mL</td>
      <td>210526</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>0.063</td>
      <td>698156.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>S2X259_59</td>
      <td>lib1</td>
      <td>S2X259</td>
      <td>59</td>
      <td>ng_per_mL</td>
      <td>201106</td>
      <td>expt_68-73-none-0-reference</td>
      <td>expt_71-S2X259-59-escape</td>
      <td>0.068</td>
      <td>700753.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>S2X259_59</td>
      <td>lib2</td>
      <td>S2X259</td>
      <td>59</td>
      <td>ng_per_mL</td>
      <td>201106</td>
      <td>expt_68-73-none-0-reference</td>
      <td>expt_71-S2X259-59-escape</td>
      <td>0.064</td>
      <td>600000.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb1_139</td>
      <td>lib1</td>
      <td>mAb1</td>
      <td>139</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_135-mAb1-139-escape</td>
      <td>0.077</td>
      <td>842165.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb1_139</td>
      <td>lib2</td>
      <td>mAb1</td>
      <td>139</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_135-mAb1-139-escape</td>
      <td>0.079</td>
      <td>851124.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb10_147</td>
      <td>lib1</td>
      <td>mAb10</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_144-mAb10-147-escape</td>
      <td>0.046</td>
      <td>467946.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb10_147</td>
      <td>lib2</td>
      <td>mAb10</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_144-mAb10-147-escape</td>
      <td>0.044</td>
      <td>441191.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb11_125</td>
      <td>lib1</td>
      <td>mAb11</td>
      <td>125</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_145-mAb11-125-escape</td>
      <td>0.054</td>
      <td>538149.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb11_125</td>
      <td>lib2</td>
      <td>mAb11</td>
      <td>125</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_145-mAb11-125-escape</td>
      <td>0.044</td>
      <td>446148.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb12_267</td>
      <td>lib1</td>
      <td>mAb12</td>
      <td>267</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_146-mAb12-267-escape</td>
      <td>0.053</td>
      <td>536557.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb12_267</td>
      <td>lib2</td>
      <td>mAb12</td>
      <td>267</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_146-mAb12-267-escape</td>
      <td>0.053</td>
      <td>532129.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb13_147</td>
      <td>lib1</td>
      <td>mAb13</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_147-mAb13-147-escape</td>
      <td>0.043</td>
      <td>430775.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb13_147</td>
      <td>lib2</td>
      <td>mAb13</td>
      <td>147</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_147-mAb13-147-escape</td>
      <td>0.047</td>
      <td>472865.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb14_256</td>
      <td>lib1</td>
      <td>mAb14</td>
      <td>256</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_148-mAb14-256-escape</td>
      <td>0.137</td>
      <td>1384538.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb14_256</td>
      <td>lib2</td>
      <td>mAb14</td>
      <td>256</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_148-mAb14-256-escape</td>
      <td>0.148</td>
      <td>1506071.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb2_232</td>
      <td>lib1</td>
      <td>mAb2</td>
      <td>232</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_136-mAb2-232-escape</td>
      <td>0.057</td>
      <td>617689.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb2_232</td>
      <td>lib2</td>
      <td>mAb2</td>
      <td>232</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_136-mAb2-232-escape</td>
      <td>0.055</td>
      <td>614856.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb3_234</td>
      <td>lib1</td>
      <td>mAb3</td>
      <td>234</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_137-mAb3-234-escape</td>
      <td>0.110</td>
      <td>1190414.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb3_234</td>
      <td>lib2</td>
      <td>mAb3</td>
      <td>234</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_137-mAb3-234-escape</td>
      <td>0.118</td>
      <td>1253385.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb4_462</td>
      <td>lib1</td>
      <td>mAb4</td>
      <td>462</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_138-mAb4-462-escape</td>
      <td>0.057</td>
      <td>604056.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb4_462</td>
      <td>lib2</td>
      <td>mAb4</td>
      <td>462</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_138-mAb4-462-escape</td>
      <td>0.059</td>
      <td>622698.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb5_1694</td>
      <td>lib1</td>
      <td>mAb5</td>
      <td>1694</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_139-mAb5-1694-escape</td>
      <td>0.206</td>
      <td>2222036.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb5_1694</td>
      <td>lib2</td>
      <td>mAb5</td>
      <td>1694</td>
      <td>ng_per_mL</td>
      <td>211105</td>
      <td>expt_135-139-none-0-reference</td>
      <td>expt_139-mAb5-1694-escape</td>
      <td>0.214</td>
      <td>2245393.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb6_584</td>
      <td>lib1</td>
      <td>mAb6</td>
      <td>584</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_140-mAb6-584-escape</td>
      <td>0.100</td>
      <td>1002651.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb6_584</td>
      <td>lib2</td>
      <td>mAb6</td>
      <td>584</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_140-mAb6-584-escape</td>
      <td>0.101</td>
      <td>1011280.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb7_103</td>
      <td>lib1</td>
      <td>mAb7</td>
      <td>103</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_141-mAb7-103-escape</td>
      <td>0.171</td>
      <td>1959806.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb7_103</td>
      <td>lib2</td>
      <td>mAb7</td>
      <td>103</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_141-mAb7-103-escape</td>
      <td>0.180</td>
      <td>1799047.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb8_2000</td>
      <td>lib1</td>
      <td>mAb8</td>
      <td>2000</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_142-mAb8-2000-escape</td>
      <td>0.142</td>
      <td>1421301.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb8_2000</td>
      <td>lib2</td>
      <td>mAb8</td>
      <td>2000</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_142-mAb8-2000-escape</td>
      <td>0.136</td>
      <td>1374728.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb9_154</td>
      <td>lib1</td>
      <td>mAb9</td>
      <td>154</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_143-mAb9-154-escape</td>
      <td>0.038</td>
      <td>429649.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>mAb9_154</td>
      <td>lib2</td>
      <td>mAb9</td>
      <td>154</td>
      <td>ng_per_mL</td>
      <td>211108</td>
      <td>expt_140-148-none-0-reference</td>
      <td>expt_143-mAb9-154-escape</td>
      <td>0.040</td>
      <td>400943.0</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>


Compute the escape scores for variants of the primary target and classify the variants, then compute escape scores for homologs:


```python
print(f"Computing escape scores for {primary_target} variants using {config['escape_score_type']} "
      f"score type with a pseudocount of {config['escape_score_pseudocount']} and "
      f"an escape fraction floor {config['escape_score_floor_E']}, an escape fraction ceiling "
      f"{config['escape_score_ceil_E']}, and grouping variants by {config['escape_score_group_by']}.")

escape_scores = (variants.escape_scores(score_sample_df,
                                        score_type=config['escape_score_type'],
                                        pseudocount=config['escape_score_pseudocount'],
                                        floor_E=config['escape_score_floor_E'],
                                        ceil_E=config['escape_score_ceil_E'],
                                        by=config['escape_score_group_by'],
                                        )
                 .query('target == @primary_target')
                 .pipe(variants.classifyVariants,
                       primary_target=variants.primary_target,
                       syn_as_wt=(config['escape_score_group_by'] == 'aa_substitutions'),
                       )
                 )
print('Here are the first few lines of the resulting escape scores:')
display(HTML(escape_scores.head().to_html(index=False)))

print(f"\nComputing scores for homologs grouping by {config['escape_score_homolog_group_by']}")

escape_scores_homologs = (
        variants.escape_scores(score_sample_df,
                               score_type=config['escape_score_type'],
                               pseudocount=config['escape_score_pseudocount'],
                               floor_E=config['escape_score_floor_E'],
                               ceil_E=config['escape_score_ceil_E'],
                               by=config['escape_score_homolog_group_by'],
                               )
        .query('(target != @primary_target) | (n_aa_substitutions == 0)')
        )

print('Here are the first few lines of the resulting homolog escape scores:')
display(HTML(escape_scores_homologs.head().to_html(index=False)))
```

    Computing escape scores for SARS-CoV-2 variants using frac_escape score type with a pseudocount of 0.5 and an escape fraction floor 0, an escape fraction ceiling 1, and grouping variants by barcode.
    Here are the first few lines of the resulting escape scores:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>name</th>
      <th>target</th>
      <th>library</th>
      <th>pre_sample</th>
      <th>post_sample</th>
      <th>barcode</th>
      <th>score</th>
      <th>score_var</th>
      <th>pre_count</th>
      <th>post_count</th>
      <th>codon_substitutions</th>
      <th>n_codon_substitutions</th>
      <th>aa_substitutions</th>
      <th>n_aa_substitutions</th>
      <th>variant_class</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>S2K146_63</td>
      <td>SARS-CoV-2</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>TTCCAAAATATTGTCA</td>
      <td>0.024382</td>
      <td>2.301607e-07</td>
      <td>89848</td>
      <td>2652</td>
      <td>GAT59AAC TTT156TCG</td>
      <td>2</td>
      <td>D59N F156S</td>
      <td>2</td>
      <td>&gt;1 nonsynonymous</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>SARS-CoV-2</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>TAGTAACAATGCGGTA</td>
      <td>0.007342</td>
      <td>2.658893e-07</td>
      <td>23003</td>
      <td>204</td>
      <td></td>
      <td>0</td>
      <td></td>
      <td>0</td>
      <td>wildtype</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>SARS-CoV-2</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>ATAAAAAGTCCATATG</td>
      <td>0.008466</td>
      <td>4.032818e-07</td>
      <td>17511</td>
      <td>179</td>
      <td>CTT5AAG</td>
      <td>1</td>
      <td>L5K</td>
      <td>1</td>
      <td>1 nonsynonymous</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>SARS-CoV-2</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>TTAATTAGTATCAGGT</td>
      <td>0.010568</td>
      <td>5.175740e-07</td>
      <td>17075</td>
      <td>218</td>
      <td>GAC34CGC CTG125GTC</td>
      <td>2</td>
      <td>D34R L125V</td>
      <td>2</td>
      <td>&gt;1 nonsynonymous</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>SARS-CoV-2</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>TTAGATGAAGCCAGTA</td>
      <td>0.215394</td>
      <td>1.639754e-05</td>
      <td>13640</td>
      <td>3557</td>
      <td>GAC90TAC GAC98TTC GGG174GAG</td>
      <td>3</td>
      <td>D90Y D98F G174E</td>
      <td>3</td>
      <td>&gt;1 nonsynonymous</td>
    </tr>
  </tbody>
</table>


    
    Computing scores for homologs grouping by aa_substitutions
    Here are the first few lines of the resulting homolog escape scores:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>name</th>
      <th>target</th>
      <th>library</th>
      <th>pre_sample</th>
      <th>post_sample</th>
      <th>aa_substitutions</th>
      <th>score</th>
      <th>score_var</th>
      <th>pre_count</th>
      <th>post_count</th>
      <th>n_aa_substitutions</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>S2K146_63</td>
      <td>BM48-31</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>BM48-31</td>
      <td>1.000000</td>
      <td>0.000000e+00</td>
      <td>2888</td>
      <td>3712</td>
      <td>0</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>GD-Pangolin</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>GD-Pangolin</td>
      <td>0.000998</td>
      <td>1.302323e-08</td>
      <td>63644</td>
      <td>76</td>
      <td>0</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>HKU3-1</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>HKU3-1</td>
      <td>1.000000</td>
      <td>0.000000e+00</td>
      <td>3462</td>
      <td>4856</td>
      <td>0</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>LYRa11</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>LYRa11</td>
      <td>0.138134</td>
      <td>8.073441e-06</td>
      <td>16525</td>
      <td>2750</td>
      <td>0</td>
    </tr>
    <tr>
      <td>S2K146_63</td>
      <td>RaTG13</td>
      <td>lib1</td>
      <td>expt_130-none-0-reference</td>
      <td>expt_130-S2K146-63-escape</td>
      <td>RaTG13</td>
      <td>0.001613</td>
      <td>2.403815e-08</td>
      <td>55809</td>
      <td>108</td>
      <td>0</td>
    </tr>
  </tbody>
</table>


## Apply pre-selection count filter to variant escape scores
Now determine a pre-selection count filter in order to flag for removal variants with counts that are so low that the estimated score is probably noise.
We know that stop codons should be largely purged pre-selection, and so the counts for them are a good indication of the "noise" threshold.
We therefore set the filter using the number of pre-selection counts for the stop codons.

To do this, we first compute the number of pre-selection counts for stop-codon variants at various quantiles and look at these.
We then take the number of pre-selection counts at the specified quantile as the filter cutoff, and filter scores for all variants with pre-selection counts less than this filter cutoff:


```python
filter_quantile = config['escape_score_stop_quantile_filter']
assert 0 <= filter_quantile <= 1

quantiles = sorted(set([0.5, 0.9, 0.95, 0.98, 0.99, 0.995, 0.999] + [filter_quantile]))

stop_score_counts = (
    escape_scores
    .query('variant_class == "stop"')
    .groupby(['library', 'pre_sample'], observed=True)
    ['pre_count']
    .quantile(q=quantiles)
    .reset_index()
    .rename(columns={'level_2': 'quantile'})
    .pivot_table(index=['pre_sample', 'library'],
                 columns='quantile',
                 values='pre_count')
    )

print('Quantiles of the number of pre-selection counts per variant for stop variants:')
display(HTML(stop_score_counts.to_html(float_format='%.1f')))

print(f"\nSetting the pre-count filter cutoff to the {filter_quantile} quantile:")
pre_count_filter_cutoffs = (
    stop_score_counts
    [filter_quantile]
    .rename('pre_count_filter_cutoff')
    .reset_index()
    )
display(HTML(pre_count_filter_cutoffs.to_html(float_format='%.1f')))
```

    Quantiles of the number of pre-selection counts per variant for stop variants:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>quantile</th>
      <th>0.5</th>
      <th>0.9</th>
      <th>0.95</th>
      <th>0.98</th>
      <th>0.99</th>
      <th>0.995</th>
      <th>0.999</th>
    </tr>
    <tr>
      <th>pre_sample</th>
      <th>library</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">expt_68-73-none-0-reference</th>
      <th>lib1</th>
      <td>31.0</td>
      <td>100.0</td>
      <td>128.0</td>
      <td>165.0</td>
      <td>190.4</td>
      <td>216.0</td>
      <td>300.6</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>32.0</td>
      <td>113.0</td>
      <td>146.0</td>
      <td>192.0</td>
      <td>226.4</td>
      <td>258.6</td>
      <td>392.7</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_130-none-0-reference</th>
      <th>lib1</th>
      <td>30.0</td>
      <td>99.0</td>
      <td>126.0</td>
      <td>162.9</td>
      <td>199.0</td>
      <td>236.2</td>
      <td>407.1</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>29.0</td>
      <td>102.0</td>
      <td>135.0</td>
      <td>177.7</td>
      <td>209.0</td>
      <td>247.2</td>
      <td>479.8</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_135-139-none-0-reference</th>
      <th>lib1</th>
      <td>62.0</td>
      <td>195.0</td>
      <td>254.0</td>
      <td>326.0</td>
      <td>383.0</td>
      <td>454.0</td>
      <td>559.0</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>54.0</td>
      <td>185.0</td>
      <td>244.0</td>
      <td>320.0</td>
      <td>382.0</td>
      <td>429.9</td>
      <td>622.0</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">expt_140-148-none-0-reference</th>
      <th>lib1</th>
      <td>66.0</td>
      <td>209.0</td>
      <td>265.0</td>
      <td>348.0</td>
      <td>402.0</td>
      <td>469.0</td>
      <td>641.0</td>
    </tr>
    <tr>
      <th>lib2</th>
      <td>60.0</td>
      <td>202.0</td>
      <td>260.0</td>
      <td>346.0</td>
      <td>412.0</td>
      <td>459.0</td>
      <td>681.0</td>
    </tr>
  </tbody>
</table>


    
    Setting the pre-count filter cutoff to the 0.99 quantile:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>pre_sample</th>
      <th>library</th>
      <th>pre_count_filter_cutoff</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>expt_68-73-none-0-reference</td>
      <td>lib1</td>
      <td>190.4</td>
    </tr>
    <tr>
      <th>1</th>
      <td>expt_68-73-none-0-reference</td>
      <td>lib2</td>
      <td>226.4</td>
    </tr>
    <tr>
      <th>2</th>
      <td>expt_130-none-0-reference</td>
      <td>lib1</td>
      <td>199.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>expt_130-none-0-reference</td>
      <td>lib2</td>
      <td>209.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>expt_135-139-none-0-reference</td>
      <td>lib1</td>
      <td>383.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>expt_135-139-none-0-reference</td>
      <td>lib2</td>
      <td>382.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>expt_140-148-none-0-reference</td>
      <td>lib1</td>
      <td>402.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>expt_140-148-none-0-reference</td>
      <td>lib2</td>
      <td>412.0</td>
    </tr>
  </tbody>
</table>


Apply the filter to the escape scores, so that scores that fail the pre-selection count filter are now marked with `pass_pre_count_filter` of `False`:


```python
escape_scores = (
    escape_scores
    .merge(pre_count_filter_cutoffs,
           on=['library', 'pre_sample'],
           how='left',
           validate='many_to_one')
    .assign(pass_pre_count_filter=lambda x: x['pre_count'] >= x['pre_count_filter_cutoff'])
    )

escape_scores_homologs = (
    escape_scores_homologs
    .merge(pre_count_filter_cutoffs,
           on=['library', 'pre_sample'],
           how='left',
           validate='many_to_one')
    .assign(pass_pre_count_filter=lambda x: x['pre_count'] >= x['pre_count_filter_cutoff'])
    )
```

Plot the fraction of variants of each type that pass the pre-selection count filter in each pre-selection sample.
The ideal filter would have the property such that no *stop* variants pass, all *wildtype* (or *synonymous*) variants pass, and some intermediate fraction of *nonsynonymous* variants pass.
However, if the variant composition in the pre-selection samples is already heavily skewed by jackpotting, there will be some deviation from this ideal behavior.
Here is what the plots actually look like:


```python
frac_pre_pass_filter = (
    escape_scores
    [['pre_sample', 'library', 'target', config['escape_score_group_by'],
      'pre_count', 'pass_pre_count_filter', 'variant_class']]
    .drop_duplicates()
    .groupby(['pre_sample', 'library', 'variant_class'], observed=True)
    .aggregate(n_variants=pd.NamedAgg('pass_pre_count_filter', 'count'),
               n_pass_filter=pd.NamedAgg('pass_pre_count_filter', 'sum')
               )
    .reset_index()
    .assign(frac_pass_filter=lambda x: x['n_pass_filter'] / x['n_variants'],
            pre_sample=lambda x: pd.Categorical(x['pre_sample'], x['pre_sample'].unique(), ordered=True))
    )

p = (ggplot(frac_pre_pass_filter) +
     aes('variant_class', 'frac_pass_filter', fill='variant_class') +
     geom_bar(stat='identity') +
     facet_grid('library ~ pre_sample') +
     theme(axis_text_x=element_text(angle=90),
           figure_size=(3.3 * frac_pre_pass_filter['pre_sample'].nunique(),
                        2 * frac_pre_pass_filter['library'].nunique()),
           panel_grid_major_x=element_blank(),
           ) +
     scale_fill_manual(values=CBPALETTE[1:]) +
     expand_limits(y=(0, 1))
     )

_ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_53_0.png)
    


## Apply ACE2-binding / expression filter to variant mutations
In [Starr et al (2020)](https://www.biorxiv.org/content/10.1101/2020.06.17.157982v1), we used deep mutational scanning to estimate how each mutation affected ACE2 binding and expression.
Here we flag for removal any variants of the primary target that have (or have mutations) that were measured to decrease ACE2-binding or expression beyond a minimal threshold, in order to avoid these variants muddying the signal as spurious escape mutants.

To do this, we first determine all mutations that do / do-not having binding that exceeds the thresholds.
We then flag variants as passing the ACE2-binding / expression filter if all of the mutations they contain exceed both thresholds, and failing as otherwise.
In addition, we look at the measured ACE2-binding / expression for each variant in each library, and flag as passing the filter any variants that have binding / expression that exeed the thresholds, and failing as otherwise.
If we are grouping by amino-acid substitutions, we use the average value for variants with those substitutions.

Note that these filters are only applied to mutants of the primary target; homologs are specified manually for filtering in the configuration file:


```python
print(f"Reading ACE2-binding and expression for mutations from {config['mut_bind_expr']}, "
      f"and for variants from {config['variant_bind']} and {config['variant_expr']}, "
      f"and filtering for variants with binding >={config['escape_score_min_bind_variant']}."
      f"and expression >= {config['escape_score_min_expr_variant']}, and also variants that "
      f"only have mutations with binding >={config['escape_score_min_bind_mut']} and "
      f"expression >={config['escape_score_min_expr_mut']}.")

# filter on mutations
mut_bind_expr = pd.read_csv(config['mut_bind_expr'])
assert mut_bind_expr['mutation_RBD'].nunique() == len(mut_bind_expr)
for prop in ['bind', 'expr']:
    muts_adequate = set(mut_bind_expr
                        .query(f"{prop}_avg >= {config[f'escape_score_min_{prop}_mut']}")
                        ['mutation_RBD']
                        )
    print(f"{len(muts_adequate)} of {len(mut_bind_expr)} mutations have adequate {prop}.")
    escape_scores[f"muts_pass_{prop}_filter"] = (
        escape_scores
        ['aa_substitutions']
        .map(lambda s: set(s.split()).issubset(muts_adequate))
        ) 

# filter on variants
for prop, col in [('bind', 'delta_log10Ka'), ('expr', 'delta_ML_meanF')]:
    filter_name = f"variant_pass_{prop}_filter"
    variant_pass_df = (
        pd.read_csv(config[f"variant_{prop}"], keep_default_na=False, na_values=['NA'])
        .groupby(['library', 'target', config['escape_score_group_by']])
        .aggregate(val=pd.NamedAgg(col, 'mean'))
        .reset_index()
        .assign(pass_filter=lambda x: x['val'] >= config[f"escape_score_min_{prop}_variant"])
        .rename(columns={'pass_filter': filter_name,
                         'val': f"variant_{prop}"})
        )
    print(f"\nTotal variants of {primary_target} that pass {prop} filter:")
    display(HTML(
        variant_pass_df
        .groupby(['library', filter_name])
        .aggregate(n_variants=pd.NamedAgg(config['escape_score_group_by'], 'count'))
        .to_html()
        ))
    escape_scores = (
        escape_scores
        .drop(columns=filter_name, errors='ignore')
        .merge(variant_pass_df,
               how='left',
               validate='many_to_one',
               on=['library', 'target', config['escape_score_group_by']],
               )
        )
    assert escape_scores[filter_name].notnull().all()

# annotate as passing overall filter if passes all mutation and binding filters:
escape_scores['pass_ACE2bind_expr_filter'] = (
        escape_scores['muts_pass_bind_filter'] &
        escape_scores['muts_pass_expr_filter'] &
        escape_scores['variant_pass_bind_filter'] &
        escape_scores['variant_pass_expr_filter']
        )
```

    Reading ACE2-binding and expression for mutations from results/prior_DMS_data/mutant_ACE2binding_expression.csv, and for variants from results/prior_DMS_data/variant_ACE2binding.csv and results/prior_DMS_data/variant_expression.csv, and filtering for variants with binding >=-2.35.and expression >= -1.0, and also variants that only have mutations with binding >=-2.35 and expression >=-1.0.
    3422 of 4221 mutations have adequate bind.
    2328 of 4221 mutations have adequate expr.
    
    Total variants of SARS-CoV-2 that pass bind filter:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>n_variants</th>
    </tr>
    <tr>
      <th>library</th>
      <th>variant_pass_bind_filter</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">lib1</th>
      <th>False</th>
      <td>56963</td>
    </tr>
    <tr>
      <th>True</th>
      <td>41606</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">lib2</th>
      <th>False</th>
      <td>55964</td>
    </tr>
    <tr>
      <th>True</th>
      <td>40548</td>
    </tr>
  </tbody>
</table>


    
    Total variants of SARS-CoV-2 that pass expr filter:



<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>n_variants</th>
    </tr>
    <tr>
      <th>library</th>
      <th>variant_pass_expr_filter</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">lib1</th>
      <th>False</th>
      <td>75269</td>
    </tr>
    <tr>
      <th>True</th>
      <td>23300</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">lib2</th>
      <th>False</th>
      <td>74439</td>
    </tr>
    <tr>
      <th>True</th>
      <td>22073</td>
    </tr>
  </tbody>
</table>


Plot the fraction of variants that **have already passed the pre-count filter** that are filtered by the ACE2-binding or expression thresholds:


```python
frac_ACE2bind_expr_pass_filter = (
    escape_scores
    .query('pass_pre_count_filter == True')
    [['pre_sample', 'library', 'target', config['escape_score_group_by'],
      'pre_count', 'pass_ACE2bind_expr_filter', 'variant_class']]
    .drop_duplicates()
    .groupby(['pre_sample', 'library', 'variant_class'], observed=True)
    .aggregate(n_variants=pd.NamedAgg('pass_ACE2bind_expr_filter', 'count'),
               n_pass_filter=pd.NamedAgg('pass_ACE2bind_expr_filter', 'sum')
               )
    .reset_index()
    .assign(frac_pass_filter=lambda x: x['n_pass_filter'] / x['n_variants'],
            pre_sample=lambda x: pd.Categorical(x['pre_sample'], x['pre_sample'].unique(), ordered=True))
    )

p = (ggplot(frac_ACE2bind_expr_pass_filter) +
     aes('variant_class', 'frac_pass_filter', fill='variant_class') +
     geom_bar(stat='identity') +
     facet_grid('library ~ pre_sample') +
     theme(axis_text_x=element_text(angle=90),
           figure_size=(3.3 * frac_ACE2bind_expr_pass_filter['pre_sample'].nunique(),
                        2 * frac_ACE2bind_expr_pass_filter['library'].nunique()),
           panel_grid_major_x=element_blank(),
           ) +
     scale_fill_manual(values=CBPALETTE[1:]) +
     expand_limits(y=(0, 1))
     )

_ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_57_0.png)
    


## Examine and write escape scores
Plot the distribution of escape scores across variants of different classes **among those that pass both the pre-selection count filter and the ACE2-binding / expression filter**.
If things are working correctly, we don't expect escape in wildtype (or synonymous variants), but do expect escape for some small fraction of nonsynymous variants.
Also, we do not plot the scores for the stop codon variant class, as most stop-codon variants have already been filtered out so this category is largely noise:


```python
nfacets = len(escape_scores.groupby(['library', 'name']).nunique())
ncol = min(8, nfacets)
nrow = math.ceil(nfacets / ncol)

df = (escape_scores
      .query('(pass_pre_count_filter == True) & (pass_ACE2bind_expr_filter == True)')
      .query('variant_class != "stop"')
      )
     
p = (ggplot(df) +
     aes('variant_class', 'score', color='variant_class') +
     geom_boxplot(outlier_size=1.5, outlier_alpha=0.1) +
     facet_wrap('~ name + library', ncol=ncol) +
     theme(axis_text_x=element_text(angle=90),
           figure_size=(2.35 * ncol, 3 * nrow),
           panel_grid_major_x=element_blank(),
           ) +
     scale_fill_manual(values=CBPALETTE[1:]) +
     scale_color_manual(values=CBPALETTE[1:])
     )

_ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_59_0.png)
    


Also, we want to see how much the high escape scores are correlated with simple coverage.
To do this, we plot the correlation between escape score and pre-selection count just for the nonsynonymous variants (which are the ones that we expect to have true escape).
The plots below have a lot of overplotting, but are still sufficient to test of the score is simply correlated with the pre-selection counts or not.
The hoped for result is that the escape score doesn't appear to be strongly correlated with pre-selection counts:


```python
p = (ggplot(escape_scores
            .query('(pass_pre_count_filter == True) & (pass_ACE2bind_expr_filter == True)')
            .query('variant_class in ["1 nonsynonymous", ">1 nonsynonymous"]')
            ) +
     aes('pre_count', 'score') +
     geom_point(alpha=0.1, size=1) +
     facet_wrap('~ name + library', ncol=ncol) +
     theme(axis_text_x=element_text(angle=90),
           figure_size=(2.35 * ncol, 2.35 * nrow),
           ) +
     scale_fill_manual(values=CBPALETTE[1:]) +
     scale_color_manual(values=CBPALETTE[1:]) +
     scale_x_log10()
     )

_ = p.draw()
```


    
![png](counts_to_scores_files/counts_to_scores_61_0.png)
    


Write the escape scores to a file:


```python
print(f"Writing escape scores for {primary_target} to {config['escape_scores']}")
escape_scores.to_csv(config['escape_scores'], index=False, float_format='%.4g', compression='gzip')

print(f"Writing escape scores for homologs to {config['escape_scores_homologs']}")
escape_scores_homologs.to_csv(config['escape_scores_homologs'], index=False, float_format='%.4g')
```

    Writing escape scores for SARS-CoV-2 to results/escape_scores/scores.csv.gz
    Writing escape scores for homologs to results/escape_scores/scores_homologs.csv

