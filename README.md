# Overview
## Assumptions
This repository provides guidance on how to connect to the IDR Snowflake instance using Python. We assume that:
1. You're planning to execute your Python code locally, on a CMS-issued laptop.
2. You have a current version of Python already installed.
3. You have the ability to connect to pip to install open-source Python packages.
4. You already have all relevant EUA job codes needed to connect to the IDR and access any relevant data assets.

For assistance installing Python on your CMS-issued laptop, reach out to the CMS help desk. If you need help installing Python packages or managing your Python environment, consult other members of your component, since environment management practices can vary across teams.

For help with IDR job codes or IDR data documentation, please refer to the [IDRC Communications Confluence page](https://confluenceent.cms.gov/pages/viewpage.action?spaceKey=IDRCC&title=IDRC+Communications) for up-to-date documentation.

## Quickstart

If all you're looking for is a quick guide on querying and fetching data from the IDR Snowflake instance, here's a primer:
1. Install the [Python Snowflake connector](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector-install). The easiest approach is to run `pip install "snowflake-connector-python[pandas]"` from a command line window. Make sure to include the `[pandas]` argument to include Pandas extensions in your installation.
2. Import the Python Snowflake connector, and create a connection using the IDR's host and the `externalbrowser` authenticator. Make sure to populate your EUA ID in the `user` argument below. Running this command should launch a browser window in your default browser, which will ask you to authenticate to the IDR using CMS SSO. Complete the prompt on the browser window, and navigate back to your execution location.
```
import snowflake.connector

ctx = snowflake.connector.connect(
          account="cms-idr.privatelink",
          user="<your_eua>",
          authenticator="externalbrowser"
    )
```

3. Specify a query. For example, to fetch counts of living Medicare beneficiaries by zip code, you might do the following:
```
query = """
    select geo_zip5_cd
    , count(1)
    from idrc_prd.cms_vdm_view_mdcr_prd.v2_mdcr_bene
    where idr_ltst_trans_flg = 'Y'
    and idr_trans_obslt_ts > current_date()
    and (bene_death_dt is null or bene_death_dt > current_date())
    group by geo_zip5_cd
"""
```
4. Execute your query using a cursor object generated from your connection object, and fetch your results as a Pandas dataframe. Be careful not to fetch too much data with this command, since you'll download all results of your query! For example, don't fetch an entire Part B claims table; instead, try to fetch a summary (e.g. counts of claims by HCPCS code or something similar).
```
with ctx.cursor() as cur:
    cur.execute(query)
    df = cur.fetch_pandas_all()
```
At this point, you can manipulate your results (saved in the `df` object) as a standard Pandas dataframe in any way that you'd like. See [/examples/quickstart.ipynb](/examples/quickstart.ipynb) for a notebook containing the above code.

# Connection and usage details
## Connection options
The first step in connecting to the IDR's Snowflake instance is to specify your connection parameters. The three required parameters are:
1. `account="cms-idr.privatelink"`
2. `user = "<your_eua>"`. For example, if your EUA is TRTY, specify `user="TRTY"`.
3. `authenticator="externalbrowser"`.

In addition to these required parameters, you can also specify a set of other parameters if you'd like. These paramters are entirely optional, and will populate or replace IDR-specified defaults:
1. `warehouse`: compute warehouse to use when running queries in the IDR. For most users, likely defaults to `"IDRC_PRD_COMM_WH"`. Update if you'd like to use something other than your account default.
2. `database`: default database to use, if the database is not specified in a query. For most IDR end users, the only database you'll interact with is `"IDRC_PRD"`.
3. `schema`: default schema to use, if the schema is not specified in a query. For example, if you wanted to query tables in the Medicare VDM, you would specify `"CMS_VDM_VIEW_MDCR_PRD"`.
4. `role`: role to use when executing queries in the IDR. This role will default to your default IDR Snowflake role. As a result, most users should not need to change this option unless you have multiple roles that you regularly use.

Each parameter should be specified as an additional argument to the Snowflake connector function shown above. Here's a more extensive example showing a few more of the optional credentials listed above:
```
ctx = snowflake.connector.connect(
          host="cms-idr.privatelink",
          user="<your_eua>",
          authenticator="externalbrowser",
          warehouse="IDRC_PRD_COMM_WH",
          database="IDRC_PRD",
          schema="CMS_VDM_VIEW_MDCR_PRD"
    )
```
## Connection steps
To retrieve results from a database connection (Snowflake or otherwise), we need to use a *cursor*. A *cursor* is an ephemeral object that allows a user to retrieve the results of a query from a database connection.

When working with database cursors, the best practice is to explicitly "close" a cursor when you're done with it. In Python, you can do this explicitly as follows:
```
cur = ctx.cursor()
cur.execute(sql)
df = cur.fetch_pandas_all()
cur.close()
```
You can make this a little more readable using a [`with` statement](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement). The `with` statement below to the code block above, but is a bit more concise:
```
with ctx.cursor() as cur:
    cur.execute(sql)
    df = cur.fetch_pandas_all()
```
The `cur.close()` function is implicitly invoked at the end of the indented block following the `with` statement. Using `with` statements helps organize your code and saves you the trouble of remembering to close cursors and conduct other cleanup steps.

## Reading and saving IDR data
If all you need to do is read data from the IDR and save it locally, you're nearly done! Using the above examples, we've saved our results as a Pandas dataframe. See the [Pandas documentation](https://pandas.pydata.org/docs/) for infomration on how to manipulate a Pandas dataframe, but here are some basic examples:
1. View the results of your query with `print(df)`.
2. Save your results to a csv using `df.to_csv('path/to/your/output.csv')`

Be cautious about running queries that will return large amounts of data. For instance, in the example above, we were interested in counting the number of living beneficiaries by zip code. This query will return approximately 40,000 records (one record for each zip code), which is managable. However, do *not* download a full list of all living Medicare beneficiaries, since this would require you to download over sixty million database records to your computer! 

In general, try to aggregate data before downloading. Conduct your heavier computation in the IDR's Snowflake database, and download aggregated or summarized results to your computer.

## Writing data to IDR Snowflake
If you're authorized to write data to an IDR ADM/VDM, you can do so in Python using the [`write_pandas()` function](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector-api#module-snowflake-connector-pandas-tools). For example, using the same connection object as above:
```
import pandas as pd

df = pd.read_csv("path/to/your/file.csv")
write_pandas(
    conn = cnx, 
    df = df, 
    table_name = "<your_table>",
    database = "IDRC_PRD",
    schema = "<your_adm_or_vdm>"
)
```
Here, `df` should be a pandas dataframe. In this example, we're reading the pandas dataframe from a local CSV, but you can load any Pandas dataframe to the IDR in this fashion.

## Running arbitrary SQL statements
If you're interested in running somehting other than a simple `SELECT` statement - such as a `CREATE TABLE` statement or something similar - you might not want to use Pandas to execute your SQL command. Instead, you can run a SQL command with a cursor directly:
```
with ctx.cursor() as cur:
    cur.execute("""
        CREATE TABLE IDRC_PRD.<your_adm_or_vdm>.<your_table> AS
            SELECT * FROM IDRC_PRD.CMS_VDM_VIEW_MDCR_PRD.V2_MDCR_BENE
            LIMIT 10
    """)
```
In this case, there are no results to fetch, so there's no need to use the `cur.fetch_pandas_all()` method.


## Further reading
- [Snowflake documentation](https://docs.snowflake.com/)
 - Specific information on the [Snowflake Python connector](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector)
- [Pandas documentation](https://pandas.pydata.org/docs/)
- [IDRC Confluence](https://confluenceent.cms.gov/pages/viewpage.action?spaceKey=IDRCC&title=IDRC+Communications)
 - IDR's ["best practices"](https://confluenceent.cms.gov/download/attachments/489926811/IDR-User-Guide-Other-Access-Best-Practices.pdf?version=1&modificationDate=1726499246390&api=v2) guide for non-browser Snowflake connections.
 - IDR [job code information](https://confluenceent.cms.gov/display/IDRCC/IDRC+Onboarding+-+EUA+Job+Codes+for+IDRC) 
