# Streamlining Data Access Part 2: Generating Synthetic Data in Python for Preliminary Analysis

<br />

A continuation from the first part of steamlining data access, in this part I'll be discussing using a technique called synthetic data where we can generate fake data that follows similar properties of the original data. Ideally the goal of this document is to educate the basics of synthetic data and provide usable coding that the reader can use to get started on doing surface level analysis without real data.

<br /> 

## What is Synthetic Data?
Synthetic data is artificially generated to mimic real-world data. It is based on statistical analysis of real data. They can be useful for augmenting small real datasets for training machine learning models, running simulations when real data is not available or very minimal, testing systems and models without exposing sensitive real data, sharing datasets while preserving privacy. This is widely used within Industries that have a lot of sensitive personal data like the financial centers and healthcare industry. <br>

The key advantage of synthetic data is it looks and behaves like real data but does not contain private information. It also lets you generate large datasets even with a small sample of real data.


<br />

## What are the 5 synthesizers in the SDV package?
<br />
For this demonstration I am going to use a python package called the Synthetic Data Vault or SDV to create synthetic data in a tabular format (aka csv/excel). Within the package there are 5 different types of models that can be used to generate data, and they each have their own strength and weakness to consider.

<br />

1. **Fast ML Preset:**
   - Produces the fastest speed to create synthetic data
   - Great for first time SDV users, can be used to get quickly started with the synthetic data

``` python

synthesizer = SingleTablePreset(
    metadata, # required
    name='FAST_ML', # required
    locales=['en_US', 'en_CA']
)

```

<br />


2. **GaussianCopulaSynthesizer:**
   - Uses classical, statistical methods to train a model and generate synthetic data
   - The most customizable one out of the 5 models
   - Can be used to work with normal distribution, binomial distribution, or other kinds depending on the data
``` python

synthesizer = GaussianCopulaSynthesizer(
    metadata, # required
    enforce_min_max_values=True,
    enforce_rounding=False,
    numerical_distributions={
        'amenities_fee': 'beta',
        'checkin_date': 'uniform'
    },
    default_distribution='norm'
)

```
<br />

3. **CTGANSynthesizer:**
   - The CTGAN Synthesizer uses GAN-based, deep learning methods to train a model and generate synthetic data
   - GANs (Generative Adversarial Networks) are a type of unsupervised machine learning model, consisting of a generator that creates data and a discriminator that differentiates between real and generated data, improving iteratively through adversarial training until the generated data is indistinguishable from real data.

``` python
synthesizer = CTGANSynthesizer(
    metadata, # required
    enforce_rounding=False,
    epochs=500,
    verbose=True
)
```
<br />

4. **TVAESynthesizer:**
   - The TVAE Synthesizer uses a variational autoencoder (VAE)-based, neural network techniques to train a model and generate synthetic data.
   - Variational Autoencoders (VAEs) are a type of generative model in machine learning that use a neural network architecture to encode input data into a latent space and then reconstruct it back to the original data, with a focus on learning the distribution of data for generating new, similar data.
   - For more info: https://www.cs.columbia.edu/~zemel/Class/Nndl-2021/files/lec13.pdf
     
``` python
synthesizer = TVAESynthesizer(
    metadata, # required
    enforce_min_max_values=True,
    enforce_rounding=False,
    epochs=500
)
```
<br />

5. **CopulaGANSynthesizer:**
   - The Copula GAN Synthesizer uses a mix of classic, statistical methods and GAN-based deep learning methods to train a model and generate synthetic data.
   - Experimental, results may vary

``` python
synthesizer = CopulaGANSynthesizer(
    metadata, # required
    enforce_min_max_values=True,
    enforce_rounding=False,
    numerical_distributions={
        'amenities_fee': 'beta',
        'checkin_date': 'uniform'
    },
    epochs=500,
    verbose=True
)
```
<br />

## Preparation for the demonstration
Some context for the data we are using for the demonstration: I will be using the Emergency Connectivity Fund open source data Form 477, Covid relief pandemic program led by the FCC federal agency to ensure that schools and libraries around the nation continue to have access to technology such as computers and internet services by reimbursing them with funds. 

<br />

**The data is filtered by choosing only Form Versions that are current and choosing 7 columns from the dataset:**
   - Application Number (Identifier)
   - unique_key (Unique Identifier, Primary Key)
   - Total Funding Commitment Request Amount (Continuous Variable)
   - Total Student Count (Continuous Variable)
   - Monthly Recurring Unit Cost (Continuous Variable)
   - Product Type (Categorical Variable)
   - Urban/ Rural Status (Binary Variable)

<br />

