from pathlib import Path
from datetime import date
import math

import plotly.express as px
import plotly.graph_objects as go
import duckdb
import pandas as pd
import streamlit as st
import pydeck as pdk
import re

# import libraries for streamlit app 

APP_DIR = Path(__file__).resolve().parent

NO_LICENSE_DEID_LPN = 187


def color_to_rgba(color, alpha=180):
    if isinstance(color, (list, tuple)) and len(color) >= 3:
        r, g, b = color[0], color[1], color[2]
        return [int(r), int(g), int(b), int(alpha)]

    if not isinstance(color, str):
        return [200, 200, 200, int(alpha)]

    c = color.strip()
    if c.startswith("#"):
        r, g, b = px.colors.hex_to_rgb(c)
        return [int(r), int(g), int(b), int(alpha)]

    m = re.match(r"^rgba?\(([^)]+)\)$", c, flags=re.IGNORECASE)
    if m:
        parts = [p.strip() for p in m.group(1).split(",")]
        nums = [float(p) for p in parts[:4]]
        r, g, b = nums[0], nums[1], nums[2]
        if len(nums) >= 4:
            a = nums[3]
            a255 = int(round(a * 255)) if a <= 1 else int(round(a))
            return [int(r), int(g), int(b), a255]
        return [int(r), int(g), int(b), int(alpha)]

    return [200, 200, 200, int(alpha)]

@st.cache_resource
def get_con():
    con = duckdb.connect()

    con.execute(f"CREATE OR REPLACE VIEW agg_map_grid_day_norm AS SELECT * FROM '{(APP_DIR / 'agg_map_grid_day_norm.parquet').as_posix()}'")
    con.execute(f"CREATE OR REPLACE VIEW agg_offenders_day_norm AS SELECT * FROM '{(APP_DIR / 'agg_offenders_day_norm.parquet').as_posix()}'")
    con.execute(f"CREATE OR REPLACE VIEW agg_streets_day_norm AS SELECT * FROM '{(APP_DIR / 'agg_streets_day_norm.parquet').as_posix()}'")
    con.execute(f"CREATE OR REPLACE VIEW agg_offender_violation_day AS SELECT * FROM '{(APP_DIR / 'agg_offender_violation_day.parquet').as_posix()}'")
    con.execute(f"CREATE OR REPLACE VIEW agg_offender_hour_day AS SELECT * FROM '{(APP_DIR / 'agg_offender_hour_day.parquet').as_posix()}'")
    con.execute(f"CREATE OR REPLACE VIEW tickets_enriched AS SELECT * FROM '{(APP_DIR / 'tickets_2024_enriched.parquet').as_posix()}'")

    return con

@st.cache_data
def get_violation_values():
    con = get_con()
    df = con.execute(
        """
        SELECT DISTINCT violation_desc_long
        FROM agg_map_grid_day_norm
        ORDER BY 1
        """
    ).fetchdf()
    return df["violation_desc_long"].tolist()

