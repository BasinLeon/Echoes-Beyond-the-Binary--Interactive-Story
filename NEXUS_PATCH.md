# BASIN::NEXUS Patch - Fix Garbled Text & Add Resume Analysis

## Issue 1: Garbled "keyḃ̈a͠BATTLESTATION" Text

The garbled text is Streamlit's Material Icons (`keyboard_arrow_right`) rendering incorrectly.

### Fix: Add this CSS to your existing style block (around line 530)

Replace the existing icon-hiding CSS with this more aggressive version:

```css
/* === CRITICAL FIX: Hide Material Icons rendering as garbled text === */
/* This fixes the "keyḃ̈a͠BATTLESTATION" issue */

/* Hide the SVG toggle icons completely */
[data-testid="stExpander"] svg,
.streamlit-expanderHeader svg {
    display: none !important;
    visibility: hidden !important;
    width: 0 !important;
    height: 0 !important;
}

/* Hide any text that starts with 'key' (the garbled icon text) */
[data-testid="stExpander"] summary::before,
.streamlit-expanderHeader::before {
    content: "" !important;
}

/* Force clean display of expander header text only */
.streamlit-expanderHeader {
    font-size: 0 !important;  /* Hide everything first */
}

.streamlit-expanderHeader p,
.streamlit-expanderHeader span[data-testid="stMarkdownContainer"] {
    font-size: 0.9rem !important;  /* Then show only the text */
    font-family: 'Orbitron', sans-serif !important;
}

/* Nuclear option: Hide Material Icons font-face entirely if needed */
@font-face {
    font-family: 'Material Icons';
    src: none !important;
}

/* Hide stray icon text patterns */
[data-testid="stSidebar"] *:not(p):not(span):not(label):not(h1):not(h2):not(h3):not(input):not(textarea) {
    font-family: inherit !important;
}
```

---

## Issue 2: Resume vs JD Analysis Not Working

The UI shows "Ready for analysis!" but nothing actually analyzes the content.

### Fix: Add this analysis function BEFORE line 2860

Add this complete function somewhere around line 100-200 (after imports):

```python
def analyze_resume_vs_jd(resume_text: str, jd_text: str) -> dict:
    """
    Analyze resume against job description to find matches, gaps, and recommendations.
    Returns a comprehensive analysis dictionary.
    """
    import re
    from collections import Counter
    
    # Common skill keywords to look for
    SKILL_KEYWORDS = {
        # Technical
        'python', 'javascript', 'typescript', 'react', 'node', 'sql', 'aws', 'gcp', 'azure',
        'docker', 'kubernetes', 'api', 'rest', 'graphql', 'machine learning', 'ai', 'data',
        'analytics', 'tableau', 'salesforce', 'hubspot', 'marketo', 'segment',
        # GTM/Sales
        'pipeline', 'quota', 'revenue', 'arr', 'mrr', 'saas', 'enterprise', 'smb', 'mid-market',
        'outbound', 'inbound', 'prospecting', 'cold calling', 'discovery', 'demo', 'closing',
        'negotiation', 'stakeholder', 'c-suite', 'executive', 'vp', 'director',
        'account executive', 'ae', 'sdr', 'bdr', 'account manager', 'csm', 'customer success',
        'sales ops', 'revops', 'gtm', 'go-to-market', 'partner', 'channel', 'alliances',
        # Leadership
        'leadership', 'management', 'team', 'strategy', 'planning', 'coaching', 'mentoring',
        'hiring', 'scaling', 'growth', 'optimization', 'process', 'playbook', 'enablement',
        # Metrics
        'yoy', 'growth', 'increase', 'improvement', 'reduction', 'efficiency', 'roi', 'ltv',
        'cac', 'churn', 'retention', 'conversion', 'win rate', 'deal size', 'cycle time'
    }
    
    # Normalize text
    resume_lower = resume_text.lower()
    jd_lower = jd_text.lower()
    
    # Extract keywords from JD
    jd_keywords = set()
    for keyword in SKILL_KEYWORDS:
        if keyword in jd_lower:
            jd_keywords.add(keyword)
    
    # Extract keywords from Resume
    resume_keywords = set()
    for keyword in SKILL_KEYWORDS:
        if keyword in resume_lower:
            resume_keywords.add(keyword)
    
    # Calculate matches and gaps
    matched_keywords = jd_keywords & resume_keywords
    missing_keywords = jd_keywords - resume_keywords
    bonus_keywords = resume_keywords - jd_keywords
    
    # Calculate match score
    if len(jd_keywords) > 0:
        match_score = int((len(matched_keywords) / len(jd_keywords)) * 100)
    else:
        match_score = 0
    
    # Extract years of experience mentioned
    exp_pattern = r'(\d+)\+?\s*(?:years?|yrs?)'
    jd_exp = re.findall(exp_pattern, jd_lower)
    resume_exp = re.findall(exp_pattern, resume_lower)
    
    jd_years = max([int(y) for y in jd_exp]) if jd_exp else None
    resume_years = max([int(y) for y in resume_exp]) if resume_exp else None
    
    # Extract numbers/metrics from resume
    metrics_pattern = r'(\d+(?:\.\d+)?)\s*(%|percent|million|m\b|k\b|x\b|\$)'
    resume_metrics = re.findall(metrics_pattern, resume_lower)
    
    # Determine fit level
    if match_score >= 80:
        fit_level = "🟢 STRONG FIT"
        fit_color = "#00ff88"
    elif match_score >= 60:
        fit_level = "🟡 GOOD FIT"
        fit_color = "#FFD700"
    elif match_score >= 40:
        fit_level = "🟠 MODERATE FIT"
        fit_color = "#ff9500"
    else:
        fit_level = "🔴 WEAK FIT"
        fit_color = "#ff4444"
    
    return {
        "match_score": match_score,
        "fit_level": fit_level,
        "fit_color": fit_color,
        "matched_keywords": sorted(matched_keywords),
        "missing_keywords": sorted(missing_keywords),
        "bonus_keywords": sorted(bonus_keywords),
        "jd_keywords_count": len(jd_keywords),
        "resume_keywords_count": len(resume_keywords),
        "jd_years_required": jd_years,
        "resume_years_mentioned": resume_years,
        "metrics_found": len(resume_metrics),
        "resume_word_count": len(resume_text.split()),
        "jd_word_count": len(jd_text.split())
    }
```