Another important change I made was creating a unique identifier that mixes with Funding Request Number, FRN Line Item ID, and One time Unit Cost. The unique identifier step is important because the package relies on ensuring each row of data is unique and that that there are no duplicates in the unique identifier when generating the data. If there are duplicates, you will receive an error. If your variable is completely unique that would be ideal, but in case you do not have one be sure to create it by mixing with other variables to truly make it unique.


<br />
The last thing that's recommended before starting the test is to visualize the variables you're going to synthesize in order to have a good reference point to use.
<br/>

Continuous Variables:
![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/e9a6567b-df0a-49a5-b2d1-865bac55850f)

<br />


Categorical Variables:
![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/4f467319-3710-40c1-abac-4d579ccaa852)

<br />

Note: The 95% quantile and above has been removed in order to get a closer look into the distribution of the continuous variables. The extreme skewness beforehand made it very hard to read it.


<br />

## Demonstration
Here is a rough draft of the coding that your client can use to generate their Synthetic data:

<br />

``` python
from sdv.single_table import TVAESynthesizer, GaussianCopulaSynthesizer, CTGANSynthesizer, CopulaGANSynthesizer
from sdv.lite import SingleTablePreset
from sdv.metadata import SingleTableMetadata
from sdv.evaluation.single_table import get_column_plot, evaluate_quality, get_column_pair_plot

def generate_synthetic_data(dataset, sdv_type, unique_id, num_samples):
    """
    Generates synthetic data using the specified SDV synthesizer.

    Args:
        dataset (pd.DataFrame): The dataset to base the synthetic data on.
        sdv_type (str): Type of SDV synthesizer to use ('TVAE', 'GaussianCopula', 'CopulaGAN', 'fast_ml', or 'CTGAN').
        unique_id (str): Column name to set as a primary key in the dataset.
        num_samples (int): Number of synthetic data rows to generate.

    Returns:
        pd.DataFrame: The generated synthetic dataset.
    """
    # Create metadata object and detect from dataframe
    metadata = SingleTableMetadata()
    metadata.detect_from_dataframe(data=dataset)
    
    # Update metadata for the unique key column
    metadata.update_column(column_name=unique_id, sdtype='id')
    metadata.set_primary_key(column_name=unique_id)
    
    # Validate metadata
    metadata.validate_data(data=dataset)
    
    # Select the synthesizer based on sdv_type
    if sdv_type == 'TVAE':
        synthesizer = TVAESynthesizer(
                        metadata, # required
                        enforce_min_max_values=True,
                        enforce_rounding=True,
                        epochs=500) 
    elif sdv_type == 'GaussianCopula':
        synthesizer = GaussianCopulaSynthesizer(
                        metadata, 
                        enforce_min_max_values=True,
                        enforce_rounding=True,
                        #numerical_distributions={
                        #    'amenities_fee': 'beta',
                        #    'checkin_date': 'uniform'
                        #        },
                        #default_distribution='norm' #lets you determine what distrubution your working  with, read file for more information: https://docs.sdv.dev/sdv/single-table-data/modeling/synthesizers/gaussiancopulasynthesizer
                        )
    elif sdv_type == 'CTGAN':
        synthesizer = CTGANSynthesizer(
                        metadata, 
                        enforce_rounding=True,
                        epochs=500,
                        verbose=True)
    elif sdv_type == 'CopulaGAN':
        synthesizer = CopulaGANSynthesizer(
                        metadata, 
                        enforce_min_max_values=True,
                        enforce_rounding=True,
                     #   numerical_distributions={
                     #       'amenities_fee': 'beta',
                     #       'checkin_date': 'uniform'
                     #                       },
                        epochs=500,
                        verbose=True)
    elif sdv_type == 'fast_ml':
        synthesizer = SingleTablePreset(metadata, name='FAST_ML')
    else:
        return "Error: Invalid SDV synthesizer type. Choose 'TVAE', 'GaussianCopula', 'CopulaGAN', 'fast_ml', or 'CTGAN'."

    # Fit the synthesizer on the dataset
    synthesizer.fit(dataset)
    
    # Generate synthetic data
    synthetic_data = synthesizer.sample(num_rows=num_samples)
    
    return synthetic_data

# Example usage
# synthetic_dataset = generate_synthetic_data(ecf_data_sample_selected, 'TVAE', 'unique_key', 25000)

```

<br /> 


## Synthetic Data Generation Function Explanation

This Python function demonstrates how to generate synthetic data using various synthesizers from the SDV (Synthetic Data Vault) library. It allows the creation of a synthetic dataset that mimics the structure of a real dataset. Below is a detailed explanation of the function:

### Importing Required Libraries and Modules:

