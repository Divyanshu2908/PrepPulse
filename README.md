# prepPulse

A full-stack AI-powered interview preparation tool. Users upload their resume and paste a job description to receive a personalized interview report — including a match score, predicted technical and behavioral questions (with model answers), skill gap analysis, a day-by-day preparation roadmap, and an AI-tailored resume PDF download.

The AI is powered by Google Gemini, and the resume PDF is rendered via Puppeteer.

---

## Features

- **AI interview report** — Gemini analyzes the resume, self-description, and job description and returns a structured JSON report with:
    - A match score (0–100)
    - Technical questions with interviewer intent and suggested answers
    - Behavioral questions with intent and suggested answers
    - Skill gaps with severity ratings (low / medium / high)
    - A day-by-day preparation plan
- **AI resume PDF** — Gemini generates a tailored, ATS-friendly HTML resume based on the candidate's profile and the target job, which is then rendered to a downloadable PDF via Puppeteer
- **JWT authentication** — Cookie-based login/register/logout with a token blacklist
- **React frontend** — Feature-sliced SPA (React 19 + Vite) with context-based state management and protected routes
- **Interview history** — Users can browse all their past reports

---

## Tech Stack

### Backend

| Layer             | Technology                         |
| ----------------- | ---------------------------------- |
| Runtime           | Node.js                            |
| Framework         | Express 5                          |
| Database          | MongoDB (Mongoose 9)               |
| AI                | Google Gemini (`@google/genai`)    |
| Schema validation | Zod + `zod-to-json-schema`         |
| PDF rendering     | Puppeteer                          |
| File upload       | Multer (memory storage, 3MB limit) |
| PDF parsing       | `pdf-parse`                        |
| Auth              | JWT (`jsonwebtoken`), `bcryptjs`   |

### Frontend

| Layer       | Technology     |
| ----------- | -------------- |
| Framework   | React 19       |
| Build tool  | Vite 7         |
| Routing     | React Router 7 |
| HTTP client | Axios          |
| Styling     | SCSS (Sass)    |

---

## Project Structure

```
interview-ai-yt-main/
├── Backend/
│   ├── server.js                          # Entry point
│   └── src/
│       ├── app.js                         # Express app, CORS, routes
│       ├── config/
│       │   └── database.js                # MongoDB connection
│       ├── controllers/
│       │   ├── auth.controller.js         # Register, login, logout, get-me
│       │   └── interview.controller.js    # Generate report, get report(s), generate resume PDF
│       ├── middlewares/
│       │   ├── auth.middleware.js         # JWT guard (cookie-based)
│       │   └── file.middleware.js         # Multer memory upload (3MB max)
│       ├── models/
│       │   ├── user.model.js              # User (username, email, password)
│       │   ├── interviewReport.model.js   # Report schema (questions, skillGaps, preparationPlan, etc.)
│       │   └── blacklist.model.js         # Invalidated JWT tokens
│       ├── routes/
│       │   ├── auth.routes.js
│       │   └── interview.routes.js
│       └── services/
│           └── ai.service.js              # Gemini API calls + Puppeteer PDF generation
│
└── Frontend/
    ├── index.html
    ├── vite.config.js
    └── src/
        ├── App.jsx                        # Root — wraps providers and router
        ├── app.routes.jsx                 # Route definitions
        ├── main.jsx
        └── features/
            ├── auth/
            │   ├── auth.context.jsx       # Auth state (user, loading)
            │   ├── components/
            │   │   └── Protected.jsx      # Route guard
            │   ├── hooks/
            │   │   └── useAuth.js         # Login, register, logout, getMe
            │   ├── pages/
            │   │   ├── Login.jsx
            │   │   └── Register.jsx
            │   └── services/
            │       └── auth.api.js        # Axios calls to /api/auth
            └── interview/
                ├── interview.context.jsx  # Report/reports state
                ├── hooks/
                │   └── useInterview.js    # Generate, fetch, download
                ├── pages/
                │   ├── Home.jsx           # Input form + past reports list
                │   └── Interview.jsx      # Report viewer (questions, roadmap, PDF download)
                └── services/
                    └── interview.api.js   # Axios calls to /api/interview
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- A running MongoDB instance (local or Atlas)
- A Google Gemini API key ([get one here](https://aistudio.google.com/app/apikey))

### Backend Setup

```bash
cd Backend
npm install
```

Create a `.env` file in `Backend/`:

```env
MONGO_URI=mongodb://localhost:27017/interview-ai
JWT_SECRET=your_jwt_secret_here
GOOGLE_GENAI_API_KEY=your_gemini_api_key_here
```

Start the backend:

```bash
npm run dev
```

The server runs on **port 3000**.

### Frontend Setup

```bash
cd Frontend
npm install
npm run dev
```

The frontend runs on **http://localhost:5173** (Vite default). The backend is expected at `http://localhost:3000` — this is hardcoded in the Axios instances and the CORS config in `app.js`.

