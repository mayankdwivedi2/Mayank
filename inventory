import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import trino
from trino.auth import BasicAuthentication
from datetime import datetime

# Page Config
st.set_page_config(page_title="Paytm MMB Inventory Dashboard", layout="wide")

# ------------------------------------------------------------------
# Global CSS for center-aligned tables (fast — no per-cell Styler)
# ------------------------------------------------------------------
st.markdown("""
<style>
[data-testid="stDataFrame"] div[data-testid="stDataFrameResizable"] {
    text-align: center;
}
[data-testid="stDataFrame"] td, [data-testid="stDataFrame"] th {
    text-align: center !important;
    justify-content: center !important;
}
</style>
""", unsafe_allow_html=True)

BRAND_COLORS = ["#00baf2", "#0072a3", "#66d1ff", "#003b57"]
ALERT = "#e74c3c"
EDC_COLOR = "#00baf2"
SB_COLOR = "#f39c12"

# Hardcoded device cost (Point 10)
DEVICE_COST = {'EDC': 5000, 'SB': 1000}

# ------------------------------------------------------------------
# Column label map + table helper (renaming only — CSS handles centering)
# ------------------------------------------------------------------
COLUMN_LABELS = {
    'e_code': 'E-Code', 'cust_id': 'Cust ID', 'name': 'Name', 'doj': 'DOJ',
    'role_designation': 'Role Designation', 'city': 'City', 'state': 'State',
    'sub_department': 'Dept', 'status': 'Status', 'l1': 'L1', 'role': 'Role',
    'l1_name': 'L1 Name', 'zone': 'Zone (RH)', 'device_type': 'Device Type',
    'last_30_days_consumption': 'Last 30 Days Cons.', 'unmapped': 'Unmapped',
    'available_a50': 'Available A50', 'available_others': 'Available Others',
    'available_stock_a910_dx8000': 'Available A910/DX8000',
    'available_stock_t9': 'Available T9', 'available_unmapped': 'Available + Unmapped',
    'intransit': 'Intransit', 'warehouse_pickup_pending': 'Delivered in WH Pickup Pending',
    'total_stock': 'Total Stock', 'new_sales': 'New Sales', 'repl': 'Repl.', 'dos': 'DOS',
    'plan_devices': 'Plan Devices', 'yet_to_be_dispatched': 'Yet To Be Dispatched',
    'without_t9_dos': 'DOS (Without T9)', 'barcode': 'Barcode', 'device_model': 'Device Model',
    'barcode_status_label': 'Status', 'updated_at': 'Updated Date', 'aging_days': 'Aging (Days)',
    'aging_bucket': 'Aging Bucket', 'spoc_e_code': 'SPOC E-Code', 'spoc_name': 'SPOC Name',
    'employee_status': 'Employee Status', 'device_cost': 'Device Cost (₹)',
}

def styled_table(df):
    """Renames columns to readable labels — no Styler, CSS handles centering (fast)."""
    return df.rename(columns=COLUMN_LABELS)

def styled_table_colored(df, dos_col='dos', unmapped_col='unmapped'):
    """Only for SMALL summary tables (few rows) — DOS green(high)->red(low),
    Unmapped red(high)->green(low). Kept off large tables to protect speed."""
    renamed = df.rename(columns=COLUMN_LABELS)
    dos_label = COLUMN_LABELS.get(dos_col, dos_col)
    unmapped_label = COLUMN_LABELS.get(unmapped_col, unmapped_col)
    styler = renamed.style
    if dos_label in renamed.columns and renamed[dos_label].notna().any() and len(renamed) > 0:
        vmin, vmax = renamed[dos_label].min(), renamed[dos_label].max()
        if vmin != vmax:
            styler = styler.background_gradient(subset=[dos_label], cmap='RdYlGn', vmin=vmin, vmax=vmax)
    if unmapped_label in renamed.columns and renamed[unmapped_label].notna().any() and len(renamed) > 0:
        vmin, vmax = renamed[unmapped_label].min(), renamed[unmapped_label].max()
        if vmin != vmax:
            styler = styler.background_gradient(subset=[unmapped_label], cmap='RdYlGn_r', vmin=vmin, vmax=vmax)
    return styler

def search_table(df, key, columns):
    """Search box that filters rows by identifier columns (e.g. e_code, barcode, name)."""
    query = st.text_input("🔍 Search (E-Code / Name / Barcode)", key=key)
    if query and not df.empty:
        cols = [c for c in columns if c in df.columns]
        mask = pd.Series(False, index=df.index)
        for c in cols:
            mask = mask | df[c].astype(str).str.contains(query, case=False, na=False)
        return df[mask]
    return df

# CSV Download Helper
def convert_df_to_csv(df):
    return df.to_csv(index=False).encode('utf-8')

# Trino session properties — shared across connections
TRINO_SESSION_PROPERTIES = {
    "distinct_aggregations_strategy": "single_step",
    "join_reordering_strategy": "NONE",
}

# ------------------------------------------------------------------
# Module-level helpers (usable across tabs)
# ------------------------------------------------------------------
def compute_group_summary(data, group_col):
    summary = (
        data.groupby(group_col)[
            ['last_30_days_consumption', 'unmapped', 'available_a50',
             'available_others', 'available_stock_a910_dx8000',
             'available_stock_t9', 'intransit']
        ]
        .sum()
        .reset_index()
    )
    available_total = (
        summary['unmapped'] + summary['available_a50'] + summary['available_others']
        + summary['available_stock_a910_dx8000'] + summary['available_stock_t9']
    )
    summary['dos'] = (summary['last_30_days_consumption'] / available_total.replace(0, pd.NA)).fillna(0).round(2)
    return summary

def compute_dept_summary(data, group_col='sub_department'):
    cols = ['new_sales', 'repl', 'last_30_days_consumption', 'unmapped',
            'available_a50', 'available_others', 'available_stock_a910_dx8000',
            'available_stock_t9', 'intransit', 'warehouse_pickup_pending', 'total_stock']
    summary = data.groupby(group_col)[cols].sum().reset_index()
    available_unmapped = (
        summary['unmapped'] + summary['available_a50'] + summary['available_others']
        + summary['available_stock_a910_dx8000'] + summary['available_stock_t9']
    )
    summary['dos'] = (summary['last_30_days_consumption'] / available_unmapped.replace(0, pd.NA)).fillna(0).round(0).astype(int)
    summary['available_unmapped'] = available_unmapped
    return summary[[group_col, 'repl', 'new_sales', 'last_30_days_consumption',
                     'available_unmapped', 'intransit', 'warehouse_pickup_pending',
                     'total_stock', 'dos']]

def add_total_row(df, group_col):
    total = df.drop(columns=[group_col]).sum(numeric_only=True)
    avail = total.get('available_unmapped', 0)
    cons = total.get('last_30_days_consumption', 0)
    total['dos'] = round(cons / avail, 2) if avail > 0 else 0
    total_row = pd.DataFrame([total])
    total_row[group_col] = 'Total'
    return pd.concat([df, total_row], ignore_index=True)


