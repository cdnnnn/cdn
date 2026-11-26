//environment.js
// Environment settings of the UI. Change the API server here.
// Plain constants only: vite.config.js imports this file too, so keep it free of browser or Vite-only code.

// Address of the data generation API server (uvicorn app.main:app).
export const API_HOST = "";

// Path prefix of the data generation API on that server.
export const API_PATH = "/api/datagen";

// How the browser reaches API_HOST:
//   false (default): calls go to /api/datagen on the page's own origin. `npm run dev` / `npm run preview`
//                    forward them to API_HOST, and a build served by the API server itself is same-origin.
//                    No CORS setup is needed on the server.
//   true:            calls go straight to API_HOST from the browser. Use it when the built UI is hosted
//                    somewhere else; the API server must then allow this origin (CORS).
export const CALL_API_HOST_DIRECTLY = false;












//api.js
// Thin client for /api/datagen (same API as the HTML UI). The server address is set in ./environment.js.
import { API_HOST, API_PATH, CALL_API_HOST_DIRECTLY } from "./environment.js";

export const API = (CALL_API_HOST_DIRECTLY ? API_HOST.replace(/\/+$/, "") : "") + API_PATH;

export class ApiError extends Error {
  constructor(message, status) {
    super(message);
    this.status = status;
  }
}

export async function api(path, opts = {}) {
  const headers = opts.body && !(opts.body instanceof FormData) ? { "Content-Type": "application/json" } : {};
  let r;
  try {
    r = await fetch(API + path, { ...opts, headers });
  } catch (e) {
    throw new ApiError(`cannot reach ${API_HOST} (${e.message})`, 0);
  }
  if (!r.ok) {
    let msg = `${r.status} ${r.statusText}`;
    try {
      const j = await r.json();
      msg = typeof j.detail === "string" ? j.detail
        : (j.detail || []).map((d) => `${(d.loc || []).filter((x) => x !== "body").join(".")}: ${String(d.msg).replace(/^Value error, /, "")}`).join("\n");
    } catch { /* not JSON */ }
    throw new ApiError(msg, r.status);
  }
  return (r.headers.get("content-type") || "").includes("json") ? r.json() : r.text();
}

export const post = (path, body) => api(path, { method: "POST", body: body === undefined ? undefined : JSON.stringify(body) });

export function upload(files) {
  const fd = new FormData();
  files.forEach((f) => fd.append("files", f));
  return api("/uploads", { method: "POST", body: fd });
}

export const fileUrl = (jobId, name) => `${API}/jobs/${encodeURIComponent(jobId)}/files/${encodeURIComponent(name)}`;
export const imageUrl = (jobId, name) => `${API}/jobs/${encodeURIComponent(jobId)}/images/${encodeURIComponent(name)}`;

















//vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { API_HOST } from "./src/environment.js";

// API server of the POC (uvicorn app.main:app), set in src/environment.js.
// Override it for one run with DATAGEN_API=http://host:port npm run dev
const API = process.env.DATAGEN_API || API_HOST;

export default defineConfig({
  plugins: [react()],
  base: "./",
  server: {
    port: 5173,
    proxy: { "/api": { target: API, changeOrigin: true } },
  },
  preview: {
    port: 4173,
    proxy: { "/api": { target: API, changeOrigin: true } },
  },
  build: {
    // Builds into ./dist for now, so it never replaces the UI served at ../ui/react.
    // Point it at "../ui/react" once this version replaces the old one.
    outDir: "dist",
    emptyOutDir: true,
  },
});