def q_map(con, date_start, date_end, violations):
    if violations:
        placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        SELECT
          lat_bin,
          lon_bin,
          SUM(ticket_count) AS total_count,
                    SUM(day_weight) AS active_days,
          SUM(ticket_count) / SUM(day_weight) AS avg_per_day
        FROM agg_map_grid_day_norm
        WHERE ticket_date BETWEEN ? AND ?
          AND violation_desc_long IN ({placeholders})
        GROUP BY 1,2
        """
        params = [date_start, date_end] + list(violations)
    else:
        sql = """
        SELECT
          lat_bin,
          lon_bin,
          SUM(ticket_count) AS total_count,
                    SUM(day_weight) AS active_days,
          SUM(ticket_count) / SUM(day_weight) AS avg_per_day
        FROM agg_map_grid_day_norm
        WHERE ticket_date BETWEEN ? AND ?
        GROUP BY 1,2
        """
        params = [date_start, date_end]
    return con.execute(sql, params).fetchdf()

def q_top_offenders(con, date_start, date_end, violations, top_n, metric="total", exclude_no_license=False):
    top_n = int(top_n)
    score_expr = "SUM(ticket_count)" if metric == "total" else "SUM(ticket_count) / SUM(day_weight)"

    exclude_clause = f" AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""

    if violations:
        placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        SELECT
          deid_lpn,
          {score_expr} AS score,
          SUM(ticket_count) AS total_tickets,
          SUM(day_weight) AS active_days
        FROM agg_offenders_day_norm
                WHERE ticket_date BETWEEN ? AND ?
                    AND violation_desc_long IN ({placeholders})
                    {exclude_clause}
        GROUP BY 1
        ORDER BY score DESC
        LIMIT ?
        """
        params = [date_start, date_end] + list(violations) + [top_n]
    else:
        sql = f"""
        SELECT
          deid_lpn,
          {score_expr} AS score,
          SUM(ticket_count) AS total_tickets,
          SUM(day_weight) AS active_days
        FROM agg_offenders_day_norm
                WHERE ticket_date BETWEEN ? AND ?
                    {exclude_clause}
        GROUP BY 1
        ORDER BY score DESC
        LIMIT ?
        """
        params = [date_start, date_end, top_n]

    return con.execute(sql, params).fetchdf()

def q_top_streets(con, date_start, date_end, violations, top_n, metric="total"):
    top_n = int(top_n)
    score_expr = "SUM(ticket_count)" if metric == "total" else "SUM(ticket_count) / SUM(day_weight)"

    if violations:
        placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        SELECT
          street_key,
          {score_expr} AS score,
          SUM(ticket_count) AS total_tickets,
          SUM(day_weight) AS active_days
        FROM agg_streets_day_norm
        WHERE ticket_date BETWEEN ? AND ?
          AND violation_desc_long IN ({placeholders})
        GROUP BY 1
        ORDER BY score DESC
        LIMIT ?
        """
        params = [date_start, date_end] + list(violations) + [top_n]
    else:
        sql = f"""
        SELECT
          street_key,
          {score_expr} AS score,
          SUM(ticket_count) AS total_tickets,
          SUM(day_weight) AS active_days
        FROM agg_streets_day_norm
        WHERE ticket_date BETWEEN ? AND ?
        GROUP BY 1
        ORDER BY score DESC
        LIMIT ?
        """
        params = [date_start, date_end, top_n]

    return con.execute(sql, params).fetchdf()


def q_top_offender_ids(con, date_start, date_end, violations, top_n, exclude_no_license=False):
        top_n = int(top_n)
        exclude_clause = f" AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""
        if violations:
                placeholders = ",".join(["?"] * len(violations))
                sql = f"""
                SELECT
                    deid_lpn
                FROM agg_offenders_day_norm
                WHERE ticket_date BETWEEN ? AND ?
                    AND violation_desc_long IN ({placeholders})
                    {exclude_clause}
                GROUP BY 1
                ORDER BY SUM(ticket_count) DESC
                LIMIT ?
                """
                params = [date_start, date_end] + list(violations) + [top_n]
        else:
                sql = """
                SELECT
                    deid_lpn
                FROM agg_offenders_day_norm
                WHERE ticket_date BETWEEN ? AND ?
                    """ + exclude_clause + """
                GROUP BY 1
                ORDER BY SUM(ticket_count) DESC
                LIMIT ?
                """
                params = [date_start, date_end, top_n]

        df = con.execute(sql, params).fetchdf()
        return df["deid_lpn"].tolist()


def q_offender_points(con, date_start, date_end, violations, offender_ids, exclude_no_license=False):
    offender_ids = list(offender_ids or [])
    if not offender_ids:
        return pd.DataFrame(
            columns=[
                "latitude",
                "longitude",
                "deid_lpn",
                "ticket_issue_date",
                "ticket_issue_time",
                "violation_desc_long",
                "location",
                "ticket_number",
            ]
        )

    id_placeholders = ",".join(["?"] * len(offender_ids))

    if violations:
        vio_placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        SELECT
            latitude,
            longitude,
            deid_lpn,
            ticket_issue_date,
            ticket_issue_time,
            violation_desc_long,
            location,
            ticket_number
        FROM tickets_enriched
        WHERE CAST(ticket_issue_date AS DATE) BETWEEN ? AND ?
            AND violation_desc_long IN ({vio_placeholders})
            AND deid_lpn IN ({id_placeholders})
            {f"AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""}
        """
        params = [date_start, date_end] + list(violations) + offender_ids
    else:
        sql = f"""
        SELECT
            latitude,
            longitude,
            deid_lpn,
            ticket_issue_date,
            ticket_issue_time,
            violation_desc_long,
            location,
            ticket_number
        FROM tickets_enriched
        WHERE CAST(ticket_issue_date AS DATE) BETWEEN ? AND ?
            AND deid_lpn IN ({id_placeholders})
            {f"AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""}
        """
        params = [date_start, date_end] + offender_ids

    df = con.execute(sql, params).fetchdf()
    df = df.dropna(subset=["latitude", "longitude", "deid_lpn"])
    return df


