# Lab 1: Create and Deploy Your First Azure Function

## Student Information
- **Name:** Akash Patel
- **Student ID:** 041269598
- **Course:** CST8917 - Spring 2026

## Setup

### Prerequisites

1. Install [Python 3.12](https://www.python.org/downloads/)
2. Install Azure Functions VS Code extension
3. Setup Python `.venv` in `MyFunctionProject` by `F1` -> `Python: Create environment...` -> `Venv` -> `Python 3.12.*` -> `requirements.txt`
4. Install the [Azure Cosmos DB emulator](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-develop-emulator?tabs=docker-linux%2Ccsharp&pivots=api-nosql). You have options between Docker images or MSI installer.
5. When the Cosmos DB Emulator is running (either by container or by installed app), run the helper script to set up the `cst8917lab1cosmosdb` and `analysis_results` container:
   ```bash
   cd MyFunctionProject/local_helper
   python init_emulator.py
   ```

### Environment

1. Navigate to `local.settings.json` and ensure the following variables are configured; you shouldn't have to change anything if you are running locally with the Cosmos DB Emulator.
   ```json
    {
        "IsEncrypted": false,
        "Values": {
            "FUNCTIONS_WORKER_RUNTIME": "python",
            "DATABASE_CONNECTION_STRING": "AccountEndpoint=https://localhost:8081/;AccountKey=C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==;",
            "COSMOS_DB_NAME": "TextAnalysisDB"
        }
    }
   ```

### Run (Local)

1. Open `Terminal` and start Azure Function:
   ```bash
   func start
   ```
2. Open `http://localhost:7071/api/TextAnalyzer?text=Hello, world!` in browser, or run the script below in `New Terminal`:
   ```bash
   curl "http://localhost:7071/api/TextAnalyzer?text=Hello, world!"
   ```
   You should see the following output:
   ```json
   {
        "id": "<random-uuid>",
        "analysis": {
            "wordCount": 2,
            "characterCount": 13,
            "characterCountNoSpaces": 12,
            "sentenceCount": 1,
            "paragraphCount": 1,
            "averageWordLength": 6,
            "longestWord": "Hello,",
            "readingTimeMinutes": 0
        },
        "metadata": {
            "analyzedAt": "<datetime>",
            "textPreview": "Hello, world!"
        },
        "originalText": "Hello, world!"
    }
   ```
3. Go to `https://localhost:8081/_explorer/index.html` to view the Cosmos DB Emulator Data Explorer; you should see the above output stored in the `analysis_results` container in the `cst8917lab1cosmosdb`.
4. Open `http://localhost:7071/api/GetAnalysisHistory` in browser to view the results you output in step 2. Play around with the `TextAnalyzer` function more, and open `http://localhost:7071/api/GetAnalysisHistory?limit=2` to view the 2 most recent outputs. Alternatively, run the script(s) below in the terminal opened in step 2.
    ```bash
    curl "http://localhost:7071/api/GetAnalysisHistory" # get all results stored in cst8917lab1cosmosdb
    curl "http://localhost:7071/api/GetAnalysisHistory?limit=2" # get the most recent 2 results stored in cst8917lab1cosmosdb
    ```
