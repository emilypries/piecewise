# Statistics Generated by Piecewise

Piecewise digests Measurement Lab data, computes aggregate statistics of the selected data, and makes both the data and aggregate statistics available in an API. Here we document how Piecewise does that aggregation, where to access the resulting statistics and an example that illustrates how to use this feature.

## Computing Statistics

The computing of aggregate statistics is based on the idea of **composable statistics:** combining the results from multiple samples to get a valid result for the combination of the samples.

For example, by tracking the sample count and total, we can compute a count and total for the overall population. This can then be easily converted to an arithmetic mean.

The samples are selected along configurable dimensions such as time slices or a spatial grid, so Piecewise can support histogram and heatmap types visualizations at varying granularity.

## Bin Types

Piecewise uses the term **bin** to refer to a combination of results from multiple samples. The types of bins available in Piecewise are listed below.

* **time_slices** - bin over the timestamps of test results.
  * **resolution** (required) - specifies a time unit by name.
  The timestamps are defined by truncating any time information of lower granularity by this unit (so truncating 2015-06-15T05:37:34 to hours gives 2015-06-15T05:00:00.)

  The time units supported are those defined by Postgres:
  * minute
  * hour
  * day
  * week
  * quarter
  * year
  * decade
  * century
  * millennium

* **spatial_grid** - bins over the GeoIP locations from test results.
  * **resolution** (required) - configures the spatial_grid bin with a number of degrees determining the size of the (square) grid cells.
    This may be fractional.
* **spatial_join** - bins by joining the GeoIP locations with a spatial table in postgres.
  * **table** (required) - specifies the name of the geometry table.
  * **geometry_column** (required) - specifies the name of the geometry table.
  * **key** (required) - specifies the name of the column that identifies features in the geometry table. This will be stored in the aggregate table rather than the full geometry. Regardless of the name in the geometry table the column in the aggregate table will always be named 'key.'
  * **key_type** (optional, default 'integer') - specifies the type of the key column. This currently accepts a string whose value can be either "string" or "integer".
* **isp_bins** bins over the ISP names associated with each IP according to the Maxmind database.
  * **maxmind_file** (required) - configures the isp_bins bin with the name of a postgres table used containing the Maxmind database.
  The 'setup.sql' script included with the Piecewise sources includes a COPY command that can ingest this table from the Maxmind CSV.
  * **rewrites** (required) - configures how the maxmind database names will be consolidated into shorter names. This is specified as an object where the keys are short names and the values are lists of strings that will be matched in the long names.

## Statistic Types

* **AverageRTT** - Average Round Trip Time is computed using the total of round trip times and count of all raw test results.
* **MedianRTT** - Median Round Trip Time is computed using a sample of test results stored in a PostgreSQL ARRAY column. It is slower than AverageRTT but less sensitive to skew from outliers in the test results.
* **MinRTT** - The Minimum Round Trip Time of test results selected.
* **AverageUpload** - The average upload bandwidth in ?bits per second?.
* **MedianUpload** - The median upload bandwidth.
* **UploadMin** - The lowest value for upload bandwidth in a given selection of test results.
* **UploadMax** - The highest value for upload bandwidth in a given selection of test results.
* **UploadCount** - The total number of test results in each bin for upload tests.
* **AverageDownload** - The average download bandwidth in bits per second.
* **MedianDownload** - The median download bandwidth.
* **DownloadMin** - The lowest value for download bandwidth in a given selection of test results.
* **DownloadMax** - The highest value for download bandwidth in a given selection of test results.
* **DownloadCount** - The total number of test results in each bin for download tests.

## Filter Types

* **temporal** - Filters the test data by timestamp.
  * **after** (required) Specified as a string in the format "MMM dd HH:mm:ss" in the UTC time zone, timestamps must be at or after this time in order to be included.
  * **before** (required) Specified as a string in the format "MMM dd HH:mm:ss" in the UTC time zone, timestamps must be before this time in order to be included.
  * Example: `{ "type": "temporal", "after": "Jan 1 2015 00:00:00", "before" : "Jan 2 2015 00:00:00" }`
* **bbox** - Filters the test data by geoip location.
  * **bbox** (required) - Specified as a list of four latitude and longitude numbers, in the order **West, South, East, North**. These form a bounding box defining the area from which tests must originate.
  * Example: `{ "type": "bbox", "bbox": [-122.6733398438,47.3630134401,-121.9509887695,47.8076208172] }`