def q_one_time_offender_points(con, date_start, date_end, violations, exclude_no_license=False):
    exclude_one_time = f" AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""
    exclude_t = f" AND t.deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""

    if violations:
        vio_placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        WITH one_time AS (
            SELECT deid_lpn
            FROM agg_offenders_day_norm
            WHERE ticket_date BETWEEN ? AND ?
              AND violation_desc_long IN ({vio_placeholders})
              {exclude_one_time}
            GROUP BY 1
            HAVING SUM(ticket_count) = 1
        )
        SELECT
            t.latitude,
            t.longitude,
            t.deid_lpn,
            t.ticket_issue_date,
            t.ticket_issue_time,
            t.violation_desc_long,
            t.location,
            t.ticket_number
        FROM tickets_enriched t
        JOIN one_time o ON o.deid_lpn = t.deid_lpn
        WHERE CAST(t.ticket_issue_date AS DATE) BETWEEN ? AND ?
          AND t.violation_desc_long IN ({vio_placeholders})
          {exclude_t}
        """
        params = [date_start, date_end] + list(violations) + [date_start, date_end] + list(violations)
    else:
        sql = f"""
        WITH one_time AS (
            SELECT deid_lpn
            FROM agg_offenders_day_norm
            WHERE ticket_date BETWEEN ? AND ?
              {exclude_one_time}
            GROUP BY 1
            HAVING SUM(ticket_count) = 1
        )
        SELECT
            t.latitude,
            t.longitude,
            t.deid_lpn,
            t.ticket_issue_date,
            t.ticket_issue_time,
            t.violation_desc_long,
            t.location,
            t.ticket_number
        FROM tickets_enriched t
        JOIN one_time o ON o.deid_lpn = t.deid_lpn
        WHERE CAST(t.ticket_issue_date AS DATE) BETWEEN ? AND ?
          {exclude_t}
        """
        params = [date_start, date_end, date_start, date_end]

    df = con.execute(sql, params).fetchdf()
    df = df.dropna(subset=["latitude", "longitude", "deid_lpn"])
    return df


def assign_offender_colors(df_points: pd.DataFrame, offender_ids):
        offender_ids = list(offender_ids or [])
        if df_points.empty or not offender_ids:
                df_points["fill_color"] = []
                return df_points

        palette = px.colors.qualitative.Safe

        color_map = {}
        for i, oid in enumerate(offender_ids):
            color_map[int(oid)] = color_to_rgba(palette[i % len(palette)])

        df_points = df_points.copy()
        df_points["deid_lpn"] = df_points["deid_lpn"].astype(int)
        default_color = [200, 200, 200, 160]
        df_points["fill_color"] = df_points["deid_lpn"].apply(lambda x: color_map.get(int(x), default_color))
        return df_points

def q_offender_violation_breakdown(con, deid_lpn, date_start, date_end, violations=None, exclude_no_license=False):
    deid_lpn = int(deid_lpn)
    violations = violations or []

    exclude_clause = f" AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""

    if violations:
        placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        SELECT
          violation_desc_long,
          SUM(ticket_count) AS tickets
        FROM agg_offender_violation_day
        WHERE deid_lpn = ?
          AND ticket_date BETWEEN ? AND ?
          AND violation_desc_long IN ({placeholders})
                    {exclude_clause}
        GROUP BY 1
        ORDER BY tickets DESC
        """
        params = [deid_lpn, date_start, date_end] + list(violations)
    else:
        sql = """
        SELECT
          violation_desc_long,
          SUM(ticket_count) AS tickets
        FROM agg_offender_violation_day
        WHERE deid_lpn = ?
          AND ticket_date BETWEEN ? AND ?
                """ + exclude_clause + """
        GROUP BY 1
        ORDER BY tickets DESC
        """
        params = [deid_lpn, date_start, date_end]

    return con.execute(sql, params).fetchdf()

