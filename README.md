# AI Chatbot for Placement Assistance using RAG

An AI-powered chatbot that allows users to interact with placement data using natural-language queries. The system combines a Groq-powered LLM with a Firebase Firestore database and an agent-based retrieval layer to understand questions, retrieve the required placement information, process large datasets, and generate user-friendly responses.

The application also supports JWT authentication and real-time Server-Sent Events (SSE) streaming so users can see the AI agent's processing steps and results as they are generated.

> **Note:** This implementation uses an agentic/tool-based retrieval architecture over Firestore rather than a traditional vector-database RAG pipeline based on embeddings. The project can therefore be described as an **AI-assisted retrieval/RAG-style chatbot for structured placement data**.

---

## Features

- 🤖 Natural-language AI chatbot for placement-related queries
- 🔎 AI-driven retrieval from Firebase Firestore
- 🧠 Multi-step AI agent for complex database questions
- 📊 Student, company, placement, round, and yearly analytics queries
- 🔍 Deep search for fields stored inside student round data
- ⚡ Real-time response streaming using Server-Sent Events (SSE)
- 🔐 JWT-based authentication
- 🍪 Secure HTTP-only authentication cookies
- 📦 Large dataset handling with in-memory dataset references
- 🔃 Filtering, sorting, limiting, selecting fields, and combining datasets
- 💬 Conversational handling for greetings and general assistant questions
- 📈 Token usage tracking for Groq API calls
- 🩺 Health-check endpoint for deployment monitoring
- 🐳 Docker support
- 🚀 Gunicorn production server support
- 🌐 CORS configuration for frontend/backend integration

---

## System Architecture

```text
                         ┌──────────────────────┐
                         │      Frontend        │
                         │  Chatbot Interface   │
                         └──────────┬───────────┘
                                    │
                              HTTP / SSE
                                    │
                                    ▼
                     ┌───────────────────────────┐
                     │       Flask API           │
                     │     streaming_api.py      │
                     └────────────┬──────────────┘
                                  │
                           JWT Authentication
                                  │
                                  ▼
                     ┌───────────────────────────┐
                     │        AI Agent           │
                     │        agent.py            │
                     └────────────┬──────────────┘
                                  │
                      Natural Language Query
                                  │
                                  ▼
                     ┌───────────────────────────┐
                     │       Groq LLM            │
                     │ llama-3.1-8b-instant      │
                     │ llama-3.3-70b-versatile   │
                     └────────────┬──────────────┘
                                  │
                         Tool / Function Decision
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
             ▼                    ▼                    ▼
      Firestore Query       Deep Field Search     Data Operations
      db_functions.py       deep_search.py       data_operations.py
             │                    │                    │
             └────────────────────┼────────────────────┘
                                  ▼
                         Firebase Firestore
                                  │
                                  ▼
                      Retrieved Placement Data
                                  │
                                  ▼
                           AI Final Response
                                  │
                                  ▼
                         SSE Stream to Client
```

---

## How the Chatbot Works

The chatbot follows an agent-based retrieval workflow:

1. The user enters a natural-language question.
2. Flask receives the request through the `/api/stream` endpoint.
3. JWT authentication verifies the user's access token.
4. The `Agent` analyzes the query.
5. Conversational queries such as greetings are handled directly.
6. Database-related questions are passed to the Groq LLM.
7. The LLM decides which available function/tool should be used.
8. The selected function retrieves data from Firestore.
9. Large datasets can be stored temporarily and manipulated using filtering, sorting, limiting, or field selection.
10. For fields hidden inside round-level student data, the deep-search module searches nested `rowData`.
11. The retrieved information is passed back to the agent.
12. The agent can perform additional iterations when a query requires multiple steps.
13. Once sufficient information is available, the agent terminates the retrieval process.
14. A final natural-language response is generated for the user.
15. The response and intermediate events are streamed to the frontend using SSE.

The agent allows up to **5 iterations** for complex requests.

---

## Example Queries

The chatbot can answer questions such as:

