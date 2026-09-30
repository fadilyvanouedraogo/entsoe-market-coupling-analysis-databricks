# Power BI — D2 measures

Connect Power BI Desktop to the SQL warehouse (*Get data → Databricks*), import the tables of `energy.d2_gold`
(9 tables: 5 dimensions, 4 facts), then create the relationships below and the measures (in a `D2 Measures` table).

## Relationships

| From (many) | To (one) | Active |
|---|---|---|
| `fact_price_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_price_hourly.hour_key` | `dim_hour.hour_key` | yes |
| `fact_price_hourly.zone_code` | `dim_zone.zone_code` | yes |
| `fact_border_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_border_hourly.hour_key` | `dim_hour.hour_key` | yes |
| `fact_border_hourly.border_key` | `dim_border.border_key` | yes |
| `fact_route_flow_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_route_flow_hourly.hour_key` | `dim_hour.hour_key` | yes |
| `fact_route_flow_hourly.route_key` | `dim_route.route_key` | yes |
| `fact_region_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_region_hourly.hour_key` | `dim_hour.hour_key` | yes |

Mark `dim_date` as date table (column `date`).

## Flow map

Install a flow-map visual from AppSource (e.g. *Flow Map* or *Route Map*) and map: origin = `dim_route[from_label]`, destination = `dim_route[to_label]`, origin latitude/longitude = `from_lat`/`from_lon`, destination latitude/longitude = `to_lat`/`to_lon`, width = `Net physical flow (GWh)`. Over the selected period only the dominant direction of each border has a non-zero net flow, so the map shows one arrow per border, pointing to the net importer.

## Prices

```dax
Average price (€/MWh) =
AVERAGE ( fact_price_hourly[price_eur_mwh] )
```
Format: `0.00`

```dax
Min price (€/MWh) =
MIN ( fact_price_hourly[price_eur_mwh] )
```
Format: `0.00`

```dax
Max price (€/MWh) =
MAX ( fact_price_hourly[price_eur_mwh] )
```
Format: `0.00`

```dax
Volatility (std dev €/MWh) =
STDEV.P ( fact_price_hourly[price_eur_mwh] )
```
Format: `0.00`

```dax
Negative price hours =
CALCULATE ( COUNTROWS ( fact_price_hourly ), fact_price_hourly[is_negative_price] = TRUE () ) + 0
```
Format: `#,0`

```dax
% negative price hours =
DIVIDE ( [Negative price hours], COUNTROWS ( fact_price_hourly ) )
```
Format: `0.0%`

```dax
BE vs CWE average gap (€/MWh) =
VAR be = CALCULATE ( AVERAGE ( fact_price_hourly[price_eur_mwh] ), dim_zone[zone_code] = "BE" )
VAR cwe = CALCULATE ( AVERAGE ( fact_price_hourly[price_eur_mwh] ), dim_zone[is_cwe] = TRUE () )
RETURN be - cwe
```
Format: `+0.00;-0.00;0.00`

## Net position

```dax
Average net position (MW) =
AVERAGE ( fact_price_hourly[net_position_mwh] )
```
Format: `#,0`

```dax
Net exports (GWh) =
DIVIDE ( SUM ( fact_price_hourly[net_position_mwh] ), 1000 )
```
Format: `#,0.0`

```dax
% exporting hours =
DIVIDE (
    CALCULATE ( COUNTROWS ( fact_price_hourly ), fact_price_hourly[net_position_mwh] > 0 ),
    CALCULATE ( COUNTROWS ( fact_price_hourly ), NOT ISBLANK ( fact_price_hourly[net_position_mwh] ) )
)
```
Format: `0.0%`

```dax
Import/export status =
IF ( ISBLANK ( [Net exports (GWh)] ), BLANK (),
    IF ( [Net exports (GWh)] >= 0, "Net exporter", "Net importer" ) )
```

## Borders

```dax
% converged hours =
DIVIDE (
    CALCULATE ( COUNTROWS ( fact_border_hourly ), fact_border_hourly[is_converged] = TRUE () ),
    CALCULATE ( COUNTROWS ( fact_border_hourly ), NOT ISBLANK ( fact_border_hourly[spread_a_b_eur_mwh] ) )
)
```
Format: `0.0%`

