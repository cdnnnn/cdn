{(b.required_capabilities?.length ?? 0) > 0 && (
  <div className="run-eval__dataset-caps">
    {b.required_capabilities.slice(0, 4).map((c) => (
      <span key={c} className="run-eval__chip run-eval__chip--static">
        {c}
      </span>
    ))}
  </div>
)}










setBenchmarks(res.benchmarks.map((b) => ({
  ...b,
  required_capabilities: b.required_capabilities ?? [],
  tasks: b.tasks ?? [],
})));