```text
How many students are placed?

Show me the placement statistics for 2024.

How many students have more than 2 offers?

Show me the companies in 2024.

How many students were placed in TCS?

Show me Google's round details.

Show me the final round students for Infosys.

Which students have received multiple offers?

Show the students who were placed in 2024.

Find the mobile number of a particular student.

Give me company-wise placement statistics.
```

The AI determines the appropriate Firestore collection, filters, subcollections, or deep-search operation based on the question.

---

## Firestore Data Model

The project works with three main Firestore collections.

### 1. `companies`

Stores company and recruitment-round information.

```text
companies
 └── companyYearId
      ├── companyName
      ├── year
      ├── status
      ├── currentRound
      ├── finalRound
      ├── totalRounds
      ├── totalPlaced
      ├── totalApplied
      ├── createdAt
      ├── updatedAt
      │
      ├── rounds
      │    └── roundId
      │         ├── roundNumber
      │         ├── roundName
      │         ├── rawColumns
      │         ├── studentCount
      │         ├── isFinalRound
      │         └── data
      │              └── rowId
      │                   ├── rowData
      │                   ├── studentId
      │                   └── status
      │
      └── placements
           └── studentId
                ├── rowData
                └── timestamp
```

### 2. `students`

Stores student-level placement information.

```text
students
 └── studentId
      ├── name
      ├── rollNumber
      ├── email
      ├── companyStatus
      ├── selectedCompanies
      ├── currentStatus
      ├── totalOffers
      └── updatedAt
```

### 3. `years`

Stores yearly placement analytics.

```text
years
 └── year
      ├── totalCompanies
      ├── completedCompanies
      ├── runningCompanies
      ├── totalPlaced
      ├── totalStudentsParticipated
      └── companyWise
```

---

## AI Agent Tools

The agent can work with the following retrieval/data operations.

### `query_database()`

Retrieves information from the main Firestore collections.

Supported collections:

- `companies`
- `students`
- `years`

Supports:

- `get`
- `count`
- filters
- selected fields
- subcollections

---

### `deep_search_field()`

Searches nested student round data when information is stored inside dynamic `rowData`.

For example:

```text
Find the phone number of Rahul
```

The deep-search layer can search field variations such as:

```text
mobile
phone
contact
```

and inspect student round data to locate the requested value.

---

### `manipulate_stored_data()`

Processes datasets already retrieved by the agent.

Supported operations include:

- Filter
- Select fields
- Limit
- Sort
- Combine datasets

This helps prevent unnecessary repeated database queries when working with large result sets.

---

### `get_metadata()`

Returns information about the database structure and available data.

---

## Project Structure

```text
Ai_to_db/
│
├── agent.py
│   └── Main AI agent/orchestrator
│
├── auth_utils.py
│   └── JWT token verification and authentication decorators
│
├── data_operations.py
│   └── Filtering, sorting, limiting, selecting and combining datasets
│
├── db_functions.py
│   └── Firestore database queries and subcollection operations
│
├── deep_search.py
│   └── Deep search inside nested student round data
│
├── firebase_config.py
│   └── Firebase Admin SDK and Firestore initialization
│
├── groq_client.py
│   └── Groq API client and token usage tracking
│
├── prompts.py
│   └── AI system prompts, iteration prompts and final-response prompts
│
├── streaming_api.py
│   └── Flask API and SSE streaming endpoints
│
├── utils.py
│   └── Firestore and JSON data utility functions
│
├── requirements.txt
│   └── Python dependencies
│
└── Dockerfile
    └── Docker production configuration
```

---

## Technology Stack

| Technology | Purpose |
|---|---|
| Python | Backend programming language |
| Flask | REST API framework |
| Groq | LLM inference |
| Llama | Language models used by the chatbot |
| Firebase Admin SDK | Firebase/Firestore integration |
| Cloud Firestore | Placement database |
| JWT | Authentication |
| Server-Sent Events | Real-time response streaming |
| Flask-CORS | Cross-origin communication |
| Gunicorn | Production WSGI server |
| Docker | Containerization |
| python-dotenv | Environment variable management |

---

## AI Models

