import streamlit as st
import pandas as pd
import plotly.express as px

# --------------------------------------------------
# Page Config
# --------------------------------------------------
st.set_page_config(page_title="CGL-2 Process Dashboard", layout="wide")

st.title("CGL-2 Process Dashboard")
st.subheader("Op Batch No Wise Analysis")

# --------------------------------------------------
# Load Production + Clearance Data
# --------------------------------------------------

@st.cache_data
def load_data():

    # ---------------- Production ----------------
    prod_df = pd.read_excel(
        "CGL2_prod_rpt_01.01.2024_31.01.2026_11AM.xlsx"
    )

    # ---------------- Clearance ----------------
    clr_df = pd.read_excel(
        "3.1 Final clerance CRM1 01.01.2023 to 31.12.2025.xlsx"
    )

    # Clean column names
    prod_df.columns = prod_df.columns.str.strip()
    clr_df.columns = clr_df.columns.str.strip()

    # Ensure datatype match
    prod_df["Op Batch No"] = prod_df["Op Batch No"].astype(str)
    clr_df["BatchNo"] = clr_df["BatchNo"].astype(str)

    # Create Clearance DateTime (if both columns exist)
    if "Clearance Date" in clr_df.columns and "TIME" in clr_df.columns:

        clr_df["Clearance Date"] = pd.to_datetime(
            clr_df["Clearance Date"],
            errors="coerce"
        )

        clr_df["TIME"] = pd.to_datetime(
            clr_df["TIME"],
            errors="coerce"
        ).dt.time

        clr_df["ClearanceDateTime"] = pd.to_datetime(
            clr_df["Clearance Date"].astype(str) + " " +
            clr_df["TIME"].astype(str),
            errors="coerce"
        )

        # Keep latest clearance entry per batch
        clr_df = (
            clr_df.sort_values("ClearanceDateTime")
                  .groupby("BatchNo", as_index=False)
                  .last()
        )

    # Select required clearance columns
    clearance_columns = [
        "BatchNo",
        "Ord Ys", "Ys",
        "Ord Uts", "Uts",
        "OrdElongation", "Elongation",
        "OrderHardness", "HardnessValue",
        "SurfaceRemark"
    ]

    clr_selected = clr_df[clearance_columns]

    # Merge
    merged_df = prod_df.merge(
        clr_selected,
        left_on="Op Batch No",
        right_on="BatchNo",
        how="left"
    )

    return merged_df
df = load_data()


# --------------------------------------------------
# Create End DateTime
# --------------------------------------------------

df["End Date"] = pd.to_datetime(
    df["End Date"], format="%d/%m/%Y", errors="coerce"
)

df["End Time"] = pd.to_datetime(
    df["End Time"], format="%H:%M:%S", errors="coerce"
).dt.time

df["End DateTime"] = pd.to_datetime(
    df["End Date"].astype(str) + " " + df["End Time"].astype(str),
    errors="coerce"
)

# --------------------------------------------------
# Sidebar - Op Batch Selection
# --------------------------------------------------

batch_list = sorted(df["Op Batch No"].dropna().unique())

selected_batch = st.sidebar.selectbox(
    "Search / Select Op Batch No",
    batch_list
)

# --------------------------------------------------
# Coil Sequence (3 Before + Selected + 3 After)
# --------------------------------------------------

coil_sequence = (
    df.sort_values("End DateTime")
      .groupby("Op Batch No", as_index=False)
      .last()
      .sort_values("End DateTime")
      .reset_index(drop=True)
)

selected_index = coil_sequence[
    coil_sequence["Op Batch No"] == selected_batch
].index[0]

start_index = max(selected_index - 3, 0)
end_index = selected_index + 4

window_coils = coil_sequence.iloc[start_index:end_index]

# --------------------------------------------------
# Display Coil Window
# --------------------------------------------------

