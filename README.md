# CRISP-DM Phase 4: Creating an Interactive Netflix Dashboard

In the previous tutorials, we created interactive charts and maps with Plotly Express. We will now combine selected figures and key performance indicators (KPIs) into a single interactive dashboard that can be opened in a web browser.

## Learning objectives

By the end of this tutorial, you will be able to:

* create KPIs from a DataFrame;
* convert Plotly figures into HTML fragments;
* describe a dashboard layout in a prompt;
* use generative AI to create the HTML and CSS structure of a dashboard; and
* export the complete dashboard as an HTML file.

## Business question

How can we present Netflix audience trends, content performance, distributions, and geographic information in a single interactive page that supports business analysis?

---

## 1. Load the libraries and datasets

Upload the following files to Google Colab:

* `netflix_weekly_clean.csv`;
* `country_weekly.tsv`.

Load the libraries:

```python
import pandas as pd
import plotly.express as px
import requests
from plotly.io import to_html
```

Load the datasets:

```python
df = pd.read_csv("netflix_weekly_clean.csv")
netflix_df = pd.read_csv("country_weekly.tsv", sep="\t")
```
Change the format of the data:

```python
category_names = {
    1: "Films (English)",
    2: "Films (Non-English)",
    3: "TV (English)",
    4: "TV (Non-English)"
}
df["week"] = pd.to_datetime(df["week"])
df["month"] = df["week"].dt.month
df["year"] = df["week"].dt.year
df["category"] = df["category"].map(category_names)
df.info()
```

---

## 2. Calculate the KPIs

The dashboard will display the top title in the latest week, its weekly views, its cumulative number of weeks in the Top 10, and the all-time average weekly audience.

First, select the latest top title:
```python
latest_top = df[
    (df["week"] == df["week"].max())
    & (df["category"] == "Films (English)")
    & (df["weekly_rank"] == 1)
]
print(latest_top)
```

Now, select data from the columns, format the results, and compute the all-time average weekly audience:
```python
top_title = latest_top["show_title"][0]
top_title_views = latest_top["weekly_views"][0]
top_title_cumulative = latest_top["cumulative_weeks_in_top_10"][0]
mean_weekly_views = df["weekly_views"].mean()

top_title_views_text = f"{top_title_views / 1000000:.1f}M"
mean_weekly_views_text = f"{mean_weekly_views / 1000000:.1f}M"

print("Top title:", top_title)
print("Weekly views:", top_title_views_text)
print("Mean weekly views:", mean_weekly_views_text)
print("Cumulative weeks in the Top 10:", top_title_cumulative)
```

The variables ending in `_text` contain abbreviated values that will be displayed directly in the dashboard.

---

## 3. Create the figures

The dashboard will use figures created in the previous tutorials. Run the following code to recreate the selected figures.


### Line chart 1

```python
df["week"] = pd.to_datetime(df["week"])
weekly_audience = (df.groupby("week", as_index=False)["weekly_views"].sum())
print(weekly_audience)

fig_line_1 = px.line(
    weekly_audience,
    x="week",
    y="weekly_views",
    title="Netflix Weekly Audience Over Time",
    labels={"week": "Week", "weekly_views": "Weekly Views"},
    markers=True,
    template="xgridoff"
)
fig_line_1.update_layout(title_x=0.5)
fig_line_1.show()
```


### Line chart 2

```python
selected_category = "Films (English)"
category_data = df[df["category"] == selected_category]
latest_week = df["week"].max()
latest_ranking = category_data[category_data["week"] == latest_week]
title_history = category_data[
    category_data["show_title"].isin(latest_ranking["show_title"])
].sort_values("week")

fig_line_4 = px.line(
    title_history,
    x="week",
    y="weekly_views",
    color="show_title",
    markers=True,
    title="Audience Trends of the Latest Top 10 Titles",
    labels={
        "week": "Week",
        "weekly_views": "Weekly Views",
        "show_title": "Title"
    },
    template="plotly_white"
)
fig_line_4.update_layout(title_x=0.5)
fig_line_4.show()

```


### Line chart 3

