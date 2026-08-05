# EUROCONTROL Airport-Arrival ATFM Delay Codes

The EUROCONTROL Airport Arrival ATFM Delay dataset assigns delay minutes to the regulation reason considered responsible for the delay. The following table documents the reason codes represented by columns such as `DLY_APT_ARR_C_1` and `DLY_APT_ARR_W_1`.

| Full delay code | Delay reason | Explanation |
|---|---|---|
| `DLY_APT_ARR_A_1` | Accident/Incident | Delay caused by an aviation accident, incident, emergency, or related operational response that reduces airport or terminal-area capacity. This can include runway closures, emergency inspections, or restrictions needed to manage the event safely. |
| `DLY_APT_ARR_C_1` | ATC Capacity | Delay imposed when expected traffic exceeds the amount that the responsible air traffic control unit can safely handle. The constraint may result from traffic volume, complexity, sector configuration, or a temporarily reduced arrival rate. |
| `DLY_APT_ARR_D_1` | De-icing | Delay associated with aircraft de-icing or anti-icing operations, or with treatment of runways and taxiways. These activities increase turnaround and ground-movement times and may reduce the airport's sustainable arrival rate. |
| `DLY_APT_ARR_E_1` | Equipment (non-ATC) | Delay caused by the failure or reduced availability of airport equipment or services not operated as part of ATC. Examples can include airfield lighting, navigation-related airport infrastructure, ground systems, or other essential aerodrome facilities. |
| `DLY_APT_ARR_G_1` | Aerodrome Capacity | Delay caused by limitations in airport infrastructure or ground operations. Typical constraints include insufficient runway, taxiway, stand, or gate capacity; runway works; closures; or surface congestion. |
| `DLY_APT_ARR_I_1` | Industrial Action (ATC) | Delay caused by strikes, work stoppages, or other organised labour action involving air traffic controllers or ATC personnel, resulting in reduced ATC capacity. |
| `DLY_APT_ARR_M_1` | Airspace Management | Delay caused by the allocation, configuration, or temporary restriction of available airspace. Examples include military reservations, restricted areas, or reduced civil-route availability that limits the flow of traffic into an airport. |
| `DLY_APT_ARR_N_1` | Industrial Action (non-ATC) | Delay caused by strikes or other organised labour action involving personnel outside ATC, such as airport operators, ground handlers, security staff, firefighters, or baggage handlers. |
| `DLY_APT_ARR_O_1` | Other | Delay caused by an identified operational reason that does not fit any of the more specific ATFM reason categories. |
| `DLY_APT_ARR_P_1` | Special Event | Delay associated with an unusual planned event that increases traffic demand or introduces temporary operating restrictions. Examples include major sporting events, international summits, state visits, air shows, or large public gatherings. |
| `DLY_APT_ARR_R_1` | ATC Routeing | Delay caused by routes imposed, changed, or made unavailable by ATC to manage traffic safely. Alternative or reduced routing options can concentrate traffic onto fewer corridors and require the flow to be regulated. |
| `DLY_APT_ARR_S_1` | ATC Staffing | Delay caused by insufficient ATC personnel to operate the required number of controller positions or sectors, reducing the amount of traffic that can be handled safely. |
| `DLY_APT_ARR_T_1` | Equipment (ATC) | Delay caused by the failure, degradation, maintenance, or reduced availability of ATC systems such as radar, communications, surveillance, flight-data-processing, or controller working-position equipment. |
| `DLY_APT_ARR_V_1` | Environmental Issues | Delay caused by environmental restrictions or conditions that limit airport operations or capacity. Examples can include noise-abatement requirements, emissions-related restrictions, or other environmental constraints. |
| `DLY_APT_ARR_W_1` | Weather | Delay caused by weather that reduces airport or terminal-area capacity, such as fog, low visibility, strong winds, thunderstorms, snow, or adverse runway conditions. De-icing-specific constraints are recorded separately under code `D`. |
| `DLY_APT_ARR_NA_1` | Not specified | Delay for which the regulation reason was not specified, not available, or not assigned to another reason category. |

## Dataset field convention

Each reason code is embedded in its corresponding dataset column name:

```text
DLY_APT_ARR_<CODE>_1
```

For example, `DLY_APT_ARR_C_1` contains airport-arrival ATFM delay minutes attributed to **ATC Capacity**, while `DLY_APT_ARR_W_1` contains minutes attributed to **Weather**. `DLY_APT_ARR_1` contains the total airport-arrival ATFM delay minutes across all reason codes.

## Terminology note

This documentation follows the labels in EUROCONTROL's **Airport Arrival ATFM Delay dataset dictionary**. EUROCONTROL's newer general ATFM taxonomy labels code `E` as **Aerodrome Services** and code `M` as **Military Activity**. When processing historical dataset columns, use the definitions supplied with the particular dataset release.

## Sources

- [EUROCONTROL Aviation Intelligence Portal — Airport Arrival ATFM Delay Dataset](https://ansperformance.eu/reference/dataset/airport-arrival-atfm-delay/)
- [EUROCONTROL Aviation Intelligence Portal — ATFM Delay Codes](https://ansperformance.eu/definition/atfm-delay-codes/)
