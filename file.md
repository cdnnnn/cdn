//Harnessscorecard.tsx
import { Fragment, useEffect, useMemo, useState } from 'react';
import { Award, Check, CheckCircle2, ChevronDown, ChevronRight, Clock, Coins, Footprints, Loader2, Repeat, Timer, X, XCircle } from 'lucide-react';
import { harnessResultsApi } from '../../api/endpoints/harnessResults';
import type {
  HarnessCell,
  HarnessErrorType,
  HarnessResultsResponse,
} from '../../api/endpoints/harnessResults';
import styles from './HarnessScorecard.module.scss';

// ---------------------------------------------------------------------------
// HarnessScorecard — the detail view for a Document-agent harness run
// (eval_type 'agent_harness'), rendered by History.tsx in place of the
// per-model results table. Four parts, matching the Workstream 2 outputs:
//   1. Agent scorecard   — one column per agent (Hermes vs SEMCo-Work), task
//                          success rate as the headline metric
//   2. By use case       — success rate per agent x model x suite
//   3. Error analysis    — failed tasks by error type, per agent
//   4. Trace store       — stored runs, expandable to their tool calls
//
// Self-contained on purpose: it fetches its own data through
// harnessResultsApi (mocked for now) into local state, so the evaluations
// slice and the existing results flow stay untouched.
// ---------------------------------------------------------------------------

const ERROR_LABEL: Record<HarnessErrorType, string> = {
  wrong_tool: 'Wrong tool',
  wrong_argument: 'Wrong argument',
  gave_up: 'Gave up',
  loop: 'Loop',
  file_broken: 'File broken',
};
const ERROR_TYPES = Object.keys(ERROR_LABEL) as HarnessErrorType[];

// Plain-language reading of each error type, shown in an opened trace.
const ERROR_HELP: Record<HarnessErrorType, string> = {
  wrong_tool: 'The agent picked a tool that could not do what the step needed.',
  wrong_argument: 'The agent picked the right tool but passed it a wrong value.',
  gave_up: 'The agent stopped before the task was finished.',
  loop: 'The agent kept repeating the same call without making progress.',
  file_broken: 'The agent produced a file, but the checker could not open it.',
};

interface AgentSummary {
  id: string;
  title: string;
  tasks: number;
  passed: number;
  rate: number;
  toolAcc: number;
  steps: number;
  wasted: number;
  faithfulness: number | null;
  cost: number;
  time: number;
  errors: Record<HarnessErrorType, number>;
}

// Task-weighted roll-up of a set of cells.
function summarize(cells: HarnessCell[]) {
  const tasks = cells.reduce((n, c) => n + c.tasks, 0);
  const w = (pick: (c: HarnessCell) => number) =>
    tasks > 0 ? cells.reduce((n, c) => n + pick(c) * c.tasks, 0) / tasks : 0;
  const faithCells = cells.filter((c) => c.faithfulness !== null);
  const faithTasks = faithCells.reduce((n, c) => n + c.tasks, 0);
  const errors = { wrong_tool: 0, wrong_argument: 0, gave_up: 0, loop: 0, file_broken: 0 };
  cells.forEach((c) => ERROR_TYPES.forEach((t) => (errors[t] += c.errors?.[t] ?? 0)));
  const passed = cells.reduce((n, c) => n + c.passed, 0);
  return {
    tasks,
    passed,
    rate: tasks > 0 ? passed / tasks : 0,
    toolAcc: w((c) => c.tool_call_accuracy),
    steps: w((c) => c.avg_steps),
    wasted: w((c) => c.wasted_steps),
    faithfulness:
      faithTasks > 0 ? faithCells.reduce((n, c) => n + (c.faithfulness ?? 0) * c.tasks, 0) / faithTasks : null,
    cost: w((c) => c.cost_per_task),
    time: w((c) => c.time_per_task),
    errors,
  };
}

const pct = (n: number) => `${Math.round(n * 100)}%`;
const tier = (n: number) => (n >= 0.8 ? 'high' : n >= 0.6 ? 'mid' : 'low');

interface Props {
  evaluationId: string;
  // History's own id -> display-name lookup, so model names match the rest
  // of the page.
  modelName: (id: string) => string;
}