# ------------------------------------------------------------------
# 1a. Data Loading — Inventory & DOS tab (EDC / SB / Aggregate split)
# ------------------------------------------------------------------
@st.cache_data(ttl=3600)
def load_data():
    try:
        conn = trino.dbapi.connect(
            host="ppsl-trino-query-platform.paytmpayments.com",
            port=443,
            user="mayank2.dwivedi@paytmpayments.com",
            auth=BasicAuthentication("mayank2.dwivedi@paytmpayments.com", "mayank2#dwivedi51"),
            catalog="hive_offline",
            schema="team_og",
            http_scheme="https",
            session_properties=TRINO_SESSION_PROPERTIES
        )

        query = """
        WITH active_manpower AS (
            SELECT 
                CAST(cust_id AS VARCHAR) AS cust_id, e_code, agent_name AS name, 
                doj AS state,
                role_designation, city, sub_department, status, l1_manger AS l1,
                role_designation AS role, l1_manger AS l1_name, zone1 AS zone
            FROM team_og.manpower1
            WHERE lower(status) != 'inactive' AND cust_id IS NOT NULL
        ),
        stock_pivot AS (
            SELECT
                CAST(s.cust_id AS VARCHAR) AS cust_id,
                COUNT(CASE WHEN lower(s.device_type) = 'edc' AND (s.barcode_status = 5 OR lower(s.stage) = 'unmapped') THEN 1 END) AS edc_unmapped,
                COUNT(CASE WHEN lower(s.device_type) = 'sb'  AND (s.barcode_status = 5 OR lower(s.stage) = 'unmapped') THEN 1 END) AS sb_unmapped,
                COUNT(CASE WHEN lower(s.device_type) = 'edc' AND upper(s.device_model) LIKE '%A50%' AND lower(s.stage) = 'available' THEN 1 END) AS edc_a50,
                COUNT(CASE WHEN lower(s.device_type) = 'sb'  AND upper(s.device_model) LIKE '%A50%' AND lower(s.stage) = 'available' THEN 1 END) AS sb_a50,
                COUNT(CASE WHEN lower(s.device_type) = 'edc' AND (upper(s.device_model) LIKE '%A910%' OR upper(s.device_model) LIKE '%DX8000%') AND lower(s.stage) = 'available' THEN 1 END) AS edc_a910_dx8000,
                COUNT(CASE WHEN lower(s.device_type) = 'sb'  AND (upper(s.device_model) LIKE '%A910%' OR upper(s.device_model) LIKE '%DX8000%') AND lower(s.stage) = 'available' THEN 1 END) AS sb_a910_dx8000,
                COUNT(CASE WHEN lower(s.device_type) = 'edc' AND upper(s.device_model) LIKE '%T9%' AND lower(s.stage) = 'available' THEN 1 END) AS edc_t9,
                COUNT(CASE WHEN lower(s.device_type) = 'sb'  AND upper(s.device_model) LIKE '%T9%' AND lower(s.stage) = 'available' THEN 1 END) AS sb_t9,
                COUNT(CASE WHEN lower(s.device_type) = 'edc' AND upper(s.device_model) NOT LIKE '%A50%' AND upper(s.device_model) NOT LIKE '%A910%' AND upper(s.device_model) NOT LIKE '%DX8000%' AND upper(s.device_model) NOT LIKE '%T9%' AND lower(s.stage) = 'available' THEN 1 END) AS edc_others,
                COUNT(CASE WHEN lower(s.device_type) = 'sb'  AND upper(s.device_model) NOT LIKE '%A50%' AND upper(s.device_model) NOT LIKE '%A910%' AND upper(s.device_model) NOT LIKE '%DX8000%' AND upper(s.device_model) NOT LIKE '%T9%' AND lower(s.stage) = 'available' THEN 1 END) AS sb_others,
                COUNT(CASE WHEN lower(s.device_type) = 'edc' AND lower(s.stage) = 'forward_in_transit' THEN 1 END) AS edc_intransit,
                COUNT(CASE WHEN lower(s.device_type) = 'sb'  AND lower(s.stage) = 'forward_in_transit' THEN 1 END) AS sb_intransit
            FROM user_mayank2_dwivedi.ats_stock s
            INNER JOIN active_manpower am ON CAST(s.cust_id AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_edc_dep AS (
            SELECT CAST(final_agent_custid AS VARCHAR) AS cust_id, COUNT(DISTINCT device_id) AS edc_dep
            FROM user_mayank2_dwivedi.ats_edc_dep d
            INNER JOIN active_manpower am ON CAST(d.final_agent_custid AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_edc_rep AS (
            SELECT CAST(Agent_Cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT NEW_Device_number) AS edc_rep
            FROM user_mayank2_dwivedi.ats_edc_rep r
            INNER JOIN active_manpower am ON CAST(r.Agent_Cust_id AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_edc_upg AS (
            SELECT CAST(Agent_Cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT NEW_Device_number) AS edc_upg
            FROM user_mayank2_dwivedi.ats_edc_upg u
            INNER JOIN active_manpower am ON CAST(u.Agent_Cust_id AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_sb_dep AS (
            SELECT CAST(agent_cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT device_id) AS sb_dep
            FROM user_mayank2_dwivedi.ats_sb_dep sd
            INNER JOIN active_manpower am ON CAST(sd.agent_cust_id AS VARCHAR) = am.cust_id
            WHERE dl_last_updated >= date '1900-01-01'
            GROUP BY 1
        ),
        c_sb_rep AS (
            SELECT CAST(agent_cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT mapped_dsn) AS sb_upg
            FROM user_mayank2_dwivedi.ats_sb_rep sr
            INNER JOIN active_manpower am ON CAST(sr.agent_cust_id AS VARCHAR) = am.cust_id
            WHERE dl_last_updated >= date '1900-01-01'
            GROUP BY 1
        ),
        dispatch_intransit AS (
            SELECT
                CAST(receiver_e_code AS VARCHAR) AS e_code,
                SUM(
                    COALESCE(TRY_CAST(a910 AS INTEGER), 0) + 
                    COALESCE(TRY_CAST(a910s AS INTEGER), 0) +
                    COALESCE(TRY_CAST(dx800 AS INTEGER), 0) + 
                    COALESCE(TRY_CAST(t9_devices_non_emi_devices AS INTEGER), 0) + 
                    COALESCE(TRY_CAST(a50 AS INTEGER), 0)
                ) AS dispatch_intransit_devices
            FROM user_mayank2_dwivedi.edc_dispatch_overall_sept26
            GROUP BY 1
        ),
        employee_agg AS (
            SELECT
                m.e_code, m.cust_id, m.name, m.role_designation, m.city, m.state,
                m.sub_department, m.status, m.l1, m.role, m.l1_name, m.zone,
                COALESCE(b.edc_dep, 0) AS edc_dep,
                COALESCE(c.edc_rep, 0) AS edc_rep,
                COALESCE(d.edc_upg, 0) AS edc_upg,
                COALESCE(f.sb_dep, 0) AS sb_dep,
                COALESCE(g.sb_upg, 0) AS sb_upg,
                COALESCE(sp.edc_unmapped, 0) AS edc_unmapped,
                COALESCE(sp.sb_unmapped, 0) AS sb_unmapped,
                COALESCE(sp.edc_a50, 0) AS edc_a50,
                COALESCE(sp.sb_a50, 0) AS sb_a50,
                COALESCE(sp.edc_others, 0) AS edc_others,
                COALESCE(sp.sb_others, 0) AS sb_others,
                COALESCE(sp.edc_a910_dx8000, 0) AS edc_a910_dx8000,
                COALESCE(sp.sb_a910_dx8000, 0) AS sb_a910_dx8000,
                COALESCE(sp.edc_t9, 0) AS edc_t9,
                COALESCE(sp.sb_t9, 0) AS sb_t9,
                COALESCE(sp.edc_intransit, 0) AS edc_intransit,
                COALESCE(sp.sb_intransit, 0) AS sb_intransit,
                COALESCE(disp.dispatch_intransit_devices, 0) AS dispatch_intransit_devices
            FROM active_manpower m
            LEFT JOIN stock_pivot sp ON m.cust_id = sp.cust_id
            LEFT JOIN c_edc_dep b ON m.cust_id = b.cust_id
            LEFT JOIN c_edc_rep c ON m.cust_id = c.cust_id
            LEFT JOIN c_edc_upg d ON m.cust_id = d.cust_id
            LEFT JOIN c_sb_dep f ON m.cust_id = f.cust_id
            LEFT JOIN c_sb_rep g ON m.cust_id = g.cust_id
            LEFT JOIN dispatch_intransit disp ON m.e_code = disp.e_code
        ),
        final_metrics AS (
            SELECT
                ea.e_code, ea.cust_id, ea.name, ea.role_designation, ea.city, ea.state,
                ea.sub_department, ea.status, ea.l1, ea.role, ea.l1_name, ea.zone,
                dt.device_type,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_dep
                    WHEN 'SB'  THEN ea.sb_dep
                    ELSE ea.edc_dep + ea.sb_dep
                END AS new_sales,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_rep + ea.edc_upg
                    WHEN 'SB'  THEN ea.sb_upg
                    ELSE ea.edc_rep + ea.edc_upg + ea.sb_upg
                END AS repl,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_dep + ea.edc_rep + ea.edc_upg
                    WHEN 'SB'  THEN ea.sb_dep + ea.sb_upg
                    ELSE ea.edc_dep + ea.edc_rep + ea.edc_upg + ea.sb_dep + ea.sb_upg
                END AS last_30_days_consumption,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_unmapped
                    WHEN 'SB'  THEN ea.sb_unmapped
                    ELSE ea.edc_unmapped + ea.sb_unmapped
                END AS unmapped,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_a50
                    WHEN 'SB'  THEN ea.sb_a50
                    ELSE ea.edc_a50 + ea.sb_a50
                END AS available_a50,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_others
                    WHEN 'SB'  THEN ea.sb_others
                    ELSE ea.edc_others + ea.sb_others
                END AS available_others,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_a910_dx8000
                    WHEN 'SB'  THEN ea.sb_a910_dx8000
                    ELSE ea.edc_a910_dx8000 + ea.sb_a910_dx8000
                END AS available_stock_a910_dx8000,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_t9
                    WHEN 'SB'  THEN ea.sb_t9
                    ELSE ea.edc_t9 + ea.sb_t9
                END AS available_stock_t9,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.edc_intransit
                    WHEN 'SB'  THEN ea.sb_intransit
                    ELSE ea.edc_intransit + ea.sb_intransit
                END AS intransit,
                CASE dt.device_type
                    WHEN 'EDC' THEN ea.dispatch_intransit_devices
                    WHEN 'SB'  THEN 0
                    ELSE ea.dispatch_intransit_devices
                END AS warehouse_pickup_pending
            FROM employee_agg ea
            CROSS JOIN UNNEST(ARRAY['EDC', 'SB', 'Aggregate']) AS dt(device_type)
        )
        SELECT
            e_code, cust_id, name, role_designation, city, state, sub_department,
            status, l1, role, l1_name, zone, device_type,
            new_sales, repl,
            last_30_days_consumption, unmapped, available_a50, available_others,
            available_stock_a910_dx8000, available_stock_t9, intransit, warehouse_pickup_pending,
            (unmapped + available_a50 + available_others + available_stock_a910_dx8000 + available_stock_t9
             + intransit + warehouse_pickup_pending) AS total_stock,
            COALESCE(
                last_30_days_consumption 
                / NULLIF((unmapped + available_a50 + available_others + available_stock_a910_dx8000 + available_stock_t9), 0)
            , 0) AS dos,
            last_30_days_consumption * 1.10 AS plan_devices,
            (last_30_days_consumption * 1.10) - (intransit + warehouse_pickup_pending) AS yet_to_be_dispatched,
            COALESCE(
                last_30_days_consumption 
                / NULLIF((unmapped + available_a50 + available_others + available_stock_a910_dx8000), 0)
            , 0) AS without_t9_dos
        FROM final_metrics
        """
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        error_msg = str(e)
        if "message=" in error_msg:
            clean_err = error_msg.split("message=")[-1].split(", query_id")[0]
            st.error(f"Backend Database Error: {clean_err}")
        else:
            st.error(f"Query fail ho gayi: {e}")
        st.stop()