The project uses Groq-hosted Llama models.

### Main structured-query model

```text
llama-3.1-8b-instant
```

Used through the `GroqClient` for structured AI-agent interactions.

The client requests JSON-formatted responses when required so that the agent can interpret the LLM's decisions and function calls.

### Conversational model

```text
llama-3.3-70b-versatile
```

Used for conversational/general assistant interactions such as greetings and capability questions.

---

## Authentication

The API uses JWT-based authentication.

The `/api/stream` endpoint is protected using:

```python
@token_required
```

The access token is read from the HTTP-only cookie:

```text
accessToken
```

The authentication utility:

1. Reads the access token.
2. Verifies the JWT signature.
3. Checks token validity/expiration.
4. Attaches the decoded user information to the request.
5. Allows the AI query to execute only for authenticated users.

There is also support for an admin-only decorator through:

```python
@admin_required
```

---

## API Endpoints

### Health Check

```http
GET /health
```

Example response:

```json
{
  "status": "healthy",
  "message": "AI Streaming API is running"
}
```

---

### API Information

```http
GET /
```

Returns basic information about the AI streaming API.

---

### Stream AI Query

```http
POST /api/stream
```

Requires a valid JWT access token.

Request:

```json
{
  "query": "How many students are placed?"
}
```

The endpoint returns a Server-Sent Events stream containing events such as:

```text
iteration
ai_decision
function_start
function_result
huge_data_init
huge_data_row
final
error
```

A final response event contains the AI-generated answer.

---

### Set Authentication Token

```http
POST /api/auth/set-token
```

Receives access and refresh tokens from the authentication service and stores them as secure cookies for the AI service domain.

Request:

```json
{
  "accessToken": "YOUR_ACCESS_TOKEN",
  "refreshToken": "YOUR_REFRESH_TOKEN"
}
```

---

### Logout

```http
POST /api/auth/logout
```

Clears the authentication cookies.

---

## Environment Variables

Create a `.env` file in the project directory.

Example:

```env
GROQ_API_KEY=your_groq_api_key

JWT_SECRET_KEY=your_jwt_secret_key
JWT_REFRESH_SECRET_KEY=your_jwt_refresh_secret_key

FIREBASE_CREDS_PATH=path/to/firebase-service-account.json
```

### Important

Do **not** commit:

```text
.env
firebase-service-account.json
serviceAccountKey.json
```

or any other private credentials to GitHub.

Add them to `.gitignore`.

---

## Firebase Setup

1. Create or select a Firebase project.
2. Enable Cloud Firestore.
3. Create a Firebase service account.
4. Generate a service-account JSON key.
5. Store the JSON file securely.
6. Set its path using:

```env
FIREBASE_CREDS_PATH=path/to/service-account.json
```

The application initializes Firebase using the Firebase Admin SDK.

---

## Groq Setup

1. Create a Groq account.
2. Generate an API key.
3. Add the key to `.env`:

```env
GROQ_API_KEY=your_groq_api_key
```

Never hard-code the API key in source code.

---

## Local Installation

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/AI-ChatBot-using-RAG.git
cd AI-ChatBot-using-RAG
```

### 2. Create a virtual environment

Windows:

```bash
python -m venv venv
```

Activate:

```bash
venv\Scripts\activate
```

Linux/macOS:

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment variables

Create:

```text
.env
```

and add the required Firebase, Groq and JWT configuration.

### 5. Start the application

```bash
python streaming_api.py
```

The application uses the `PORT` environment variable when available and defaults to:

```text
5004
```

Local server:

```text
http://localhost:5004
```

Health check:

```text
http://localhost:5004/health
```

---

## Running with Gunicorn

For production-style execution:

```bash
gunicorn --bind 0.0.0.0:${PORT:-5004} --workers 2 --threads 4 --timeout 120 streaming_api:app
```

---

## Docker

The project includes a `Dockerfile`.

### Build

```bash
docker build -t ai-placement-chatbot .
```

### Run

```bash
docker run -p 5004:5004 --env-file .env ai-placement-chatbot
```

For deployment platforms that provide a dynamic `PORT`, the Docker configuration automatically uses the platform's port.

---

## Large Dataset Handling

The agent includes temporary in-memory storage for large Firestore query results.

Instead of repeatedly sending large datasets back to the LLM, the system can store retrieved data using a generated dataset ID.

Example:

```text
dataset_1
```

The agent can then perform operations such as:

```text
filter
sort
limit
select_fields
combine
```

This improves query handling for large placement datasets and reduces unnecessary repeated database retrieval.

---

## Real-Time Streaming

The application uses **Server-Sent Events (SSE)** to provide real-time updates.

The agent runs in a background thread while the Flask endpoint streams events through a queue.

Example event flow:

```text
Starting AI agent
       ↓
