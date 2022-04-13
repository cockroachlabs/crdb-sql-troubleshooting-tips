# Query slow because GROUP BY columns within STORING clause of index

## Version(s): 
All

## Issue Indicators/Example Errors
Indicators for this issue are:
    - a slow query has a GROUP BY clause,
    - the slow query uses an index that has a STORING clause, and
    - some or all of the columns in the query’s GROUP BY clause are part of the index’s STORING clause and are not index key columns.

## Description
Here is an example query that was running slower than expected:

```sql
SELECT
  cnt, organization, concat(os, '-', version) AS bucket
FROM
  (
    SELECT
      count(1)::FLOAT8 AS cnt, organization, os, version
    FROM
      nodes
    WHERE
      lastseen > ($1)::TIMESTAMPTZ AND lastseen <= ($2)::TIMESTAMPTZ
    GROUP BY
      organization, os, version
  )
  
Arguments:
  $1: '2021-07-27 13:22:09.000058Z'
  $2: '2021-10-25 13:22:09.000058Z'
```

The columns in the GROUP BY clause are organization, os, and version.

The query plan shows that it is using index nodes_lastseen_organization_storing:

```sql
                     distribution         full
                     vectorized           true
render                                                                                                      (cnt float, organization varchar, bucket string)
 │                   estimated row count  3760
 │                   render 0             (concat((os)[string], ('-')[string], (version)[string]))[string]
 │                   render 1             ((count_rows)[int]::FLOAT8)[float]
 │                   render 2             (organization)[varchar]
 └── group                                                                                                  (organization varchar, os string, version string, count_rows int)
      │              estimated row count  3760
      │              aggregate 0          count_rows()
      │              group by             organization, os, version
      └── project                                                                                           (organization varchar, os string, version string)
           └── scan                                                                                         (organization varchar, lastseen timestamptz, os string, version string)
                     estimated row count  2330245
                     table                nodes@nodes_lastseen_organization_storing
                     spans                /2021-07-27T13:22:09.000059Z-/2021-10-25T13:22:09.000058001Z
```

Here is the table schema for the example query:

```sql
CREATE TABLE public.nodes (
	id VARCHAR(60) NOT NULL,
	ampuuid UUID NULL,
	organization VARCHAR(60) NULL,
	created TIMESTAMPTZ NULL DEFAULT now():::TIMESTAMPTZ,
	disabled BOOL NOT NULL DEFAULT false,
	lastseen TIMESTAMPTZ NULL DEFAULT now():::TIMESTAMPTZ,
	os STRING NOT NULL,
	arch STRING NOT NULL,
	autotags JSONB NULL,
	version STRING NOT NULL DEFAULT '':::STRING,
	clone BOOL NOT NULL DEFAULT false,
	cloneof VARCHAR(60) NOT NULL DEFAULT '':::STRING,
	endpoint_type STRING NOT NULL DEFAULT 'amp':::STRING,
	ip INET NULL,
	osqueryversion STRING NOT NULL DEFAULT '':::STRING,
	CONSTRAINT "primary" PRIMARY KEY (id ASC),
	INDEX nodes_organization_ampuuid (organization ASC, ampuuid ASC),
	INDEX nodes_created_asc_organization (created ASC, organization ASC),
	INDEX nodes_created_desc_organization (created DESC, organization ASC),
	INDEX nodes_organization_os_version (organization ASC, os ASC, version ASC),
	INDEX nodes_organization_version (organization ASC, version ASC),
	INDEX nodes_lastseen_organization_storing (lastseen ASC, organization ASC) STORING (os, version),
	FAMILY "primary" (id, ampuuid, organization, created, disabled, lastseen, os, arch, autotags, version, clone, cloneof, endpoint_type, ip, osqueryversion)
);
```

The `nodes_lastseen_organization_storing` index has the `GROUP BY` column organization as an index key column, however the `STORING` clause includes the GROUP BY columns os and version.

## Solution
Create a new secondary index that has all of the `GROUP BY` columns as key columns in the index. This is beneficial for the efficiency of a query since it allows crdb to perform a streaming `GROUP BY` rather than a hash `GROUP BY`.

There was an improvement in the latency of the example query when the os and version fields were added as index key columns to a new secondary index:

```sql
INDEX "nodes_lastseen_organization_os_version" (lastseen, organization, os, version)
```

## References
Docs: https://www.cockroachlabs.com/docs/stable/indexes#storing-columns
GitHub: https://github.com/cockroachlabs/support/issues/1297#issuecomment-952036351
