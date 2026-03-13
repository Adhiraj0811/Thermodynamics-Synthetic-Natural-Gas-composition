# Thermodynamics-Synthetic-Natural-Gas-composition
import streamlit as st
import numpy as np
import matplotlib.pyplot as plt
from thermopack.cubic import cubic
from thermopack.pcsaft import pcsaft

# --- STREAMLIT UI SETUP ---
st.set_page_config(
    page_title="SNG1 Phase Envelope", 
    page_icon="📈", 
    layout="wide",
    initial_sidebar_state="expanded"
)

# --- SIDEBAR DETAILS ---
with st.sidebar:
    st.image("https://cdn-icons-png.flaticon.com/512/1998/1998650.png", width=60) 
    st.title("Model Details")
    st.markdown("Recreation of Figure 1 from **Alfradique and Castier (2007)**: *Calculation of Phase Equilibrium of Natural Gases...*")
    
    st.divider()
    
    st.subheader("Mixture: SNG1")
    st.markdown("""
    **Composition (Mole Fraction):**
    * **Methane (C1):** 0.94085
    * **Ethane (C2):** 0.04468
    * **Isopentane (iC5):** 0.01447
    
    **Why this composition?**
    Synthetic Natural Gas (SNG) mixtures are studied to model real reservoir fluids:
    * **C1:** The dominant bulk component, representing typical "dry" natural gas.
    * **C2 & iC5:** The heavier hydrocarbon fractions. Even a tiny amount of a heavier component like Isopentane (~1.45%) dramatically stretches the phase envelope and dictates the dew point curve. This is critical for predicting when liquid condensation will occur!
    """)
    
    st.divider()
    st.caption("Powered by Thermopack & Streamlit")

# --- MAIN PAGE HEADER ---
st.title("Phase Equilibria of SNG1 Mixture")

# --- PROJECT CONTEXT ---
st.markdown("""
**Project Context & Industrial Relevance:**
This dashboard recreates a thermodynamic analysis comparing the Peng-Robinson and PC-SAFT Equations of State (EOS). 
Accurate phase equilibrium prediction is essential in the natural gas industry to design separation facilities, properly size equipment, and prevent costly liquid condensation ("dropout") during pipeline transportation.

The objective here is to calculate the phase envelopes and critical points of the Synthetic Natural Gas (SNG1) mixture. To strictly evaluate the pure predictive capabilities of these models, all binary interaction parameters ($k_{ij}$) are set to zero. Ultimately, this compares whether the complex, molecular-based PC-SAFT model offers better phase behavior predictions than the industry-standard cubic Peng-Robinson EOS.
""")

# --- NEW MODEL DESCRIPTIONS ---
desc_col1, desc_col2 = st.columns(2)

with desc_col1:
    st.info("""
    **Peng–Robinson Model (PR EOS)**
    * A cubic equation of state commonly used in the oil and gas industry.
    * Used to predict pressure–temperature–volume relationships and phase behavior of gases.
    * Simple and fast to calculate, so widely used in engineering simulations.
    * Less accurate near the critical region and for complex gas mixtures.
    """)

with desc_col2:
    st.warning("""
    **PC-SAFT Model**
    * An advanced molecular-based equation of state derived from statistical thermodynamics.
    * Represents molecules as chains of segments, giving more realistic molecular interactions.
    * Provides more accurate prediction of dew points, critical points, and phase behavior.
    * Generally shows better agreement with experimental data for natural gas mixtures.
    """)

st.divider()

