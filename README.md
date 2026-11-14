
"""
========================================================================================
SkillForge AI - Autonomous Multi-Agent Personalized Learning Platform (ALL-IN-ONE)
========================================================================================
Zero external pip dependencies (pure standard Python library).
NEW: Users can register from the website (name, email, target role, skills) and get
     their own personalized gap analysis, roadmap, assessments and mentor chat.

HOW TO RUN:
  python app.py          --> Starts server at http://127.0.0.1:8000
  python app.py --test   --> Runs automated test suite
========================================================================================
"""

import http.server
import socketserver
import json
import sqlite3
import urllib.parse
import os
import sys
import webbrowser
from typing import Dict, Any, List, Optional

PORT = 8000
DB_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "platform.db")

# ======================================================================================
# 1. DATA CATALOG & ONTOLOGY
# ======================================================================================

CAREER_ROLES = {
    "Full Stack Developer": {
        "title": "Full Stack Developer",
        "description": "Designs, builds, and maintains complete end-to-end web applications covering responsive frontends, robust REST/GraphQL backends, databases, and deployment.",
        "category": "Software Engineering",
        "required_skills": {
            "HTML/CSS": {"weight": 1.0, "level": "Basic", "tier": "Core"},
            "JavaScript": {"weight": 2.0, "level": "Intermediate", "tier": "Core"},
            "React": {"weight": 2.0, "level": "Intermediate", "tier": "Frontend"},
            "Node.js": {"weight": 2.0, "level": "Intermediate", "tier": "Backend"},
            "SQL": {"weight": 1.5, "level": "Intermediate", "tier": "Database"},
            "Python": {"weight": 1.5, "level": "Intermediate", "tier": "Backend"},
            "Git": {"weight": 1.0, "level": "Basic", "tier": "Tooling"},
            "REST APIs": {"weight": 1.5, "level": "Intermediate", "tier": "Architecture"}
        },
        "prerequisites": {
            "React": ["JavaScript", "HTML/CSS"],
            "Node.js": ["JavaScript"],
            "REST APIs": ["Node.js", "Python"]
        }
    },
    "Data Scientist": {
        "title": "Data Scientist",
        "description": "Extracts actionable insights from data using statistics, machine learning, data wrangling, and predictive modeling.",
        "category": "Data & AI",
        "required_skills": {
            "Python": {"weight": 2.0, "level": "Advanced", "tier": "Core"},
            "SQL": {"weight": 2.0, "level": "Advanced", "tier": "Core"},
            "Data Analysis": {"weight": 2.0, "level": "Intermediate", "tier": "Core"},
            "Machine Learning": {"weight": 2.5, "level": "Intermediate", "tier": "Core"},
            "Statistics": {"weight": 1.5, "level": "Intermediate", "tier": "Mathematics"},
            "Data Visualization": {"weight": 1.0, "level": "Intermediate", "tier": "Analytics"}
        },
        "prerequisites": {
            "Machine Learning": ["Python", "Statistics", "Data Analysis"],
            "Data Visualization": ["Python"]
        }
    },
    "DevOps Engineer": {
        "title": "DevOps Engineer",
        "description": "Automates software delivery pipelines, manages cloud infrastructure, ensures high reliability, CI/CD, and system monitoring.",
        "category": "Infrastructure & Cloud",
        "required_skills": {
            "Linux": {"weight": 1.5, "level": "Intermediate", "tier": "Core"},
            "Git": {"weight": 1.0, "level": "Intermediate", "tier": "Tooling"},
            "Docker": {"weight": 2.0, "level": "Intermediate", "tier": "Containers"},
            "Kubernetes": {"weight": 2.0, "level": "Intermediate", "tier": "Orchestration"},
            "CI/CD": {"weight": 1.5, "level": "Intermediate", "tier": "Automation"},
            "Cloud (AWS/GCP)": {"weight": 2.0, "level": "Intermediate", "tier": "Cloud"},
            "Python": {"weight": 1.0, "level": "Basic", "tier": "Scripting"}
        },
        "prerequisites": {
            "Kubernetes": ["Docker", "Linux"],
            "CI/CD": ["Git", "Docker"]
        }
    }
}

