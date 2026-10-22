
import io
from pathlib import Path

import pandas as pd
import plotly.express as px
import streamlit as st

st.set_page_config(
    page_title="India PSU Intelligence",
    page_icon="🏛️",
    layout="wide",
)

st.markdown("""
<style>
    .block-container {padding-top: 1.5rem; padding-bottom: 2rem;}
    [data-testid="stMetric"] {
        background: var(--secondary-background-color);
        border: 1px solid rgba(128,128,128,.2);
        padding: 16px;
        border-radius: 12px;
    }
</style>
""", unsafe_allow_html=True)

st.title("🏛️ India PSU Intelligence Dashboard")
st.caption(
    "Explore public sector enterprises across India. "
    "Metrics are calculated from the dataset you provide."
)

# ---------- Data loading ----------
@st.cache_data(show_spinner=False)
def load_excel(file_bytes, filename):
    excel = pd.ExcelFile(io.BytesIO(file_bytes))
    sheets = {
        sheet: pd.read_excel(
            io.BytesIO(file_bytes), sheet_name=sheet
        )
        for sheet in excel.sheet_names
    }
    return sheets


def normalize_name(value):
    return str(value).strip().lower()


def find_column(df, candidates):
    """Find a column by matching normalized names."""
    lookup = {
        normalize_name(col).replace("_", " "): col
        for col in df.columns
    }
    for candidate in candidates:
        key = normalize_name(candidate).replace("_", " ")
        if key in lookup:
            return lookup[key]
    return None


with st.sidebar:
    st.header("📂 Data source")
    uploaded = st.file_uploader(
        "Upload your PSU Excel workbook",
        type=["xlsx", "xls"],
    )

    default_path = Path("data/Master All PSUs.xlsx")
    st.caption(
        "Upload an Excel workbook, or place your file at "
        "`data/Master All PSUs.xlsx` in the repository."
    )

    st.divider()
    st.header("🔎 Dashboard filters")

if uploaded:
    file_bytes = uploaded.getvalue()
    filename = uploaded.name
elif default_path.exists():
    file_bytes = default_path.read_bytes()
    filename = default_path.name
else:
    file_bytes = None
    filename = None

if file_bytes is None:
    st.info(
        "Upload your PSU Excel workbook in the sidebar to begin. "
        "You can also add it to the repository under "
        "`data/Master All PSUs.xlsx`."
    )
    st.markdown("""
    ### Dashboard features
    - Total entities and unique PSU names
    - State/UT and sector-wise analysis
    - Operating-status and ownership breakdowns
    - Interactive charts and filters
    - Searchable records and CSV export
    - Data-quality checks for missing values and duplicates
    """)
    st.stop()

try:
    sheets = load_excel(file_bytes, filename)
except Exception as exc:
    st.error(f"Could not read the Excel workbook: {exc}")
    st.stop()

with st.sidebar:
    selected_sheet = st.selectbox(
        "Select worksheet",
        list(sheets.keys()),
    )

df = sheets[selected_sheet].copy()
df.columns = [str(col).strip() for col in df.columns]
df = df.dropna(how="all")

if df.empty:
    st.warning("The selected worksheet contains no records.")
    st.stop()

# ---------- Detect likely columns ----------
name_col = find_column(
    df, ["PSU Name", "Company Name", "Enterprise Name",
         "Name of CPSE", "Name", "Entity Name"]
)
state_col = find_column(
    df, ["State", "State/UT", "State / UT", "UT",
         "Location State", "State or UT"]
)
sector_col = find_column(
    df, ["Sector", "Industry", "Industry Sector",
         "Ministry/Sector", "Business Sector"]
)
status_col = find_column(
    df, ["Operating Status", "Operational Status",
         "Status", "Operating / Non-operating"]
)
ownership_col = find_column(
    df, ["Ownership", "PSU Type", "Enterprise Type",
         "Central/State", "Central or State",
         "Government Level"]
)

# ---------- Filters ----------
with st.sidebar:
    filtered = df.copy()

    def apply_filter(label, col):
        global filtered
        if col:
            values = sorted(
                filtered[col].dropna().astype(str).unique().tolist()
            )
            choices = st.multiselect(label, values)
            if choices:
                filtered = filtered[
                    filtered[col].astype(str).isin(choices)
                ]

    apply_filter("State / UT", state_col)
    apply_filter("Sector", sector_col)
    apply_filter("Operating status", status_col)
    apply_filter("Ownership", ownership_col)

# ---------- KPI calculations ----------
total_records = len(filtered)
unique_entities = (
    filtered[name_col].dropna().astype(str).nunique()
    if name_col else None
)
state_count = (
    filtered[state_col].dropna().astype(str).nunique()
    if state_col else None
)
sector_count = (
    filtered[sector_col].dropna().astype(str).nunique()
    if sector_col else None
)
missing_cells = int(filtered.isna().sum().sum())
total_cells = int(filtered.shape[0] * filtered.shape[1])
completeness = (
    100 * (1 - missing_cells / total_cells)
    if total_cells else 0
)