- `from sdv.single_table` imports `TVAESynthesizer`, `GaussianCopulaSynthesizer`, `CTGANSynthesizer`, `CopulaGANSynthesizer` for different types of data synthesis.
- `from sdv.lite` imports `SingleTablePreset` for lightweight synthesizers.
- `from sdv.metadata` imports `SingleTableMetadata` for handling metadata of the dataset.
- `from sdv.evaluation.single_table` imports evaluation tools.

### Function Definition:

- **Function Name**: `generate_synthetic_data`
- **Arguments**:
  - `dataset`: A pandas DataFrame representing the real dataset.
  - `sdv_type`: A string specifying the type of SDV synthesizer to use. (Options: TVAE, GaussianCopula, CTGAN, fast_ml, CopulaGAN)
  - `unique_id`: A string representing the column name to be set as the primary key.
  - `num_samples`: An integer indicating the number of synthetic data rows to generate.

### Steps Within the Function:

1. **Metadata Creation and Detection**:
   - Create a `SingleTableMetadata` object.
   - Detects and extract metadata from the provided dataset using `metadata.detect_from_dataframe(data=dataset)`.

``` python
metadata = SingleTableMetadata()
metadata.detect_from_dataframe(data=dataset)

```

2. **Updating Metadata**:
   - Update metadata for the unique key column using `metadata.update_column(column_name=unique_id, sdtype='id')`.
   - Make sure that whichever unique identifier you used is truly unique and has no duplicates. If there is one there will be an error.
   - Set the primary key for the metadata using `metadata.set_primary_key(column_name=unique_id)`.
  
``` python
metadata.update_column(column_name=unique_id, sdtype='id')
metadata.set_primary_key(column_name=unique_id)


```

3. **Metadata Validation**:
   - Validate the metadata with `metadata.validate_data(data=dataset)`.
   - The object compares both the metadata and the original data to see if the former accurately describes the latter.
   - If validation fails, return a descriptive error message on where it happened.

``` python
metadata.validate_data(data=dataset)
```


5. **Synthesizer Selection and Initialization**:
   - Based on `sdv_type`, select and initialize the appropriate synthesizer (TVAESynthesizer, GaussianCopulaSynthesizer, etc.).
   - Configure synthesizer parameters like `enforce_min_max_values`, `enforce_rounding`, and `epochs`.
   - Note: Each model are customizable so feel free to tune the hyperparameter to fit your need. In the main function I've provided all the parts you can modify within the model.

``` python
if sdv_type == 'TVAE':
    synthesizer = TVAESynthesizer(metadata, enforce_min_max_values=True, ...)
elif sdv_type == 'GaussianCopula':
    synthesizer = GaussianCopulaSynthesizer(metadata, ...)
elif sdv_type == 'CTGAN':
    synthesizer = CTGANSynthesizer(metadata, ...)
elif sdv_type == 'CopulaGAN':
    synthesizer = CopulaGANSynthesizer(metadata, ...)
elif sdv_type == 'fast_ml':
    synthesizer = SingleTablePreset(metadata, name='FAST_ML')
else:
    return "Error: Invalid SDV synthesizer type."
``` 

6. **Fitting the Synthesizer**:
   - Fit the selected synthesizer to the real dataset using `synthesizer.fit(dataset)`.
   - This will allow you to apply your trained model to the real dataset itself and start the process of generating synthetic data.

``` python
   synthesizer.fit(dataset)
``` 

7. **Generating Synthetic Data**:
   - Generate synthetic data by sampling from the fitted synthesizer with `synthetic_data = synthesizer.sample(num_rows=num_samples)`.
   - However many rows of data you want to produce can be written down in the num_samples.
   - The time it takes to generate new data is dependent on which model you use, the size of the data including rows and columns, and the hyperparameter tuning on the model.

``` python
synthetic_data = synthesizer.sample(num_rows=num_samples)

``` 

8. **Returning Synthetic Data**:
   - The function returns the generated synthetic DataFrame.
     
``` python
   return synthetic_data
``` 
### Example Usage:

```python

synthetic_dataset = generate_synthetic_data(ecf_data_sample_selected, 'TVAE', 'unique_key', 25000)
```
<br />

## Validating the results
It's important to ensure that the output from the model has worked effectively after running through the original data. There's three steps to check the validity and effectiveness of the results:


1. **Diagnosing the model**:
   - Use run_diagnostic object to compare all three datasets (real data, synthetic data, metadata) to see if they have the same quality among each other.
   - The results have to be 100% in this case, if its lower than 100 then something has gone wrong with the process.

```python
from sdv.evaluation.single_table import run_diagnostic

diagnostic_report = run_diagnostic( #this is to ensure that the model has worked properly. Anything lower than a 100 is a problem and should be taken care of
    real_data=ecf_data_sample_selected,
    synthetic_data=synthetic_dataset,
    metadata=metadata)
```

