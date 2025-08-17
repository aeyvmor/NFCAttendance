# ARCHITECTURE.md  

## 📌 Project Overview  
This project is an **NFC-Based Attendance System** designed for university classrooms.  
It uses a **Raspberry Pi + NFC Reader** to capture attendance data, which is sent to a **Node.js backend** with a **MongoDB database**.  
Instructors access a **React + Material UI web dashboard** to manage sessions, track attendance, and generate reports.  

The project emphasizes:  
- **Automation** → Eliminates manual roll calls and paper logs.  
- **Flexibility** → Configurable grace periods, session creation, and multi-university support.  
- **Scalability** → Online database for cross-campus deployment.  

---

## 🛠️ Final Tech Stack  

| Layer              | Technology | Reasoning |
|--------------------|------------|-----------|
| **Device (Edge)**  | **Python** on Raspberry Pi | Best support for NFC hardware (`nfcpy`, `pn532`), simple to deploy, easy to maintain. |
| **Backend API**    | **Node.js + Express** | Handles REST APIs, good for scalability, integrates well with MongoDB, async by default. |
| **Database**       | **MongoDB (Cloud Atlas or self-hosted)** | Flexible schema for students/instructors/sessions, easy integration with Node.js. |
| **Frontend**       | **React + Material UI** | Responsive, modern UI, scalable component-based structure. |
| **Reporting**      | **Python scripts integrated into backend** | For CSV/PDF exports, analytics using `pandas` or `reportlab`. |
| **Hosting**        | **Cloud (Heroku / Render / Railway / VPS)** | Ensures accessibility from multiple universities. |
| **Authentication** | JWT (JSON Web Tokens) | Lightweight, stateless session management for instructors/admins. |

---

## 📡 System Architecture Diagram  

```text
   +--------------------+        +---------------------+        +--------------------+
   |   Raspberry Pi     |        |     Backend API     |        |     Frontend       |
   | (Python + NFC)     |  --->  | (Node.js + Express) |  --->  | (React + MUI)      |
   | - Reads NFC UID    |        | - Attendance APIs   |        | - Instructor Dash  |
   | - Sends via HTTPS  |        | - Session Mgmt      |        | - Reports          |
   +--------------------+        | - User Auth (JWT)   |        +--------------------+
              |                  | - Connects to DB    |
              | HTTPS            +---------------------+
              v
       +------------------+
       |   MongoDB Cloud  |
       | - Students       |
       | - Instructors    |
       | - Schedules      |
       | - Attendance     |
       +------------------+