### Then update the Arsenal section (around line 2964) to call this function

Find this code:

```python
if resume_text_input and jd_text_input:
    st.success("🚀 Ready for analysis!")
```

Replace with:

```python
if resume_text_input and jd_text_input:
    st.success("🚀 Ready for analysis!")
    
    # === AUTOMATIC ANALYSIS ===
    analysis = analyze_resume_vs_jd(resume_text_input, jd_text_input)
    
    st.markdown("---")
    st.markdown("### 🎯 RESUME vs JD ANALYSIS")
    
    # Main Score Card
    st.markdown(f"""
    <div style="background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%); 
                border: 2px solid {analysis['fit_color']}; border-radius: 15px; 
                padding: 25px; margin: 20px 0; text-align: center;">
        <h1 style="color: {analysis['fit_color']}; margin: 0; font-size: 3rem;">{analysis['match_score']}%</h1>
        <p style="color: {analysis['fit_color']}; font-size: 1.5rem; margin: 10px 0;">{analysis['fit_level']}</p>
        <p style="color: #8892b0; margin: 0;">
            {len(analysis['matched_keywords'])} of {analysis['jd_keywords_count']} key requirements matched
        </p>
    </div>
    """, unsafe_allow_html=True)
    
    # Detailed Breakdown
    col_a1, col_a2, col_a3 = st.columns(3)
    
    with col_a1:
        st.markdown("#### ✅ MATCHED SKILLS")
        if analysis['matched_keywords']:
            for kw in analysis['matched_keywords'][:10]:
                st.markdown(f"<span style='color: #00ff88;'>✓ {kw.title()}</span>", unsafe_allow_html=True)
        else:
            st.caption("No direct keyword matches found")
    
    with col_a2:
        st.markdown("#### ⚠️ MISSING FROM RESUME")
        if analysis['missing_keywords']:
            for kw in analysis['missing_keywords'][:10]:
                st.markdown(f"<span style='color: #ff6b6b;'>✗ {kw.title()}</span>", unsafe_allow_html=True)
            st.caption("Consider adding these to your resume")
        else:
            st.success("No critical gaps!")
    
    with col_a3:
        st.markdown("#### 💎 YOUR BONUS SKILLS")
        if analysis['bonus_keywords']:
            for kw in analysis['bonus_keywords'][:10]:
                st.markdown(f"<span style='color: #00d4ff;'>★ {kw.title()}</span>", unsafe_allow_html=True)
            st.caption("Highlight these as differentiators")
        else:
            st.caption("All your skills are JD-aligned")
    
    # Experience Check
    st.markdown("---")
    st.markdown("#### 📊 QUICK INTEL")
    
    intel_col1, intel_col2, intel_col3, intel_col4 = st.columns(4)
    
    with intel_col1:
        if analysis['jd_years_required']:
            st.metric("JD Requires", f"{analysis['jd_years_required']}+ years")
        else:
            st.metric("JD Requires", "Not specified")
    
    with intel_col2:
        if analysis['resume_years_mentioned']:
            st.metric("Your Experience", f"{analysis['resume_years_mentioned']}+ years")
        else:
            st.metric("Your Experience", "Add metrics!")
    
    with intel_col3:
        st.metric("Metrics in Resume", analysis['metrics_found'])
    
    with intel_col4:
        st.metric("Resume Length", f"{analysis['resume_word_count']} words")
    
    # Recommendations
    st.markdown("---")
    st.markdown("#### 🎯 ACTION ITEMS")
    
    recommendations = []
    
    if analysis['match_score'] < 60:
        recommendations.append("🔴 **Low Match**: Consider adding more JD keywords to your resume")
    
    if analysis['missing_keywords']:
        top_missing = ', '.join(analysis['missing_keywords'][:3])
        recommendations.append(f"📝 **Add Keywords**: {top_missing}")
    
    if analysis['metrics_found'] < 5:
        recommendations.append("📊 **Add Metrics**: Include more quantifiable achievements (%, $, x improvement)")
    
    if analysis['resume_word_count'] < 300:
        recommendations.append("📄 **Expand Resume**: Your resume may be too brief for the role level")
    elif analysis['resume_word_count'] > 800:
        recommendations.append("✂️ **Trim Resume**: Consider condensing to highlight key achievements")
    
    if not recommendations:
        recommendations.append("✅ **Strong Position**: Your resume aligns well with this JD!")
    
    for rec in recommendations:
        st.markdown(rec)
```

---

## How to Apply This Patch

1. Open your local `basin-signal-engine/app.py`
2. Add the `analyze_resume_vs_jd` function near the top (after imports)
3. Find line ~2964 and replace the status section with the analysis code
4. Add the CSS fix to the existing style block
5. Commit and push to GitHub
6. Streamlit Cloud will auto-redeploy

The analysis will now run automatically when both resume and JD are pasted!