# --- CACHED CALCULATIONS (STRICTLY UNTOUCHED) ---
@st.cache_data
def calculate_sng1():
    fluids_1 = 'C1,C2,IC5'
    z_1 = [0.94085, 0.04468, 0.01447]

    # --- PENG-ROBINSON (PR) CALCULATION ---
    eos_pr = cubic(fluids_1, 'PR')
    for i in range(1, 4):
        for j in range(1, 4):
            if i != j:  
                eos_pr.set_kij(i, j, 0.0)

    T_pr, P_pr = eos_pr.get_envelope_twophase(1e5, z_1)
    T_pr = np.array(T_pr)
    P_pr_MPa = np.array(P_pr) / 1e6

    T_guess_pr = T_pr[np.argmax(P_pr_MPa)]
    Tc_pr, vc_pr, Pc_pr = eos_pr.critical(z_1, temp=T_guess_pr)
    Pc_pr_MPa = Pc_pr / 1e6

    # --- PC-SAFT CALCULATION ---
    eos_saft = pcsaft(fluids_1)
    for i in range(1, 4):
        for j in range(1, 4):
            if i != j:
                eos_saft.set_kij(i, j, 0.0)

    T_saft, P_saft = eos_saft.get_envelope_twophase(1e5, z_1)
    T_saft = np.array(T_saft)
    P_saft_MPa = np.array(P_saft) / 1e6

    T_guess_saft = T_saft[np.argmax(P_saft_MPa)]
    Tc_saft, vc_saft, Pc_saft = eos_saft.critical(z_1, temp=T_guess_saft)
    Pc_saft_MPa = Pc_saft / 1e6

    return (T_pr, P_pr_MPa, Tc_pr, Pc_pr_MPa, 
            T_saft, P_saft_MPa, Tc_saft, Pc_saft_MPa)

with st.spinner("Calculating rigorous phase envelopes and critical points..."):
    (T_pr, P_pr_MPa, Tc_pr, Pc_pr_MPa, 
     T_saft, P_saft_MPa, Tc_saft, Pc_saft_MPa) = calculate_sng1()

# --- LAYOUT: COLUMNS ---
col1, col2 = st.columns([3, 1]) 

with col1:
    # --- UPGRADED MATPLOTLIB VISUALS (UNTOUCHED FROM PREVIOUS) ---
    fig, ax = plt.subplots(figsize=(10, 6), dpi=150)
    ax.set_facecolor('#fafafa')
    fig.patch.set_facecolor('#ffffff')

    # Plot Phase Envelopes 
    ax.plot(T_pr, P_pr_MPa, color='#1f77b4', linestyle='-', linewidth=2.5, label='Peng-Robinson EOS')
    ax.plot(T_saft, P_saft_MPa, color='#ff7f0e', linestyle='--', linewidth=2.5, label='PC-SAFT EOS')

    # Plot Exact Calculated Critical Points 
    ax.plot(Tc_pr, Pc_pr_MPa, marker='o', markersize=9, color='#1f77b4', 
            markeredgecolor='black', markeredgewidth=1.5, label='PR Critical Point')
    ax.plot(Tc_saft, Pc_saft_MPa, marker='^', markersize=10, color='#ff7f0e', 
            markeredgecolor='black', markeredgewidth=1.5, label='PC-SAFT Critical Point')

    # Formatting 
    ax.set_title('Figure 1: Phase Envelope for SNG1', fontsize=16, fontweight='bold', pad=15)
    ax.set_xlabel('Temperature (K)', fontsize=13, fontweight='500')
    ax.set_ylabel('Pressure (MPa)', fontsize=13, fontweight='500')

    # Specific Axes limits
    ax.set_xlim(200, 280)  
    ax.set_ylim(0, 10)

    # Styling ticks and grid
    ax.tick_params(direction='in', length=6, width=1, labelsize=11, top=True, right=True) 
    ax.grid(True, linestyle=':', alpha=0.6, color='gray') 

    ax.legend(loc='lower left', frameon=True, facecolor='white', edgecolor='lightgray', fontsize=11)
    fig.tight_layout()

    # Render Graph
    st.pyplot(fig)

with col2:
    # --- METRICS PANEL (UNTOUCHED FROM PREVIOUS) ---
    st.markdown("### Critical Points")
    st.markdown("Calculated dynamically using exact Newton-Raphson solvers.")
    
    st.info("**Peng-Robinson (PR)**")
    st.metric(label="Temperature", value=f"{Tc_pr:.2f} K")
    st.metric(label="Pressure", value=f"{Pc_pr_MPa:.2f} MPa")
    
    st.warning("**PC-SAFT**")
    st.metric(label="Temperature", value=f"{Tc_saft:.2f} K")
    st.metric(label="Pressure", value=f"{Pc_saft_MPa:.2f} MPa")
