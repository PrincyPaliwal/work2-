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
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%sql\n",
    "WITH ResearchCosts AS (\n",
    "    SELECT \n",
    "        A2.ParentBroker AS Brokers,\n",
    "        A2.research_splits AS SplitTeam,\n",
    "        A2.ResourceProvider AS Research_Name,\n",
    "        CAST(SUM(A2.AdjustedValue) AS BIGINT) AS Research_Cost\n",
    "    FROM freerider.sigma_input.a2rawdata A2\n",
    "    WHERE A2.ParentBroker <> 'Other'\n",
    "     AND A2.StartDate >= {{dateRangeFilter-1}}.start -- Parameter substitution\n",
    "      AND A2.StartDate <= {{dateRangeFilter-1}}.end\n",
    "    GROUP BY A2.ParentBroker, A2.ResourceProvider, A2.research_splits\n",
    "\n",
    "),\n",
    "Commissions AS (\n",
    "    SELECT \n",
    "        CRD.ParentBroker AS Brokers,\n",
    "        CRD.SplitTeam AS SplitTeam,\n",
    "        CAST(SUM(CRD.TotalCommission) AS INT) AS Commission\n",
    "    FROM freerider.sigma_input.commission CRD\n",
    "where  CRD.TradeDate >= {{dateRangeFilter-1}}.start -- Parameter substitution\n",
    "      AND CRD.TradeDate <= {{dateRangeFilter-1}}.end\n",
    "    GROUP BY CRD.ParentBroker, CRD.SplitTeam\n",
    ")\n",
    "SELECT \n",
    "    COALESCE(RC.Brokers, C.Brokers) AS Brokers,\n",
    "    COALESCE(RC.SplitTeam, C.SplitTeam) AS SplitTeam,\n",
    "    COALESCE(RC.Research_Name, 'N/A') AS Research_Name,\n",
    "    COALESCE(RC.Research_Cost, 0) AS Research_Cost,\n",
    "    COALESCE(C.Commission, 0) AS Total_Commission,\n",
    "    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta\n",
    "FROM ResearchCosts RC\n",
    "FULL OUTER JOIN Commissions C\n",
    "    ON RC.Brokers = C.Brokers AND RC.SplitTeam = C.SplitTeam\n",
    "ORDER BY Brokers ASC"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%sql\n",
    "WITH ResearchCosts AS (\n",
    "    SELECT \n",
    "        CS.FULLNAME AS Individual,\n",
    "        CS.Account AS Account,\n",
    "        CAST(SUM(A2.AdjustedValue) AS INT) AS Research_Cost\n",
    "    FROM freerider.sigma_input.a2rawdata AS A2\n",
    "    INNER JOIN freerider.sigma_input.Commission_split AS CS\n",
    "        ON A2.Attendee = CS.FULLNAME\n",
    "    WHERE A2.ParentBroker <> 'Other'\n",
    "      AND A2.StartDate >= {{dateRangeFilter-1}}.start -- Parameter substitution\n",
    "      AND A2.StartDate <= {{dateRangeFilter-1}}.end -- Parameter substitution\n",
    "    GROUP BY CS.FULLNAME, CS.Account\n",
    "),\n",
    "Commissions AS (\n",
    "    SELECT \n",
    "        CS.FULLNAME AS Individual,\n",
    "        CS.Account AS Account,\n",
    "        CAST(SUM(CRD.TotalCommission) AS INT) AS Commission\n",
    "    FROM freerider.sigma_input.commission AS CRD\n",
    "    INNER JOIN freerider.sigma_input.Commission_split AS CS\n",
    "        ON CRD.Account = CS.Account\n",
    "    WHERE CRD.ResearchTeam <> 'Other'\n",
    "      AND CRD.TradeDate >= {{dateRangeFilter-1}}.start -- Parameter substitution\n",
    "      AND CRD.TradeDate <= {{dateRangeFilter-1}}.end -- Parameter substitution\n",
    "    GROUP BY CS.FULLNAME, CS.Account\n",
    ")\n",
    "SELECT \n",
    "    COALESCE(RC.Individual, C.Individual) AS Individual,\n",
    "    COALESCE(RC.Account, C.Account) AS Account,\n",
    "    COALESCE(RC.Research_Cost, 0) AS Research_Cost,\n",
    "    COALESCE(C.Commission, 0) AS Commission,\n",
    "    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta\n",
    "FROM ResearchCosts AS RC\n",
    "FULL OUTER JOIN Commissions AS C\n",
    "    ON RC.Individual = C.Individual AND RC.Account = C.Account\n",
    "ORDER BY Individual ASC"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%sql\n",
    "WITH ResearchCosts AS (\n",
    "    SELECT \n",
    "        research_splits AS Team,\n",
    "        CAST(SUM(AdjustedValue) AS INT) AS Research_Cost\n",
    "    FROM freerider.sigma_input.a2rawdata\n",
    "    WHERE ParentBroker <> 'Other' \n",
    "      AND research_splits <> 'Compliance / Operations'\n",
    "      AND `StartDate` >= COALESCE({{dateRangeFilter-1}}.start, DATE_TRUNC('year', CURRENT_DATE))\n",
    "      AND `StartDate` <= COALESCE({{dateRangeFilter-1}}.end, CURRENT_DATE)\n",
    "    GROUP BY research_splits\n",
    "),\n",
    "Commissions AS (\n",
    "    SELECT \n",
    "        SplitTeam AS Team,\n",
    "        CAST(SUM(TotalCommission) AS INT) AS Commission\n",
    "    FROM freerider.sigma_input.commission\n",
    "    WHERE ResearchTeam <> 'Other'\n",
    "    AND `TradeDate` >= COALESCE({{dateRangeFilter-1}}.start, DATE_TRUNC('year', CURRENT_DATE))\n",
    "    AND `TradeDate` <= COALESCE({{dateRangeFilter-1}}.end, CURRENT_DATE)\n",
    "    GROUP BY SplitTeam\n",
    ")\n",
    "SELECT \n",
    "    COALESCE(RC.Team, C.Team) AS Team,\n",
    "    COALESCE(RC.Research_Cost, 0) AS Research_Cost,\n",
    "    COALESCE(C.Commission, 0) AS Commission,\n",
    "    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta\n",
    "FROM ResearchCosts RC\n",
    "FULL OUTER JOIN Commissions C\n",
    "    ON RC.Team = C.Team\n",
    "ORDER BY Team ASC"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%sql\n",
    "WITH ResearchCosts AS (\n",
    "    SELECT \n",
    "        A2.ParentBroker AS Brokers,\n",
    "        A2.ResourceProvider AS Research_Name,\n",
    "        CAST(SUM(A2.AdjustedValue) AS BIGINT) AS Research_Cost\n",
    "    FROM freerider.sigma_input.a2rawdata AS A2\n",
    "    WHERE A2.ParentBroker <> 'Other'\n",
    "     AND A2.StartDate >= {{dateRangeFilter-1}}.start -- Parameter substitution\n",
    "      AND A2.StartDate <= {{dateRangeFilter-1}}.end\n",
    "    GROUP BY A2.ParentBroker, A2.ResourceProvider\n",
    "),\n",
    "Commissions AS (\n",
    "    SELECT \n",
    "        CRD.ParentBroker AS Brokers,\n",
    "        CAST(SUM(CRD.TotalCommission) AS INT) AS Commission\n",
    "    FROM freerider.sigma_input.commission AS CRD\n",
    "      where CRD.TradeDate >= {{dateRangeFilter-1}}.start -- Parameter substitution\n",
    "      AND CRD.TradeDate <= {{dateRangeFilter-1}}.end\n",
    "    GROUP BY CRD.ParentBroker\n",
    ")\n",
    "SELECT \n",
    "    COALESCE(RC.Brokers, C.Brokers) AS Brokers,\n",
    "    COALESCE(RC.Research_Name, 'N/A') AS Research_Name,\n",
    "    COALESCE(RC.Research_Cost, 0) AS Research_Cost,\n",
    "    COALESCE(C.Commission, 0) AS Total_Commission,\n",
    "    COALESCE(C.Commission, 0) - COALESCE(RC.Research_Cost, 0) AS Delta\n",
    "FROM ResearchCosts AS RC\n",
    "FULL OUTER JOIN Commissions AS C\n",
    "    ON RC.Brokers = C.Brokers\n",
    "ORDER BY Brokers ASC"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%sql\n",
    "WITH ResearchCosts AS (\n",
    "    SELECT \n",
    "        A2.research_splits AS Team,\n",
    "        A2.Attendee AS Name,\n",
    "        CS.Account AS Account,\n",
    "        ROUND(SUM(A2.AdjustedValue), 0) AS ResearchCost\n",
    "    FROM freerider.sigma_input.a2rawdata A2\n",
    "    JOIN freerider.sigma_input.commission_split CS \n",
    "        ON A2.Attendee = CS.FULLNAME\n",
    "    WHERE A2.StartDate >= {{dateRangeFilter}}.start \n",
    "      AND A2.StartDate <= {{dateRangeFilter}}.end\n",
    "      AND A2.ParentBroker ={{Parent-Broker}}\n",
    "    GROUP BY A2.research_splits, A2.Attendee, CS.Account\n",
    "),\n",
    "Commissions AS (\n",
    "    SELECT\n",
    "        C.SplitTeam AS Team,\n",
    "        CS.FULLNAME AS Name,\n",
    "        C.Account AS Account,\n",
    "        ROUND(SUM(C.TotalCommission), 0) AS Commission\n",
    "    FROM freerider.sigma_input.commission C\n",
    "    JOIN freerider.sigma_input.commission_split CS \n",
    "        ON C.Account = CS.Account\n",
    "    WHERE C.TradeDate >= {{dateRangeFilter}}.start \n",
    "      AND C.TradeDate <= {{dateRangeFilter}}.end\n",
    "      AND C.ParentBroker ={{Parent-Broker}}\n",
    "    GROUP BY C.SplitTeam, CS.FULLNAME, C.Account\n",
    ")\n",
    "SELECT \n",
    "    COALESCE(RC.Team, C.Team) AS Team,\n",
    "    COALESCE(RC.Name, C.Name) AS Name,\n",
    "    COALESCE(C.Account, RC.Account) AS Account,\n",
    "    COALESCE(RC.ResearchCost, 0) AS ResearchCost,\n",
    "    COALESCE(C.Commission, 0) AS Commission,\n",
    "    COALESCE(C.Commission, 0) - COALESCE(RC.ResearchCost, 0) AS Delta\n",
    "FROM ResearchCosts RC\n",
    "RIGHT JOIN Commissions C \n",
    "    ON RC.Name = C.Name\n",
    "ORDER BY Team, Name, Account"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%sql\n",
    "SELECT \n",
    "    Attendee, \n",
    "    COUNT(*) AS Unreconciled\n",
    "FROM \n",
    "    freerider.sigma_input.a2rawdata A2\n",
    "WHERE \n",
    "    Status = 'None'\n",
    "    AND (BuysideStatus != 'Canceled' OR BuysideStatus IS NULL)\n",
    "    AND IsReviewRequired = true\n",
    "    AND StartDate >= make_date(YEAR({{dateRangeFilter}}.start), 1, 1)\n",
    "    AND `Value` > 750\n",
    "GROUP BY \n",
    "    Attendee\n",
    "ORDER BY \n",
    "    Unreconciled DESC"
   ]
  }
 ],
 "metadata": {
  "application/vnd.databricks.v1+notebook": {
   "computePreferences": null,
   "dashboards": [],
   "environmentMetadata": {
    "base_environment": "",
    "environment_version": "2"
   },
   "language": "python",
   "notebookMetadata": {
    "pythonIndentUnit": 4
   },
   "notebookName": "Research And Commission Cost Dashboard For Traders",
   "widgets": {}
  },
  "language_info": {
   "name": "python"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 0
}

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