---

## API Reference

### Auth — `/api/auth`

| Method | Endpoint             | Auth   | Description                             |
| ------ | -------------------- | ------ | --------------------------------------- |
| POST   | `/api/auth/register` | None   | Register with username, email, password |
| POST   | `/api/auth/login`    | None   | Login and receive a JWT cookie          |
| GET    | `/api/auth/logout`   | Cookie | Blacklist token and clear cookie        |
| GET    | `/api/auth/get-me`   | Cookie | Get current user details                |

---

### Interview — `/api/interview`

All routes require a valid JWT cookie.

| Method | Endpoint                                       | Description                                                                                           |
| ------ | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| POST   | `/api/interview/`                              | Generate a new interview report (multipart form: `resume` PDF + `jobDescription` + `selfDescription`) |
| GET    | `/api/interview/`                              | List all reports for the logged-in user (summarized — questions and plan excluded)                    |
| GET    | `/api/interview/report/:interviewId`           | Get full report by ID                                                                                 |
| POST   | `/api/interview/resume/pdf/:interviewReportId` | Generate and download a tailored resume PDF                                                           |

---

## How It Works

### Report Generation

1. User submits a PDF resume, a self-description, and a job description via the frontend form.
2. The backend extracts plain text from the PDF using `pdf-parse`.
3. The extracted text, self-description, and job description are sent to Gemini with a structured JSON schema (defined with Zod and converted to JSON Schema for the API).
4. Gemini returns a validated JSON object with the match score, questions, skill gaps, and preparation plan.
5. The report is saved to MongoDB and returned to the frontend.

### Resume PDF Generation

1. A saved report's resume content, job description, and self-description are sent to Gemini with a prompt instructing it to generate a tailored, ATS-friendly HTML resume.
2. Gemini returns the HTML as a JSON field.
3. Puppeteer renders the HTML to an A4 PDF and streams the buffer back to the client as a download.

---

## Data Models

### InterviewReport

| Field                 | Type            | Notes                                    |
| --------------------- | --------------- | ---------------------------------------- |
| `user`                | ObjectId → User | Owner                                    |
| `title`               | String          | Job title extracted by AI                |
| `matchScore`          | Number          | 0–100                                    |
| `technicalQuestions`  | Array           | `{ question, intention, answer }`        |
| `behavioralQuestions` | Array           | `{ question, intention, answer }`        |
| `skillGaps`           | Array           | `{ skill, severity: low\|medium\|high }` |
| `preparationPlan`     | Array           | `{ day, focus, tasks[] }`                |
| `resume`              | String          | Extracted PDF text                       |
| `selfDescription`     | String          | User-supplied                            |
| `jobDescription`      | String          | User-supplied                            |

---

## Notes

- **Gemini model**: The AI service currently uses `gemini-3-flash-preview`. Check the [Google GenAI docs](https://ai.google.dev/) for available model strings and update accordingly.
- **CORS**: The backend CORS origin is hardcoded to `http://localhost:5173`. Update `src/app.js` before deploying to any other environment.
- **API base URL**: The frontend Axios instances hardcode `http://localhost:3000`. Move this to a Vite environment variable (`import.meta.env.VITE_API_URL`) for production builds.
- **Token blacklist TTL**: JWT tokens expire after 1 day, but blacklisted tokens accumulate indefinitely in MongoDB. Add a TTL index on the `blacklist` collection to auto-purge them after 24 hours.
- **Error handling**: Auth hook catch blocks are silent (`catch(err) {}`). Adding user-facing error state would improve UX significantly.
- **Puppeteer in production**: Puppeteer requires Chromium. If deploying to a containerized environment, ensure the appropriate Chromium dependencies are installed or use `puppeteer-core` with a managed browser.
