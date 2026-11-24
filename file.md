// Thin client for /api/datagen (same API as the HTML UI).
export const API = "/api/datagen";

export class ApiError extends Error {
  constructor(message, status) {
    super(message);
    this.status = status;
  }
}

export async function api(path, opts = {}) {
  const headers = opts.body && !(opts.body instanceof FormData) ? { "Content-Type": "application/json" } : {};
  const r = await fetch(API + path, { ...opts, headers });
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