# ------------------------------------------------------------------
# 1b. Data Loading — Inventory tab (combined EDC+SB, no device_type split)
# ------------------------------------------------------------------
@st.cache_data(ttl=3600)
def load_inventory_data():
    try:
        conn = trino.dbapi.connect(
            host="ppsl-trino-query-platform.paytmpayments.com",
            port=443,
            user="mayank2.dwivedi@paytmpayments.com",
            auth=BasicAuthentication("mayank2.dwivedi@paytmpayments.com", "mayank2#dwivedi51"),
            catalog="hive_offline",
            schema="team_og",
            http_scheme="https",
            session_properties=TRINO_SESSION_PROPERTIES
        )

        query = """
        WITH active_manpower AS (
            SELECT 
                CAST(cust_id AS VARCHAR) AS cust_id, e_code, agent_name AS name, 
                doj AS state,
                role_designation, city, sub_department, status, l1_manger AS l1,
                role_designation AS role, l1_manger AS l1_name, zone1 AS zone
            FROM team_og.manpower1
            WHERE lower(status) != 'inactive' AND cust_id IS NOT NULL
        ),
        stock_agg AS (
            SELECT 
                CAST(s.cust_id AS VARCHAR) AS cust_id,
                COUNT(CASE WHEN s.barcode_status = 5 OR lower(s.stage) = 'unmapped' THEN 1 END) AS unmapped_stock,
                COUNT(CASE WHEN upper(s.device_model) LIKE '%A50%' AND lower(s.stage) = 'available' THEN 1 END) AS available_a50,
                COUNT(CASE WHEN (upper(s.device_model) LIKE '%A910%' OR upper(s.device_model) LIKE '%DX8000%') AND lower(s.stage) = 'available' THEN 1 END) AS available_a910_dx8000,
                COUNT(CASE WHEN upper(s.device_model) LIKE '%T9%' AND lower(s.stage) = 'available' THEN 1 END) AS available_t9,
                COUNT(CASE WHEN upper(s.device_model) NOT LIKE '%A50%' AND upper(s.device_model) NOT LIKE '%A910%' AND upper(s.device_model) NOT LIKE '%DX8000%' AND upper(s.device_model) NOT LIKE '%T9%' AND lower(s.stage) = 'available' THEN 1 END) AS available_others,
                COUNT(CASE WHEN lower(s.stage) = 'forward_in_transit' THEN 1 END) AS intransit_stock
            FROM user_mayank2_dwivedi.ats_stock s
            INNER JOIN active_manpower am ON CAST(s.cust_id AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_edc_dep AS (
            SELECT CAST(final_agent_custid AS VARCHAR) AS cust_id, COUNT(DISTINCT device_id) AS edc_dep
            FROM user_mayank2_dwivedi.ats_edc_dep d
            INNER JOIN active_manpower am ON CAST(d.final_agent_custid AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_edc_rep AS (
            SELECT CAST(Agent_Cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT NEW_Device_number) AS edc_rep
            FROM user_mayank2_dwivedi.ats_edc_rep r
            INNER JOIN active_manpower am ON CAST(r.Agent_Cust_id AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_edc_upg AS (
            SELECT CAST(Agent_Cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT NEW_Device_number) AS edc_upg
            FROM user_mayank2_dwivedi.ats_edc_upg u
            INNER JOIN active_manpower am ON CAST(u.Agent_Cust_id AS VARCHAR) = am.cust_id
            GROUP BY 1
        ),
        c_sb_dep AS (
            SELECT CAST(agent_cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT device_id) AS sb_dep
            FROM user_mayank2_dwivedi.ats_sb_dep sd
            INNER JOIN active_manpower am ON CAST(sd.agent_cust_id AS VARCHAR) = am.cust_id
            WHERE dl_last_updated >= date '1900-01-01'
            GROUP BY 1
        ),
        c_sb_rep AS (
            SELECT CAST(agent_cust_id AS VARCHAR) AS cust_id, COUNT(DISTINCT mapped_dsn) AS sb_upg
            FROM user_mayank2_dwivedi.ats_sb_rep sr
            INNER JOIN active_manpower am ON CAST(sr.agent_cust_id AS VARCHAR) = am.cust_id
            WHERE dl_last_updated >= date '1900-01-01'
            GROUP BY 1
        ),
        edc_dispatch_intransit AS (
            SELECT
                CAST(receiver_e_code AS VARCHAR) AS e_code,
                SUM(
                    COALESCE(TRY_CAST(a910 AS INTEGER), 0) + COALESCE(TRY_CAST(a910s AS INTEGER), 0) + COALESCE(TRY_CAST(dx800 AS INTEGER), 0) + COALESCE(TRY_CAST(t9_devices_non_emi_devices AS INTEGER), 0) + COALESCE(TRY_CAST(a50 AS INTEGER), 0)
                ) AS dispatch_intransit_devices
            FROM user_mayank2_dwivedi.edc_dispatch_overall_sept26
            GROUP BY 1
        ),
        sb_dispatch_intransit AS (
            SELECT
                CAST(receiver_e_code AS VARCHAR) AS e_code,
                SUM(COALESCE(TRY_CAST(sb_quantity AS INTEGER), 0)) AS dispatch_intransit_devices
            FROM user_mayank2_dwivedi.Sb_dispatch_overall_sept26
            GROUP BY 1
        )
        SELECT 
            m.e_code, m.cust_id, m.name, m.role_designation, m.city, m.state,
            m.sub_department, m.status, m.l1, m.role, m.l1_name, m.zone,
            (COALESCE(b.edc_dep, 0) + COALESCE(c.edc_rep, 0) + COALESCE(d.edc_upg, 0) + COALESCE(f.sb_dep, 0) + COALESCE(g.sb_upg, 0)) AS last_30_days_consumption,
            COALESCE(s.unmapped_stock, 0) AS unmapped,
            COALESCE(s.available_a50, 0) AS available_a50,
            COALESCE(s.available_others, 0) AS available_others,
            COALESCE(s.available_a910_dx8000, 0) AS available_stock_a910_dx8000,
            COALESCE(s.available_t9, 0) AS available_stock_t9,
            COALESCE(s.intransit_stock, 0) + COALESCE(edc_disp.dispatch_intransit_devices, 0) + COALESCE(sb_disp.dispatch_intransit_devices, 0) AS intransit,
            COALESCE(
                (COALESCE(b.edc_dep, 0) + COALESCE(c.edc_rep, 0) + COALESCE(d.edc_upg, 0) + COALESCE(f.sb_dep, 0) + COALESCE(g.sb_upg, 0))
                / NULLIF(COALESCE(s.unmapped_stock, 0) + COALESCE(s.available_a50, 0) + COALESCE(s.available_others, 0) + COALESCE(s.available_a910_dx8000, 0) + COALESCE(s.available_t9, 0), 0)
            , 0) AS dos,
            (COALESCE(b.edc_dep, 0) + COALESCE(c.edc_rep, 0) + COALESCE(d.edc_upg, 0) + COALESCE(f.sb_dep, 0) + COALESCE(g.sb_upg, 0)) * 1.10 AS plan_devices,
            ((COALESCE(b.edc_dep, 0) + COALESCE(c.edc_rep, 0) + COALESCE(d.edc_upg, 0) + COALESCE(f.sb_dep, 0) + COALESCE(g.sb_upg, 0)) * 1.10) - (COALESCE(s.intransit_stock, 0) + COALESCE(edc_disp.dispatch_intransit_devices, 0) + COALESCE(sb_disp.dispatch_intransit_devices, 0)) AS yet_to_be_dispatched,
            COALESCE(
                (COALESCE(b.edc_dep, 0) + COALESCE(c.edc_rep, 0) + COALESCE(d.edc_upg, 0) + COALESCE(f.sb_dep, 0) + COALESCE(g.sb_upg, 0))
                / NULLIF(COALESCE(s.unmapped_stock, 0) + COALESCE(s.available_a50, 0) + COALESCE(s.available_others, 0) + COALESCE(s.available_a910_dx8000, 0), 0)
            , 0) AS without_t9_dos
        FROM active_manpower m
        LEFT JOIN stock_agg s ON m.cust_id = s.cust_id
        LEFT JOIN c_edc_dep b ON m.cust_id = b.cust_id
        LEFT JOIN c_edc_rep c ON m.cust_id = c.cust_id
        LEFT JOIN c_edc_upg d ON m.cust_id = d.cust_id
        LEFT JOIN c_sb_dep f ON m.cust_id = f.cust_id
        LEFT JOIN c_sb_rep g ON m.cust_id = g.cust_id
        LEFT JOIN edc_dispatch_intransit edc_disp ON m.e_code = edc_disp.e_code
        LEFT JOIN sb_dispatch_intransit sb_disp ON m.e_code = sb_disp.e_code
        """
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        error_msg = str(e)
        if "message=" in error_msg:
            clean_err = error_msg.split("message=")[-1].split(", query_id")[0]
            st.error(f"Backend Database Error: {clean_err}")
        else:
            st.error(f"Query fail ho gayi: {e}")
        st.stop()


