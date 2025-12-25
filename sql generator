import streamlit as st
import pandas as pd
import duckdb
import os
import google.generativeai as genai

# Set page configuration
st.set_page_config(
    page_title="SQL File Query App with AI",
    page_icon="📊",
    layout="wide"
)

st.title("📊 SQL Query on Files with AI")
st.markdown("""
Upload your data files (CSV, Excel, JSON, Parquet) and run SQL queries with AI-powered help using DuckDB and Google Gemini.
""")

# Configure Gemini API
api_key = os.environ.get('AI_INTEGRATIONS_GEMINI_API_KEY')
if api_key:
    genai.configure(api_key=api_key)

# Initialize session state for DuckDB connection and tables
if 'duckdb_con' not in st.session_state:
    st.session_state.duckdb_con = duckdb.connect(database=':memory:')
    st.session_state.loaded_tables = []

con = st.session_state.duckdb_con
loaded_tables = st.session_state.loaded_tables

# File uploader
uploaded_files = st.file_uploader(
    "Upload files", 
    accept_multiple_files=True, 
    type=['csv', 'xlsx', 'json', 'parquet']
)

if uploaded_files:
    with st.spinner("Processing files..."):
        for uploaded_file in uploaded_files:
            # Check if file already loaded
            existing = [t for t in loaded_tables if t['File'] == uploaded_file.name]
            if existing:
                continue
                
            try:
                file_ext = os.path.splitext(uploaded_file.name)[1].lower()
                
                if file_ext == '.csv':
                    df = pd.read_csv(uploaded_file)
                elif file_ext == '.xlsx':
                    df = pd.read_excel(uploaded_file)
                elif file_ext == '.json':
                    df = pd.read_json(uploaded_file)
                elif file_ext == '.parquet':
                    df = pd.read_parquet(uploaded_file)
                else:
                    st.warning(f"Skipping unsupported file: {uploaded_file.name}")
                    continue
                
                table_name = os.path.splitext(uploaded_file.name)[0]
                table_name = "".join(c if c.isalnum() else "_" for c in table_name)
                
                con.register(table_name, df)
                
                loaded_tables.append({
                    "File": uploaded_file.name,
                    "Table Name": table_name,
                    "Rows": len(df),
                    "Columns": ", ".join(df.columns)
                })
                
            except Exception as e:
                st.error(f"Error processing {uploaded_file.name}: {str(e)}")
    
    if loaded_tables:
        st.success(f"Successfully loaded {len(loaded_tables)} files!")
        
        # Display table info
        with st.expander("📋 Available Tables", expanded=False):
            st.dataframe(pd.DataFrame(loaded_tables), use_container_width=True)
        
        # Get schema info for AI context
        schema_info = ""
        for table in loaded_tables:
            table_name = table['Table Name']
            columns_result = con.execute(f"SELECT * FROM {table_name} LIMIT 1").df()
            columns = list(columns_result.columns)
            schema_info += f"Table: {table_name}\n  Columns: {', '.join(columns)}\n\n"
        
        # Sidebar for AI assistance
        st.sidebar.header("🤖 AI Assistant")
        
        # AI Query Helper
        st.sidebar.subheader("📝 Query Helper")
        query_request = st.sidebar.text_area(
            "Describe what you want to do:",
            placeholder="e.g., Show me the top 5 customers by sales",
            height=100,
            key="query_helper"
        )
        
        if st.sidebar.button("Generate Query", key="generate_btn"):
            if query_request:
                try:
                    if not api_key:
                        st.sidebar.warning("AI service not configured")
                    else:
                        model = genai.GenerativeModel('gemini-2.5-flash')
                        
                        prompt = f"""Given these database tables and their structure:

{schema_info}

Generate a SQL query for DuckDB to: {query_request}

Return ONLY the SQL query, no explanation."""
                        
                        response = model.generate_content(prompt)
                        generated_query = response.text.strip()
                        
                        # Remove markdown code blocks if present
                        if generated_query.startswith('```'):
                            generated_query = generated_query.split('```')[1]
                            if generated_query.startswith('sql'):
                                generated_query = generated_query[3:]
                            generated_query = generated_query.strip()
                        
                        st.session_state.generated_query = generated_query
                        st.sidebar.success("Query generated!")
                        st.sidebar.code(generated_query, language="sql")
                except Exception as e:
                    st.sidebar.error(f"Error generating query: {str(e)}")
        
        # Main query interface
        st.subheader("🔍 Run SQL Query")
        
        default_query = st.session_state.get('generated_query', 
                                             f"SELECT * FROM {loaded_tables[0]['Table Name']} LIMIT 10")
        
        query = st.text_area(
            "Enter your SQL query:",
            value=default_query,
            height=150,
            key="sql_query"
        )
        
        col1, col2, col3 = st.columns([1, 1, 4])
        
        with col1:
            run_button = st.button("▶ Run Query", type="primary")
        
        with col2:
            explain_button = st.button("💡 Explain Query")
        
        if run_button:
            try:
                result = con.execute(query).df()
                
                st.subheader(f"✓ Results ({len(result)} rows)")
                st.dataframe(result, use_container_width=True)
                
                # Download button
                csv = result.to_csv(index=False).encode('utf-8')
                st.download_button(
                    "⬇️ Download as CSV",
                    csv,
                    "query_results.csv",
                    "text/csv",
                    key='download-csv'
                )
                
                # Store for insights
                st.session_state.last_result = result
                st.session_state.last_query = query
                
            except Exception as e:
                st.error(f"SQL Error: {str(e)}")
        
        # Query explanation
        if explain_button:
            if query:
                try:
                    if not api_key:
                        st.warning("AI service not configured")
                    else:
                        model = genai.GenerativeModel('gemini-2.5-flash')
                        
                        prompt = f"""Explain this SQL query in simple terms. What data is it retrieving and why?

Query:
{query}

Database schema:
{schema_info}

Provide a clear, concise explanation."""
                        
                        response = model.generate_content(prompt)
                        
                        with st.expander("📖 Query Explanation", expanded=True):
                            st.write(response.text)
                except Exception as e:
                    st.error(f"Error explaining query: {str(e)}")
        
        # Data insights
        if 'last_result' in st.session_state and len(st.session_state.last_result) > 0:
            if st.button("🔬 Get Data Insights"):
                try:
                    if not api_key:
                        st.warning("AI service not configured")
                    else:
                        model = genai.GenerativeModel('gemini-2.5-flash')
                        
                        # Convert result to string for analysis
                        result_summary = st.session_state.last_result.head(20).to_string()
                        
                        prompt = f"""Analyze this query result and provide insights:

Query: {st.session_state.last_query}

Data sample (first 20 rows):
{result_summary}

Total rows: {len(st.session_state.last_result)}

Provide 3-5 key insights about this data."""
                        
                        response = model.generate_content(prompt)
                        
                        with st.expander("🔬 Data Insights", expanded=True):
                            st.write(response.text)
                except Exception as e:
                    st.error(f"Error analyzing data: {str(e)}")
    
    else:
        st.info("No valid files loaded.")
else:
    st.info("👆 Upload files to begin")