display_cols = [
    "Op Batch No", "Actual Tdc",
    "End Date",
    "End Time", "Holding Section Strip Actual Temperature",
"Surface Conditioning Mill Elongation",
"Tension Leveller Elongation",
"Furnace Entry Speed",
"Tube Treatment 6 Strip Actual Temperature",
     "Customer", "Total Zn/AlZn Coating",
"Passivation_Type",
"Logo",
"Liner Marking",
    "Ord Ys", "Ys",
    "Ord Uts", "Uts", 
    "GL",
    "OrdElongation", "Elongation",
    "OrderHardness", "HardnessValue", "End DateTime"
]

available_cols = [col for col in display_cols if col in window_coils.columns]

st.dataframe(
    window_coils[available_cols],
    use_container_width=True
)



# --------------------------------------------------
# Selected Coil Full Data
# --------------------------------------------------

filtered_df = df[df["Op Batch No"] == selected_batch]

st.markdown("## Full Production + Clearance Data")

st.write(f"Showing data for Batch: {selected_batch}")
st.dataframe(filtered_df, use_container_width=True)


# --------------------------------------------------
# Surface Remark Table
# --------------------------------------------------

st.markdown("## Surface Remark for Selected Batch")

selected_surface = (
    df[df["Op Batch No"] == selected_batch]
    [["Op Batch No", "SurfaceRemark"]]
    .drop_duplicates()
)

st.dataframe(selected_surface, use_container_width=True)


# --------------------------------------------------
# Optional Example Plot (Clearance Status Count)
# --------------------------------------------------



# --------------------------------------------------
# Mechanical Property Comparison Plot
# --------------------------------------------------



if selected_batch:

    row = filtered_df.iloc[0]

    properties = [
        ("YS", "Ord Ys", "Ys"),
        ("UTS", "Ord Uts", "Uts"),
        ("Elongation", "OrdElongation", "Elongation")
    ]

    col1, col2, col3 = st.columns(3)

    for idx, (prop_name, order_col, actual_col) in enumerate(properties):

        order_range = str(row.get(order_col)).replace(" ", "")
        actual_value = pd.to_numeric(row.get(actual_col), errors="coerce")

        if "-" in order_range:
            min_val, max_val = order_range.split("-")
            min_val = pd.to_numeric(min_val, errors="coerce")
            max_val = pd.to_numeric(max_val, errors="coerce")
        else:
            continue  # skip if invalid format

        if pd.isna(actual_value) or pd.isna(min_val) or pd.isna(max_val):
            continue

        # Create figure
        fig = px.bar(
            x=[prop_name],
            y=[actual_value],
            text=[actual_value],
            title=f"{prop_name} Control Chart"
        )

        # Show value label
        fig.update_traces(textposition="outside")

        # Add Minimum line (GREEN)
        fig.add_shape(
            type="line",
            x0=-0.5,
            x1=0.5,
            y0=min_val,
            y1=min_val,
            line=dict(color="green", width=3),
        )

        # Add Maximum line (RED)
        fig.add_shape(
            type="line",
            x0=-0.5,
            x1=0.5,
            y0=max_val,
            y1=max_val,
            line=dict(color="red", width=3),
        )

        fig.update_layout(
            yaxis_title="Value",
            showlegend=False
        )

        # Display in horizontal columns
        if idx == 0:
            col1.plotly_chart(fig, use_container_width=True)
        elif idx == 1:
            col2.plotly_chart(fig, use_container_width=True)
        else:
            col3.plotly_chart(fig, use_container_width=True)



# --------------------------------------------------
# 6-Month Backward Scatter Analysis (Final Robust Version)
# --------------------------------------------------