# ------------------------------------------------------------------
# 1c. Data Loading — Device Aging tab
# ------------------------------------------------------------------
@st.cache_data(ttl=3600)
def load_aging_data():
    try:
        conn = trino.dbapi.connect(
            host="ppsl-trino-query-platform.paytmpayments.com",
            port=443,
            user="mayank2.dwivedi@paytmpayments.com",
            auth=BasicAuthentication("mayank2.dwivedi@paytmpayments.com", "mayank2#dwivedi51"),
            catalog="hive_offline",
            schema="team_og",
            http_scheme="https",
            session_properties=TRINO_SESSION_PROPERTIES
        )

        query = """
        WITH active_manpower AS (
            SELECT 
                CAST(cust_id AS VARCHAR) AS cust_id, e_code, agent_name AS name, 
                doj AS state,
                role_designation, city, sub_department, status, l1_manger AS l1,
                role_designation AS role, l1_manger AS l1_name, zone1 AS zone
            FROM team_og.manpower1
            WHERE cust_id IS NOT NULL
        )
        SELECT
            am.e_code AS spoc_e_code,
            am.name AS spoc_name,
            am.status AS employee_status,
            am.zone,
            am.state,
            am.city,
            am.role,
            am.l1_name,
            am.sub_department,
            s.barcode,
            s.device_type,
            s.device_model,
            s.barcode_status,
            CASE 
                WHEN lower(s.stage) = 'available' THEN 'Available'
                WHEN lower(s.stage) = 'unmapped' THEN 'Unmapped'
                WHEN lower(s.stage) = 'forward_in_transit' THEN 'In-Transit'
                ELSE 'Unknown'
            END AS barcode_status_label,
            s.updated_at,
            date_diff('day', CAST(s.updated_at AS date), CURRENT_DATE) AS aging_days
        FROM user_mayank2_dwivedi.ats_stock s
        INNER JOIN active_manpower am ON CAST(s.cust_id AS VARCHAR) = am.cust_id
        WHERE s.updated_at IS NOT NULL
        AND lower(s.stage) IN ('available', 'unmapped', 'forward_in_transit')
        """
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        error_msg = str(e)
        if "message=" in error_msg:
            clean_err = error_msg.split("message=")[-1].split(", query_id")[0]
            st.error(f"Backend Database Error: {clean_err}")
        else:
            st.error(f"Query fail ho gayi: {e}")
        st.stop()


