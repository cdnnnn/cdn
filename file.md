Project setup and architecture. React + TypeScript + Vite project, Material UI component library, API client layer, data fetching and caching (TanStack Query), routing, and the build setup.
Portal shell, branding and authentication. Layout, side navigation, “Model Eval” name and logo, login/logout, session expiry handling, and protected pages.
Dashboard. Live view of running and queued jobs, GPU server status, recent results and key numbers, updating automatically while jobs run.
Job submission. A form to submit a model (model path, Docker image, base accuracy, draft model, options), with input validation and clear error messages.
Job detail and live tracking. A progress stepper (Checks → Accuracy → Performance → Re-test → Stability → Report) with live updates, a trial list with status and reasons, the recipe agent’s tab (sources, candidates, why each was accepted or rejected), and cancel.
Trial results and charts. Per-trial page with the launch command, GSM8K accuracy, benchmark results by concurrency, and time-series charts (tokens/s per GPU, GPU utilization, KV cache, running and queued requests).
Reports. Validation report view (7 sections) with download (Markdown, JSON, Excel); daily and weekly reports list and view.
Metrics page. GPU utilization by hour and server, daily token counts, job completion rate, and trends by week.
Settings and recipe management. GPU servers (add, edit, test connection, enable/disable); accuracy, performance and recipe agent settings, with secrets hidden; and a recipes page to review, approve or reject recipes drafted by the agent.
Deployment, testing and production readiness. Production build in a container (nginx), API address configurable per environment, error and empty states, browser testing, end-to-end testing with real jobs on the company servers, and user documentation.
