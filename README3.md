# Solution Architecture LLM + IOT DATA
```
┌────────────────────────────────────────────────────────────────────────────┐
│                               App Bootstrap                                │
│────────────────────────────────────────────────────────────────────────────│
│ • Imports: os, json, pandas, numpy, dateutil.parser, datetime, gemini API  │
│ • Constants:                                                                │
│     CSV_CLEAN="smartcity_env_log_cleaned.csv"                               │
│     Columns: temp_c, hum, (optional) temp_c_norm, hum_norm                  │
│     MODEL_NAME="gemini-1.5-flash"                                           │
│     SYSTEM_STYLE="Answer strictly from JSON… be concise…"                   │
│ • Entry: main()                                                             │
└────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌────────────────────────────────────────────────────────────────────────────┐
│                          Gemini Configuration                              │
│────────────────────────────────────────────────────────────────────────────│
│ • Read GEMINI_API_KEY from environment                                     │
│ • If present → genai.configure(api_key=…)                                  │
│ • model = GenerativeModel(MODEL_NAME, system_instruction=SYSTEM_STYLE)     │
│ • If missing → run in “no-LLM” mode (print computed JSON only)             │
└────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌────────────────────────────────────────────────────────────────────────────┐
│                              CSV Loading                                   │
│────────────────────────────────────────────────────────────────────────────│
│ load_df():                                                                  │
│  • Read CSV_CLEAN                                                           │
│  • Coerce numeric: temp_c, hum, temp_c_norm?, hum_norm?                     │
│  • Return df.reset_index(drop=True)                                         │
│ Assumption: no explicit timestamp column; operates on full dataset rows     │
└────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌────────────────────────────────────────────────────────────────────────────┐
│                         Query Routing (Rule-based)                          │
│────────────────────────────────────────────────────────────────────────────│
│ route(df, q):                                                               │
│  • Normalize q → lowercase                                                  │
│  • If “latest|current|now” → latest_reading(df)                            │
│  • Else if “average|avg|mean|min|max|stats|summary” → stats_over_all(df,…) │
│      – metric = "temp" if “temp” in q; "hum" if “hum|humidity” in q; else both │
│  • Else → fallback to latest_reading                                        │
│  ⇒ Returns a small JSON dict + _intent + optional _metric                   │
└────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌────────────────────────────────────────────────────────────────────────────┐
│                         Pandas “Tool” Functions                             │
│────────────────────────────────────────────────────────────────────────────│
│ latest_reading(df):                                                         │
│  • Take last row → {"temp_c", "hum", "temp_c_norm"?, "hum_norm"?}           │
│                                                                              │
│ stats_over_all(df, metric):                                                  │
│  • Compute count                                                             │
│  • For selected metric(s): min, max, avg over entire dataframe               │
│  • Include normalized stats if columns exist                                 │
│ ⇒ Produces authoritative JSON for the LLM                                    │
└────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌────────────────────────────────────────────────────────────────────────────┐
│                          LLM Answer (Gemini)                                │
│────────────────────────────────────────────────────────────────────────────│
│ • Build prompt:                                                             │
│    - User question text                                                     │
│    - “Context JSON (authoritative)” = results from pandas                   │
│    - Instruction: “Write a short, direct answer with numbers and ranges.”   │
│ • Call model.generate_content(prompt)                                       │
│ • If LLM unavailable → print the JSON instead                               │
└────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌────────────────────────────────────────────────────────────────────────────┐
│                        CLI Interaction & Control                            │
│────────────────────────────────────────────────────────────────────────────│
│ • Print banner + usage                                                      │
│ • Loop:                                                                     │
│    - Read input "You> "                                                     │
│    - results = route(df, q)                                                 │
│    - If model: print Gemini’s phrased answer                                │
│      else: print computed JSON                                              │
│ • Handle KeyboardInterrupt/EOF → graceful exit                              │
└────────────────────────────────────────────────────────────────────────────┘



'''