df_base = load_data()

st.title("Paytm MMB Inventory Dashboard")

tab_dos, tab_inventory, tab_edc_dispatch, tab_sb_dispatch, tab_aging = st.tabs([
    "Inventory Dos", "Inventory", "EDC Dispatch", "SB Dispatch", "Device Aging"
])

# ====================================================================
# TAB 1: DOS & Consumption Details
# ====================================================================
with tab_dos:
    st.markdown("### 🔍 Filters")
    f_col1, f_col2, f_col3, f_col4 = st.columns(4)
    selected_device_type = f_col1.selectbox(
        "Select View", options=['Aggregate', 'EDC', 'SB'], index=0, key="dos_view"
    )
    selected_zone = f_col2.multiselect(
        "Select Zone", options=sorted(df_base['zone'].dropna().unique()), key="dos_zone"
    )
    selected_state = f_col3.multiselect(
        "Select State", options=sorted(df_base['state'].dropna().unique()), key="dos_state"
    )
    selected_role = f_col4.multiselect(
        "Select Role", options=sorted(df_base['role'].dropna().unique()), key="dos_role"
    )

    df_filtered = df_base[df_base['device_type'] == selected_device_type].copy()
    if selected_zone:
        df_filtered = df_filtered[df_filtered['zone'].isin(selected_zone)]
    if selected_state:
        df_filtered = df_filtered[df_filtered['state'].isin(selected_state)]
    if selected_role:
        df_filtered = df_filtered[df_filtered['role'].isin(selected_role)]

    st.markdown(f"**Current View:** {selected_device_type} Devices — DOS & Consumption Details")
    st.markdown("---")

    total_available_filtered = (
        df_filtered['available_a50'].sum() + df_filtered['available_others'].sum()
        + df_filtered['available_stock_a910_dx8000'].sum() + df_filtered['available_stock_t9'].sum()
    )
    total_consumption_filtered = df_filtered['last_30_days_consumption'].sum()
    total_stock_for_dos = total_available_filtered + df_filtered['unmapped'].sum()
    overall_dos = (total_consumption_filtered / total_stock_for_dos) if total_stock_for_dos > 0 else 0

    col1, col2, col3, col4 = st.columns(4)
    col1.metric("Total Consumption (30 Days)", f"{total_consumption_filtered:,.0f}")
    col2.metric("Total In-Transit", f"{df_filtered['intransit'].sum():,.0f}")
    col3.metric("Total Available Stock", f"{total_available_filtered:,.0f}")
    col4.metric("Overall DOS", f"{overall_dos:.1f}")

    st.markdown("---")

    col_chart1, col_chart2 = st.columns(2)

    with col_chart1:
        st.subheader("Consumption by Zone")
        zone_df = (
            df_filtered.groupby('zone')['last_30_days_consumption']
            .sum()
            .reset_index()
            .sort_values('last_30_days_consumption', ascending=True)
        )
        if not zone_df.empty and zone_df['last_30_days_consumption'].sum() > 0:
            fig_zone = px.bar(
                zone_df, x='last_30_days_consumption', y='zone', orientation='h',
                text='last_30_days_consumption', color='last_30_days_consumption',
                color_continuous_scale='Blues',
            )
            fig_zone.update_traces(texttemplate='%{text:,.0f}', textposition='outside', marker_line_width=0)
            fig_zone.update_layout(
                coloraxis_showscale=False, xaxis_title="Devices Consumed", yaxis_title="",
                margin=dict(l=10, r=30, t=10, b=10), height=max(300, 32 * len(zone_df)),
                uniformtext_minsize=10,
            )
            st.plotly_chart(fig_zone, use_container_width=True)

    with col_chart2:
        st.subheader("Device Distribution")
        device_totals = {
            'A50': df_filtered['available_a50'].sum(),
            'A910/DX8000': df_filtered['available_stock_a910_dx8000'].sum(),
            'T9': df_filtered['available_stock_t9'].sum(),
            'Others': df_filtered['available_others'].sum()
        }
        device_totals = {k: v for k, v in device_totals.items() if v > 0}

        if device_totals:
            total_devices = sum(device_totals.values())
            fig_pie = go.Figure(data=[go.Pie(
                labels=list(device_totals.keys()),
                values=list(device_totals.values()),
                hole=0.55,
                marker=dict(colors=BRAND_COLORS, line=dict(color='#ffffff', width=2)),
                textinfo='label+value+percent',
                texttemplate='<b>%{label}</b><br>%{value:,.0f} (%{percent})',
                textposition='inside',
                insidetextorientation='radial',
                sort=False,
            )])
            fig_pie.update_layout(
                showlegend=True,
                legend=dict(orientation="h", yanchor="bottom", y=-0.15, xanchor="center", x=0.5),
                margin=dict(l=10, r=10, t=10, b=10),
                annotations=[dict(
                    text=f"<b>{total_devices:,.0f}</b><br>Total",
                    x=0.5, y=0.5, font_size=16, showarrow=False
                )],
            )
            st.plotly_chart(fig_pie, use_container_width=True)
        else:
            st.info("No available device data for selected filters.")

    st.markdown("---")
    st.subheader("🚨 Critical Zones — Lowest DOS")
    critical_zone_dos = compute_group_summary(df_filtered, 'zone')
    critical_zone_dos = (
        critical_zone_dos[critical_zone_dos['last_30_days_consumption'] > 0]
        .sort_values('dos', ascending=True)
        .head(5)
        .sort_values('dos', ascending=False)
    )
    if not critical_zone_dos.empty:
        fig_critical_dos = px.bar(
            critical_zone_dos, x='dos', y='zone', orientation='h', text='dos',
            color_discrete_sequence=[ALERT],
        )
        fig_critical_dos.update_traces(texttemplate='%{text:.1f}', textposition='outside')
        fig_critical_dos.update_layout(
            xaxis_title="DOS (Consumption/Stock)", yaxis_title="",
            margin=dict(l=10, r=30, t=10, b=10), height=280,
        )
        st.plotly_chart(fig_critical_dos, use_container_width=True)
    else:
        st.info("Not enough consumption data to compute DOS.")

    st.markdown("---")
    st.subheader("📍 State-wise Summary")
    state_summary = compute_group_summary(df_filtered, 'state')
    st.dataframe(
        styled_table_colored(state_summary.sort_values('last_30_days_consumption', ascending=False)),
        use_container_width=True, height=300
    )

    st.markdown("---")
    st.subheader("👤 RH-wise Summary")
    rh_summary = compute_group_summary(df_filtered, 'zone')
    st.dataframe(
        styled_table_colored(rh_summary.sort_values('last_30_days_consumption', ascending=False)),
        use_container_width=True, height=300
    )

    st.markdown("---")
    st.subheader("📉 RH-wise DOS")
    rh_dos = (
        rh_summary[rh_summary['last_30_days_consumption'] > 0]
        .sort_values('dos', ascending=True)
    )
    if not rh_dos.empty:
        fig_rh_dos = px.bar(
            rh_dos, x='dos', y='zone', orientation='h', text='dos',
            color_discrete_sequence=[ALERT],
        )
        fig_rh_dos.update_traces(texttemplate='%{text:.1f}', textposition='outside')
        fig_rh_dos.update_layout(
            xaxis_title="DOS (Consumption/Stock)", yaxis_title="",
            margin=dict(l=10, r=30, t=10, b=10),
            height=max(300, 28 * len(rh_dos)),
        )
        st.plotly_chart(fig_rh_dos, use_container_width=True)
    else:
        st.info("Not enough consumption data to compute RH-wise DOS.")

    st.markdown("---")
    st.subheader(f"🏢 Department-wise Summary — {selected_device_type}")
    dept_summary = compute_dept_summary(df_filtered)
    dept_summary = dept_summary.sort_values('last_30_days_consumption', ascending=False)
    dept_summary = add_total_row(dept_summary, 'sub_department')
    st.dataframe(styled_table_colored(dept_summary), use_container_width=True, height=280)

    st.markdown("---")
    st.subheader("Agent Level Details")
    df_filtered_display = search_table(df_filtered, key="search_agent_details", columns=['e_code', 'name'])
    st.dataframe(styled_table(df_filtered_display), use_container_width=True, height=400)
    st.download_button(
        label="📥 Download Excel/CSV", data=convert_df_to_csv(df_filtered),
        file_name="dos_agent_data.csv", mime="text/csv"
    )