Generating report ...
(1/2) Evaluating Data Validity: : 100%|██████████| 7/7 [00:00<00:00, 368.31it/s]
(2/2) Evaluating Data Structure: : 100%|██████████| 1/1 [00:00<00:00, 499.14it/s]

Overall Score: 100.0%

Properties:
- Data Validity: 100.0%
- Data Structure: 100.0%


2. **Quality Check**:
   - Use evaluate_quality to check the similarities of the synthetic and real data.
   - Column shapes show the similarity statistically between the synthetic and real data for one column.
   - Column Pair Trends show the similarity statistically between two columns for both data.

```python
quality_report = evaluate_quality( 
    real_data=ecf_data_sample_selected, 
    synthetic_data=synthetic_dataset,
    metadata=metadata 
)
```
Generating report ...
(1/2) Evaluating Column Shapes: : 100%|██████████| 7/7 [00:00<00:00, 87.19it/s]
(2/2) Evaluating Column Pair Trends: : 100%|██████████| 21/21 [00:00<00:00, 56.76it/s] 

Overall Score: 63.6%

Properties:
- Column Shapes: 88.66%
- Column Pair Trends: 38.53%

<br />

If you want to look at the accuracy for each particular columns, use the object quality_report.get_details for closer inspection:
<br/>

```python
quality_report.get_details(property_name='Column Shapes')
```
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Column</th>
      <th>Metric</th>
      <th>Score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Total Funding Commitment Request Amount</td>
      <td>KSComplement</td>
      <td>0.932573</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Product Type</td>
      <td>TVComplement</td>
      <td>0.919209</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Urban/ Rural Status</td>
      <td>TVComplement</td>
      <td>0.983609</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Monthly Recurring Unit Cost</td>
      <td>KSComplement</td>
      <td>0.669842</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Total Student Count</td>
      <td>KSComplement</td>
      <td>0.927928</td>
    </tr>
  </tbody>
</table>
</div>
  
3. **Visualization**:
   - There's three ways to look at the results visually
   - Using get_column_plot (shows the distribution for the real and synthetic data for one column)
   - Using get_column_pair_plot (visualize the trends between a pair of columns for real and synthetic data)
   - Using a histogram and bar graphs method

<br/>
get_column_plot
<br/>

```python
from sdv.evaluation.single_table import get_column_plot
import plotly.io as pio

# This ensures Plotly plots show up in Jupyter notebooks
pio.renderers.default = 'notebook'

fig = get_column_plot(
    real_data=ecf_data_sample_selected,
    synthetic_data=synthetic_dataset,
    metadata=metadata,
    column_name='Total Funding Commitment Request Amount'
)

# Show the figure using Plotly's show method
fig.show()
```

![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/b1e31b70-faba-4435-b183-da6659e16561)


<br />
get_column_pair_plot

<br/>

```python
from sdv.evaluation.single_table import get_column_pair_plot

fig = get_column_pair_plot(
    real_data=ecf_data_sample_selected,
    synthetic_data=synthetic_dataset,
    metadata=metadata,
    column_names=['Total Funding Commitment Request Amount', 'Total Student Count'],
    )
    
fig.show()

```

<br />

Histograms and Bar Graphs

Real Data Continuous Variables:
![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/e9a6567b-df0a-49a5-b2d1-865bac55850f)

<br />


Real Data Categorical Variables:
![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/4f467319-3710-40c1-abac-4d579ccaa852)

<br />

Synthetic Data Continuous Variables:
![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/77795585-f884-405e-8995-953f1a43bb00)

<br />

Synthetic Data Categorical Variables:
![image](https://github.com/DanielKim15/Handling-PII-Data-Part-2-Synthetic-Data-/assets/59937761/61252a29-4425-48b0-b7cf-8de1d34ec5b8)

<br />
Note: For the histograms and bar graphs method, I've used a TVAE method to generate 25,000 rows out of 30,000+ compared to the original. From our previous diagnostic report it seems that most of the continuous variables and categorical variables seem to have high similarities with the real data, but the Monthly Recurring cost seems to have a different outcome. The real data seems to have occasional bumps on the x-axis for cost, but when looking at the synthetic data the x-axis seems to be almost completely smoothed out, unimodal like at this point. There will be other columns like this one where the model won't produce a high level of accuracy when producing the synthetic data, so it is important to keep in mind when doing the analysis. The column can also be excluded from the analysis too in case the accuracy is way too low for the user to use.


<br />

## Final Thoughts
Synthetic Data can provide alternative ways to get started on the data without the risk of handling PII data. Considering how the future of generative models are progressing fast, having the accuracy between real and syntehtic is only going to get better over time.