AI decision
       ↓
Function started
       ↓
Function result
       ↓
Additional processing
       ↓
Final response
```

For very large datasets, rows can also be streamed individually rather than waiting for the complete dataset before displaying results.

---

## Error Handling

The application includes error handling for:

- Missing API keys
- Missing Firebase credentials
- Invalid JWT tokens
- Expired JWT tokens
- Missing user queries
- Unknown AI functions
- Firestore query failures
- Groq API failures
- Missing datasets
- Streaming errors

The API returns JSON error responses for authentication and request-validation failures and sends SSE error events during streaming failures.

---

## Security Considerations

Before deploying publicly:

- Keep `.env` out of GitHub.
- Keep Firebase service-account JSON files private.
- Rotate exposed API keys immediately.
- Use strong JWT secrets.
- Keep authentication enabled for protected endpoints.
- Use HTTPS in production.
- Review CORS origins and avoid allowing unnecessary domains.
- Avoid logging sensitive user or database information.
- Apply appropriate Firestore access controls.
- Do not expose service-account credentials in frontend code.

---

## Why This Project Is Useful

Traditional database systems require users to understand the database structure and write specific queries.

This project provides a natural-language interface over structured placement data.

Instead of asking a user to know:

```text
Which collection contains placement statistics?
Which field stores total offers?
How are company rounds represented?
```

the user can simply ask:

```text
Show me the placement statistics for 2024.
```

The AI agent interprets the request, chooses the appropriate retrieval operation, accesses the Firestore data, processes the result, and returns a readable answer.

---

## Example End-to-End Query

### User

```text
Show me students who received more than two offers.
```

### Processing

```text
User Query
    ↓
Flask API
    ↓
JWT Verification
    ↓
AI Agent
    ↓
Groq LLM
    ↓
Select query_database()
    ↓
Students Collection
    ↓
Filter: totalOffers > 2
    ↓
Retrieved Data
    ↓
AI Formats Result
    ↓
SSE Stream
    ↓
User
```

---

## Project Highlights

- Built an AI chatbot for natural-language interaction with placement data.
- Implemented an agent-based retrieval workflow using Groq-hosted Llama models.
- Integrated Firebase Firestore for structured placement-data retrieval.
- Added multi-step reasoning and controlled tool/function execution.
- Implemented deep search for dynamically stored student round fields.
- Added large-dataset processing with filtering, sorting, limiting, and field selection.
- Implemented JWT authentication with secure HTTP-only cookies.
- Added Server-Sent Events for real-time AI response streaming.
- Containerized the backend using Docker and configured Gunicorn for production deployment.

---

## Future Improvements

Possible future enhancements include:

- Vector embeddings and a vector database for unstructured documents.
- Hybrid retrieval combining structured Firestore queries with semantic search.
- Conversation memory for multi-turn questions.
- Role-based access control for different placement users.
- Query caching for frequently requested analytics.
- Persistent chat history.
- Advanced analytics dashboards.
- Automated report generation.
- Observability and centralized logging.
- Rate limiting and request monitoring.

---

## Author

**Shreyas Chittawar**

Electronics and Communication Engineering (ECE)

Interested in AI, backend development, VLSI and software engineering.

---

## License

This project is intended for educational, academic and demonstration purposes. Add an appropriate open-source license if you plan to distribute the project publicly.