# ====================================================================
# TAB 2: Inventory
# ====================================================================
with tab_inventory:
    df_inv = load_inventory_data()

    st.markdown("### 🔍 Filters")
    f_col1, f_col2, f_col3 = st.columns(3)
    inv_zone = f_col1.multiselect(
        "Select Zone", options=sorted(df_inv['zone'].dropna().unique()), key="inv_zone"
    )
    inv_state = f_col2.multiselect(
        "Select State", options=sorted(df_inv['state'].dropna().unique()), key="inv_state"
    )
    inv_role = f_col3.multiselect(
        "Select Role", options=sorted(df_inv['role'].dropna().unique()), key="inv_role"
    )

    df_inv_f = df_inv.copy()
    if inv_zone:
        df_inv_f = df_inv_f[df_inv_f['zone'].isin(inv_zone)]
    if inv_state:
        df_inv_f = df_inv_f[df_inv_f['state'].isin(inv_state)]
    if inv_role:
        df_inv_f = df_inv_f[df_inv_f['role'].isin(inv_role)]

    st.markdown("**Combined EDC + Soundbox Inventory View**")
    st.markdown("---")

    total_available = (
        df_inv_f['available_a50'].sum() + df_inv_f['available_others'].sum()
        + df_inv_f['available_stock_a910_dx8000'].sum() + df_inv_f['available_stock_t9'].sum()
    )
    total_unmapped_inv = df_inv_f['unmapped'].sum()
    total_consumption_inv = df_inv_f['last_30_days_consumption'].sum()
    total_stock_for_dos_inv = total_available + total_unmapped_inv
    overall_dos_inv = (total_consumption_inv / total_stock_for_dos_inv) if total_stock_for_dos_inv > 0 else 0

    k1, k2, k3, k4 = st.columns(4)
    k1.metric("Total Unmapped", f"{total_unmapped_inv:,.0f}")
    k2.metric("Total Available", f"{total_available:,.0f}")
    k3.metric("Overall DOS", f"{overall_dos_inv:.1f}")
    k4.metric("Yet To Be Dispatched", f"{df_inv_f['yet_to_be_dispatched'].sum():,.0f}")

    m1, m2, m3, m4 = st.columns(4)
    m1.metric("A50", f"{df_inv_f['available_a50'].sum():,.0f}")
    m2.metric("A910/A910S/DX8000", f"{df_inv_f['available_stock_a910_dx8000'].sum():,.0f}")
    m3.metric("T9", f"{df_inv_f['available_stock_t9'].sum():,.0f}")
    m4.metric("Others", f"{df_inv_f['available_others'].sum():,.0f}")

    st.markdown("---")

    col_i1, col_i2 = st.columns(2)

    with col_i1:
        st.subheader("Available Stock by Device Model")
        model_totals = {
            'A50': df_inv_f['available_a50'].sum(),
            'A910/DX8000': df_inv_f['available_stock_a910_dx8000'].sum(),
            'T9': df_inv_f['available_stock_t9'].sum(),
            'Others': df_inv_f['available_others'].sum(),
        }
        model_df = pd.DataFrame(
            [(k, v) for k, v in model_totals.items() if v > 0], columns=['model', 'count']
        ).sort_values('count', ascending=True)

        if not model_df.empty:
            fig_model = px.bar(
                model_df, x='count', y='model', orientation='h', text='count',
                color_discrete_sequence=[BRAND_COLORS[0]],
            )
            fig_model.update_traces(texttemplate='%{text:,.0f}', textposition='outside')
            fig_model.update_layout(
                xaxis_title="Available Devices", yaxis_title="",
                margin=dict(l=10, r=30, t=10, b=10), height=280,
            )
            st.plotly_chart(fig_model, use_container_width=True)
        else:
            st.info("No available stock for selected filters.")

    with col_i2:
        st.subheader("📍 Top 5 States — Critical Inventory (Lowest DOS)")
        critical_state_dos = compute_group_summary(df_inv_f, 'state')
        critical_state_dos = (
            critical_state_dos[critical_state_dos['last_30_days_consumption'] > 0]
            .sort_values('dos', ascending=True)
            .head(5)
            .sort_values('dos', ascending=False)
        )
        if not critical_state_dos.empty:
            fig_critical_state = px.bar(
                critical_state_dos, x='dos', y='state', orientation='h', text='dos',
                color_discrete_sequence=[ALERT],
            )
            fig_critical_state.update_traces(texttemplate='%{text:.1f}', textposition='outside')
            fig_critical_state.update_layout(
                xaxis_title="DOS (Consumption/Stock)", yaxis_title="",
                margin=dict(l=10, r=30, t=10, b=10), height=280,
            )
            st.plotly_chart(fig_critical_state, use_container_width=True)
        else:
            st.info("Not enough consumption data to compute state-wise DOS.")

    st.markdown("---")

    st.subheader("🔴 Zero Inventory Alert")
    zero_inv = df_inv_f[
        (df_inv_f['unmapped'] == 0) &
        (df_inv_f['available_a50'] == 0) &
        (df_inv_f['available_others'] == 0) &
        (df_inv_f['available_stock_a910_dx8000'] == 0) &
        (df_inv_f['available_stock_t9'] == 0) &
        (df_inv_f['intransit'] == 0)
    ]
    st.markdown(f"**{len(zero_inv):,} employees with zero inventory** (no stock, nothing in-transit)")
    zero_inv_display = search_table(
        zero_inv[['e_code', 'name', 'role', 'zone', 'state', 'l1_name']],
        key="search_zero_inv", columns=['e_code', 'name']
    )
    st.dataframe(styled_table(zero_inv_display), use_container_width=True, height=250)

    st.markdown("---")
    st.subheader("📦 Unmapped Device Details")
    st.caption("Row-level list of every currently Unmapped device, respecting the filters above.")

    df_aging_for_unmapped = load_aging_data()
    unmapped_devices = df_aging_for_unmapped[df_aging_for_unmapped['barcode_status_label'] == 'Unmapped'].copy()
    if inv_zone:
        unmapped_devices = unmapped_devices[unmapped_devices['zone'].isin(inv_zone)]
    if inv_state:
        unmapped_devices = unmapped_devices[unmapped_devices['state'].isin(inv_state)]
    if inv_role:
        unmapped_devices = unmapped_devices[unmapped_devices['role'].isin(inv_role)]

    unmapped_cols = ['spoc_e_code', 'spoc_name', 'zone', 'state', 'role', 'l1_name',
                      'barcode', 'device_type', 'device_model', 'updated_at', 'aging_days']

    unmapped_devices_display = search_table(
        unmapped_devices[unmapped_cols], key="search_unmapped_devices",
        columns=['spoc_e_code', 'spoc_name', 'barcode']
    )
    st.markdown(f"**{len(unmapped_devices):,} unmapped devices found**")
    st.dataframe(
        styled_table(unmapped_devices_display.sort_values('aging_days', ascending=False)),
        use_container_width=True, height=400
    )
    st.download_button(
        label="📥 Download Unmapped Devices", data=convert_df_to_csv(unmapped_devices[unmapped_cols]),
        file_name="unmapped_devices.csv", mime="text/csv"
    )