st.subheader("Overview")
metrics = st.columns(5)
metrics[0].metric("Total records", f"{total_records:,}")
metrics[1].metric(
    "Unique entities",
    f"{unique_entities:,}" if unique_entities is not None else "N/A",
)
metrics[2].metric(
    "States / UTs",
    f"{state_count:,}" if state_count is not None else "N/A",
)
metrics[3].metric(
    "Sectors",
    f"{sector_count:,}" if sector_count is not None else "N/A",
)
metrics[4].metric("Data completeness", f"{completeness:.1f}%")

st.caption(
    f"Worksheet: {selected_sheet} · "
    f"{len(df):,} records before filters · "
    f"{len(filtered):,} after filters"
)

# ---------- Charts ----------
st.subheader("PSU distribution")

chart1, chart2 = st.columns(2)

with chart1:
    if state_col:
        state_data = (
            filtered[state_col].fillna("Not specified")
            .astype(str).value_counts()
            .head(15).rename_axis("State / UT")
            .reset_index(name="Records")
        )
        fig = px.bar(
            state_data,
            x="Records",
            y="State / UT",
            orientation="h",
            title="Top 15 States / UTs by record count",
            text="Records",
        )
        fig.update_layout(yaxis={"categoryorder": "total ascending"})
        st.plotly_chart(fig, use_container_width=True)
    else:
        st.warning("State/UT chart unavailable: no matching column found.")

with chart2:
    if sector_col:
        sector_data = (
            filtered[sector_col].fillna("Not specified")
            .astype(str).value_counts().head(15)
            .rename_axis("Sector").reset_index(name="Records")
        )
        fig = px.pie(
            sector_data,
            names="Sector",
            values="Records",
            title="Sector distribution (top 15 categories)",
            hole=0.4,
        )
        st.plotly_chart(fig, use_container_width=True)
    else:
        st.warning("Sector chart unavailable: no matching column found.")

chart3, chart4 = st.columns(2)

with chart3:
    if status_col:
        status_data = (
            filtered[status_col].fillna("Not specified")
            .astype(str).value_counts()
            .rename_axis("Status").reset_index(name="Records")
        )
        fig = px.bar(
            status_data, x="Status", y="Records",
            title="Operating status"
        )
        st.plotly_chart(fig, use_container_width=True)
    else:
        st.info("Status chart unavailable: no matching column found.")

with chart4:
    if ownership_col:
        ownership_data = (
            filtered[ownership_col].fillna("Not specified")
            .astype(str).value_counts()
            .rename_axis("Ownership").reset_index(name="Records")
        )
        fig = px.bar(
            ownership_data, x="Ownership", y="Records",
            title="Ownership breakdown"
        )
        st.plotly_chart(fig, use_container_width=True)
    else:
        st.info("Ownership chart unavailable: no matching column found.")

# ---------- Data quality ----------
st.subheader("Data quality")
quality1, quality2, quality3 = st.columns(3)
quality1.metric("Missing cells", f"{missing_cells:,}")
quality2.metric(
    "Duplicate rows",
    f"{int(filtered.duplicated().sum()):,}"
)
quality3.metric(
    "Rows missing PSU name",
    f"{int(filtered[name_col].isna().sum()):,}"
    if name_col else "N/A"
)

with st.expander("Missing values by column"):
    missing_table = pd.DataFrame({
        "Column": filtered.columns,
        "Missing values": [
            int(filtered[col].isna().sum())
            for col in filtered.columns
        ],
        "Missing %": [
            round(100 * filtered[col].isna().mean(), 2)
            for col in filtered.columns
        ],
    }).sort_values("Missing values", ascending=False)
    st.dataframe(missing_table, use_container_width=True, hide_index=True)

# ---------- Search and table ----------
st.subheader("PSU register")

search = st.text_input(
    "Search across all displayed fields",
    placeholder="Enter PSU name, state, sector, or keyword...",
)

display_df = filtered.copy()
if search.strip():
    mask = display_df.astype(str).apply(
        lambda col: col.str.contains(
            search.strip(), case=False, regex=False, na=False
        )
    ).any(axis=1)
    display_df = display_df[mask]

st.caption(f"{len(display_df):,} matching records")
st.dataframe(
    display_df,
    use_container_width=True,
    hide_index=True,
    height=450,
)

csv_bytes = display_df.to_csv(index=False).encode("utf-8-sig")
st.download_button(
    "⬇️ Download filtered records (CSV)",
    data=csv_bytes,
    file_name="india_psu_filtered.csv",
    mime="text/csv",
)

st.caption(
    "Important: Dashboard counts describe the selected worksheet and "
    "current filters. Column detection is automatic and may need "
    "adjustment for your workbook. This dashboard does not independently "
    "verify PSU status or establish that the dataset is complete."
)