if selected_batch:

    selected_row = filtered_df.iloc[0]

    # Convert numeric columns safely
    numeric_cols = ["Op Thk", "Op Width", "Ys", "Uts", "Elongation"]
    for col in numeric_cols:
        df[col] = pd.to_numeric(df[col], errors="coerce")

    # Selected coil values
    selected_tdc = str(selected_row["Actual Tdc"])  # String match
    selected_thk = pd.to_numeric(selected_row["Op Thk"], errors="coerce")
    selected_width = pd.to_numeric(selected_row["Op Width"], errors="coerce")
    selected_end_datetime = selected_row["End DateTime"]

    if pd.isna(selected_end_datetime):
        st.warning("Selected coil has invalid End DateTime.")
    else:

        # 6-month backward window
        six_months_before = selected_end_datetime - pd.DateOffset(months=6)

        # Filter same spec + date range
        spec_df = df[
            (df["Actual Tdc"] == selected_tdc) &
            (abs(df["Op Thk"] - selected_thk) <= 0.02) &
            (abs(df["Op Width"] - selected_width) <= 5) &
            (df["End DateTime"] <= selected_end_datetime) &
            (df["End DateTime"] >= six_months_before)
        ].copy()

        # If nothing found
        if len(spec_df) == 0:
            st.warning("No matching coils found in 6-month backward window.")
        else:

            # Historical coils must have valid mechanical data
            historical_df = spec_df.dropna(subset=["Ys", "Uts", "Elongation"])

            # Selected coil row separately
            selected_point = spec_df[
                spec_df["Op Batch No"] == selected_batch
            ]

            # Combine safely (ensures selected coil stays)
            plot_df = pd.concat([historical_df, selected_point]).drop_duplicates()

            # Remove rows still missing required mechanical data
            plot_df = plot_df.dropna(subset=["Ys", "Uts", "Elongation"])

            if len(plot_df) == 0:
                st.warning("Mechanical data missing for selected coil.")
            else:

                # Highlight selected
                plot_df["Highlight"] = "Other Coils"
                plot_df.loc[
                    plot_df["Op Batch No"] == selected_batch,
                    "Highlight"
                ] = "Selected Coil"

                # ---------------- Scatter 1: Ys vs Uts ----------------
                fig1 = px.scatter(
                    plot_df,
                    x="Ys",
                    y="Uts",
                    color="Highlight",
                    color_discrete_map={
                        "Other Coils": "blue",
                        "Selected Coil": "red"
                    },
                    hover_data=["Op Batch No", "End DateTime"],
                    title="Ys vs Uts (6-Month Backward Production)"
                )

                fig1.update_traces(marker=dict(size=10))
                fig1.update_layout(
                    xaxis_title="Yield Strength (Ys)",
                    yaxis_title="Ultimate Tensile Strength (Uts)"
                )

                # ---------------- Scatter 2: Ys vs Elongation ----------------
                fig2 = px.scatter(
                    plot_df,
                    x="Ys",
                    y="Elongation",
                    color="Highlight",
                    color_discrete_map={
                        "Other Coils": "blue",
                        "Selected Coil": "red"
                    },
                    hover_data=["Op Batch No", "End DateTime"],
                    title="Ys vs Elongation (6-Month Backward Production)"
                )

                fig2.update_traces(marker=dict(size=10))
                fig2.update_layout(
                    xaxis_title="Yield Strength (Ys)",
                    yaxis_title="Elongation"
                )

                col1, col2 = st.columns(2)
                col1.plotly_chart(fig1, use_container_width=True)
                col2.plotly_chart(fig2, use_container_width=True)

                st.caption(f"Matched coils in 6-month window: {len(plot_df)}")
# --------------------------------------------------
# Heat-Wise Linked Coils (Traceability View)
# --------------------------------------------------

st.markdown("## Coils Linked to Same Heat No")

if selected_batch:

    selected_row = filtered_df.iloc[0]

    selected_heat = str(selected_row.get("Heat No"))

    if pd.isna(selected_heat) or selected_heat.strip() == "":
        st.warning("Selected coil has no Heat No.")
    else:

        # Ensure Heat No treated as string
        df["Heat No"] = df["Heat No"].astype(str)

        heat_df = df[
            df["Heat No"] == selected_heat
        ].copy()

        heat_df = heat_df.sort_values("End DateTime")

        st.write(f"Heat No: {selected_heat}")
        st.write(f"Total coils from same heat: {len(heat_df)}")

        st.dataframe(
            heat_df[
                [
                    "Op Batch No",
                    "Heat No",
                    "Actual Tdc",
                    "Op Thk",
                    "Op Width",
                    "End DateTime",
                    "Ys",
                    "Uts",
                    "Elongation"
                ]
            ],
            use_container_width=True
        )