# ====================================================================
# TAB 3: EDC Dispatch Planning
# ====================================================================
with tab_edc_dispatch:
    st.markdown("### 🔍 Filters")
    f_col1, f_col2, f_col3 = st.columns(3)
    edc_zone = f_col1.multiselect("Select Zone", options=sorted(df_base['zone'].dropna().unique()), key="edc_zone")
    edc_state = f_col2.multiselect("Select State", options=sorted(df_base['state'].dropna().unique()), key="edc_state")
    edc_role = f_col3.multiselect("Select Role", options=sorted(df_base['role'].dropna().unique()), key="edc_role")

    df_edc = df_base[df_base['device_type'] == 'EDC'].copy()
    if edc_zone:
        df_edc = df_edc[df_edc['zone'].isin(edc_zone)]
    if edc_state:
        df_edc = df_edc[df_edc['state'].isin(edc_state)]
    if edc_role:
        df_edc = df_edc[df_edc['role'].isin(edc_role)]

    st.markdown("---")

    c1, c2, c3, c4 = st.columns(4)
    c1.metric("30 Days Consumption (EDC)", f"{df_edc['last_30_days_consumption'].sum():,.0f}")
    c2.metric("Total Plan Devices (EDC)", f"{df_edc['plan_devices'].sum():,.0f}")
    c3.metric("Already In-Transit (EDC)", f"{(df_edc['intransit'] + df_edc['warehouse_pickup_pending']).sum():,.0f}")
    c4.metric("Yet To Be Dispatched (EDC)", f"{df_edc['yet_to_be_dispatched'].sum():,.0f}")

    st.markdown("---")
    st.subheader("Agent Level Dispatch Requirements (EDC)")

    dispatch_cols = ['e_code', 'name', 'zone', 'state', 'role', 'l1_name',
                      'last_30_days_consumption', 'plan_devices', 'intransit',
                      'warehouse_pickup_pending', 'yet_to_be_dispatched']

    df_edc_display = search_table(df_edc[dispatch_cols], key="search_edc_dispatch", columns=['e_code', 'name'])
    st.dataframe(styled_table(df_edc_display), use_container_width=True, height=400)
    st.download_button(
        label="📥 Download Dispatch Data", data=convert_df_to_csv(df_edc[dispatch_cols]),
        file_name="edc_dispatch.csv", mime="text/csv"
    )


# ====================================================================
# TAB 4: SB Dispatch Planning
# ====================================================================
with tab_sb_dispatch:
    st.markdown("### 🔍 Filters")
    f_col1, f_col2, f_col3 = st.columns(3)
    sb_zone = f_col1.multiselect("Select Zone", options=sorted(df_base['zone'].dropna().unique()), key="sb_zone")
    sb_state = f_col2.multiselect("Select State", options=sorted(df_base['state'].dropna().unique()), key="sb_state")
    sb_role = f_col3.multiselect("Select Role", options=sorted(df_base['role'].dropna().unique()), key="sb_role")

    df_sb = df_base[df_base['device_type'] == 'SB'].copy()
    if sb_zone:
        df_sb = df_sb[df_sb['zone'].isin(sb_zone)]
    if sb_state:
        df_sb = df_sb[df_sb['state'].isin(sb_state)]
    if sb_role:
        df_sb = df_sb[df_sb['role'].isin(sb_role)]

    st.markdown("---")

    c1, c2, c3, c4 = st.columns(4)
    c1.metric("30 Days Consumption (SB)", f"{df_sb['last_30_days_consumption'].sum():,.0f}")
    c2.metric("Total Plan Devices (SB)", f"{df_sb['plan_devices'].sum():,.0f}")
    c3.metric("Already In-Transit (SB)", f"{df_sb['intransit'].sum():,.0f}")
    c4.metric("Yet To Be Dispatched (SB)", f"{df_sb['yet_to_be_dispatched'].sum():,.0f}")

    st.markdown("---")
    st.subheader("Agent Level Dispatch Requirements (SB)")

    dispatch_cols_sb = ['e_code', 'name', 'zone', 'state', 'role', 'l1_name',
                         'last_30_days_consumption', 'plan_devices', 'intransit',
                         'yet_to_be_dispatched']

    df_sb_display = search_table(df_sb[dispatch_cols_sb], key="search_sb_dispatch", columns=['e_code', 'name'])
    st.dataframe(styled_table(df_sb_display), use_container_width=True, height=400)
    st.download_button(
        label="📥 Download Dispatch Data", data=convert_df_to_csv(df_sb[dispatch_cols_sb]),
        file_name="sb_dispatch.csv", mime="text/csv"
    )


