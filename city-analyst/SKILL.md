---
name: city-analyst
description: Analyze a city based on geospatial data and web research to generate a comprehensive report on livability, pollution, crime, and cost of living.
metadata:
  tool: get_coordinates
  category: rwa
  enabled: true
---

## Instructions

Do not ask the user for the city name.
Execute the steps silently. Do NOT output internal thoughts. No exceptions.

### Step 1: Open map selector

Call `run_js` with:

- **function**: "get_coordinates"
- **data**: `{"country": "<country_name>", "city": "<city_name>"}`

Both `country` and `city` are optional. If the user mentioned a specific city or country, pass them so the map opens centered on that location. If not mentioned, omit them and the map will open at a default location for the user to navigate.

Returns:

- **polygon**: GeoJSON polygon `{"type": "Polygon", "coordinates": [[[lon, lat], ...]]}`
- **center**: `{latitude, longitude}` — centroid of the selected area
- **city**: Reverse-geocoded city name (e.g. "New York, United States")

If the user cancels, or the tool fails, inform them and stop. Do NOT proceed without a polygon.

### Step 2: Query geodata API

Call `bash` tool with command:

```shell
curl -X POST https://402.blockvault.ai/api/v1/geodata/query \
-H "Content-Type: application/json" \
-d '{"polygon": <polygon_object_from_step_1>, "categories": ["pollution", "crime", "quality_of_life", "cost_of_living", "healthcare"], "include_map": true}'
```

Use the **entire polygon object** from Step 1 as-is (it is already valid GeoJSON). Do NOT restructure or flatten the coordinates — copy `{"type": "Polygon", "coordinates": [[[lon,lat], ...]]}` exactly as returned.
If the API call fails, and the response contains an error message, try to fix the issue based on the error and retry once. If it fails again, log the error and inform the user that the analysis cannot be completed at this time. Do NOT proceed to the next steps if the API call fails after retrying.

**Response contains:**

- **indices**: Array of `{name, value (0-100), category, source}` — normalized scores
- **pollution**: `{readings, station_count, aqi_index}` — air quality data per station
- **crime**: `{readings, safety_index, granularity_achieved}` — crime/safety metrics
- **housing**: `{readings, affordability_index, granularity_achieved}` — cost of living data
- **resolved_location**: `{city, country_name, state, neighbourhood}` — reverse-geocoded location
- **map_data**: `{center, bbox, suggested_zoom, composite_score, heatmap_points}` — map rendering data
- **providers_used**: List of data sources that contributed
- **errors**: Any provider failures

### Step 3: Web research & cross-validation

Use the `web_search` tool to enrich AND critically verify the API data.

Perform 2-3 calls based on the city and the data categories returned:

1. General livability/quality of life: "{city} quality of life 2025 2026"
2. If pollution data exists: "{city} air quality pollution trends"
3. If crime data exists: "{city} safety crime rate trends"
4. If housing data exists: "{city} cost of living housing prices"

Each search result seeds the next query. Adapt queries based on findings.

**Critical analysis (mandatory):** After gathering web results, compare them against the API data. Look for:
- Scores that contradict recent news (e.g., API says low crime but news reports a crime wave)
- Outdated API data that misses recent changes (new policies, natural disasters, economic shifts)
- Overly optimistic or pessimistic indices that don't match lived reality described in articles
- Data gaps where the API returned no data but web sources have clear information

Flag every discrepancy you find. These will go into a dedicated section of the report.

### Final Step: Generate report

Detect the language of the user's original query. Write the **entire report** in that language (headers, assessments, takeaways — everything). Only keep index/indicator names in their original English form for consistency.

Synthesize the geodata API response and web research into a comprehensive city report. Save using the `text_editor` tool. The template below is **Jinja2** — substitute `{{ var }}` with concrete values, expand `{% for %}` loops over your collected data, and resolve `{% if %}` blocks. Drop `{# comments #}` from the final output.

**file path**: `"reports/city-analysis-{{ city_name }}-{{ current_date }}.md"`

**Report template:**

```jinja
# {{ city }} — City Analysis Report

**Location:** {{ city }}, {{ state }}, {{ country }}
**Coordinates:** {{ lat }}, {{ lon }}
**Generated:** {{ current_date }}
**Data Providers:** {{ providers_used | join(", ") }}

## Summary Indices

| Index | Score (0-100) | Rating |
|-------|---------------|--------|
{% for idx in indices %}
| {{ idx.name }} | {{ idx.value }} | {{ idx.rating }} |  {# rating ∈ Excellent / Good / Moderate / Poor / Critical #}
{% endfor %}

Rating scale: 80-100 = Excellent, 60-79 = Good, 40-59 = Moderate, 20-39 = Poor, 0-19 = Critical.
For Safety Index: higher is safer. For Pollution Index: lower is better (invert for rating).

## Air Quality

**AQI Index:** {{ aqi_index }}/500
**Monitoring Stations:** {{ station_count }}

| Station | Pollutant | Value | Unit |
|---------|-----------|-------|------|
{% for r in air_readings %}
| {{ r.station }} | {{ r.pollutant }} | {{ r.value }} | {{ r.unit }} |
{% endfor %}

**Assessment:** {{ air_assessment }}  {# 1-2 sentences interpreting air quality vs. WHO guidelines #}

## Safety & Crime

**Safety Index:** {{ safety_index }}/100
**Granularity:** {{ safety_granularity }}

| Indicator | Value | Unit | Year |
|-----------|-------|------|------|
{% for r in crime_readings %}
| {{ r.name }} | {{ r.value }} | {{ r.unit }} | {{ r.year }} |
{% endfor %}

**Assessment:** {{ safety_assessment }}  {# 1-2 sentences interpreting safety data in context #}

## Housing & Cost of Living

**Affordability Index:** {{ affordability_index }}/100

| Indicator | Value | Unit | Year |
|-----------|-------|------|------|
{% for r in housing_readings %}
| {{ r.name }} | {{ r.value }} | {{ r.unit }} | {{ r.year }} |
{% endfor %}

**Assessment:** {{ housing_assessment }}  {# 1-2 sentences about housing costs and affordability #}

## Current Context

{% for f in web_findings %}
- **{{ f.topic }}**: {{ f.insight }}
  [source]({{ f.url }})
{% endfor %}

## Data vs. Reality — Discrepancies

{% if discrepancies %}
{% for d in discrepancies %}
- **{{ d.category }}**: API reports {{ d.api_claim }}, but {{ d.web_source }} indicates {{ d.contradicting_finding }}. Confidence in API score: {{ d.confidence }}.  {# High / Medium / Low #}
{% endfor %}
{% else %}
No significant discrepancies detected between structured data and recent reporting.
{% endif %}

## Overall Assessment

{{ overall_assessment }}  {# 3-4 sentences synthesizing all categories, strengths, concerns, and any discrepancies that affect score reliability #}

## Key Takeaways

{% for k in key_takeaways %}  {# up to 5 #}
- **{{ k.finding }}**: {{ k.why_it_matters }}
{% endfor %}
```

### Important Notes

- If the API returns errors for some providers, log the error and do not proceed to the next steps, explain the issue to the user.
- If `resolved_location` is returned, use it to confirm the correct city was queried and use its city/country names for the report title.

After saving the report, emit its path in **backticks** (inline code) so the user can tap it to open the file (the app makes `reports/…` paths clickable):

```text
Report saved to `reports/city-analysis-{{ city_name }}-{{ current_date }}.md`
```