def q_offender_hour_profile(con, deid_lpn, date_start, date_end, exclude_no_license=False):
        deid_lpn = int(deid_lpn)
        exclude_clause = f" AND deid_lpn <> {NO_LICENSE_DEID_LPN}" if exclude_no_license else ""
        sql = f"""
        SELECT
            issue_hour,
            SUM(ticket_count) AS tickets
        FROM agg_offender_hour_day
        WHERE deid_lpn = ?
            AND ticket_date BETWEEN ? AND ?
            {exclude_clause}
        GROUP BY 1
        ORDER BY 1
        """
        return con.execute(sql, [deid_lpn, date_start, date_end]).fetchdf()


def q_offenders_time_series(con, date_start, date_end, violations, offender_ids, exclude_no_license=False):
    offender_ids = list(offender_ids or [])
    if not offender_ids:
        return pd.DataFrame(columns=["ticket_date", "deid_lpn", "tickets"])

    if exclude_no_license:
        offender_ids = [oid for oid in offender_ids if int(oid) != NO_LICENSE_DEID_LPN]
        if not offender_ids:
            return pd.DataFrame(columns=["ticket_date", "deid_lpn", "tickets"])

    id_placeholders = ",".join(["?"] * len(offender_ids))

    if violations:
        vio_placeholders = ",".join(["?"] * len(violations))
        sql = f"""
        SELECT
          ticket_date,
          deid_lpn,
          SUM(ticket_count) AS tickets
        FROM agg_offenders_day_norm
        WHERE ticket_date BETWEEN ? AND ?
          AND violation_desc_long IN ({vio_placeholders})
          AND deid_lpn IN ({id_placeholders})
        GROUP BY 1,2
        ORDER BY 1,2
        """
        params = [date_start, date_end] + list(violations) + offender_ids
    else:
        sql = f"""
        SELECT
          ticket_date,
          deid_lpn,
          SUM(ticket_count) AS tickets
        FROM agg_offenders_day_norm
        WHERE ticket_date BETWEEN ? AND ?
          AND deid_lpn IN ({id_placeholders})
        GROUP BY 1,2
        ORDER BY 1,2
        """
        params = [date_start, date_end] + offender_ids

    return con.execute(sql, params).fetchdf()