SKILL_RESOURCES = {
    "JavaScript": {
        "description": "The programming language of the Web, powering client & server execution.",
        "courses": [
            {"title": "Modern JavaScript From The Beginning", "provider": "Coursera / FreeCodeCamp", "duration": "14 hours"},
            {"title": "JavaScript: Closures, Promises & Event Loop", "provider": "Frontend Masters", "duration": "8 hours"}
        ],
        "tutorials": [
            {"title": "The Modern JavaScript Tutorial", "url": "https://javascript.info"},
            {"title": "MDN JavaScript Documentation", "url": "https://developer.mozilla.org"}
        ],
        "exercises": [
            {"title": "Async/Await & Fetch API Practice", "platform": "LeetCode / CodeWars", "difficulty": "Intermediate"},
            {"title": "DOM Dynamic Task Board", "platform": "Frontend Mentor", "difficulty": "Hands-on Project"}
        ],
        "remediation_resources": [
            {"title": "JavaScript Basics Booster: Variables, Scope & Array Methods", "provider": "Micro-Lesson", "focus": "Remediation for weak test score"},
            {"title": "Interactive JS Sandbox: Debugging Pitfalls", "provider": "Guided Lab", "focus": "Hands-on drill"}
        ]
    },
    "React": {
        "description": "Declarative component-based UI library for scalable web applications.",
        "courses": [
            {"title": "React - The Complete Guide (Hooks & State)", "provider": "Scrimba", "duration": "20 hours"}
        ],
        "tutorials": [
            {"title": "Thinking in React & State Patterns", "url": "https://react.dev"}
        ],
        "exercises": [
            {"title": "Build a Real-Time Kanban Board", "platform": "Guided Project", "difficulty": "Intermediate"}
        ]
    },
    "Node.js": {
        "description": "Asynchronous event-driven JavaScript runtime for scalable backend services.",
        "courses": [
            {"title": "Complete Node.js Developer & REST APIs", "provider": "NodeUniversity", "duration": "18 hours"}
        ],
        "tutorials": [
            {"title": "Event Loop Architecture & Non-blocking I/O", "url": "https://nodejs.org"}
        ],
        "exercises": [
            {"title": "Develop an Express Authentication API", "platform": "Capstone Project", "difficulty": "Intermediate"}
        ]
    }
}

ASSESSMENT_BANK = {
    "JavaScript": [
        {
            "id": "js_q1",
            "question": "What will be the output of `console.log(typeof null)` in JavaScript?",
            "options": ["null", "undefined", "object", "number"],
            "correct_index": 2,
            "explanation": "In JavaScript, `typeof null` evaluates to 'object' due to a legacy design quirk preserved for backwards compatibility."
        },
        {
            "id": "js_q2",
            "question": "Which mechanism does JavaScript use to handle asynchronous operations on a single thread?",
            "options": ["Multi-threading OS threads", "Event Loop and Task Queues", "Hardware interrupts", "Synchronous polling"],
            "correct_index": 1,
            "explanation": "JavaScript relies on the Event Loop along with Microtask (Promises) and Macrotask queues to execute non-blocking callbacks."
        },
        {
            "id": "js_q3",
            "question": "What is the key difference between `==` and `===` operators?",
            "options": [
                "There is no difference",
                "`==` checks value without type coercion, `===` converts types",
                "`===` checks both value and type without coercion, while `==` performs type coercion",
                "`===` is only used for objects"
            ],
            "correct_index": 2,
            "explanation": "Strict equality `===` verifies that both operands are of the same type and have the same value without type conversion."
        },
        {
            "id": "js_q4",
            "question": "What is a Closure in JavaScript?",
            "options": [
                "A method to terminate script execution",
                "A function that retains access to its lexical scope even when executed outside that scope",
                "A private variable only accessible inside classes",
                "A function that takes no arguments and returns nothing"
            ],
            "correct_index": 1,
            "explanation": "A closure is the combination of a function bundled together with references to its surrounding lexical state."
        }
    ],
    "React": [
        {
            "id": "react_q1",
            "question": "Why should state variables never be modified directly (e.g. `state.count = 5`) in React?",
            "options": [
                "It causes a runtime syntax error",
                "Direct mutation does not trigger re-rendering and breaks reactivity",
                "React state is read-only at the OS level",
                "Direct mutation only works in Class components"
            ],
            "correct_index": 1,
            "explanation": "React detects state changes by comparing references. Mutating state directly bypasses the scheduler and will not trigger component re-render."
        },
        {
            "id": "react_q2",
            "question": "What is the purpose of the dependency array in the `useEffect` hook?",
            "options": [
                "To list NPM packages required by the component",
                "To control when the effect function re-executes based on value changes between renders",
                "To define prop types for validation",
                "To pass parameters to the parent component"
            ],
            "correct_index": 1,
            "explanation": "The dependency array informs React to only re-run the effect if one or more values in that array have changed between renders."
        }
    ],
    "Node.js": [
        {
            "id": "node_q1",
            "question": "In Node.js, what is the role of the `libuv` library?",
            "options": [
                "It compiles JavaScript to C++ assembly",
                "It provides the cross-platform abstraction for asynchronous I/O and thread pooling",
                "It serves as a relational database driver",
                "It renders HTML templates on the server"
            ],
            "correct_index": 1,
            "explanation": "libuv is a C library that powers Node.js event-driven non-blocking I/O and thread pool operations."
        }
    ]
}

