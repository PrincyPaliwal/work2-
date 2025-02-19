# Real-Time Data Flow: SQL Server to Databricks to Sigma

## Overview

This project demonstrates a real-time data flow pipeline between SQL Server, Databricks, and Sigma. The pipeline ensures that transactional data from SQL Server is processed in Databricks and visualized in Sigma, enabling real-time analytics.

## Components

- **SQL Server**: Source database containing transactional data.
- **Databricks**: Processes and transforms data for analytics.
- **Sigma**: Visualization tool for real-time data insights.
- **Azure Event Hub (Optional)**: Used for real-time data streaming.
- **Delta Lake**: Stores processed data for efficient querying.

## Workflow

1. **Data Ingestion**: SQL Server transactions are streamed using Change Data Capture (CDC) or an ETL pipeline.
2. **Databricks Processing**: Data is ingested into Databricks, cleaned, and stored in Delta Lake.
3. **Data Transformation**: Business logic is applied to process raw data.
4. **Sigma Integration**: The processed data is visualized using Sigma dashboards.

## Prerequisites

- SQL Server instance with transactional data.
- Databricks workspace setup.
- Sigma account for visualization.
- (Optional) Azure Event Hub for real-time streaming.

## Databricks SQL Queries

### A2RawData Query
```sql
%sql
  select
    c.Ticker,
    c.Issuer,
    c.ResourceProvider,
    c.StartDate,
    c.EndDate,
    c.AdjustedValue,
    c.Location,
    c.Region,
    c.Format,
    c.Value,
    c.Currency,
    c.Title,
    c.AggregatedStatus,
    c.Attendee,
    c.Status,
    c.Note,
    c.Team,
    c.CreatedAt,
    c.LastUpdate,
    c.CreatedBy,
    c.LastUpdateBy,
    c.InOffice,
    c.AllDay,
    c.CreatedByIngestor,
    c.BuysideStatus,
    c.IsReviewRequired,
    c.Id,
    c.SplitTeam as research_splits,
    case
      when bc.broker is not null then bc.broker
      else 'Other'
    END AS ParentBroker
  from
    (
      select
        max(resource_provider) as resource_provider,
        broker
      from
        freerider.sigma_input.broker_mapping
      group by
        broker
    ) bc
      right join
        (
          select
            r.*,
            r.Value * CAST(rs.weighting AS DOUBLE) as AdjustedValue,
            rs.SplitTeam
          from
            risk_sql_prod.dbo.a2arecords r
              inner join
                (
                  SELECT distinct
                    fullname,
                    SplitTeam,
                    weighting
                  FROM
                    (
                      SELECT
                        fullname,
                        team as SplitTeam,
                        weighting,
                        ROW_NUMBER() OVER (
                            PARTITION BY fullname, team
                            ORDER BY fullname, team DESC
                          ) AS rn
                      FROM
                        freerider.sigma_input.research_split
                    ) rs
                  WHERE
                    rn = 1
                ) rs
                on r.Attendee = rs.fullname
        ) c
        on c.ResourceProvider = bc.resource_provider;
```

