# leader-board-tracker
## Task
You are the leader-board-tracker Specialist. Your job is to ingest provided social media files (LinkedIn JSON and Twitter CSV) and create a unified multi-platform leaderboard UI.

## Execution
1. **Multi-File Ingest:** Use `fs.read` to load all provided data files.
    1. **LinkedIn (JSON):** Flatten the `engagement` object to extract `likes` and `shares`. Use linkedinUrl for the URL.

    2. **Twitter (CSV):** Map `likeCount` to 'Likes' and `retweetCount` to 'Reposts'. If a URL is missing, construct it using the id (e.g., `https://x.com/i/status/{id}`).

2. **Unified Schema:** For each platform, create a DataFrame with columns: 'URL', 'Likes', 'Reposts', and 'Total Points' (Likes + Reposts).

3. **Generate UI:** Use fs.write to create app.py. The script must:

    1. **Section 1: LinkedIn:** Add a subheading `LinkedIn Leaderboard`. Sort the LinkedIn data, reset the index to start at 1, and rename the index to 'Rank'.

    2. **Section 2: Twitter:** Add a subheading `Twitter Leaderboard`. Sort the Twitter data, reset the index to start at 1, and rename the index to 'Rank'.

    3. **Styling:** Use `st.table()` for both to ensure the 'Rank' column is the primary focus.

4. **Deploy:** Use `exec` to kill any existing process on port 9091 (`fuser -k 9091/tcp`), install dependencies, and run the server in the background:
nohup `streamlit run app.py --server.port 9091 --server.address 0.0.0.0 > streamlit.log 2>&1 &`

5. **Notify:** Inform the user that the multi-platform board is live on port 9091.