* **raw** - Filters the test data by an arbitrary BigQuery expression.
  * **query** (required) - Specifies the BigQuery expression that is to be included in WHERE clause of the query.
  * Example: `{ "type": "raw", "query": "project == 0" }`

## Accessing Computed Statistics

In addition to storing computed statistics in a database table on your server, Piecewise provides a web service consisting of a single endpoint that supports several query parameters, depending on which bins and statistics are configured. The default bins and statistics are described below.

```
**TO DO:**
How to create new or customize existing bins and statistics.
```

The Piecewise web service is located: ``<url-where-you-host-piecewise>/stats/q/<aggregation>``. Replace ``<url-where-you-host-piecewise>`` with your server host name or IP address and ``<aggregation>`` with the name of one of the aggregations from the configuration.

``&stats=`` contains a comma-separated list of the statistics to be returned, using the same names as in the configuration file.

``&format=`` may be 'csv' or 'geojson' to specify the output format.
GeoJSON is the default, even when no spatial information is included.

Each configured bin may be used as a bin or a filter, or both, over the consolidated data for the computed statistics.
When specifying a rebinning, prefix the bin name with ``b.`` to get the query parameter name.
When specifying a filtering, prefix the bin name with ``f.`` to get the query parameter name.
For example, to get average round trip time with timestamp binning at 3600 second resolution and return only time chunks from timestamp 10000 to timestamp 20000, the query string would be:

``&stats=AverageRTT&b.time_slices=3600&f.time_slices=10000,20000``

Multiple bin and filter parameters may be used simultaneously.

The interpretation of the rebinning and filter parameters varies by bin type.

* **``f.time_slices``** contains the after and before timestamps as simple numbers separated by a comma.
  This represents the time in seconds since the Unix epoch.
* **``b.time_slices``** takes a string specifying a resolution (as used in the time_slices configuration) to rebin at.
  Note that although rebinning to a finer resolution succeeds, it will only produce values at the first cell of each time slice.
* **``f.spatial_grid``** takes four numeric values separated by commas.
  These specify a bounding box limiting the area for which statistics are computed.
  As with the 'bbox' filter, this must be in West, South, East, North order.
* **``b.spatial_grid``** takes a single numeric value specifying the resolution to rebin at, in degrees.
* **``f.isp_bins``** takes a list of the ISP names to be included in the results, separated by commas.
* **``b.isp_bins``** simply activates ISP binning.
  It does not require a value.
* **``b.spatial_join``** activates the spatial_join binning.
  If the parameter value is 'key' then the key will be returned in the results.
  Otherwise it will be the full geometries represented as GeoJSON.
* **``f.spatial_join``** has no effect - the spatial join bins cannot be used for filtering.

## Composable Statistics Example

To be more clear about "composable" statistics, let's work through an example. Let's say we want to be able to query for average round trip times for tests submitted over time.

We start with a full dataset contains timestamps and round trip times:

| Timestamp (s) | RTT (ms) |
| -------------:| --------:|
|             1 |       30 |
|             2 |       50 |
|            25 |       40 |
|            25 |       80 |
|            28 |       90 |

And we group the data by 10-second intervals, then there are two groups: one containing seconds 0 through 9 and a second containing 20 through 29. There are no values in the 10-19 bin so it is not tracked.

Piecewise will consolidate the data so that each timestamp is only represented once in the database, and track a count and total of the round-trip-time values so that the mean can be recovered.

| Snapped timestamp (s) | Total RTT (ms) | Sample count |
| ---------------------:| --------------:| ------------:|
|                      0|              80|             2|
|                     20|             210|             3|

With this consolidated table we can get the mean round trip times over 10-second intervals by fetching only 1 row for each interval, something like this:

```
SELECT snapped_timestamp AS ts, rtt / sample_count AS AverageRTTFROM consolidated_statistics;
```

| ts | AverageRTT |
| --:| ----------:|
|  0 |         40 |
| 20 |         70 |

Additionally we can create lower-granularity results by grouping the rows of the consolidated table.
Dividing the timestamp by 30, taking the floor, and multiplying by 30 again gives 30-second granularity, with each cell bearing the timestamp of the first second in the bin.

```
SELECT (FLOOR(snapped_timestamp / 30) * 30) AS ts, SUM(rtt) / SUM(sample_count) AS AverageRTT from consolidated_statistics GROUP BY ts;
```

| ts | AverageRTT |
| --:| ----------:|
|  0 |         58 |