def q_weekday_hour(con, date_start, date_end, violations, offender_ids=None, exclude_no_license=False):
    offender_ids = list(offender_ids or [])
    if exclude_no_license and offender_ids:
        offender_ids = [oid for oid in offender_ids if int(oid) != NO_LICENSE_DEID_LPN]

    params = [date_start, date_end]
    clauses = ["CAST(ticket_issue_date AS DATE) BETWEEN ? AND ?"]

    if violations:
        placeholders = ",".join(["?"] * len(violations))
        clauses.append(f"violation_desc_long IN ({placeholders})")
        params.extend(list(violations))

    if offender_ids:
        id_placeholders = ",".join(["?"] * len(offender_ids))
        clauses.append(f"deid_lpn IN ({id_placeholders})")
        params.extend(offender_ids)

    if exclude_no_license:
        clauses.append(f"deid_lpn <> {NO_LICENSE_DEID_LPN}")

    where_sql = " AND ".join(clauses)

    sql = f"""
    SELECT
      CAST(strftime(CAST(ticket_issue_date AS DATE), '%w') AS INTEGER) AS dow,
      EXTRACT('hour' FROM TRY_CAST(ticket_issue_time AS TIME)) AS hour,
      COUNT(*) AS tickets
    FROM tickets_enriched
    WHERE {where_sql}
      AND ticket_issue_time IS NOT NULL
      AND TRY_CAST(ticket_issue_time AS TIME) IS NOT NULL
    GROUP BY 1,2
    ORDER BY 1,2
    """
    return con.execute(sql, params).fetchdf()


def weekday_hour_heatmap_figure(df_wh: pd.DataFrame, *, title: str | None = None):
    if df_wh is None or df_wh.empty:
        return None

    # DuckDB strftime('%w'): 0=Sun ... 6=Sat
    dow_to_day = {0: "Sun", 1: "Mon", 2: "Tue", 3: "Wed", 4: "Thu", 5: "Fri", 6: "Sat"}
    day_order = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
    hours = list(range(24))

    df = df_wh.copy()
    df["dow"] = df["dow"].astype(int)
    df["hour"] = df["hour"].astype(int)
    df["day"] = df["dow"].map(dow_to_day)

    pivot = (
        df.pivot_table(index="day", columns="hour", values="tickets", aggfunc="sum", fill_value=0)
        .reindex(index=day_order, columns=hours, fill_value=0)
    )

    fig = go.Figure(
        data=[
            go.Heatmap(
                z=pivot.values,
                x=[str(h) for h in hours],
                y=pivot.index.tolist(),
                colorscale="Viridis",
                colorbar=dict(title="Tickets"),
                xgap=2,
                ygap=2,
                hovertemplate="%{y} %{x}:00<br>Tickets: %{z}<extra></extra>",
            )
        ]
    )
    fig.update_layout(
        title=dict(text=title, x=0.0, xanchor="left") if title else None,
        margin=dict(l=0, r=0, t=28 if title else 10, b=0),
        height=320,
    )
    fig.update_xaxes(title_text="Hour", tickmode="linear")
    fig.update_yaxes(title_text="Day of week")
    return fig

st.set_page_config(page_title="Boston Violations Dashboard", layout="wide")
from PIL import Image
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent
IMAGES_DIR = BASE_DIR / "images"

header_img = Image.open(IMAGES_DIR / "header.png")
st.image(header_img, use_container_width=True)
st.markdown(
    "<h1 style='text-align: center;'>City of Boston Parking Violations</h1>",
    unsafe_allow_html=True,
)

st.markdown(
    "<p style='text-align:center; font-weight:600; margin-top:-8px;'>Unofficial - Only intended as an academic exercise</p>",
    unsafe_allow_html=True,
)

st.write(
    "This dashboard explores City of Boston parking violations in 2024 using pre-aggregated Parquet artifacts "
    "and a de-identified ticket dataset. Use the date and violation filters to see how enforcement patterns "
    "shift across neighborhoods, streets, and time. The map can be viewed as either point clusters (binned cells) "
    "or a heatmap, and the offender views let you compare all vehicles vs. first-time vs. one-time vs. top repeat "
    "offenders. The time-pattern charts summarize when violations happen (by day and hour) and help highlight "
    "consistent routines versus spikes driven by specific offenders or dates."
)


con = get_con()

c1, c2, c3, c4 = st.columns([2, 2, 3, 2])

with c1:
    date_start = st.date_input("Start date", value=date(2024, 1, 1)).strftime("%Y-%m-%d")
with c2:
    date_end = st.date_input("End date", value=date(2024, 12, 31)).strftime("%Y-%m-%d")

viol_vals = get_violation_values()

with c3:
    violations = st.multiselect("Violations", options=viol_vals, default=[])