```python
monthly_audience = df.groupby(["month", "category"], as_index=False)["weekly_views"].sum()
print(monthly_audience)

fig_line_checkpoint = px.line(
    monthly_audience,
    x="month",
    y="weekly_views",
    title="Views by month and category",
    labels={
        "month": "Month",
        "weekly_views": "Weekly Views",
        "category": "Category"
    },
    markers=True,
    template="plotly_white",
    color="category",

)
fig_line_checkpoint.update_xaxes(range=[1, 12], dtick=1)
fig_line_checkpoint.update_layout(title_x=0.5)
fig_line_checkpoint.show()
```


### Bar chart

```python
rank_category_audience = df.groupby(["weekly_rank", "category"], as_index=False)["weekly_views"].mean()
print(rank_category_audience)

fig_bar_3 = px.bar(
    rank_category_audience,
    x="weekly_rank",
    y="weekly_views",
    title="Average Weekly Audience by Rank and Category",
    labels={
        "weekly_rank": "Weekly Rank",
        "weekly_views": "Average Weekly Views",
        "category": "Category"
    },
    color = "category",
    barmode="group",
    animation_frame="category",
    range_y=[0,rank_category_audience["weekly_views"].max() * 1.1]
)

fig_bar_3.update_layout(title_x=0.5)
fig_bar_3.show()
```


### Histogram

```python
title_longevity = df.groupby(["show_title", "category"], as_index=False)["week"].nunique()
title_longevity = title_longevity.rename(columns={"week": "weeks_in_top_10"})
print(title_longevity)

fig_hist_1 = px.histogram(
    title_longevity,
    x="weeks_in_top_10",
    title="Time Spent in the Netflix Top 10",
    labels={
        "weeks_in_top_10": "Weeks in the Top 10"
    },
    histnorm ="percent",
    marginal = "box",
    template = "presentation"
    )

fig_hist_1.update_layout(title_x=0.5)
fig_hist_1.update_layout(bargap=0.2)
fig_hist_1.update_traces(
    marker_line_color="black",
    marker_line_width=1,
    selector={"type": "histogram"}
)
fig_hist_1.show()
```


### Pie chart

```python
category_audience = df.groupby("category", as_index=False)["weekly_views"].sum()
print(category_audience)

fig_pie_1 = px.pie(
    category_audience,
    names="category",
    values="weekly_views",
    title="Netflix Audience Share by Category",
    labels={
        "category": "Category",
        "weekly_views": "Total Views",    },
    hole = 0.1,
    color_discrete_sequence=px.colors.qualitative.Vivid,
    template  = "presentation"
)
fig_pie_1.update_layout(title_x=0.5)
fig_pie_1.update_traces(textposition="outside",
                        textinfo="percent+label",
                        textfont_size=14,
                        insidetextorientation="tangential",
                        pull = [0.1, 0.1, 0.1, 0.1],
                        marker_line_color = "black",
                        marker_line_width = 2,
                        sort =True,
                        rotation = 90,
                       direction = "clockwise"
                        )
fig_pie_1.show()
```


### World map

```python
selected_title = "KPop Demon Hunters"
netflix_map = netflix_df[netflix_df["show_title"] == selected_title]
print(netflix_map)

fig_world = px.choropleth(
    netflix_map,
    locations="country_name",
    locationmode="country names",
    color="weekly_rank",
    animation_frame="week",
    title=f"Weekly Ranking of {selected_title} by Country",
    labels={
        "weekly_rank": "Weekly Rank",
        "week": "Week"
    },
    height = 1000,
    template = "presentation",
    color_continuous_scale= "purples_r",
    projection ="orthographic"
)
fig_world.update_layout(title_x=0.5)
fig_world.show()
```

---


### 4. Adjust the graphs