# ======================================================================================
# 2. SQLITE DATABASE LAYER
# ======================================================================================

def get_connection():
    conn = sqlite3.connect(DB_FILE)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_connection()
    c = conn.cursor()
    c.execute("""
    CREATE TABLE IF NOT EXISTS learners (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        target_role TEXT NOT NULL,
        experience_level TEXT DEFAULT 'Student / Entry-Level',
        bio TEXT
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS learner_skills (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        skill_name TEXT NOT NULL,
        proficiency TEXT NOT NULL,
        proficiency_pct INTEGER DEFAULT 50,
        verified INTEGER DEFAULT 0,
        verified_score REAL DEFAULT 0,
        UNIQUE(learner_id, skill_name)
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS learner_projects (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        title TEXT NOT NULL,
        description TEXT,
        skills_used TEXT
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS assessment_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        skill TEXT NOT NULL,
        score_pct REAL NOT NULL,
        total_questions INTEGER NOT NULL,
        correct_count INTEGER NOT NULL,
        passed INTEGER NOT NULL,
        feedback TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS roadmaps (
        learner_id INTEGER PRIMARY KEY,
        target_role TEXT NOT NULL,
        roadmap_json TEXT NOT NULL,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS agent_logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        agent_name TEXT NOT NULL,
        action TEXT NOT NULL,
        summary TEXT NOT NULL,
        details_json TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );""")
    conn.commit()
    conn.close()

def reset_to_sample_scenario():
    """Resets ONLY the demo learner (id=1, Alex Morgan). Other registered users are untouched."""
    conn = get_connection()
    c = conn.cursor()
    for table in ["agent_logs", "roadmaps", "assessment_history", "learner_projects", "learner_skills"]:
        c.execute(f"DELETE FROM {table} WHERE learner_id = 1")
    c.execute("DELETE FROM learners WHERE id = 1")
    # free the demo email if a registered user previously held it
    c.execute("DELETE FROM learners WHERE email = 'alex.morgan@university.edu'")

    c.execute("""
    INSERT INTO learners (id, name, email, target_role, experience_level, bio)
    VALUES (1, 'Alex Morgan', 'alex.morgan@university.edu', 'Full Stack Developer', 'Undergraduate Student',
            'Aspiring developer with a foundation in backend scripts and data extraction, eager to master modern web stacks.')
    """)
    skills_data = [
        (1, 'Python', 'Intermediate', 75, 1, 78.0),
        (1, 'SQL', 'Intermediate', 70, 1, 72.0),
        (1, 'HTML/CSS', 'Basic', 45, 0, 0.0)
    ]
    c.executemany("""
    INSERT INTO learner_skills (learner_id, skill_name, proficiency, proficiency_pct, verified, verified_score)
    VALUES (?, ?, ?, ?, ?, ?)
    """, skills_data)
    c.execute("""
    INSERT INTO learner_projects (learner_id, title, description, skills_used)
    VALUES (1, 'Data Scraping & Analysis Engine', 'A Python pipeline that extracts course catalogs and analyzes trends in SQLite.', 'Python, SQL')
    """)
    conn.commit()
    conn.close()

LEVEL_PCT = {"Basic": 45, "Intermediate": 70, "Advanced": 90}

def create_learner(data: Dict[str, Any]) -> int:
    """Creates (or updates, if the email already exists) a learner from website form input."""
    name = (data.get("name") or "").strip()
    email = (data.get("email") or "").strip().lower()
    role = data.get("target_role")
    exp = data.get("experience_level") or "Student / Entry-Level"
    if not name:
        raise ValueError("Name is required")
    if "@" not in email or "." not in email:
        raise ValueError("Valid email is required")
    if role not in CAREER_ROLES:
        raise ValueError("Please choose a valid target role")
    skills_in = [s for s in data.get("skills", []) if (s.get("skill_name") or "").strip()]
    if not skills_in:
        raise ValueError("Please add at least one current skill")

    conn = get_connection()
    c = conn.cursor()
    c.execute("SELECT id FROM learners WHERE email = ?", (email,))
    row = c.fetchone()
    if row:
        lid = row["id"]
        c.execute("UPDATE learners SET name=?, target_role=?, experience_level=? WHERE id=?",
                  (name, role, exp, lid))
        c.execute("DELETE FROM learner_skills WHERE learner_id=?", (lid,))
    else:
        c.execute("INSERT INTO learners (name, email, target_role, experience_level, bio) VALUES (?,?,?,?,?)",
                  (name, email, role, exp, ""))
        lid = c.lastrowid

    norm = SkillMatchingModel().normalize_skill
    for s in skills_in:
        sk = norm(s.get("skill_name", ""))
        lvl = s.get("proficiency", "Basic")
        if lvl not in LEVEL_PCT:
            lvl = "Basic"
        c.execute("""INSERT OR REPLACE INTO learner_skills
                     (learner_id, skill_name, proficiency, proficiency_pct, verified, verified_score)
                     VALUES (?, ?, ?, ?, 0, 0)""", (lid, sk, lvl, LEVEL_PCT[lvl]))
    conn.commit()
    conn.close()
    return lid