with c4:
    parking_offender = st.selectbox(
        "Parking Offender",
        options=[
            "All Offenders",
            "Top 10 offenders",
            "Top 20 offenders",
            "One-time offenders",
            "Enter specific deid_lpn",
        ],
        index=0,
    )
    top_n = 20

# Heatmap toggle should be available for all Parking Offender modes
_tcol1, _tcol2 = st.columns([1, 2])
with _tcol1:
    map_as_heatmap = st.toggle("Heatmap", value=False)
with _tcol2:
    exclude_no_license = st.toggle("Cars without license plate", value=True)

if parking_offender == "All Offenders":
    df_map = q_map(con, date_start, date_end, violations)
    value_col = "total_count"
    df_map = df_map.dropna(subset=["lat_bin", "lon_bin", value_col])

    v = df_map[value_col].astype(float)
    q20, q40, q60, q80 = v.quantile([0.2, 0.4, 0.6, 0.8]).tolist()

    def color_bin(x):
        if x <= q20:
            return [173, 216, 230, 160]  # light blue
        elif x <= q40:
            return [0, 0, 139, 160]  # dark blue
        elif x <= q60:
            return [255, 215, 0, 160]  # yellow
        elif x <= q80:
            return [255, 140, 0, 160]  # orange
        else:
            return [220, 20, 60, 160]  # red

    df_map["fill_color"] = v.apply(color_bin)

    bin_tooltip = {
        "html": """
        <b>Grid bin</b><br/>
        Total tickets: {total_count}<br/>
        Avg/day: {avg_per_day}<br/>
        Active days: {active_days}<br/>
        (lat, lon): ({lat_bin}, {lon_bin})
        """,
        "style": {"backgroundColor": "rgba(0,0,0,0.8)", "color": "white"},
    }

    if map_as_heatmap:
        # Log-scale the weights to prevent the heatmap from saturating (everything turning yellow)
        df_map = df_map.copy()
        df_map["heat_weight"] = df_map[value_col].astype(float).apply(lambda x: math.log1p(max(x, 0.0)))

        heat_layer = pdk.Layer(
            "HeatmapLayer",
            data=df_map,
            get_position="[lon_bin, lat_bin]",
            get_weight="heat_weight",
            radiusPixels=35,
            intensity=0.8,
            threshold=0.25,
            colorRange=[
                [0, 0, 0, 0],
                [65, 182, 196, 90],
                [127, 205, 187, 120],
                [255, 255, 204, 140],
                [253, 174, 97, 170],
                [240, 59, 32, 200],
            ],
        )
        # Overlay an invisible pickable scatter layer so hover still works
        hover_layer = pdk.Layer(
            "ScatterplotLayer",
            data=df_map,
            get_position="[lon_bin, lat_bin]",
            get_radius=25,
            pickable=True,
            auto_highlight=True,
            stroked=False,
            get_fill_color=[0, 0, 0, 0],
        )
        layers = [heat_layer, hover_layer]
        tooltip = bin_tooltip
    else:
        layer = pdk.Layer(
            "ScatterplotLayer",
            data=df_map,
            get_position="[lon_bin, lat_bin]",
            get_radius=25,
            pickable=True,
            auto_highlight=True,
            get_fill_color="fill_color",
        )
        layers = [layer]
        tooltip = bin_tooltip

    view_state = pdk.ViewState(
        latitude=float(df_map["lat_bin"].mean()) if len(df_map) else 42.3601,
        longitude=float(df_map["lon_bin"].mean()) if len(df_map) else -71.0589,
        zoom=11,
    )

    deck = pdk.Deck(layers=layers, initial_view_state=view_state, tooltip=tooltip)
    st.pydeck_chart(deck)

    if not map_as_heatmap:
        legend = [
            ("Low ≤ 20", f"≤ {q20:.2f}", "rgb(173,216,230)"),
            ("Moderate-Low 20–55", f"{q20:.2f}–{q40:.2f}", "rgb(0,0,139)"),
            ("Medium 55-120", f"{q40:.2f}–{q60:.2f}", "rgb(255,215,0)"),
            ("High 120-260", f"{q60:.2f}–{q80:.2f}", "rgb(255,140,0)"),
            ("Very High > 260", f"> {q80:.2f}", "rgb(220,20,60)"),
        ]

        legend_html = "<div style='display:flex; gap:14px; align-items:center; justify-content:center; flex-wrap:wrap; font-size:13px; margin: 0 auto;'>"
        legend_html += f"<div><b>{value_col}</b></div>"
        for name, rng, color in legend:
            legend_html += (
                "<div style='display:flex; align-items:center; gap:6px;'>"
                f"<span style='width:14px; height:14px; background:{color}; display:inline-block; border:1px solid #333;'></span>"
                f"<span>{name}: {rng}</span>"
                "</div>"
            )
        legend_html += "</div>"
        st.markdown(legend_html, unsafe_allow_html=True)