ue the following code to correct the adjustment of the figures and the size of the texts.
```python
figures = [
    fig_line_1,
    fig_line_4,
    fig_line_checkpoint,
    fig_bar_3,
    fig_hist_1,
    fig_pie_1,
    fig_world
]

for fig in figures:
    fig.update_layout(
        autosize=True,
        height=None,
        title_font_size=14,
        font_size=10,
        legend={"font": {"size": 9}},
        margin={"l": 50, "r": 30, "t": 60, "b": 50}
    )

    fig.update_xaxes(
        automargin=True,
        title_font={"size": 11},
        tickfont={"size": 9}
    )

    fig.update_yaxes(
        automargin=True,
        title_font={"size": 11},
        tickfont={"size": 9}
    )

    fig.update_annotations(font_size=9)
```


## 5. Convert the figures to HTML

The `to_html()` function converts each Plotly figure into an HTML fragment that preserves its interactivity.

Only the first figure must include the Plotly JavaScript library. The remaining figures use the same copy of the library.

```python
html_line_1 = to_html(fig_line_1, include_plotlyjs=True, full_html=False )
html_line_4 = to_html(fig_line_4, include_plotlyjs=False, full_html=False )
html_line_checkpoint = to_html( fig_line_checkpoint, include_plotlyjs=False, full_html=False )
html_bar_3 = to_html( fig_bar_3, include_plotlyjs=False, full_html=False )
html_hist_1 = to_html( fig_hist_1, include_plotlyjs=False, full_html=False )
html_pie_1 = to_html( fig_pie_1, include_plotlyjs=False, full_html=False )
html_world = to_html( fig_world, include_plotlyjs=False, full_html=False )
```

`include_plotlyjs=True` includes Plotly in the HTML file. Therefore, the dashboard can be opened without an internet connection.

`full_html=False` returns only the fragment required to insert the figure into another HTML page.

---

## 6. Ask an AI tool to create the HTML structure

Create a prompt and provide it to an AI tool:

```text
Create a single HTML page for an interactive dashboard for a data-driven merchandising company. Include the CSS inside a <style> element in the HTML <head>. Do not create separate files.

Return only one Python assignment with this structure:

html = f"""
<!DOCTYPE html>
<html lang="en">
<head>
    <style>
        ...
    </style>
</head>
<body>
    ...
</body>
</html>
"""

Insert these KPI values:

- {top_title}: number one title in the latest week
- {top_title_views_text}: weekly views of the number one title
- {mean_weekly_views_text}: mean weekly views
- {top_title_cumulative}: Cumulative weeks on top

Insert these Plotly HTML fragments:

- {html_line_1}
- {html_line_4}
- {html_line_checkpoint}
- {html_bar_3}
- {html_hist_1}
- {html_pie_1}
- {html_world}

Use the following layout:

|---------------------------------------------------------|
│ html_line_1                  |           KPI cards      | 
|---------------------------------------------------------|
| html_pie_1    |   html_bar_3   │      html_line_4       |  
|---------------------------------------------------------|
| html_line_checkpoint |   html_hist_1   |   html_world   |  
|---------------------------------------------------------|

Layout requirements:

- Make the first row shorter than the other rows.
- Make every figure fit inside its card.
- Display the dashboard in one column on small screens.
- Scale every Plotly figure to the exact width and height of its assigned grid cell, regardless of the original figure dimensions. Do not crop or hide any part of the figures.
- After loading the page and whenever the window size changes, use Plotly.relayout() to set each figure's width and height to the dimensions of its parent card.
- Design the dashboard for a browser viewport of approximately 1620 × 1000 pixels.


Design requirements:
- Use the title "Netflix Merchandising Opportunities Dashboard".
- Use the KPI labels "Latest Number One Title", "Weekly Views", "Average Weekly Views", and "Weeks in the Top 10".

- Use a clean, professional Netflix-inspired design with red (#E50914), black (#141414), white (#FFFFFF), and light gray (#F5F5F5).

Do not recreate the Plotly figures or change their data. Do not use Dash or external frameworks.

Because the result is a Python f-string, write all CSS braces as double braces: {{ and }}. Keep single braces only around the Python variables listed above.
```

Other options you can use as inspiration for specifying the colors of the webpage:

```text
- Use a light theme with a white background, light gray cards, black text, and red accents.
- Use a professional corporate palette with navy blue, white, light gray, and turquoise accents.
- Use a modern palette with dark purple, violet, white, and soft gray.
```