def get_learner_profile(learner_id: int = 1) -> Dict[str, Any]:
    conn = get_connection()
    c = conn.cursor()
    c.execute("SELECT * FROM learners WHERE id = ?", (learner_id,))
    row = c.fetchone()
    if not row:
        conn.close()
        return {}
    learner = dict(row)

    c.execute("SELECT skill_name, proficiency, proficiency_pct, verified, verified_score FROM learner_skills WHERE learner_id = ?", (learner_id,))
    learner["skills"] = [dict(r) for r in c.fetchall()]

    c.execute("SELECT title, description, skills_used FROM learner_projects WHERE learner_id = ?", (learner_id,))
    learner["projects"] = [dict(r) for r in c.fetchall()]

    c.execute("SELECT skill, score_pct, total_questions, correct_count, passed, feedback, created_at FROM assessment_history WHERE learner_id = ? ORDER BY id DESC", (learner_id,))
    learner["assessment_history"] = [dict(r) for r in c.fetchall()]

    c.execute("SELECT roadmap_json, updated_at FROM roadmaps WHERE learner_id = ?", (learner_id,))
    r_row = c.fetchone()
    learner["roadmap"] = json.loads(r_row["roadmap_json"]) if r_row else None
    conn.close()
    return learner

def save_roadmap(learner_id: int, target_role: str, roadmap: Dict[str, Any]):
    conn = get_connection()
    c = conn.cursor()
    c.execute("""
    INSERT INTO roadmaps (learner_id, target_role, roadmap_json, updated_at)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP)
    ON CONFLICT(learner_id) DO UPDATE SET
        target_role = excluded.target_role,
        roadmap_json = excluded.roadmap_json,
        updated_at = CURRENT_TIMESTAMP
    """, (learner_id, target_role, json.dumps(roadmap)))
    conn.commit()
    conn.close()

def record_agent_log(learner_id: int, agent_name: str, action: str, summary: str, details: Optional[Dict[str, Any]] = None):
    conn = get_connection()
    c = conn.cursor()
    c.execute("""
    INSERT INTO agent_logs (learner_id, agent_name, action, summary, details_json)
    VALUES (?, ?, ?, ?, ?)
    """, (learner_id, agent_name, action, summary, json.dumps(details or {})))
    conn.commit()
    conn.close()

def get_agent_logs(learner_id: int = 1, limit: int = 50) -> List[Dict[str, Any]]:
    conn = get_connection()
    c = conn.cursor()
    c.execute("SELECT agent_name, action, summary, created_at FROM agent_logs WHERE learner_id = ? ORDER BY id DESC LIMIT ?", (learner_id, limit))
    logs = [dict(r) for r in c.fetchall()]
    conn.close()
    return logs

# ======================================================================================
# 3. SKILL MATCHING & GAP MODEL
# ======================================================================================

class SkillMatchingModel:
    PROFICIENCY_LEVELS = {"None": 0.0, "Basic": 0.4, "Intermediate": 0.75, "Advanced": 1.0}
    REQUIRED_LEVEL_MAP = {"Basic": 0.4, "Intermediate": 0.75, "Advanced": 1.0}

    def __init__(self, roles=CAREER_ROLES):
        self.roles = roles

    def normalize_skill(self, name: str) -> str:
        s = name.strip().lower()
        if s in ["js", "javascript"]: return "JavaScript"
        if s in ["react", "reactjs"]: return "React"
        if s in ["node", "nodejs", "node.js"]: return "Node.js"
        if s in ["html", "css", "html/css"]: return "HTML/CSS"
        if s in ["py", "python"]: return "Python"
            if not p or not p.get("skills"):
        reset_to_sample_scenario()
        orchestrator.run_pipeline(1)

    server_address = ("127.0.0.1", PORT)
    httpd = socketserver.ThreadingTCPServer(server_address, SingleFileHandler)
    httpd.daemon_threads = True
    httpd.allow_reuse_address = True

    print("==================================================================")
    print("      SkillForge AI - Autonomous Agentic Learning Platform        ")
    print("==================================================================")
    print(f"🚀 Server running at: http://127.0.0.1:{PORT}/")
    print("Press Ctrl+C to terminate.")
    print("==================================================================")

    try:
        webbrowser.open(f"http://127.0.0.1:{PORT}/")
    except Exception:
        pass

    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\nServer shutting down gracefully.")
        httpd.server_close()

