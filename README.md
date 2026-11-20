# GES DISC OPeNDAP and Kerchunk Access Benchmark

This workflow compares two methods for remotely accessing NASA GES DISC IMERG data:

- **Cloud OPeNDAP**, using Xarray and Pydap to request data through a Hyrax server.
- **Virtual Zarr (Kerchunk)**, using annual Kerchunk reference stores and authenticated HTTP byte-range requests to the source files.

The benchmark measures two stages separately:

1. **Lazy-load time**: authentication-dependent discovery and setup, remote metadata access, construction of lazy Xarray/Dask datasets, concatenation, and subsetting.
2. **Compute time**: execution of the lazy Dask graph, remote transfer of the requested values, decoding, and assembly of the subset in memory.

These measurements represent end-to-end access workflows. Compute time is therefore not CPU time alone.

## Workflow files

- `access_with_kerchunk.py` opens annual Kerchunk reference stores, applies variable, spatial, and temporal selections, and manages authenticated HTTP range requests, retries, and token refreshes.
- `access_with_opendap.py` discovers OPeNDAP granules, opens them concurrently with Pydap, concatenates the lazy datasets, and applies the spatial selection.
- `benchmark_access_methods.py` configures and runs repeated tests for both methods, calls `.compute()`, and records results incrementally.
- `plot_benchmark_results.py` validates a benchmark-results CSV and plots every lazy-load and compute trial.
- `gesdisc-access-benchmark.yml` defines the Conda environment used by the workflow.

## Installation

Create and activate the benchmark-specific environment:

```bash
conda env create -f gesdisc-access-benchmark.yml
conda activate gesdisc-access-benchmark
```

If the environment file remains inside an `environments/` directory, use:

```bash
conda env create -f environments/gesdisc-access-benchmark.yml
```

## Earthdata authentication

The workflow requires a NASA Earthdata Login account and credentials configured in `~/.netrc`. Do not commit `.netrc`, bearer tokens, cookies, or other credentials to Git.

Earthaccess reads the credentials with:

```python
earthaccess.login(strategy="netrc")
```

The resulting bearer token is used directly for Kerchunk HTTPS requests and to create the authenticated Pydap session used for OPeNDAP.

## Configuration

Edit `BenchmarkConfig` in `benchmark_access_methods.py` before running the benchmark. Important settings include:

- `shortname` and `version`: collection identifiers.
- `bbox`: `[minimum longitude, minimum latitude, maximum longitude, maximum latitude]`.
- `subvar`: science variable to retrieve.
- `start_dates` and `end_dates`: paired inclusive test intervals.
- `num_trials`: number of repetitions for each interval.
- `nworkers_opendap`: threads used while opening OPeNDAP granules.
- `nworkers_compute`: Dask threads used during `.compute()`.
- `max_compute_days`: largest interval that will normally be materialized.
- `kerchunk_http_max_attempts` and `kerchunk_http_backoff_seconds`: retry behavior for Kerchunk HTTP requests.
- `benchmark_results_saveloc` and `subset_output_dir`: output locations.

The current output paths reflect the original repository layout. Update them if the scripts are moved into a standalone repository.

Use conservative worker counts. Both OPeNDAP opening and `.compute()` can generate concurrent requests to Hyrax; excessive concurrency can result in throttling, timeouts, or blocking.

## Run the benchmark

Keep the three access scripts in the same directory so their local imports resolve, then run:

```bash
python benchmark_access_methods.py
```

The script writes each completed result immediately to a temporary `_running.csv` file. After a successful run, it renames the file to include the start time and total elapsed duration.

Recorded fields include:

- access method
- requested number of days
- trial number
- lazy-load time
- compute time
- estimated uncompressed subset size
- Kerchunk HTTP retries and token refreshes
- optional NetCDF output path
- completion status

By default, ranges longer than `max_compute_days` measure lazy loading but skip `.compute()`. The optional save settings can force the longest range to compute and write a NetCDF subset.

## Plot the results

Pass the benchmark CSV and desired figure path explicitly:

```bash
python plot_benchmark_results.py \
  --input results/access_benchmark_results.csv \
  --output figures/access_benchmark_times.png
```

The plot contains separate panels for lazy-load and compute time. Each line represents an individual trial; the script does not replace trial variability with only an average.

## Interpretation and reproducibility

The two methods do not perform identical internal operations:

- OPeNDAP opens one remote dataset per granule and uses Hyrax to interpret indexed DAP4 requests.
- Kerchunk opens annual reference metadata and retrieves referenced byte ranges directly from the source files.

Results can vary with network conditions, server load, authentication state, cache state, worker counts, source-file chunking, and temporary retries. For a meaningful comparison:

- use the same variable, bounding box, dates, and number of trials;
- run the methods in the same environment and during the same benchmark session;
- report worker counts and skipped computations;
- retain all successful trials and document failures;
- avoid treating a single run as a general performance guarantee.

Small benchmark CSV files may be retained with the repository to support reproduction of published figures. Downloaded data, computed NetCDF subsets, credentials, and large generated outputs should remain excluded through `.gitignore`.

## AI assistance acknowledgment

Generative AI tools were used to assist with code development, debugging, documentation, and explanation of the software workflow. The human author reviewed and revised the resulting code and documentation and remains responsible for the scientific methodology, validation, interpretation, and published results.