### Example of generated code:
```text
html = f"""
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <style>
        * {{
            box-sizing: border-box;
        }}

        html,
        body {{
            width: 100%;
            height: 100%;
            margin: 0;
            overflow: hidden;
            background: #F5F5F5;
            font-family: Arial, Helvetica, sans-serif;
            color: #141414;
        }}

        .dashboard {{
            display: grid;
            grid-template-rows: auto minmax(0, 0.75fr) minmax(0, 1fr) minmax(0, 1fr);
            gap: 10px;
            width: 100vw;
            height: 100vh;
            padding: 12px;
        }}

        .dashboard-title {{
            margin: 0;
            padding: 6px 4px;
            color: #141414;
            font-size: clamp(20px, 1.8vw, 30px);
            font-weight: 700;
            line-height: 1.1;
            border-left: 7px solid #E50914;
            padding-left: 12px;
        }}

        .dashboard-row {{
            display: grid;
            gap: 10px;
            min-width: 0;
            min-height: 0;
        }}

        .top-row {{
            grid-template-columns: minmax(0, 2fr) minmax(360px, 1fr);
        }}

        .middle-row,
        .bottom-row {{
            grid-template-columns: repeat(3, minmax(0, 1fr));
        }}

        .card {{
            position: relative;
            min-width: 0;
            min-height: 0;
            overflow: visible;
            background: #FFFFFF;
            border: 1px solid #E2E2E2;
            border-radius: 10px;
            box-shadow: 0 2px 8px rgba(20, 20, 20, 0.10);
        }}

        .figure-card {{
            display: flex;
            width: 100%;
            height: 100%;
            min-width: 0;
            min-height: 0;
            padding: 4px;
        }}

        .plot-host {{
            position: relative;
            flex: 1 1 auto;
            width: 100%;
            height: 100%;
            min-width: 0;
            min-height: 0;
        }}

        .plot-host > div,
        .plot-host .plotly-graph-div,
        .plot-host .js-plotly-plot,
        .plot-host .plot-container,
        .plot-host .svg-container {{
            max-width: 100%;
            max-height: 100%;
        }}

        .kpi-panel {{
            display: grid;
            grid-template-columns: repeat(2, minmax(0, 1fr));
            grid-template-rows: repeat(2, minmax(0, 1fr));
            gap: 8px;
            min-width: 0;
            min-height: 0;
        }}

        .kpi-card {{
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-width: 0;
            min-height: 0;
            padding: 8px;
            overflow: hidden;
            text-align: center;
            background: #141414;
            border-top: 4px solid #E50914;
            border-radius: 9px;
            box-shadow: 0 2px 7px rgba(20, 20, 20, 0.18);
        }}

        .kpi-label {{
            margin-bottom: 5px;
            color: #F5F5F5;
            font-size: clamp(10px, 0.78vw, 13px);
            font-weight: 600;
            line-height: 1.15;
        }}

        .kpi-value {{
            width: 100%;
            overflow: hidden;
            color: #FFFFFF;
            font-size: clamp(15px, 1.25vw, 22px);
            font-weight: 700;
            line-height: 1.1;
            text-overflow: ellipsis;
            white-space: nowrap;
        }}

        .kpi-card:first-child .kpi-value {{
            color: #E50914;
        }}

        @media (max-width: 800px) {{
            html,
            body {{
                height: auto;
                min-height: 100%;
                overflow-x: hidden;
                overflow-y: auto;
            }}

            .dashboard {{
                display: flex;
                flex-direction: column;
                width: 100%;
                height: auto;
                min-height: 100vh;
                padding: 10px;
            }}

            .dashboard-row,
            .top-row,
            .middle-row,
            .bottom-row {{
                display: grid;
                grid-template-columns: minmax(0, 1fr);
                height: auto;
            }}

            .figure-card {{
                height: 360px;
            }}

            .kpi-panel {{
                grid-template-columns: minmax(0, 1fr);
                grid-template-rows: repeat(4, 110px);
            }}

            .kpi-value {{
                font-size: 20px;
            }}
        }}
    </style>
</head>

<body>
    <main class="dashboard">
        <h1 class="dashboard-title">
            Netflix Merchandising Opportunities Dashboard
        </h1>

        <section class="dashboard-row top-row">
            <article class="card figure-card">
                <div class="plot-host">
                    {html_line_1}
                </div>
            </article>

            <aside class="kpi-panel">
                <article class="kpi-card">
                    <div class="kpi-label">Latest Number One Title</div>
                    <div class="kpi-value" title="{top_title}">
                        {top_title}
                    </div>
                </article>

                <article class="kpi-card">
                    <div class="kpi-label">Weekly Views</div>
                    <div class="kpi-value">
                        {top_title_views_text}
                    </div>
                </article>

                <article class="kpi-card">
                    <div class="kpi-label">Average Weekly Views</div>
                    <div class="kpi-value">
                        {mean_weekly_views_text}
                    </div>
                </article>

                <article class="kpi-card">
                    <div class="kpi-label">Weeks in the Top 10</div>
                    <div class="kpi-value">
                        {top_title_cumulative}
                    </div>
                </article>
            </aside>
        </section>

        <section class="dashboard-row middle-row">
            <article class="card figure-card">
                <div class="plot-host">
                    {html_pie_1}
                </div>
            </article>

            <article class="card figure-card">
                <div class="plot-host">
                    {html_bar_3}
                </div>
            </article>

            <article class="card figure-card">
                <div class="plot-host">
                    {html_line_4}
                </div>
            </article>
        </section>

        <section class="dashboard-row bottom-row">
            <article class="card figure-card">
                <div class="plot-host">
                    {html_line_checkpoint}
                </div>
            </article>

            <article class="card figure-card">
                <div class="plot-host">
                    {html_hist_1}
                </div>
            </article>

            <article class="card figure-card">
                <div class="plot-host">
                    {html_world}
                </div>
            </article>
        </section>
    </main>

    <script>
        function resizePlot(plot) {{
            const host = plot.closest(".plot-host");

            if (!host || typeof Plotly === "undefined") {{
                return;
            }}

            const width = Math.max(1, Math.floor(host.clientWidth));
            const height = Math.max(1, Math.floor(host.clientHeight));

            Plotly.relayout(plot, {{
                width: width,
                height: height,
                autosize: false
            }});
        }}

        function resizeAllPlots() {{
            document.querySelectorAll(".plot-host .plotly-graph-div").forEach(
                function(plot) {{
                    resizePlot(plot);
                }}
            );
        }}

        function schedulePlotResize() {{
            window.requestAnimationFrame(function() {{
                window.requestAnimationFrame(resizeAllPlots);
            }});
        }}

        window.addEventListener("load", schedulePlotResize);
        window.addEventListener("resize", schedulePlotResize);

        if (typeof ResizeObserver !== "undefined") {{
            const observer = new ResizeObserver(function(entries) {{
                entries.forEach(function(entry) {{
                    const plot = entry.target.querySelector(".plotly-graph-div");

                    if (plot) {{
                        resizePlot(plot);
                    }}
                }});
            }});

            document.querySelectorAll(".plot-host").forEach(function(host) {{
                observer.observe(host);
            }});
        }}
    </script>
</body>
</html>
"""
```

---

## 7. Generate the HTML file

Paste the complete `html = f"""..."""` assignment returned by the AI tool after the `to_html()` code.

Run the cell to create the variable `html`.

Save the complete page as an HTML file:

```python
with open("netflix_dashboard.html", "w", encoding="utf-8") as file:
    file.write(html)
```

Download the file from Google Colab and open it in a web browser.

---

## Expected result

At the end of this activity, you should have:

* An interactive dashboard contained in an HTML file that can be opened in a web browser.

---

## Sources

* Netflix, *Top 10*:
  <https://www.netflix.com/tudum/top10>
* Plotly, *Interactive HTML Export in Python*:
  <https://plotly.com/python/interactive-html-export/>
* Plotly, *Plotly Express Documentation*:
  <https://plotly.com/python/plotly-express/>
* MDN Web Docs, *CSS Grid Layout*:
  <https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout>
* CRISP-DM course material, Phase 4: Modeling.