# ====================================================================
# TAB 5: Device Aging
# ====================================================================
with tab_aging:
    df_aging = load_aging_data()

    def bucket_aging(days):
        if pd.isna(days):
            return "Unknown"
        elif days <= 15:
            return "0-15 days"
        elif days <= 30:
            return "15-30 days"
        elif days <= 45:
            return "30-45 days"
        elif days <= 60:
            return "45-60 days"
        else:
            return "60+ days"

    df_aging['aging_bucket'] = df_aging['aging_days'].apply(bucket_aging)
    df_aging['device_cost'] = df_aging['device_type'].map(DEVICE_COST).fillna(0)

    st.markdown("### 🔍 Filters")
    f_col1, f_col2, f_col3, f_col4 = st.columns(4)
    aging_zone = f_col1.multiselect("Select Zone", options=sorted(df_aging['zone'].dropna().unique()), key="aging_zone")
    aging_state = f_col2.multiselect("Select State", options=sorted(df_aging['state'].dropna().unique()), key="aging_state")
    aging_device_type = f_col3.multiselect("Select Device Type", options=sorted(df_aging['device_type'].dropna().unique()), key="aging_device_type")
    aging_status = f_col4.multiselect("Select Status (Available/Unmapped/In-Transit)", options=sorted(df_aging['barcode_status_label'].dropna().unique()), key="aging_status")

    f_col5, f_col6, f_col7 = st.columns(3)
    aging_bucket_order = ["0-15 days", "15-30 days", "30-45 days", "45-60 days", "60+ days"]
    aging_bucket_filter = f_col5.multiselect("Select Aging Bucket", options=aging_bucket_order, key="aging_bucket_filter")
    min_aging_days = f_col6.number_input("Minimum Aging (Days)", min_value=0, value=0, step=1, key="aging_min_days")
    emp_status_filter = f_col7.multiselect(
        "Employee Status", options=sorted(df_aging['employee_status'].dropna().unique()), key="aging_emp_status"
    )

    df_aging_f = df_aging.copy()
    if aging_zone:
        df_aging_f = df_aging_f[df_aging_f['zone'].isin(aging_zone)]
    if aging_state:
        df_aging_f = df_aging_f[df_aging_f['state'].isin(aging_state)]
    if aging_device_type:
        df_aging_f = df_aging_f[df_aging_f['device_type'].isin(aging_device_type)]
    if aging_status:
        df_aging_f = df_aging_f[df_aging_f['barcode_status_label'].isin(aging_status)]
    if aging_bucket_filter:
        df_aging_f = df_aging_f[df_aging_f['aging_bucket'].isin(aging_bucket_filter)]
    if min_aging_days > 0:
        df_aging_f = df_aging_f[df_aging_f['aging_days'] >= min_aging_days]
    if emp_status_filter:
        df_aging_f = df_aging_f[df_aging_f['employee_status'].isin(emp_status_filter)]

    st.markdown("**Device Aging — every device currently Available/Unmapped/In-Transit against a SPOC (including inactive/notice-period employees for recovery tracking)**")
    st.markdown("---")

    cost_60plus = df_aging_f.loc[df_aging_f['aging_days'] > 60, 'device_cost'].sum()

    a1, a2, a3, a4, a5 = st.columns(5)
    a1.metric("Total Devices", f"{len(df_aging_f):,}")
    a2.metric("Avg Aging (Days)", f"{df_aging_f['aging_days'].mean():.1f}" if not df_aging_f.empty else "0")
    a3.metric("Max Aging (Days)", f"{df_aging_f['aging_days'].max():.0f}" if not df_aging_f.empty else "0")
    a4.metric("Devices Aged 60+ Days", f"{(df_aging_f['aging_days'] > 60).sum():,}")
    a5.metric("Locked Value (60+ Days)", f"₹{cost_60plus:,.0f}")

    st.markdown("---")

    col_a1, col_a2 = st.columns(2)

    with col_a1:
        st.subheader("Aging Bucket Distribution")
        bucket_counts = (
            df_aging_f['aging_bucket']
            .value_counts()
            .reindex(aging_bucket_order)
            .fillna(0)
            .reset_index()
        )
        bucket_counts.columns = ['bucket', 'count']
        if bucket_counts['count'].sum() > 0:
            fig_bucket = px.bar(
                bucket_counts, x='bucket', y='count', text='count',
                color_discrete_sequence=[BRAND_COLORS[0]],
            )
            fig_bucket.update_traces(texttemplate='%{text:,.0f}', textposition='outside')
            fig_bucket.update_layout(
                xaxis_title="Aging Bucket", yaxis_title="Device Count",
                margin=dict(l=10, r=30, t=10, b=10), height=320,
            )
            st.plotly_chart(fig_bucket, use_container_width=True)
        else:
            st.info("No data for selected filters.")

    with col_a2:
        st.subheader("Top 10 SPOCs — Highest Avg Aging")
        top_spoc_aging = (
            df_aging_f.groupby(['spoc_e_code', 'spoc_name'])['aging_days']
            .mean()
            .reset_index()
            .sort_values('aging_days', ascending=False)
            .head(10)
            .sort_values('aging_days', ascending=True)
        )
        if not top_spoc_aging.empty:
            top_spoc_aging['label'] = top_spoc_aging['spoc_name'] + " (" + top_spoc_aging['spoc_e_code'] + ")"
            fig_spoc = px.bar(
                top_spoc_aging, x='aging_days', y='label', orientation='h', text='aging_days',
                color_discrete_sequence=[ALERT],
            )
            fig_spoc.update_traces(texttemplate='%{text:.0f}', textposition='outside')
            fig_spoc.update_layout(
                xaxis_title="Avg Aging (Days)", yaxis_title="",
                margin=dict(l=10, r=30, t=10, b=10), height=320,
            )
            st.plotly_chart(fig_spoc, use_container_width=True)
        else:
            st.info("No data for selected filters.")

    st.markdown("---")
    st.subheader("Aging by Status")
    status_aging = (
        df_aging_f.groupby('barcode_status_label')['aging_days']
        .agg(['count', 'mean'])
        .reset_index()
        .rename(columns={'count': 'device_count', 'mean': 'avg_aging_days'})
        .sort_values('device_count', ascending=False)
    )
    status_aging['avg_aging_days'] = status_aging['avg_aging_days'].round(1)
    st.dataframe(styled_table(status_aging), use_container_width=True, height=250)

    st.markdown("---")
    st.subheader("🔴 Top 50 Highest Aging Devices")
    st.caption("Note: Dashboard ki speed maintain rakhne ke liye table mein sirf top 50 sabse zyada purane (highest aging) devices dikhaye gaye hain. Poora data download karne ke liye niche diye gaye button ka use karein.")

    device_cols = ['spoc_e_code', 'spoc_name', 'employee_status', 'zone', 'state', 'role', 'l1_name',
                    'barcode', 'device_type', 'device_model', 'barcode_status_label',
                    'updated_at', 'aging_days', 'aging_bucket', 'device_cost']

    display_df = df_aging_f[device_cols].sort_values('aging_days', ascending=False).head(50)
    display_df = search_table(display_df, key="search_top50_aging", columns=['spoc_e_code', 'spoc_name', 'barcode'])

    st.dataframe(
        styled_table(display_df),
        use_container_width=True, height=450
    )

    st.download_button(
        label="📥 Download All Aging Data", 
        data=convert_df_to_csv(df_aging_f[device_cols].sort_values('aging_days', ascending=False)),
        file_name="device_aging_full.csv", 
        mime="text/csv"
    )