export default function HarnessScorecard({ evaluationId, modelName }: Props) {
  const [data, setData] = useState<HarnessResultsResponse | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [traceFilter, setTraceFilter] = useState<'all' | 'failed'>('all');
  const [traceAgent, setTraceAgent] = useState<string>('all');
  const [openTraceId, setOpenTraceId] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);
    setData(null);
    setOpenTraceId(null);
    setTraceAgent('all');
    harnessResultsApi
      .get(evaluationId)
      .then((res) => {
        if (!cancelled) setData(res);
      })
      .catch((err) => {
        if (cancelled) return;
        setError(
          (err as { response?: { data?: { detail?: string } } })?.response?.data?.detail ||
            (err as Error)?.message ||
            'Failed to load the scorecard.'
        );
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });
    return () => {
      cancelled = true;
    };
  }, [evaluationId]);

  const agents: AgentSummary[] = useMemo(
    () =>
      (data?.agents ?? []).map((a) => ({
        id: a.id,
        title: a.title,
        ...summarize((data?.cells ?? []).filter((c) => c.agent === a.id)),
      })),
    [data]
  );

  // Same roll-up, grouped by the model behind the agents instead.
  const modelSums: AgentSummary[] = (data?.model_ids ?? []).map((id) => ({
    id,
    title: modelName(id),
    ...summarize((data?.cells ?? []).filter((c) => c.model_id === id)),
  }));

  if (loading) {
    return (
      <div className={styles.hs__state}>
        <Loader2 size={16} className={styles.hs__spin} /> Loading the agent scorecard…
      </div>
    );
  }
  if (error || !data) {
    return <div className={styles.hs__state}>{error || 'No scorecard available.'}</div>;
  }

  const bestRate = Math.max(0, ...agents.map((a) => a.rate));
  const agentTitle = (id: string) => data.agents.find((a) => a.id === id)?.title ?? id;
  const suiteName = (id: string) => data.suites.find((s) => s.id === id)?.name ?? id;

  // Comparison rows: label, formatted value per agent, and which direction
  // is better (so the best cell can be marked).
  const metricRows: {
    label: string;
    hint?: string;
    value: (a: AgentSummary) => number | null;
    format: (n: number) => string;
    better: 'high' | 'low';
  }[] = [
    { label: 'Tool-call accuracy', hint: 'Right tool, right arguments, no extra calls', value: (a) => a.toolAcc, format: pct, better: 'high' },
    { label: 'Answer faithfulness', hint: 'To the source document', value: (a) => a.faithfulness, format: pct, better: 'high' },
    { label: 'Steps per task', value: (a) => a.steps, format: (n) => n.toFixed(1), better: 'low' },
    { label: 'Wasted steps', hint: 'Per task', value: (a) => a.wasted, format: (n) => n.toFixed(1), better: 'low' },
    { label: 'Cost per task', value: (a) => a.cost, format: (n) => `$${n.toFixed(3)}`, better: 'low' },
    { label: 'Time per task', value: (a) => a.time, format: (n) => `${n.toFixed(1)}s`, better: 'low' },
  ];
  const isBest = (row: (typeof metricRows)[number], a: AgentSummary, group: AgentSummary[] = agents) => {
    if (group.length < 2) return false;
    const own = row.value(a);
    if (own === null) return false;
    const all = group.map((x) => row.value(x)).filter((v): v is number => v !== null);
    return row.better === 'high' ? own >= Math.max(...all) : own <= Math.min(...all);
  };

  const bestModelRate = Math.max(0, ...modelSums.map((m) => m.rate));

  const maxErrorCount = Math.max(1, ...agents.flatMap((a) => ERROR_TYPES.map((t) => a.errors[t])));
  const totalFailed = agents.reduce((n, a) => n + (a.tasks - a.passed), 0);

  const visibleTraces = data.traces.filter(
    (t) => (traceFilter === 'all' || !t.passed) && (traceAgent === 'all' || t.agent === traceAgent)
  );

  return (
    <div className={styles.hs}>
      {/* ---------- 1. Agent scorecard ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Agent scorecard</h3>
          <span>
            Same {data.suites.length} suite{data.suites.length === 1 ? '' : 's'}, same harness, averaged over{' '}
            {data.model_ids.length} model{data.model_ids.length === 1 ? '' : 's'}
          </span>
        </div>

        <div className={styles.hs__agents} style={{ gridTemplateColumns: `repeat(${Math.max(1, agents.length)}, minmax(0, 1fr))` }}>
          {agents.map((a) => {
            const best = agents.length > 1 && a.rate === bestRate;
            return (
              <div key={a.id} className={`${styles.hs__agent} ${best ? styles['hs__agent--best'] : ''}`}>
                <div className={styles['hs__agent-top']}>
                  <span className={styles['hs__agent-name']}>{a.title}</span>
                  {best && (
                    <span className={styles.hs__badge}>
                      <Award size={11} /> Best success rate
                    </span>
                  )}
                </div>
                <div className={styles['hs__agent-rate']}>{pct(a.rate)}</div>
                <div className={styles['hs__agent-label']}>
                  Task success rate · {a.passed.toLocaleString()} of {a.tasks.toLocaleString()} tasks
                </div>
                <div className={styles.hs__bar}>
                  <div className={styles['hs__bar-fill']} style={{ width: pct(a.rate) }} />
                </div>
              </div>
            );
          })}
        </div>

        <div className={styles['hs__table-wrap']}>
          <table className={styles.hs__table}>
            <thead>
              <tr>
                <th>Metric</th>
                {agents.map((a) => (
                  <th key={a.id} className={styles.hs__num}>
                    {a.title}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {metricRows.map((row) => (
                <tr key={row.label}>
                  <td>
                    <b>{row.label}</b>
                    {row.hint && <small>{row.hint}</small>}
                  </td>
                  {agents.map((a) => {
                    const v = row.value(a);
                    return (
                      <td key={a.id} className={`${styles.hs__num} ${isBest(row, a) ? styles['hs__num--best'] : ''}`}>
                        {v === null ? '—' : row.format(v)}
                      </td>
                    );
                  })}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </section>

      {/* ---------- Model comparison — same numbers, grouped by model ---------- */}
      {modelSums.length > 1 && (
        <section className={styles.hs__sec}>
          <div className={styles['hs__sec-head']}>
            <h3>Model comparison</h3>
            <span>
              {modelSums.map((m) => m.title).join(' vs ')}, averaged over {agents.length} agent
              {agents.length === 1 ? '' : 's'}
            </span>
          </div>

          <div
            className={styles.hs__agents}
            style={{ gridTemplateColumns: `repeat(${modelSums.length}, minmax(0, 1fr))` }}
          >
            {modelSums.map((m) => {
              const best = m.rate === bestModelRate;
              return (
                <div key={m.id} className={`${styles.hs__agent} ${best ? styles['hs__agent--best'] : ''}`}>
                  <div className={styles['hs__agent-top']}>
                    <span className={styles['hs__agent-name']}>{m.title}</span>
                    {best && (
                      <span className={styles.hs__badge}>
                        <Award size={11} /> Best success rate
                      </span>
                    )}
                  </div>
                  <div className={styles['hs__agent-rate']}>{pct(m.rate)}</div>
                  <div className={styles['hs__agent-label']}>
                    Task success rate · {m.passed.toLocaleString()} of {m.tasks.toLocaleString()} tasks
                  </div>
                  <div className={styles.hs__bar}>
                    <div className={styles['hs__bar-fill']} style={{ width: pct(m.rate) }} />
                  </div>
                </div>
              );
            })}
          </div>

          <div className={styles['hs__table-wrap']}>
            <table className={styles.hs__table}>
              <thead>
                <tr>
                  <th>Metric</th>
                  {modelSums.map((m) => (
                    <th key={m.id} className={styles.hs__num}>
                      {m.title}
                    </th>
                  ))}
                </tr>
              </thead>
              <tbody>
                {metricRows.map((row) => (
                  <tr key={row.label}>
                    <td>
                      <b>{row.label}</b>
                      {row.hint && <small>{row.hint}</small>}
                    </td>
                    {modelSums.map((m) => {
                      const v = row.value(m);
                      return (
                        <td
                          key={m.id}
                          className={`${styles.hs__num} ${isBest(row, m, modelSums) ? styles['hs__num--best'] : ''}`}
                        >
                          {v === null ? '—' : row.format(v)}
                        </td>
                      );
                    })}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </section>
      )}

      {/* ---------- 2. By use case, per agent and model ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Success rate by use case</h3>
          <span>Per agent and the model behind it</span>
        </div>
        <div className={styles['hs__table-wrap']}>
          <table className={styles.hs__table}>
            <thead>
              <tr>
                <th>Agent</th>
                <th>Model</th>
                {data.suites.map((s) => (
                  <th key={s.id} className={styles.hs__num} title={s.name}>
                    {s.use_case || s.name}
                    <small>{s.name}</small>
                  </th>
                ))}
                <th className={styles.hs__num}>Overall</th>
              </tr>
            </thead>
            <tbody>
              {data.agents.map((a) =>
                data.model_ids.map((modelId, mi) => {
                  const own = data.cells.filter((c) => c.agent === a.id && c.model_id === modelId);
                  const overall = summarize(own);
                  return (
                    <tr key={`${a.id}-${modelId}`}>
                      {mi === 0 && (
                        <td rowSpan={data.model_ids.length} className={styles['hs__cell-agent']}>
                          <b>{a.title}</b>
                        </td>
                      )}
                      <td>{modelName(modelId)}</td>
                      {data.suites.map((s) => {
                        const cell = own.find((c) => c.suite_id === s.id);
                        if (!cell || cell.tasks === 0) {
                          return (
                            <td key={s.id} className={styles.hs__num}>
                              —
                            </td>
                          );
                        }
                        const rate = cell.passed / cell.tasks;
                        return (
                          <td key={s.id} className={styles.hs__num} title={`${cell.passed} of ${cell.tasks} tasks passed`}>
                            <span className={`${styles.hs__rate} ${styles[`hs__rate--${tier(rate)}`]}`}>{pct(rate)}</span>
                          </td>
                        );
                      })}
                      <td className={`${styles.hs__num} ${styles['hs__num--strong']}`}>{pct(overall.rate)}</td>
                    </tr>
                  );
                })
              )}
            </tbody>
          </table>
        </div>
        <p className={styles.hs__legend}>
          <span className={`${styles.hs__rate} ${styles['hs__rate--high']}`}>80%+</span>
          <span className={`${styles.hs__rate} ${styles['hs__rate--mid']}`}>60–79%</span>
          <span className={`${styles.hs__rate} ${styles['hs__rate--low']}`}>Below 60%</span>
        </p>
      </section>

      {/* ---------- 3. Error analysis ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Error analysis</h3>
          <span>
            {totalFailed.toLocaleString()} failed task{totalFailed === 1 ? '' : 's'}, by what went wrong
          </span>
        </div>
        <div className={styles['hs__table-wrap']}>
          <table className={styles.hs__table}>
            <thead>
              <tr>
                <th>Error type</th>
                {agents.map((a) => (
                  <th key={a.id}>{a.title}</th>
                ))}
              </tr>
            </thead>
            <tbody>
              {ERROR_TYPES.map((t) => (
                <tr key={t}>
                  <td>
                    <b>{ERROR_LABEL[t]}</b>
                  </td>
                  {agents.map((a) => (
                    <td key={a.id}>
                      <span className={styles.hs__errbar}>
                        <span className={styles['hs__errbar-track']}>
                          <span
                            className={styles['hs__errbar-fill']}
                            style={{ width: `${(a.errors[t] / maxErrorCount) * 100}%` }}
                          />
                        </span>
                        <span className={styles['hs__errbar-n']}>{a.errors[t]}</span>
                      </span>
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </section>

      {/* ---------- 4. Trace store ---------- */}
      <section className={styles.hs__sec}>
        <div className={styles['hs__sec-head']}>
          <h3>Trace store</h3>
          <span>
            {data.settings.save_traces
              ? `Sample of stored runs · limits ${data.settings.max_steps} steps, ${data.settings.task_timeout}s per task`
              : 'Traces were not saved for this run'}
          </span>
        </div>

        {data.settings.save_traces && (
          <>
            <div className={styles.hs__filters}>
              {(['all', 'failed'] as const).map((f) => (
                <button
                  key={f}
                  type="button"
                  className={`${styles.hs__chip} ${traceFilter === f ? styles['hs__chip--on'] : ''}`}
                  onClick={() => setTraceFilter(f)}
                >
                  {f === 'all' ? 'All runs' : 'Failed only'}
                </button>
              ))}
              <span className={styles['hs__filters-sep']} />
              <button
                type="button"
                className={`${styles.hs__chip} ${traceAgent === 'all' ? styles['hs__chip--on'] : ''}`}
                onClick={() => setTraceAgent('all')}
              >
                All agents
              </button>
              {data.agents.map((a) => (
                <button
                  key={a.id}
                  type="button"
                  className={`${styles.hs__chip} ${traceAgent === a.id ? styles['hs__chip--on'] : ''}`}
                  onClick={() => setTraceAgent(a.id)}
                >
                  {a.title}
                </button>
              ))}
            </div>

            <div className={styles['hs__table-wrap']}>
              <table className={`${styles.hs__table} ${styles['hs__table--rows']}`}>
                <thead>
                  <tr>
                    <th />
                    <th>Task</th>
                    <th>Agent · Model</th>
                    <th>Result</th>
                    <th className={styles.hs__num}>Steps</th>
                    <th className={styles.hs__num}>Tokens</th>
                    <th className={styles.hs__num}>Time</th>
                  </tr>
                </thead>
                <tbody>
                  {visibleTraces.map((t) => {
                    const open = openTraceId === t.id;
                    return (
                      <Fragment key={t.id}>
                        <tr
                          className={open ? styles['hs__row--open'] : ''}
                          onClick={() => setOpenTraceId(open ? null : t.id)}
                        >
                          <td className={styles.hs__toggle}>
                            <button
                              type="button"
                              aria-expanded={open}
                              aria-label={open ? 'Hide trace' : 'Show trace'}
                              onClick={(e) => {
                                e.stopPropagation();
                                setOpenTraceId(open ? null : t.id);
                              }}
                            >
                              {open ? <ChevronDown size={14} /> : <ChevronRight size={14} />}
                            </button>
                          </td>
                          <td>
                            <b>{t.task}</b>
                            <small>{suiteName(t.suite_id)}</small>
                          </td>
                          <td>
                            {agentTitle(t.agent)}
                            <small>{modelName(t.model_id)}</small>
                          </td>
                          <td>
                            <span className={`${styles.hs__result} ${t.passed ? styles['hs__result--pass'] : styles['hs__result--fail']}`}>
                              {t.passed ? <CheckCircle2 size={12} /> : <XCircle size={12} />}
                              {t.passed ? 'Passed' : t.error_type ? ERROR_LABEL[t.error_type] : 'Failed'}
                            </span>
                          </td>
                          <td className={styles.hs__num}>{t.steps.length}</td>
                          <td className={styles.hs__num}>{t.tokens.toLocaleString()}</td>
                          <td className={styles.hs__num}>{t.time_sec}s</td>
                        </tr>
                        {open &&
                          (() => {
                            const maxMs = Math.max(1, ...t.steps.map((st) => st.ms));
                            const slowest = t.steps.reduce((x, st) => (st.ms > x.ms ? st : x), t.steps[0]);
                            const repeats = t.steps.filter((st, i) => i > 0 && st.tool === t.steps[i - 1].tool).length;
                            const stepShare = Math.min(1, t.steps.length / Math.max(1, data.settings.max_steps));
                            const timeShare = Math.min(1, t.time_sec / Math.max(1, data.settings.task_timeout));
                            return (
                              <tr className={styles['hs__row--trace']}>
                                <td colSpan={7}>
                                  <div className={styles.hs__trace}>
                                    {/* verdict: what happened, in one sentence */}
                                    <div
                                      className={`${styles['hs__trace-verdict']} ${
                                        t.passed ? '' : styles['hs__trace-verdict--fail']
                                      }`}
                                    >
                                      <span className={styles['hs__trace-verdict-icon']}>
                                        {t.passed ? <Check size={15} strokeWidth={3} /> : <X size={15} strokeWidth={3} />}
                                      </span>
                                      <div>
                                        <b>
                                          {t.passed
                                            ? 'Task completed'
                                            : `Task failed${t.error_type ? `: ${ERROR_LABEL[t.error_type].toLowerCase()}` : ''}`}
                                        </b>
                                        <p>
                                          {t.passed
                                            ? `${agentTitle(t.agent)} finished in ${t.steps.length} steps and the checker accepted the result.`
                                            : t.error_type
                                              ? ERROR_HELP[t.error_type]
                                              : 'The checker did not accept the result.'}
                                        </p>
                                      </div>
                                    </div>

                                    {/* how much of the budget this run used */}
                                    <div className={styles['hs__trace-stats']}>
                                      <div className={styles['hs__trace-stat']}>
                                        <span><Footprints size={12} /> Steps</span>
                                        <b>
                                          {t.steps.length} <small>of {data.settings.max_steps} allowed</small>
                                        </b>
                                        <i><em style={{ width: pct(stepShare) }} /></i>
                                      </div>
                                      <div className={styles['hs__trace-stat']}>
                                        <span><Clock size={12} /> Time</span>
                                        <b>
                                          {t.time_sec}s <small>of {data.settings.task_timeout}s allowed</small>
                                        </b>
                                        <i><em style={{ width: pct(timeShare) }} /></i>
                                      </div>
                                      <div className={styles['hs__trace-stat']}>
                                        <span><Coins size={12} /> Tokens</span>
                                        <b>{t.tokens.toLocaleString()}</b>
                                      </div>
                                      <div className={styles['hs__trace-stat']}>
                                        <span><Timer size={12} /> Slowest step</span>
                                        <b>
                                          {(slowest.ms / 1000).toFixed(1)}s <small>{slowest.tool}</small>
                                        </b>
                                      </div>
                                    </div>

                                    {/* the steps, in order */}
                                    <div className={styles['hs__trace-head']}>
                                      <span>What the agent did, step by step</span>
                                      {repeats > 0 && (
                                        <span className={styles['hs__trace-repeat']}>
                                          <Repeat size={11} /> {repeats} repeated call{repeats === 1 ? '' : 's'}
                                        </span>
                                      )}
                                    </div>
                                    <ol className={styles.hs__steps}>
                                      {t.steps.map((st, i) => {
                                        const repeat = i > 0 && st.tool === t.steps[i - 1].tool;
                                        return (
                                          <li key={i} className={st.ok ? '' : styles['hs__step--fail']}>
                                            <span className={styles['hs__step-dot']}>
                                              {st.ok ? i + 1 : <X size={11} strokeWidth={3} />}
                                            </span>
                                            <div className={styles['hs__step-card']}>
                                              <div className={styles['hs__step-main']}>
                                                <code>{st.tool}</code>
                                                {repeat && (
                                                  <span className={styles['hs__step-tag']}>
                                                    <Repeat size={10} /> Same as previous
                                                  </span>
                                                )}
                                                <span className={styles['hs__step-sum']}>{st.summary}</span>
                                              </div>
                                              <div className={styles['hs__step-time']}>
                                                <i><em style={{ width: pct(st.ms / maxMs) }} /></i>
                                                <span className={styles['hs__step-ms']}>{(st.ms / 1000).toFixed(1)}s</span>
                                              </div>
                                            </div>
                                          </li>
                                        );
                                      })}
                                    </ol>
                                  </div>
                                </td>
                              </tr>
                            );
                          })()}
                      </Fragment>
                    );
                  })}
                  {visibleTraces.length === 0 && (
                    <tr>
                      <td colSpan={7} className={styles.hs__none}>
                        No stored runs match this filter.
                      </td>
                    </tr>
                  )}
                </tbody>
              </table>
            </div>
          </>
        )}
      </section>
    </div>
  );
}


























//Harnessscorecard.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// HarnessScorecard — detail view for a Document-agent harness run, shown in
// History's detail panel. Same "ink" tokens and em-based type scale as the
// New run wizard (NewEvaluation.module.scss); nothing is redeclared here.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

.hs {
  font-size: 0.8125rem; // base every em below is relative to
  display: flex;
  flex-direction: column;
  gap: 26px;
  margin-top: 20px;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__state {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-top: 20px;
    padding: 18px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-2;
    font-size: 0.875rem;
  }

  &__spin { animation: hs-spin 0.8s linear infinite; }

  // ---- section ------------------------------------------------------------
  &__sec { display: flex; flex-direction: column; gap: 12px; }

  &__sec-head {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 6px 16px;

    h3 { font-family: $display; font-size: 1.1538em; font-weight: 800; letter-spacing: -0.01em; color: $ink; }
    span { font-size: 0.9615em; color: $ink-2; }
  }

  // ---- agent cards --------------------------------------------------------
  &__agents { display: grid; gap: 12px; }

  &__agent {
    padding: 16px 18px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $card;

    &--best { border-color: $signal; background: $wash; }
  }

  &__agent-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  &__agent-name { font-family: $display; font-size: 1.1538em; font-weight: 700; color: $ink; }

  &__badge {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-size: 0.8462em;
    font-weight: 650;
    white-space: nowrap;
  }

  &__agent-rate {
    margin-top: 10px;
    font-family: $display;
    font-size: 2.4615em;
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1;
    color: $ink;
  }
  &__agent-label { margin-top: 6px; font-size: 0.9615em; color: $ink-2; }

  &__bar { margin-top: 12px; height: 6px; border-radius: 999px; background: $line; overflow: hidden; }
  &__bar-fill { height: 100%; border-radius: 999px; background: $signal; }

  // ---- tables -------------------------------------------------------------
  &__table-wrap { overflow-x: auto; border: 1px solid $line; border-radius: 12px; }

  &__table {
    width: 100%;
    border-collapse: collapse;
    font-size: 1em;

    th {
      padding: 10px 14px;
      background: $paper;
      border-bottom: 1px solid $line;
      text-align: left;
      font-family: $sans;
      font-size: 0.9231em;
      font-weight: 650;
      color: $ink-2;
      white-space: nowrap;
      vertical-align: bottom;

      small { display: block; font-weight: 500; color: $ink-3; }
    }

    td {
      padding: 11px 14px;
      border-bottom: 1px solid $line-2;
      color: $ink;
      vertical-align: middle;

      b { display: block; font-weight: 700; }
      small { display: block; font-size: 0.9231em; color: $ink-2; }
    }
    tbody tr:last-child td { border-bottom: 0; }

    th.hs__num,
    td.hs__num { text-align: right; white-space: nowrap; }
    td.hs__num { font-family: $mono; font-size: 0.9615em; }
    td.hs__num--best { font-weight: 700; color: $signal; }
    td.hs__num--strong { font-weight: 700; }
    td.hs__cell-agent { vertical-align: top; border-right: 1px solid $line-2; }
    td.hs__none { text-align: center; color: $ink-2; }

    // clickable rows (trace store)
    &--rows tbody tr { cursor: pointer; transition: background 0.12s ease; }
    &--rows tbody tr:hover td { background: $paper; }
    &--rows tbody tr.hs__row--open td { background: $wash; border-bottom-color: transparent; }
    &--rows tbody tr.hs__row--trace { cursor: default; }
    &--rows tbody tr.hs__row--trace td,
    &--rows tbody tr.hs__row--trace:hover td { background: $wash; padding-top: 0; }
    &--rows tbody tr.hs__row--trace b,
    &--rows tbody tr.hs__row--trace small { display: revert; }
  }

  // success-rate pill, three tiers (legend under the table)
  &__rate {
    display: inline-block;
    min-width: 46px;
    padding: 3px 8px;
    border-radius: 6px;
    text-align: center;
    font-family: $mono;
    font-size: 0.9615em;
    font-weight: 700;

    &--high { color: $ok; background: $ok-wash; }
    &--mid { color: $ink-2; background: $paper; }
    &--low { color: $danger; background: $danger-wash; }
  }

  &__legend { display: flex; flex-wrap: wrap; gap: 8px; }

  // ---- error bars ---------------------------------------------------------
  &__errbar { display: flex; align-items: center; gap: 10px; }
  &__errbar-track { flex: 1; min-width: 60px; height: 8px; border-radius: 999px; background: $line-2; overflow: hidden; }
  &__errbar-fill { display: block; height: 100%; border-radius: 999px; background: $danger; }
  &__errbar-n { min-width: 22px; text-align: right; font-family: $mono; font-size: 0.9615em; font-weight: 700; color: $ink; }

  // ---- trace store --------------------------------------------------------
  &__filters { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; }
  &__filters-sep { width: 1px; height: 18px; background: $line; }

  &__chip {
    padding: 6px 12px;
    border: 1.5px solid $line;
    border-radius: 999px;
    background: $card;
    font-family: $sans;
    font-size: 0.9615em;
    font-weight: 650;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
    &--on { border-color: $signal; background: $wash; color: $signal; }
  }

  &__toggle {
    width: 34px;

    button {
      display: inline-flex;
      padding: 2px;
      border: 0;
      background: transparent;
      color: $ink-2;
      cursor: pointer;
    }
  }

  &__result {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    font-size: 0.9231em;
    font-weight: 650;
    white-space: nowrap;

    &--pass { color: $ok; background: $ok-wash; }
    &--fail { color: $danger; background: $danger-wash; }
  }

  // ---- opened trace: verdict, budget tiles, step timeline -----------------
  &__trace {
    display: flex;
    flex-direction: column;
    gap: 14px;
    margin: 4px 0 14px;
    padding: 16px;
    border: 1px solid $line;
    border-radius: 12px;
    background: $card;
    animation: hs-open 0.2s ease;
  }

  &__trace-verdict {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 12px 14px;
    border-radius: 10px;
    background: $ok-wash;

    b { display: block; font-family: $display; font-size: 1.0769em; font-weight: 700; color: $ink; }
    p { margin-top: 2px; font-size: 0.9615em; color: $ink-2; line-height: 1.45; }

    &--fail { background: $danger-wash; }
    &--fail .hs__trace-verdict-icon { background: $danger; }
  }
  &__trace-verdict-icon {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    background: $ok;
    color: #fff;
  }

  &__trace-stats { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; }
  &__trace-stat {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 10px 12px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $paper;

    > span { display: inline-flex; align-items: center; gap: 6px; font-size: 0.8846em; font-weight: 650; color: $ink-2; }
    > b {
      display: block;
      font-family: $mono;
      font-size: 1.1538em;
      font-weight: 700;
      color: $ink;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }
    small { display: inline; font-family: $sans; font-size: 0.7333em; font-weight: 500; color: $ink-2; }
    // budget meter
    > i { display: block; height: 4px; border-radius: 999px; background: $line; overflow: hidden; }
    > i em { display: block; height: 100%; border-radius: 999px; background: $signal; }
  }

  &__trace-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    font-size: 0.9615em;
    font-weight: 700;
    color: $ink;
  }
  &__trace-repeat,
  &__step-tag {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 2px 8px;
    border-radius: 999px;
    background: $danger-wash;
    font-family: $sans;
    font-size: 0.8462em;
    font-weight: 650;
    color: $danger;
    white-space: nowrap;
  }

  &__steps {
    display: flex;
    flex-direction: column;
    list-style: none;

    li {
      position: relative;
      display: grid;
      grid-template-columns: 24px minmax(0, 1fr);
      gap: 12px;
      padding-bottom: 8px;
      font-size: 0.9615em;
      color: $ink;

      &:last-child { padding-bottom: 0; }
      // rail joining one step to the next
      &:not(:last-child)::before {
        content: '';
        position: absolute;
        left: 11px;
        top: 26px;
        bottom: -4px;
        width: 2px;
        background: $line;
      }
    }

    code { font-family: $mono; font-weight: 700; white-space: nowrap; }
  }
  &__step-dot {
    position: relative;
    z-index: 1;
    width: 24px;
    height: 24px;
    margin-top: 5px;
    border-radius: 50%;
    display: grid;
    place-items: center;
    border: 1.5px solid $line;
    background: $card;
    font-family: $mono;
    font-size: 0.8462em;
    font-weight: 700;
    color: $ink-2;
  }
  &__step-card {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    min-width: 0;
    padding: 8px 12px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $paper;
  }
  &__step-main { display: flex; align-items: center; flex-wrap: wrap; gap: 4px 10px; min-width: 0; }
  &__step-sum { color: $ink-2; }
  &__step-time {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;

    // how long this step took, relative to the slowest step
    i { display: block; width: 72px; height: 4px; border-radius: 999px; background: $line; overflow: hidden; }
    em { display: block; height: 100%; border-radius: 999px; background: $ink-3; }
  }
  &__step-ms { min-width: 34px; text-align: right; font-family: $mono; font-size: 0.9231em; color: $ink-2; }

  &__steps li.hs__step--fail {
    .hs__step-dot { border-color: $danger; background: $danger; color: #fff; }
    .hs__step-card { border-color: rgba($danger, 0.35); background: $danger-wash; }
    .hs__step-sum { color: $danger; font-weight: 650; }
  }
}

@keyframes hs-spin { to { transform: rotate(360deg); } }
@keyframes hs-open { from { opacity: 0; transform: translateY(-4px); } }

@media (max-width: 900px) {
  .hs__agents { grid-template-columns: 1fr !important; }
  .hs__trace-stats { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}

@media (prefers-reduced-motion: reduce) {
  .hs * { animation: none !important; transition: none !important; }
}
























//Harnessresultsmock.ts
// api/mock/harnessResultsMock.ts
//
// MOCK results for Document-agent harness runs, so the whole flow can be
// tested without a backend: launch a harness run in the wizard -> it shows
// up in History -> its scorecard renders.
//
// Everything is generated deterministically from the launch payload (which
// agents / models / suites / sample size were picked), seeded by the run
// id — the same run always shows the same numbers. Nothing here is real
// measurement data.
import { HARNESS_AGENTS, MOCK_HARNESS_SUITES, listMockHarnessRuns, type MockHarnessRun } from './harnessMock';
import type { EvaluationListItem } from '../../types';
import type {
  HarnessCell,
  HarnessErrorType,
  HarnessResultsResponse,
  HarnessTrace,
  HarnessTraceStep,
} from '../endpoints/harnessResults';

const delay = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

// ---- seeded randomness ----------------------------------------------------
function hash(str: string): number {
  let h = 2166136261;
  for (let i = 0; i < str.length; i += 1) {
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return h >>> 0;
}
function rng(seed: string): () => number {
  let a = hash(seed);
  return () => {
    a = (a + 0x6d2b79f5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const clamp01 = (n: number) => Math.min(0.99, Math.max(0.05, n));

// ---- what the mock "knows" about each agent / use case ---------------------
// Baseline task success by agent and use case — Hermes is tuned as the
// stronger tool caller, SEMCo-Work as the stronger document worker, so the
// scorecard has a visible shape to it.
const BASE_SUCCESS: Record<string, Record<string, number>> = {
  hermes: { 'Tool Calling': 0.88, 'Document reading': 0.71, 'Spreadsheet editing': 0.62, 'Document editing': 0.55 },
  cowork: { 'Tool Calling': 0.79, 'Document reading': 0.83, 'Spreadsheet editing': 0.74, 'Document editing': 0.81 },
};
const STEP_RANGE: Record<string, [number, number]> = {
  'Tool Calling': [3, 5],
  'Document reading': [6, 9],
  'Spreadsheet editing': [9, 14],
  'Document editing': [11, 16],
};
const ERROR_WEIGHTS: Record<string, Record<HarnessErrorType, number>> = {
  hermes: { wrong_tool: 0.2, wrong_argument: 0.3, gave_up: 0.15, loop: 0.25, file_broken: 0.1 },
  cowork: { wrong_tool: 0.15, wrong_argument: 0.2, gave_up: 0.3, loop: 0.1, file_broken: 0.25 },
};
const ERROR_TYPES: HarnessErrorType[] = ['wrong_tool', 'wrong_argument', 'gave_up', 'loop', 'file_broken'];

const TASKS: Record<string, string[]> = {
  'Tool Calling': [
    'Book the cheapest direct flight and add it to the calendar',
    'Convert 250 EUR to JPY, then email the result',
    'Answer without tools: what is the capital of Australia?',
    'Look up two orders in parallel and compare delivery dates',
  ],
  'Document reading': [
    'What was FY2023 operating margin in the annual report?',
    'Find the year-over-year change in free cash flow',
    'Which segment had the highest capital expenditure?',
  ],
  'Spreadsheet editing': [
    'Add a VLOOKUP column for region and total by quarter',
    'Remove duplicate customer rows and fix the date format',
    'Rebuild the broken SUMIFS on the summary sheet',
  ],
  'Document editing': [
    'Update the pricing table in the Word proposal',
    'Make a 6-slide deck from the quarterly report',
    'Write a one-page user guide from the release notes',
    'Summarise the incident review for leadership',
  ],
};
const TOOLS: Record<string, string[]> = {
  'Tool Calling': ['search_flights', 'get_exchange_rate', 'send_email', 'calendar.create', 'orders.lookup'],
  'Document reading': ['pdf.open', 'pdf.search', 'pdf.read_page', 'calculator'],
  'Spreadsheet editing': ['xlsx.open', 'xlsx.read_range', 'xlsx.write_formula', 'xlsx.delete_rows', 'xlsx.save'],
  'Document editing': ['docx.open', 'docx.find_table', 'docx.update_cell', 'pptx.add_slide', 'file.save'],
};
const ERROR_NOTE: Record<HarnessErrorType, string> = {
  wrong_tool: 'Called a tool that cannot do this step',
  wrong_argument: 'Right tool, wrong argument value',
  gave_up: 'Stopped before the task was finished',
  loop: 'Repeated the same call again, then was stopped',
  file_broken: 'Output file could not be opened by the checker',
};

// ---- generator --------------------------------------------------------------
function buildResults(run: MockHarnessRun): HarnessResultsResponse {
  const { payload } = run;
  const suites = payload.suites
    .map((s) => ({ run: s, meta: MOCK_HARNESS_SUITES.find((m) => m.id === s.dataset_id) }))
    .filter((s): s is { run: (typeof payload.suites)[number]; meta: (typeof MOCK_HARNESS_SUITES)[number] } => Boolean(s.meta));

  const cells: HarnessCell[] = [];
  const traces: HarnessTrace[] = [];

  payload.agents.forEach((agent) => {
    payload.model_ids.forEach((modelId) => {
      suites.forEach(({ run: suiteRun, meta }) => {
        const useCase = meta.use_case || 'Tool Calling';
        const rand = rng(`${run.id}|${agent}|${modelId}|${meta.id}`);
        // run_samples 0 = the full suite (same convention as the launch payload)
        const tasks =
          suiteRun.run_samples > 0 ? Math.min(suiteRun.run_samples, meta.question_count) : meta.question_count;

        const modelShift = ((hash(modelId) % 9) - 4) / 100; // -0.04 … +0.04 per model
        const success = clamp01((BASE_SUCCESS[agent]?.[useCase] ?? 0.7) + modelShift + (rand() - 0.5) * 0.06);
        const passed = Math.round(tasks * success);
        const failed = tasks - passed;

        const [minSteps, maxSteps] = STEP_RANGE[useCase] ?? [5, 9];
        const avgSteps = minSteps + rand() * (maxSteps - minSteps);
        const wasted = avgSteps * (0.08 + rand() * 0.17);
        const tokens = Math.round(avgSteps * (1500 + rand() * 700));

        // spread the failed tasks over the error types
        const weights = { ...(ERROR_WEIGHTS[agent] ?? ERROR_WEIGHTS.hermes) };
        if (useCase === 'Tool Calling') weights.file_broken = 0; // no output file to break
        const weightSum = ERROR_TYPES.reduce((n, t) => n + weights[t], 0);
        const errors = { wrong_tool: 0, wrong_argument: 0, gave_up: 0, loop: 0, file_broken: 0 };
        let left = failed;
        ERROR_TYPES.forEach((t, i) => {
          const n = i === ERROR_TYPES.length - 1 ? left : Math.min(left, Math.round((failed * weights[t]) / weightSum));
          errors[t] = n;
          left -= n;
        });
        if (useCase === 'Tool Calling' && errors.file_broken > 0) {
          errors.wrong_argument += errors.file_broken;
          errors.file_broken = 0;
        }

        cells.push({
          agent,
          model_id: modelId,
          suite_id: meta.id,
          tasks,
          passed,
          tool_call_accuracy: clamp01(success + 0.04 + (rand() - 0.5) * 0.05),
          avg_steps: Number(avgSteps.toFixed(1)),
          wasted_steps: Number(wasted.toFixed(1)),
          faithfulness:
            useCase === 'Document reading' || useCase === 'Document editing' ? clamp01(0.8 + rand() * 0.15) : null,
          tokens_per_task: tokens,
          cost_per_task: Number(((tokens / 1_000_000) * 6).toFixed(4)),
          time_per_task: Number((avgSteps * (3.4 + rand() * 1.6)).toFixed(1)),
          errors,
        });

        // trace store sample: one passing and (if any) one failing task per cell
        if (payload.harness.save_traces) {
          const taskPool = TASKS[useCase] ?? TASKS['Tool Calling'];
          const toolPool = TOOLS[useCase] ?? TOOLS['Tool Calling'];
          // pick one of the error types this cell actually had, so the sample varies
          const present = ERROR_TYPES.filter((t) => errors[t] > 0);
          const failType = present.length > 0 ? present[Math.floor(rand() * present.length)] : null;
          [true, ...(failed > 0 ? [false] : [])].forEach((ok, k) => {
            const stepCount = Math.max(2, Math.round(avgSteps + (ok ? -1 : 2)));
            const steps: HarnessTraceStep[] = Array.from({ length: stepCount }).map((_, i) => {
              const last = i === stepCount - 1;
              const tool =
                !ok && failType === 'loop' && i > 1 ? toolPool[1 % toolPool.length] : toolPool[i % toolPool.length];
              return {
                tool,
                summary: last && !ok && failType ? ERROR_NOTE[failType] : last ? 'Returned the final answer' : `Step ${i + 1} completed`,
                ms: Math.round(400 + rand() * 4200),
                ok: ok || !last,
              };
            });
            traces.push({
              id: `${run.id}-${agent}-${hash(modelId + meta.id) % 1000}-${k}`,
              agent,
              model_id: modelId,
              suite_id: meta.id,
              task: taskPool[Math.floor(rand() * taskPool.length)],
              passed: ok,
              tokens: Math.round(stepCount * (1500 + rand() * 700)),
              time_sec: Number((steps.reduce((n, s) => n + s.ms, 0) / 1000).toFixed(1)),
              error_type: ok ? null : failType,
              steps,
            });
          });
        }
      });
    });
  });

  return {
    evaluation_id: run.id,
    name: payload.evaluation_name,
    agents: payload.agents.map((id) => ({ id, title: HARNESS_AGENTS.find((a) => a.id === id)?.title ?? id })),
    model_ids: payload.model_ids,
    suites: suites.map(({ meta }) => ({ id: meta.id, name: meta.name, use_case: meta.use_case || '' })),
    selected_metrics: payload.selected_metrics,
    settings: payload.harness,
    cells,
    traces,
  };
}

// ---- mock "API" ---------------------------------------------------------------
export async function mockGetHarnessResults(evaluationId: string): Promise<HarnessResultsResponse> {
  await delay(500);
  const run = listMockHarnessRuns().find((r) => r.id === evaluationId);
  if (!run) throw new Error('Harness run not found.');
  return buildResults(run);
}

// History list rows for the mock runs (newest first). In the real flow these
// come back from GET /evaluations like any other evaluation; while mocked,
// History.tsx merges them in on page 1.
export function listMockHarnessEvaluations(): EvaluationListItem[] {
  return listMockHarnessRuns().map((run) => {
    const res = buildResults(run);
    const byAgent = res.agents.map((a) => {
      const own = res.cells.filter((c) => c.agent === a.id);
      const tasks = own.reduce((n, c) => n + c.tasks, 0);
      const passed = own.reduce((n, c) => n + c.passed, 0);
      return { title: a.title, rate: tasks > 0 ? passed / tasks : 0 };
    });
    const best = [...byAgent].sort((x, y) => y.rate - x.rate)[0];
    return {
      id: run.id,
      name: run.payload.evaluation_name,
      description: 'Document-agent harness (mock)',
      eval_type: 'agent_harness',
      dataset_id: res.suites[0]?.id ?? '',
      datasets_config: res.suites.map((s) => ({ dataset_id: s.id })),
      benchmark: res.suites.map((s) => s.name).join(', '),
      model_ids: run.payload.model_ids,
      selected_metrics: run.payload.selected_metrics,
      run_samples: run.payload.suites[0]?.run_samples ?? 0,
      selected_category: [],
      status: 'completed',
      progress: 100,
      total_questions: res.cells.reduce((n, c) => n + c.tasks, 0),
      top_model: best ? best.title : null,
      top_score: best ? Math.round(best.rate * 100) : null,
      created_at: run.created_at,
      started_at: run.created_at,
      completed_at: run.created_at,
      report: null,
    };
  });
}
