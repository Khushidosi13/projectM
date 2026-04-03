# Architectural & Performance Optimizations Report
**Streaming Website Backend**

## 1. Advanced Video Processing (HLS & FFmpeg)
* **HLS Transcoding:** Instead of serving raw, massive `.mp4` files that buffer endlessly, the application automatically transcodes uploads into **HLS (HTTP Live Streaming)** chunks (`.m3u8` & `.ts`) using industry-standard FFmpeg. This allows clients to stream video smoothly regardless of their internet speed.
* **Background Worker Processing:** Video transcoding is completely decoupled from the main HTTP request thread. By utilizing Go's lightweight **goroutines** and localized background contexts, the API immediately returns a `201 Created` status to the user while the heavy CPU work executes entirely in the background.

## 2. High-Performance Memory Caching (Redis)
* **Metadata Interception:** Every time a user opens the homepage or clicks a video, the server intercepts the request and pulls the perfectly formatted JSON from a blazing-fast **Redis** memory store. This prevents the MySQL database from crushing under heavy loads.
* **Smart Cache Invalidation:** The caching isn't just a blind timer. The exact millisecond a user uploads a new video, or the background FFmpeg worker finishes rendering, the application natively self-destructs that user's cached feed. This guarantees the user instantly sees their new video on the homepage without waiting 5 minutes for the cache to clear naturally.

## 3. Stateful Security in a Stateless Architecture
* **JWT Authentication:** The entire API is locked behind stateless JSON Web Tokens (JWT), ensuring that route verification takes nearly zero milliseconds. 
* **Microsecond Token Blacklisting:** The biggest flaw with standard JWTs is that users can't easily "log out" before the token expires. We solved this by creating a highly optimized `POST /logout` endpoint that captures the JWT and violently throws it into a Redis Blacklist. Our custom Middleware dynamically checks this Redis list in microseconds before allowing access to protected routes.

## 4. Third-Party API Proxying (Pexels Integration)
* **Dynamic Data Mapping:** To ensure the platform always has high-quality content, we built a native integration with the Pexels API. 
* **Data Struct Abstraction:** The backend intercepts Pexels' incredibly messy and deeply nested JSON structures, siphons out the best High-Definition `video/mp4` links, and cleans the metadata into our standardized `ExploreVideoResponse` format. The frontend has absolutely no idea it is talking to an external provider!

## 5. Resilient Database & Application Infrastructure
* **Native Connection Pooling:** By tapping into Go's `database/sql` natively, the backend automatically maintains a secure pool of idle MySQL connections. If 500 users attempt to load data at the exact same time, the server recycles active connections rather than opening 500 separate pathways, preventing the database from collapsing.
* **Graceful Shutdowns:** We engineered the `main.go` bootstrapper to listen for OS terminate signals (like `SIGINT`). Rather than forcefully crashing the connections when the server closes, it gives the system 30 seconds to finish answering active user requests and cleanly shuts down the MySQL and Redis connections to prevent data corruption.