else:
    selected_offender_ids = []

    if parking_offender == "Top 10 offenders":
        selected_offender_ids = q_top_offender_ids(con, date_start, date_end, violations, top_n=10, exclude_no_license=exclude_no_license)
    elif parking_offender == "Top 20 offenders":
        selected_offender_ids = q_top_offender_ids(con, date_start, date_end, violations, top_n=20, exclude_no_license=exclude_no_license)
    elif parking_offender == "Enter specific deid_lpn":
        deid_text = st.text_input("deid_lpn")
        try:
            selected_offender_ids = [int(deid_text)] if deid_text.strip() != "" else []
        except ValueError:
            selected_offender_ids = []

    if exclude_no_license and selected_offender_ids:
        selected_offender_ids = [oid for oid in selected_offender_ids if int(oid) != NO_LICENSE_DEID_LPN]

    if parking_offender == "One-time offenders":
        df_points = q_one_time_offender_points(con, date_start, date_end, violations, exclude_no_license=exclude_no_license)
        df_points = df_points.copy()
        palette = px.colors.qualitative.Safe
        if len(df_points):
            df_points["deid_lpn"] = df_points["deid_lpn"].astype(int)
            df_points["fill_color"] = df_points["deid_lpn"].apply(
                lambda x: color_to_rgba(palette[int(x) % len(palette)], alpha=180)
            )
        else:
            df_points["fill_color"] = []
    else:
        df_points = q_offender_points(con, date_start, date_end, violations, selected_offender_ids, exclude_no_license=exclude_no_license)
        df_points = assign_offender_colors(df_points, selected_offender_ids)

    if map_as_heatmap:
        heat_layer = pdk.Layer(
            "HeatmapLayer",
            data=df_points,
            get_position="[longitude, latitude]",
            get_weight=1,
            radiusPixels=35,
            intensity=0.8,
            threshold=0.25,
            colorRange=[
                [0, 0, 0, 0],
                [65, 182, 196, 90],
                [127, 205, 187, 120],
                [255, 255, 204, 140],
                [253, 174, 97, 170],
                [240, 59, 32, 200],
            ],
        )
        hover_layer = pdk.Layer(
            "ScatterplotLayer",
            data=df_points,
            get_position="[longitude, latitude]",
            get_radius=20,
            pickable=True,
            auto_highlight=True,
            stroked=False,
            get_fill_color=[0, 0, 0, 0],
        )
        layers = [heat_layer, hover_layer]
    else:
        layer = pdk.Layer(
            "ScatterplotLayer",
            data=df_points,
            get_position="[longitude, latitude]",
            get_radius=20,
            pickable=True,
            auto_highlight=True,
            get_fill_color="fill_color",
        )
        layers = [layer]

    view_state = pdk.ViewState(
        latitude=float(df_points["latitude"].mean()) if len(df_points) else 42.3601,
        longitude=float(df_points["longitude"].mean()) if len(df_points) else -71.0589,
        zoom=12,
    )

    deck = pdk.Deck(
        layers=layers,
        initial_view_state=view_state,
        tooltip={
            "html": """
            <b>Offender ticket</b><br/>
            deid_lpn: {deid_lpn}<br/>
            Date: {ticket_issue_date}<br/>
            Time: {ticket_issue_time}<br/>
            Violation: {violation_desc_long}<br/>
            Location: {location}<br/>
            Ticket #: {ticket_number}
            """,
            "style": {"backgroundColor": "rgba(0,0,0,0.8)", "color": "white"},
        },
    )
    st.pydeck_chart(deck)

    if parking_offender in {"Top 10 offenders", "Top 20 offenders", "Enter specific deid_lpn"} and selected_offender_ids:
        df_ts = q_offenders_time_series(con, date_start, date_end, violations, selected_offender_ids, exclude_no_license=exclude_no_license)
        if len(df_ts):
            df_ts = df_ts.copy()
            df_ts["deid_lpn"] = df_ts["deid_lpn"].astype(str)
            df_ts["ticket_date"] = pd.to_datetime(df_ts["ticket_date"]).dt.date

            st.subheader("Violations Over Time")
            fig = px.line(
                df_ts,
                x="ticket_date",
                y="tickets",
                color="deid_lpn",
                markers=False,
                labels={"ticket_date": "Date", "tickets": "Violations", "deid_lpn": "Offender"},
            )
            fig.update_layout(legend_title_text="Offender")
            st.plotly_chart(fig, use_container_width=True)

            # Weekday (y) x hour (x) heatmap directly below the time graph
            df_wh = q_weekday_hour(
                con,
                date_start,
                date_end,
                violations,
                offender_ids=selected_offender_ids,
                exclude_no_license=exclude_no_license,
            )
            if len(df_wh):
                # Peak cell label (like the reference)
                peak = df_wh.loc[df_wh["tickets"].astype(int).idxmax()]
                dow_to_day = {0: "Sun", 1: "Mon", 2: "Tue", 3: "Wed", 4: "Thu", 5: "Fri", 6: "Sat"}
                peak_day = dow_to_day.get(int(peak["dow"]), "")
                peak_hour = int(peak["hour"])
                peak_tickets = int(peak["tickets"])

                st.caption("Peak hour (focused)")
                st.markdown(
                    f"<div style='font-size:36px; font-weight:650; line-height:1.1;'>{peak_day} {peak_hour:02d}:00</div>",
                    unsafe_allow_html=True,
                )
                st.markdown(
                    f"<div style='display:inline-block; padding:4px 10px; border-radius:999px; background:rgba(34,197,94,0.15); color:rgb(22,163,74); font-weight:600; font-size:13px;'>↑ {peak_tickets} tickets</div>",
                    unsafe_allow_html=True,
                )

                hm_fig = weekday_hour_heatmap_figure(df_wh)
                if hm_fig is not None:
                    st.plotly_chart(hm_fig, use_container_width=True)

st.divider()

col1, col2 = st.columns(2)

with col1:
    st.subheader("Top Offenders")
    df_off = q_top_offenders(con, date_start, date_end, violations, top_n, metric="total", exclude_no_license=exclude_no_license)
    st.dataframe(df_off, use_container_width=True, height=320)

with col2:
    st.subheader("Top Streets")
    df_st = q_top_streets(con, date_start, date_end, violations, top_n, metric="total")
    st.dataframe(df_st, use_container_width=True, height=320)

st.divider()

st.subheader("Offender Drilldown")

offender_options = df_off["deid_lpn"].tolist() if len(df_off) else []
selected = st.selectbox("Select offender", options=offender_options)

if selected is not None and selected != "":
    df_break = q_offender_violation_breakdown(con, selected, date_start, date_end, violations, exclude_no_license=exclude_no_license)
    st.write("Violation breakdown")
    st.dataframe(df_break, use_container_width=True, height=220)


st.divider()
st.caption(
    "Disclaimer: This dashboard was created as an academic exercise for the Harvard Kennedy School course "
    "Coding for Policy. Visualizations and results are for educational purposes only."
)