```dax
% congested hours =
DIVIDE (
    CALCULATE ( COUNTROWS ( fact_border_hourly ), fact_border_hourly[is_congested] = TRUE () ),
    CALCULATE ( COUNTROWS ( fact_border_hourly ), NOT ISBLANK ( fact_border_hourly[spread_a_b_eur_mwh] ) )
)
```
Format: `0.0%`

```dax
Average absolute spread (€/MWh) =
AVERAGE ( fact_border_hourly[abs_spread_eur_mwh] )
```
Format: `0.00`

```dax
Average spread A − B (€/MWh) =
AVERAGE ( fact_border_hourly[spread_a_b_eur_mwh] )
```
Format: `+0.00;-0.00;0.00`

```dax
Average spread when congested (€/MWh) =
CALCULATE ( AVERAGE ( fact_border_hourly[abs_spread_eur_mwh] ), fact_border_hourly[is_congested] = TRUE () )
```
Format: `0.00`

```dax
Estimated congestion rent (M€) =
DIVIDE ( SUM ( fact_border_hourly[congestion_rent_eur] ), 1000000 )
```
Format: `#,0.00`

```dax
% adverse flow hours =
DIVIDE (
    CALCULATE ( COUNTROWS ( fact_border_hourly ), fact_border_hourly[is_adverse_flow] = TRUE () ),
    CALCULATE ( COUNTROWS ( fact_border_hourly ), fact_border_hourly[is_congested] = TRUE (), NOT ISBLANK ( fact_border_hourly[net_flow_a_to_b_mwh] ) )
)
```
Format: `0.0%`

## Flow map

```dax
Physical flow (GWh) =
DIVIDE ( SUM ( fact_route_flow_hourly[flow_mwh] ), 1000 )
```
Format: `#,0.0`

```dax
Net physical flow (GWh) =
DIVIDE (
    SUMX (
        VALUES ( dim_route[route_key] ),
        MAX ( 0, CALCULATE ( SUM ( fact_route_flow_hourly[flow_mwh] ) - SUM ( fact_route_flow_hourly[reverse_flow_mwh] ) ) )
    ),
    1000
)
```
Format: `#,0.0`

```dax
Net scheduled exchange (GWh) =
DIVIDE (
    SUMX (
        VALUES ( dim_route[route_key] ),
        MAX ( 0, CALCULATE ( SUM ( fact_route_flow_hourly[sched_mwh] ) - SUM ( fact_route_flow_hourly[reverse_sched_mwh] ) ) )
    ),
    1000
)
```
Format: `#,0.0`

```dax
Hours with net flow on route =
CALCULATE ( COUNTROWS ( fact_route_flow_hourly ), fact_route_flow_hourly[net_flow_mwh] > 0 ) + 0
```
Format: `#,0`

## CWE region

```dax
% CWE full convergence hours =
DIVIDE ( CALCULATE ( COUNTROWS ( fact_region_hourly ), fact_region_hourly[is_full_convergence] = TRUE () ),
    COUNTROWS ( fact_region_hourly ) )
```
Format: `0.0%`

```dax
Average CWE max spread (€/MWh) =
AVERAGE ( fact_region_hourly[max_spread_eur_mwh] )
```
Format: `0.00`

```dax
Average distinct CWE prices =
AVERAGE ( fact_region_hourly[n_price_groups] )
```
Format: `0.00`

```dax
% hours BE most expensive =
DIVIDE ( CALCULATE ( COUNTROWS ( fact_region_hourly ), fact_region_hourly[be_is_highest] = TRUE () ),
    COUNTROWS ( fact_region_hourly ) )
```
Format: `0.0%`

```dax
% hours BE cheapest =
DIVIDE ( CALCULATE ( COUNTROWS ( fact_region_hourly ), fact_region_hourly[be_is_lowest] = TRUE () ),
    COUNTROWS ( fact_region_hourly ) )
```
Format: `0.0%`