### Commission Query
```sql
%sql
select
  c.TradeDate,
  c.Account,
  c.BrokerCode,
  c.ParentBroker,
  c.BrokerName,
  c.SecType,
  c.TotalCommission,
  c.Shares,
  c.Timestamp,
  c.team as SplitTeam,
  case
    when bc.resource_provider is not null then bc.resource_provider
    else 'Other'
  END AS ResearchTeam
from
  (
    select
      max(resource_provider) as resource_provider,
      broker
    from
      freerider.sigma_input.broker_mapping
    group by
      broker
  ) bc
    right join
      (
        select
          bm.TradeDate,
          bm.Account,
          bm.BrokerCode,
          bm.ParentBroker,
          bm.BrokerName,
          bm.SecType,
          bm.Shares,
          bm.Timestamp,
          cs.team,
          bm.TotalCommission * CAST(cs.weighting AS DOUBLE) as TotalCommission
        from
          risk_sql_prod.core.brokercommission bm
            inner join
              (
                select distinct
                  team,
                  weighting,
                  account
                from
                  freerider.sigma_input.commission_split
              ) cs
              on bm.account = cs.account
      ) c
      on c.ParentBroker = bc.broker;
{

   %sql
WITH ResearchCosts AS (
    SELECT 
        A2.ParentBroker AS Brokers,
        A2.research_splits AS SplitTeam,
        A2.ResourceProvider AS Research_Name,
        CAST(SUM(A2.AdjustedValue) AS BIGINT) AS Research_Cost
    FROM freerider.sigma_input.a2rawdata A2
    WHERE A2.ParentBroker <> 'Other'
     AND A2.StartDate >= {{dateRangeFilter-1}}.start -- Parameter substitution
      AND A2.StartDate <= {{dateRangeFilter-1}}.end
    GROUP BY A2.ParentBroker, A2.ResourceProvider, A2.research_splits

),
Commissions AS (
    SELECT 
        CRD.ParentBroker AS Brokers,
        CRD.SplitTeam AS SplitTeam,
        CAST(SUM(CRD.TotalCommission) AS INT) AS Commission
    FROM freerider.sigma_input.commission CRD
where  CRD.TradeDate >= {{dateRangeFilter-1}}.start -- Parameter substitution
      AND CRD.TradeDate <= {{dateRangeFilter-1}}.end
    GROUP BY CRD.ParentBroker, CRD.SplitTeam
)
SELECT 
    COALESCE(RC.Brokers, C.Brokers) AS Brokers,
    COALESCE(RC.SplitTeam, C.SplitTeam) AS SplitTeam,
    COALESCE(RC.Research_Name, 'N/A') AS Research_Name,
    COALESCE(RC.Research_Cost, 0) AS Research_Cost,
    COALESCE(C.Commission, 0) AS Total_Commission,
    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta
FROM ResearchCosts RC
FULL OUTER JOIN Commissions C
    ON RC.Brokers = C.Brokers AND RC.SplitTeam = C.SplitTeam
ORDER BY Brokers ASC
  %sql
WITH ResearchCosts AS (
    SELECT 
        CS.FULLNAME AS Individual,
        CS.Account AS Account,
        CAST(SUM(A2.AdjustedValue) AS INT) AS Research_Cost
    FROM freerider.sigma_input.a2rawdata AS A2
    INNER JOIN freerider.sigma_input.Commission_split AS CS
        ON A2.Attendee = CS.FULLNAME
    WHERE A2.ParentBroker <> 'Other'
      AND A2.StartDate >= {{dateRangeFilter-1}}.start -- Parameter substitution
      AND A2.StartDate <= {{dateRangeFilter-1}}.end -- Parameter substitution
    GROUP BY CS.FULLNAME, CS.Account
),
Commissions AS (
    SELECT 
        CS.FULLNAME AS Individual,
        CS.Account AS Account,
        CAST(SUM(CRD.TotalCommission) AS INT) AS Commission
    FROM freerider.sigma_input.commission AS CRD
    INNER JOIN freerider.sigma_input.Commission_split AS CS
        ON CRD.Account = CS.Account
    WHERE CRD.ResearchTeam <> 'Other'
      AND CRD.TradeDate >= {{dateRangeFilter-1}}.start -- Parameter substitution
      AND CRD.TradeDate <= {{dateRangeFilter-1}}.end -- Parameter substitution
    GROUP BY CS.FULLNAME, CS.Account
)
SELECT 
    COALESCE(RC.Individual, C.Individual) AS Individual,
    COALESCE(RC.Account, C.Account) AS Account,
    COALESCE(RC.Research_Cost, 0) AS Research_Cost,
    COALESCE(C.Commission, 0) AS Commission,
    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta
FROM ResearchCosts AS RC
FULL OUTER JOIN Commissions AS C
    ON RC.Individual = C.Individual AND RC.Account = C.Account
ORDER BY Individual ASC

%sql
WITH ResearchCosts AS (
    SELECT 
        research_splits AS Team,
        CAST(SUM(AdjustedValue) AS INT) AS Research_Cost
    FROM freerider.sigma_input.a2rawdata
    WHERE ParentBroker <> 'Other' 
      AND research_splits <> 'Compliance / Operations'
      AND `StartDate` >= COALESCE({{dateRangeFilter-1}}.start, DATE_TRUNC('year', CURRENT_DATE))
      AND `StartDate` <= COALESCE({{dateRangeFilter-1}}.end, CURRENT_DATE)
    GROUP BY research_splits
),
Commissions AS (
    SELECT 
        SplitTeam AS Team,
        CAST(SUM(TotalCommission) AS INT) AS Commission
    FROM freerider.sigma_input.commission
    WHERE ResearchTeam <> 'Other'
    AND `TradeDate` >= COALESCE({{dateRangeFilter-1}}.start, DATE_TRUNC('year', CURRENT_DATE))
    AND `TradeDate` <= COALESCE({{dateRangeFilter-1}}.end, CURRENT_DATE)
    GROUP BY SplitTeam
)
SELECT 
    COALESCE(RC.Team, C.Team) AS Team,
    COALESCE(RC.Research_Cost, 0) AS Research_Cost,
    COALESCE(C.Commission, 0) AS Commission,
    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta
FROM ResearchCosts RC
FULL OUTER JOIN Commissions C
    ON RC.Team = C.Team
ORDER BY Team ASC

%sql
WITH ResearchCosts AS (
    SELECT 
        A2.ParentBroker AS Brokers,
        A2.ResourceProvider AS Research_Name,
        CAST(SUM(A2.AdjustedValue) AS BIGINT) AS Research_Cost
    FROM freerider.sigma_input.a2rawdata AS A2
    WHERE A2.ParentBroker <> 'Other'
     AND A2.StartDate >= {{dateRangeFilter-1}}.start -- Parameter substitution
      AND A2.StartDate <= {{dateRangeFilter-1}}.end
    GROUP BY A2.ParentBroker, A2.ResourceProvider
),
Commissions AS (
    SELECT 
        CRD.ParentBroker AS Brokers,
        CAST(SUM(CRD.TotalCommission) AS INT) AS Commission
    FROM freerider.sigma_input.commission AS CRD
      where CRD.TradeDate >= {{dateRangeFilter-1}}.start -- Parameter substitution
      AND CRD.TradeDate <= {{dateRangeFilter-1}}.end
    GROUP BY CRD.ParentBroker
)
SELECT 
    COALESCE(RC.Brokers, C.Brokers) AS Brokers,
    COALESCE(RC.Research_Name, 'N/A') AS Research_Name,
    COALESCE(RC.Research_Cost, 0) AS Research_Cost,
    COALESCE(C.Commission, 0) AS Total_Commission,
    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta
FROM ResearchCosts AS RC
FULL OUTER JOIN Commissions AS C
    ON RC.Brokers = C.Brokers
ORDER BY Brokers ASC


%sql
WITH ResearchCosts AS (
    SELECT 
        A2.research_splits AS Team,
        A2.Attendee AS Name,
        CS.Account AS Account,
        ROUND(SUM(A2.AdjustedValue), 0) AS ResearchCost
    FROM freerider.sigma_input.a2rawdata A2
    JOIN freerider.sigma_input.commission_split CS 
        ON A2.Attendee = CS.FULLNAME
    WHERE A2.StartDate >= {{dateRangeFilter}}.start 
      AND A2.StartDate <= {{dateRangeFilter}}.end
      AND A2.ParentBroker ={{Parent-Broker}}
    GROUP BY A2.research_splits, A2.Attendee, CS.Account
),
Commissions AS (
    SELECT
        C.SplitTeam AS Team,
        CS.FULLNAME AS Name,
        C.Account AS Account,
        ROUND(SUM(C.TotalCommission), 0) AS Commission
    FROM freerider.sigma_input.commission C
    JOIN freerider.sigma_input.commission_split CS 
        ON C.Account = CS.Account
    WHERE C.TradeDate >= {{dateRangeFilter}}.start 
      AND C.TradeDate <= {{dateRangeFilter}}.end
      AND C.ParentBroker ={{Parent-Broker}}
    GROUP BY C.SplitTeam, CS.FULLNAME, C.Account
)
SELECT 
    COALESCE(RC.Team, C.Team) AS Team,
    COALESCE(RC.Name, C.Name) AS Name,
    COALESCE(C.Account, RC.Account) AS Account,
    COALESCE(RC.ResearchCost, 0) AS ResearchCost,
    COALESCE(C.Commission, 0) AS Commission,
    COALESCE(C.Commission, 0) - COALESCE(RC.ResearchCost, 0) AS Delta
FROM ResearchCosts RC
RIGHT JOIN Commissions C 
    ON RC.Name = C.Name
ORDER BY Team, Name, Account

%sql
SELECT 
    Attendee, 
    COUNT(*) AS Unreconciled
FROM 
    freerider.sigma_input.a2rawdata A2
WHERE 
    Status = 'None'
    AND (BuysideStatus != 'Canceled' OR BuysideStatus IS NULL)
    AND IsReviewRequired = true
    AND StartDate >= make_date(YEAR({{dateRangeFilter}}.start), 1, 1)
    AND `Value` > 750
GROUP BY 
    Attendee
ORDER BY 
    Unreconciled DESC 

```

## Sigma Integration

1. Connect to Databricks using JDBC.
2. Use SQL queries to extract insights from Delta tables.
3. Create real-time dashboards with the transformed data.

## Deployment

- Schedule a Databricks job to run the ETL pipeline at fixed intervals.
- Use Databricks Delta Live Tables (DLT) for continuous processing.
- Set up alerting and monitoring using Databricks or Sigma.

## Conclusion

This pipeline enables real-time data flow from SQL Server to Databricks and visualization in Sigma, ensuring up-to-date insights for decision-making.

## Contributions
Feel free to submit pull requests for improvements or additional features.

## License
This project is licensed under the **MIT License**.

## Contact
For issues or support, reach out via **GitHub Issues** or email the project maintainer.
