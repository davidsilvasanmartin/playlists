I need to create a system requirements and technology stack document for a new web project. Here are the details:

### Project Overview
*   **Project Name:** "Playlists" (this will change later)
*   **Purpose:** Saving playlists of music in a way that can be persisted forever (unlike with online services such as YouTube or Spotify, where they can delete songs without any notice)
*   **Target Audience:** Music lovers

### Core Features
*   User registration. There is only one role, USER.
*   Users can save playlists. Each song on a playlist has song title, artist, optionally an album, and an internet link. Playlists are saved to the database. Users can perform basic CRUD of the playlists.
*   The internet link of each song is a website (YouTube, Soundcloud, etc) where the user can listen to that song. We would use the API of those online services to match song metadata, and give the user the choice to save one of the offered links for that song.
*   The front-end allows the user to play playlists, according to the internet link of each song.

### Future Features - To be implemented slowly after release
*   User roles and full role-based access control.
*   The user can rate songs in the playlist. The user can rate its own playlists.
*   The user can share playlists with other users.
*   The user can search for playlists from all users by song title, artist, or album.
*   The user can follow other users and see their playlists.
*   The user can create public or private playlists.
*   The user can import/export playlists in a standard format (JSON, XML).
*   The user can rate other users' playlists.
*   [AI feature] The user can receive recommendations based on their playlists.

### Non-Functional Requirements
*   **Scalability:** Must support up to 10,000 concurrent users within the first year.
*   **Performance:** Average page load time should be under 1.5 seconds.
*   **Security:** Must implement protection against common web vulnerabilities (OWASP Top 10). All user data must be encrypted.

### Technology Preferences
*   **Backend:** Spring Boot (Java)
*   **Frontend:** No Angular. No Next.js. Maybe I can use React with some other framework (Vite?). It needs to support the latest features: signals, typescript, tailwindcss, jest for testing, etc.
*   **Database:** PostgreSQL
*   **Deployment:** Docker, so it can be self-hosted or deployed on cloud platforms.

Based on this, please generate a document that outlines:
1.  Recommended system architecture (frontend, backend, database).
2.  A detailed technology stack for each component.
3.  System requirements (hardware/cloud resources).
4.  A summary of security best practices to be implemented.