//Custommetricsdashboard.tsx
import { useNavigate } from 'react-router-dom';
import { Gauge, X } from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { SAVED_METRICS, SAVED_DATASETS } from './mockData';

export default function CustomMetricsDashboard() {
  const navigate = useNavigate();

  const typeBadgeClass = (type: string) =>
    `${styles.badge} ${styles[`badge--${type}`] || ''}`;

  return (
    <div className={`page-enter pg-shell ${styles.cm}`}>
      <div className={styles['cm__header']}>
        <div>
          <p className={styles['cm__header-eyebrow']}>Custom Metrics</p>
          <h1>Dashboard</h1>
          <p className={styles['cm__header-sub']}>Saved metrics and datasets for evaluation</p>
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-scroll']}`}>
        <div className={styles['cards-row']}>
          <div className={styles.card}>
            <div className={styles['card-header']}>
              <h3>Saved Metrics</h3>
              <button
                type="button"
                className={`${styles.btn} ${styles['btn-sm']}`}
                onClick={() => navigate('/app/custom-metrics/create')}
              >
                + New
              </button>
            </div>
            <div className={styles['card-body']}>
              {SAVED_METRICS.length === 0 ? (
                <div className={styles.empty}>No metrics saved yet.</div>
              ) : (
                <div className={styles['table-wrap']}>
                  <table className={styles.table}>
                    <thead>
                      <tr>
                        <th>Name</th>
                        <th>Type</th>
                        <th>Created</th>
                        <th></th>
                      </tr>
                    </thead>
                    <tbody>
                      {SAVED_METRICS.map((m) => (
                        <tr key={m.id}>
                          <td>{m.name}</td>
                          <td><span className={typeBadgeClass(m.type)}>{m.type === 'code' ? 'Code' : 'Visual'}</span></td>
                          <td>{m.created}</td>
                          <td>
                            <button type="button" className={styles['btn-icon']} title="Delete">
                              <X size={13} />
                            </button>
                          </td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              )}
            </div>
          </div>

          <div className={styles.card}>
            <div className={styles['card-header']}>
              <h3>Saved Datasets</h3>
              <button
                type="button"
                className={`${styles.btn} ${styles['btn-sm']}`}
                onClick={() => navigate('/app/custom-metrics/upload')}
              >
                + Upload
              </button>
            </div>
            <div className={styles['card-body']}>
              {SAVED_DATASETS.length === 0 ? (
                <div className={styles.empty}>No datasets uploaded yet.</div>
              ) : (
                <div className={styles['table-wrap']}>
                  <table className={styles.table}>
                    <thead>
                      <tr>
                        <th>Name</th>
                        <th>Type</th>
                        <th>Rows</th>
                        <th></th>
                      </tr>
                    </thead>
                    <tbody>
                      {SAVED_DATASETS.map((d) => (
                        <tr key={d.id}>
                          <td>{d.name}</td>
                          <td><span className={typeBadgeClass(d.type)}>{d.type.toUpperCase()}</span></td>
                          <td>{d.rows}</td>
                          <td>
                            <button type="button" className={styles['btn-icon']} title="Delete">
                              <X size={13} />
                            </button>
                          </td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              )}
            </div>
          </div>
        </div>

        {SAVED_METRICS.length === 0 && SAVED_DATASETS.length === 0 && (
          <div className={styles.empty}>
            <Gauge size={16} /> Nothing here yet — create a metric or upload a dataset to get started.
          </div>
        )}
      </div>
    </div>
  );
}















//Createmetric.tsx
import { useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { Loader2, Plus, X } from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { useToast } from './useToast';
import {
  EVAL_TYPES, EvalType, METRIC_TEMPLATES, RULE_FIELDS, RULE_OPERATORS,
  CODE_TEMPLATE, VALIDATION_DATASETS, VALIDATION_ROWS,
} from './mockData';

type CompareType = 'literal' | 'field';

interface Rule {
  id: number;
  field: string;
  operator: string;
  compareType: CompareType;
  value: string;
}

let ruleSeq = 2;

export default function CreateMetric() {
  const navigate = useNavigate();
  const { showToast, ToastEl } = useToast();

  const [name, setName] = useState('');
  const [evalType, setEvalType] = useState<EvalType>('model');
  const [metricMode, setMetricMode] = useState<'visual' | 'code'>('visual');
  const [selectedTemplate, setSelectedTemplate] = useState(METRIC_TEMPLATES.model[0].id);
  const [rules, setRules] = useState<Rule[]>([
    { id: 1, field: 'actual_output', operator: 'contains', compareType: 'field', value: 'expected_output' },
  ]);
  const [gate, setGate] = useState<'AND' | 'OR'>('AND');
  const [threshold, setThreshold] = useState(0.5);
  const [code, setCode] = useState(CODE_TEMPLATE);
  const [validationDataset, setValidationDataset] = useState('');
  const [validating, setValidating] = useState(false);
  const [showResults, setShowResults] = useState(false);

  const handleEvalType = (t: EvalType) => {
    setEvalType(t);
    setSelectedTemplate(METRIC_TEMPLATES[t][0].id);
  };

  const addRule = () => {
    ruleSeq += 1;
    setRules((r) => [...r, { id: ruleSeq, field: 'input', operator: 'contains', compareType: 'literal', value: '' }]);
  };

  const removeRule = (id: number) => {
    setRules((r) => (r.length > 1 ? r.filter((rule) => rule.id !== id) : r));
  };

  const updateRule = (id: number, patch: Partial<Rule>) => {
    setRules((r) => r.map((rule) => (rule.id === id ? { ...rule, ...patch } : rule)));
  };

  const generatedLogic = useMemo(() => {
    const opLabel: Record<string, string> = {
      contains: 'contains', not_contains: 'not contains', equals: '==',
      starts_with: 'starts with', ends_with: 'ends with',
      length_gt: 'length >', length_lt: 'length <', regex: 'matches',
    };
    return rules
      .map((r) => `${r.field} ${opLabel[r.operator] || r.operator} ${r.compareType === 'field' ? r.value || '<field>' : `"${r.value || '…'}"`}`)
      .join(` ${gate} `);
  }, [rules, gate]);

  const runValidation = () => {
    if (!validationDataset) {
      showToast('Select a dataset to validate against', 'error');
      return;
    }
    setValidating(true);
    setShowResults(false);
    setTimeout(() => {
      setValidating(false);
      setShowResults(true);
    }, 700);
  };

  const avgScore = useMemo(
    () => (VALIDATION_ROWS.reduce((sum, r) => sum + r.score, 0) / VALIDATION_ROWS.length).toFixed(2),
    [],
  );
  const passRate = useMemo(
    () => Math.round((VALIDATION_ROWS.filter((r) => r.score >= threshold).length / VALIDATION_ROWS.length) * 100),
    [threshold],
  );

  const saveMetric = () => {
    if (!name.trim()) {
      showToast('Give the metric a name first', 'error');
      return;
    }
    showToast(`Metric "${name}" saved`, 'ok');
    setTimeout(() => navigate('/app/custom-metrics/dashboard'), 700);
  };

  return (
    <div className={`page-enter pg-shell ${styles.cm}`}>
      <div className={styles['cm__header']}>
        <div>
          <p className={styles['cm__header-eyebrow']}>Custom Metrics</p>
          <h1>Create Custom Metric</h1>
          <p className={styles['cm__header-sub']}>Build a scoring rule with the visual builder, or write your own DeepEval metric</p>
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-scroll']}`}>
        <div className={styles.panel}>
          <div className={styles['form-group']}>
            <label>Metric Name</label>
            <input
              className={styles.input}
              placeholder="e.g., Keyword Accuracy"
              value={name}
              onChange={(e) => setName(e.target.value)}
            />
          </div>

          <div className={styles['form-group']}>
            <label>Evaluation Type</label>
            <div className={styles['btn-group']}>
              {EVAL_TYPES.map((t) => (
                <button
                  key={t.key}
                  type="button"
                  className={`${styles['btn-toggle']} ${evalType === t.key ? styles.active : ''}`}
                  onClick={() => handleEvalType(t.key)}
                >
                  {t.label}
                </button>
              ))}
            </div>
          </div>

          <div className={styles['form-group']}>
            <label>Metric Type</label>
            <div className={styles['toggle-container']}>
              <button
                type="button"
                className={`${styles['toggle-btn']} ${metricMode === 'visual' ? styles.active : ''}`}
                onClick={() => setMetricMode('visual')}
              >
                Visual Builder
              </button>
              <button
                type="button"
                className={`${styles['toggle-btn']} ${metricMode === 'code' ? styles.active : ''}`}
                onClick={() => setMetricMode('code')}
              >
                Code Editor
              </button>
            </div>
          </div>

          {metricMode === 'visual' ? (
            <>
              <p className={styles.hint}>Select a pre-built metric template based on evaluation type</p>

              <div className={styles['metric-templates']}>
                {METRIC_TEMPLATES[evalType].map((tpl) => (
                  <div
                    key={tpl.id}
                    className={`${styles['metric-card']} ${selectedTemplate === tpl.id ? styles.selected : ''}`}
                    onClick={() => setSelectedTemplate(tpl.id)}
                  >
                    <div className={styles['metric-card-header']}>
                      <input
                        type="radio"
                        name={`metric-${evalType}`}
                        checked={selectedTemplate === tpl.id}
                        onChange={() => setSelectedTemplate(tpl.id)}
                      />
                      <strong>{tpl.name}</strong>
                    </div>
                    <p>{tpl.desc}</p>
                    <code>{tpl.code}</code>
                  </div>
                ))}
              </div>

              <div className={styles['custom-rules-section']}>
                <h4>Custom Rules <span className={styles.optional}>(Optional — combine with template above)</span></h4>

                {rules.map((rule) => (
                  <div key={rule.id} className={styles['rule-item']}>
                    <div className={styles['rule-fields']}>
                      <select
                        className={`${styles.select} ${styles['rule-field-select']}`}
                        value={rule.field}
                        onChange={(e) => updateRule(rule.id, { field: e.target.value })}
                      >
                        {RULE_FIELDS.map((f) => <option key={f} value={f}>{f}</option>)}
                      </select>
                      <select
                        className={`${styles.select} ${styles['rule-operator']}`}
                        value={rule.operator}
                        onChange={(e) => updateRule(rule.id, { operator: e.target.value })}
                      >
                        {RULE_OPERATORS.map((op) => <option key={op.value} value={op.value}>{op.label}</option>)}
                      </select>
                      <select
                        className={`${styles.select} ${styles['rule-compare-type']}`}
                        value={rule.compareType}
                        onChange={(e) => updateRule(rule.id, { compareType: e.target.value as CompareType, value: '' })}
                      >
                        <option value="literal">Literal Value</option>
                        <option value="field">Compare to Field</option>
                      </select>
                      {rule.compareType === 'literal' ? (
                        <input
                          className={`${styles.input} ${styles['rule-value']}`}
                          placeholder="enter value"
                          value={rule.value}
                          onChange={(e) => updateRule(rule.id, { value: e.target.value })}
                        />
                      ) : (
                        <select
                          className={`${styles.select} ${styles['rule-value']}`}
                          value={rule.value}
                          onChange={(e) => updateRule(rule.id, { value: e.target.value })}
                        >
                          <option value="">select field…</option>
                          {RULE_FIELDS.map((f) => <option key={f} value={f}>{f}</option>)}
                        </select>
                      )}
                      <button
                        type="button"
                        className={styles['btn-icon']}
                        title="Remove"
                        onClick={() => removeRule(rule.id)}
                      >
                        <X size={14} />
                      </button>
                    </div>
                  </div>
                ))}

                <div className={styles['add-rule-row']}>
                  <select
                    className={`${styles.select} ${styles['gate-select']}`}
                    value={gate}
                    onChange={(e) => setGate(e.target.value as 'AND' | 'OR')}
                  >
                    <option value="AND">AND</option>
                    <option value="OR">OR</option>
                  </select>
                  <button type="button" className={`${styles.btn} ${styles['btn-sm']}`} onClick={addRule}>
                    <Plus size={13} /> Add Rule
                  </button>
                </div>

                <div className={styles['rule-preview']}>
                  <label>Generated Logic</label>
                  <code>{generatedLogic || 'No rules defined'}</code>
                </div>
              </div>

              <div className={styles['threshold-row']}>
                <div className={styles['rule-field']}>
                  <label>Pass Threshold</label>
                  <input
                    type="number"
                    className={styles.input}
                    min={0}
                    max={1}
                    step={0.1}
                    value={threshold}
                    onChange={(e) => setThreshold(Math.min(1, Math.max(0, Number(e.target.value))))}
                  />
                </div>
              </div>
            </>
          ) : (
            <div className={styles['code-editor']}>
              <div className={styles['editor-header']}>
                <span>Python Code</span>
                <button
                  type="button"
                  className={`${styles.btn} ${styles['btn-sm']}`}
                  onClick={() => { setCode(CODE_TEMPLATE); showToast('Template loaded', 'ok'); }}
                >
                  Load Template
                </button>
              </div>
              <textarea
                className={styles['code-area']}
                spellCheck={false}
                value={code}
                onChange={(e) => setCode(e.target.value)}
              />
            </div>
          )}

          <div className={styles['validation-section']}>
            <div className={styles['validation-header']}>
              <h3>Validate Metric</h3>
              <p>Test your metric on sample data before saving</p>
            </div>
            <div className={styles['validation-controls']}>
              <select
                className={styles.select}
                value={validationDataset}
                onChange={(e) => setValidationDataset(e.target.value)}
              >
                <option value="">Select a dataset…</option>
                {VALIDATION_DATASETS.map((d) => <option key={d.value} value={d.value}>{d.label}</option>)}
              </select>
              <button type="button" className={styles.btn} onClick={runValidation} disabled={validating}>
                {validating ? <Loader2 size={13} className={styles.spin} /> : null}
                Run Validation
              </button>
            </div>

            {showResults && (
              <div className={styles['validation-results']}>
                <table className={styles.table}>
                  <thead>
                    <tr>
                      <th>#</th>
                      <th>Input</th>
                      <th>Output</th>
                      <th>Expected</th>
                      <th>Score</th>
                      <th>Status</th>
                    </tr>
                  </thead>
                  <tbody>
                    {VALIDATION_ROWS.map((row, i) => (
                      <tr key={i}>
                        <td className={styles['cell-num']}>{i + 1}</td>
                        <td>{row.input}</td>
                        <td>{row.output}</td>
                        <td>{row.expected}</td>
                        <td className={row.score >= threshold ? styles['cell-pass'] : styles['cell-fail']}>{row.score.toFixed(2)}</td>
                        <td className={row.score >= threshold ? styles['cell-pass'] : styles['cell-fail']}>
                          {row.score >= threshold ? 'Pass' : 'Fail'}
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
                <div className={styles['validation-summary']}>
                  <span>Average Score: <strong>{avgScore}</strong></span>
                  <span>Pass Rate: <strong>{passRate}%</strong></span>
                </div>
              </div>
            )}
          </div>

          <div className={styles['form-actions']}>
            <button
              type="button"
              className={`${styles.btn} ${styles['btn-secondary']}`}
              onClick={() => navigate('/app/custom-metrics/dashboard')}
            >
              Cancel
            </button>
            <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={saveMetric}>
              Save Metric
            </button>
          </div>
        </div>
      </div>

      {ToastEl}
    </div>
  );
}

















//Uploaddataset.tsx
import { useMemo, useRef, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { Sparkles, Upload, X } from 'lucide-react';
import styles from './CustomMetrics.module.scss';
import { useToast } from './useToast';
import { EVAL_TYPES, EvalType, SAMPLE_SOURCE_DATA, MAPPING_TARGETS, AUTO_MAP_GUESS } from './mockData';

const STEP_LABELS = ['Upload', 'Preview', 'Map Fields', 'Save'];

function formatFileSize(bytes: number) {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}

export default function UploadDataset() {
  const navigate = useNavigate();
  const { showToast, ToastEl } = useToast();
  const fileInputRef = useRef<HTMLInputElement>(null);

  const [step, setStep] = useState(1);
  const [datasetName, setDatasetName] = useState('');
  const [evalType, setEvalType] = useState<EvalType>('model');
  const [file, setFile] = useState<{ name: string; size: number } | null>(null);
  const [dragging, setDragging] = useState(false);
  const [mapping, setMapping] = useState<Record<string, string>>({});

  const sourceRows = SAMPLE_SOURCE_DATA[evalType];
  const columns = useMemo(() => (sourceRows.length ? Object.keys(sourceRows[0]) : []), [sourceRows]);
  const targets = MAPPING_TARGETS[evalType];

  const pickFile = () => fileInputRef.current?.click();

  const acceptFile = (name: string, size: number) => {
    setFile({ name, size });
  };

  const handleFileInput: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    const f = e.target.files?.[0];
    if (f) acceptFile(f.name, f.size);
  };

  const handleDrop: React.DragEventHandler<HTMLDivElement> = (e) => {
    e.preventDefault();
    setDragging(false);
    const f = e.dataTransfer.files?.[0];
    if (f) acceptFile(f.name, f.size);
    else acceptFile(`${evalType}_eval.json`, 4096); // mock fallback since this is static demo data
  };

  const clearFile = () => setFile(null);

  const processFile = () => {
    if (!file) return;
    setStep(2);
  };

  const goToStep = (s: number) => setStep(s);

  const autoMapFields = () => {
    setMapping(AUTO_MAP_GUESS[evalType]);
    showToast('Fields auto-mapped with AI', 'ok');
  };

  const requiredFilled = targets.filter((t) => t.required).every((t) => mapping[t.key]);

  const previewTransform = () => {
    if (!requiredFilled) {
      showToast('Map all required fields first', 'error');
      return;
    }
    setStep(4);
  };

  const transformed = useMemo(() => {
    return sourceRows.map((row) => {
      const out: Record<string, string> = {};
      targets.forEach((t) => {
        const src = mapping[t.key];
        if (src && row[src] !== undefined) out[t.key] = row[src];
      });
      return out;
    });
  }, [sourceRows, targets, mapping]);

  const validRows = transformed.filter((r) => targets.filter((t) => t.required).every((t) => r[t.key])).length;
  const errorRows = transformed.length - validRows;

  const saveDataset = () => {
    if (!datasetName.trim()) {
      showToast('Give the dataset a name first', 'error');
      return;
    }
    showToast(`Dataset "${datasetName}" saved`, 'ok');
    setTimeout(() => navigate('/app/custom-metrics/dashboard'), 700);
  };

  const stepClass = (n: number) => `${styles.step} ${step === n ? styles.active : ''} ${step > n ? styles.done : ''}`;

  return (
    <div className={`page-enter pg-shell ${styles.cm}`}>
      <div className={styles['cm__header']}>
        <div>
          <p className={styles['cm__header-eyebrow']}>Custom Metrics</p>
          <h1>Upload Dataset</h1>
          <p className={styles['cm__header-sub']}>Import a dataset and map it to the evaluation schema</p>
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-scroll']}`}>
        <div className={styles.steps}>
          {STEP_LABELS.map((label, i) => (
            <div key={label} style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
              <div className={stepClass(i + 1)}>
                <span className={styles['step-num']}>{i + 1}</span>
                <span className={styles['step-label']}>{label}</span>
              </div>
              {i < STEP_LABELS.length - 1 && <div className={styles['step-line']} />}
            </div>
          ))}
        </div>

        <div className={styles.panel}>
          {step === 1 && (
            <>
              <div className={styles['form-group']}>
                <label>Dataset Name</label>
                <input
                  className={styles.input}
                  placeholder="e.g., Customer Support QA"
                  value={datasetName}
                  onChange={(e) => setDatasetName(e.target.value)}
                />
              </div>

              <div className={styles['form-group']}>
                <label>Evaluation Type</label>
                <div className={styles['btn-group']}>
                  {EVAL_TYPES.map((t) => (
                    <button
                      key={t.key}
                      type="button"
                      className={`${styles['btn-toggle']} ${evalType === t.key ? styles.active : ''}`}
                      onClick={() => { setEvalType(t.key); setMapping({}); }}
                    >
                      {t.label}
                    </button>
                  ))}
                </div>
              </div>

              <div
                className={`${styles.dropzone} ${dragging ? styles.drag : ''}`}
                onClick={pickFile}
                onDragOver={(e) => { e.preventDefault(); setDragging(true); }}
                onDragLeave={() => setDragging(false)}
                onDrop={handleDrop}
              >
                <div className={styles['dropzone-icon']}><Upload size={20} /></div>
                <p>Drag and drop file here</p>
                <p className={styles['dropzone-hint']}>or click to browse</p>
                <p className={styles['dropzone-formats']}>Supported: JSON, CSV, Parquet, Arrow</p>
                <input ref={fileInputRef} type="file" accept=".json,.csv,.parquet,.arrow" hidden onChange={handleFileInput} />
              </div>

              {file && (
                <div className={styles['file-info']}>
                  <span className={styles['file-name']}>{file.name}</span>
                  <span className={styles['file-size']}>{formatFileSize(file.size)}</span>
                  <button type="button" className={styles['btn-icon']} onClick={clearFile}><X size={13} /></button>
                </div>
              )}

              <div className={styles['form-actions']}>
                <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={processFile} disabled={!file}>
                  Process File
                </button>
              </div>
            </>
          )}

          {step === 2 && (
            <>
              <h3 style={{ fontFamily: 'inherit' }}>Data Preview</h3>
              <p className={styles.hint}>Showing first {sourceRows.length} rows. Total rows: {sourceRows.length}</p>

              <div className={styles['table-wrap']}>
                <table className={styles.table}>
                  <thead>
                    <tr>{columns.map((c) => <th key={c}>{c}</th>)}</tr>
                  </thead>
                  <tbody>
                    {sourceRows.map((row, i) => (
                      <tr key={i}>{columns.map((c) => <td key={c}>{row[c]}</td>)}</tr>
                    ))}
                  </tbody>
                </table>
              </div>

              <div className={styles['detected-columns']}>
                <h4>Detected Columns</h4>
                <div className={styles['column-chips']}>
                  {columns.map((c) => (
                    <span key={c} className={styles.chip}>{c} <span>string</span></span>
                  ))}
                </div>
              </div>

              <div className={styles['form-actions']}>
                <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={() => goToStep(1)}>Back</button>
                <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={() => goToStep(3)}>Continue to Mapping</button>
              </div>
            </>
          )}

          {step === 3 && (
            <>
              <h3 style={{ fontFamily: 'inherit' }}>Map Columns to Fields</h3>
              <p className={styles.hint}>Map your data columns to the required evaluation fields</p>

              {targets.map((t) => (
                <div key={t.key} className={styles['mapping-row']}>
                  <div className={styles['mapping-target']}>
                    {t.required && <span className={styles.required}>*</span>}
                    {t.label}
                    <span className={styles['mapping-hint']}>{t.hint}</span>
                  </div>
                  <span className={styles['mapping-arrow']}>&#8594;</span>
                  <select
                    className={`${styles.select} ${styles['mapping-source']}`}
                    value={mapping[t.key] || ''}
                    onChange={(e) => setMapping((m) => ({ ...m, [t.key]: e.target.value }))}
                  >
                    <option value="">— none —</option>
                    {columns.map((c) => <option key={c} value={c}>{c}</option>)}
                  </select>
                </div>
              ))}

              <div className={styles['ai-assist']}>
                <button type="button" className={styles['btn-ai']} onClick={autoMapFields}>
                  <Sparkles size={13} /> Auto-Map with AI
                </button>
                <span className={styles['ai-hint']}>Uses LLM to suggest column mappings</span>
              </div>

              <div className={styles['form-actions']}>
                <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={() => goToStep(2)}>Back</button>
                <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={previewTransform}>Preview Result</button>
              </div>
            </>
          )}

          {step === 4 && (
            <>
              <h3 style={{ fontFamily: 'inherit' }}>Transformed Data Preview</h3>
              <p className={styles.hint}>Review the transformed data before saving</p>

              <div className={styles['json-preview']}>
                <pre>{JSON.stringify(transformed, null, 2)}</pre>
              </div>

              <div className={styles['transform-summary']}>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Total Rows</span>
                  <span className={styles['summary-value']}>{sourceRows.length}</span>
                </div>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Evaluation Type</span>
                  <span className={styles['summary-value']}>{evalType[0].toUpperCase() + evalType.slice(1)}</span>
                </div>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Valid Rows</span>
                  <span className={`${styles['summary-value']} ${styles['summary-value--ok']}`}>{validRows}</span>
                </div>
                <div className={styles['summary-item']}>
                  <span className={styles['summary-label']}>Errors</span>
                  <span className={`${styles['summary-value']} ${errorRows ? styles['summary-value--danger'] : ''}`}>{errorRows}</span>
                </div>
              </div>

              <div className={styles['form-actions']}>
                <button type="button" className={`${styles.btn} ${styles['btn-secondary']}`} onClick={() => goToStep(3)}>Back</button>
                <button type="button" className={`${styles.btn} ${styles['btn-primary']}`} onClick={saveDataset}>Save Dataset</button>
              </div>
            </>
          )}
        </div>
      </div>

      {ToastEl}
    </div>
  );
}

























//Custommetrics.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Custom Metrics — Dashboard / Create Metric / Upload Dataset.
// Mirrors the History/Reports/Comparison/Sidebar design system: ink/paper
// palette, ultramarine signal accent, mono instrument labels, hover-lift.
// Shared by all three sub-pages via CSS module composition.
// ===========================================================================

$ink:      var(--ink-1);
$ink-2:    var(--ink-2);
$ink-3:    var(--ink-3);
$paper:    var(--paper);
$card:     var(--card);
$line:     var(--line);
$line-2:   var(--line-2);
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     var(--signal-wash);
$ok:       #0FA968;
$ok-wash:  var(--ok-wash);
$amber:    #E08600;
$amber-wash: var(--amber-wash);
$danger:   #DC2626;
$danger-wash: var(--danger-wash);
$violet:   #6D28D9;
$violet-wash: rgba(109, 40, 217, 0.1);
$sky:      #0369A1;
$sky-wash: var(--sky-wash);
$ink-wash: var(--ink-wash);

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-toast-in {
  from { opacity: 0; transform: translate(-50%, 8px); }
  to   { opacity: 1; transform: translate(-50%, 0); }
}

// ---- shared page header -----------------------------------------------
.cm {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.84375rem;
    color: $ink-2;
  }
}

.pg-body-scroll {
  overflow-y: auto;
  padding: 20px 32px 32px;
}

// ---- buttons -------------------------------------------------------------
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.8125rem;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease;

  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; box-shadow: $soft; }
  &:disabled { opacity: 0.45; cursor: not-allowed; }
}

.btn-sm { padding: 6px 11px; font-size: 0.75rem; border-radius: 8px; }

.btn-primary {
  border-color: $signal;
  background: $signal;
  color: #fff;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn-secondary {
  background: $paper;
}

.btn-ai {
  border-color: rgba($violet, 0.3);
  background: $violet-wash;
  color: $violet;

  &:hover:not(:disabled) { border-color: $violet; background: rgba($violet, 0.16); color: $violet; }
}

.btn-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-3;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;

  &:hover { background: $danger-wash; border-color: rgba($danger, 0.2); color: $danger; }
}

.spin { animation: cm-spin 0.8s linear infinite; }

// ---- toggle groups ---------------------------------------------------------
.btn-group {
  display: inline-flex;
  padding: 3px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 11px;
  gap: 2px;
}

.btn-toggle {
  padding: 7px 16px;
  border-radius: 8px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.8125rem;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, box-shadow 0.15s ease;

  &:hover { color: $ink; }

  &.active {
    background: $card;
    color: $signal;
    box-shadow: $soft;
    font-weight: 700;
  }
}

.toggle-container {
  display: inline-flex;
  border: 1px solid $line;
  border-radius: 11px;
  overflow: hidden;
}

.toggle-btn {
  padding: 9px 18px;
  border: none;
  background: $paper;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.8125rem;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease;

  &:hover { color: $ink; }

  &.active {
    background: $signal;
    color: #fff;
  }

  &:not(:last-child) { border-right: 1px solid $line; }
}

// ---- cards (dashboard) ------------------------------------------------------
.cards-row {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
}

.card {
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  overflow: hidden;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 18px;
  border-bottom: 1px solid $line;

  h3 {
    font-family: $display;
    font-size: 0.9375rem;
    font-weight: 700;
    color: $ink;
  }
}

.card-body { padding: 4px 0 8px; }

// ---- generic table -----------------------------------------------------
.table-wrap {
  overflow-x: auto;
}

.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.8125rem;

  thead th {
    text-align: left;
    background: $paper;
    @extend %micro;
    font-size: 0.5625rem;
    color: $ink-3;
    padding: 10px 18px;
    white-space: nowrap;
  }

  tbody tr {
    border-top: 1px solid $line-2;
    transition: background 0.13s ease;
    &:hover { background: $paper; }
  }

  tbody td {
    padding: 11px 18px;
    color: $ink;
    vertical-align: middle;
  }
}

.badge {
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.625rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  border-radius: 6px;
  padding: 3px 8px;
  white-space: nowrap;
  color: $signal;
  background: $wash;

  &--code  { color: $violet; background: $violet-wash; }
  &--model { color: $signal; background: $wash; }
  &--rag   { color: $sky; background: $sky-wash; }
  &--agent { color: $amber; background: $amber-wash; }
}

// ---- forms ---------------------------------------------------------------
.form-group {
  margin-bottom: 20px;

  label {
    display: block;
    @extend %micro;
    font-size: 0.6875rem;
    color: $ink-2;
    margin-bottom: 8px;
  }
}

.input,
.select {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 12px;
  font-size: 0.8125rem;
  font-family: $sans;
  color: $ink;
  background: $card;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.select { cursor: pointer; }

.hint {
  font-size: 0.78125rem;
  color: $ink-3;
  margin: -4px 0 16px;
}

.panel {
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  padding: 24px;
}

.panel + .panel { margin-top: 16px; }

// ---- metric template cards --------------------------------------------
.metric-templates {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.metric-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }

  &.selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.metric-card-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.84375rem;
  color: $ink;

  input[type='radio'] { accent-color: $signal; }
}

.metric-card p {
  font-size: 0.75rem;
  color: $ink-2;
  line-height: 1.45;
  margin-bottom: 8px;
}

.metric-card code {
  display: block;
  font-family: $mono;
  font-size: 0.6875rem;
  color: $signal;
  background: $card;
  border: 1px solid $line;
  border-radius: 7px;
  padding: 6px 8px;
  overflow-x: auto;
  white-space: nowrap;
}

// ---- custom rule builder ---------------------------------------------
.custom-rules-section {
  border-top: 1px solid $line;
  padding-top: 20px;
  margin-bottom: 20px;

  h4 {
    font-family: $display;
    font-size: 0.875rem;
    font-weight: 700;
    color: $ink;
    margin-bottom: 12px;
  }
}

.optional {
  @extend %micro;
  font-size: 0.625rem;
  color: $ink-3;
  margin-left: 6px;
}

.rule-item {
  margin-bottom: 8px;
}

.rule-fields {
  display: flex;
  align-items: center;
  gap: 8px;

  select, input {
    flex-shrink: 0;
  }
}

.rule-field-select { width: 150px; }
.rule-operator { width: 140px; }
.rule-compare-type { width: 140px; }
.rule-value { flex: 1; min-width: 0; }

.add-rule-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 10px 0 16px;
}

.gate-select { width: 90px; font-weight: 700; color: $signal; }

.rule-preview {
  label {
    display: block;
    @extend %micro;
    font-size: 0.625rem;
    color: $ink-3;
    margin-bottom: 6px;
  }

  code {
    display: block;
    font-family: $mono;
    font-size: 0.75rem;
    color: $ink;
    background: $paper;
    border: 1px solid $line;
    border-radius: 9px;
    padding: 10px 12px;
  }
}

.threshold-row {
  display: flex;
  gap: 24px;
  margin-top: 16px;

  .rule-field {
    label {
      display: block;
      @extend %micro;
      font-size: 0.625rem;
      color: $ink-2;
      margin-bottom: 6px;
    }

    input {
      width: 100px;
    }
  }
}

// ---- code editor -----------------------------------------------------
.code-editor {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
  margin-bottom: 24px;
}

.editor-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  background: $paper;
  border-bottom: 1px solid $line;
  @extend %micro;
  font-size: 0.6875rem;
  color: $ink-2;
}

.code-area {
  width: 100%;
  min-height: 320px;
  border: none;
  resize: vertical;
  padding: 16px;
  font-family: $mono;
  font-size: 0.78125rem;
  line-height: 1.6;
  color: $ink;
  background: $card;

  &:focus { outline: none; }
}

// ---- validation ---------------------------------------------------------
.validation-section {
  border-top: 1px solid $line;
  padding-top: 20px;
  margin-bottom: 24px;
}

.validation-header {
  margin-bottom: 12px;

  h3 {
    font-family: $display;
    font-size: 0.9375rem;
    font-weight: 700;
    color: $ink;
  }

  p {
    font-size: 0.78125rem;
    color: $ink-3;
    margin-top: 2px;
  }
}

.validation-controls {
  display: flex;
  gap: 10px;
  margin-bottom: 16px;

  select { max-width: 280px; }
}

.validation-results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}

.validation-summary {
  display: flex;
  gap: 24px;
  padding: 12px 18px;
  background: $paper;
  border-top: 1px solid $line;
  font-size: 0.8125rem;
  color: $ink-2;

  strong { color: $ink; font-family: $mono; }
}

.cell-pass { font-family: $mono; font-weight: 700; color: $ok; }
.cell-fail { font-family: $mono; font-weight: 700; color: $danger; }
.cell-num { font-family: $mono; font-weight: 700; color: $ink; }

// ---- form actions ---------------------------------------------------------
.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding-top: 8px;
}

// ---- upload wizard: steps -------------------------------------------------
.steps {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 28px;
}

.step {
  display: flex;
  align-items: center;
  gap: 8px;
  opacity: 0.55;
  transition: opacity 0.15s ease;

  &.active, &.done { opacity: 1; }
}

.step-num {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: $mono;
  font-size: 0.75rem;
  font-weight: 700;
  color: $ink-3;
  background: $paper;
  border: 1.5px solid $line;
}

.step.active .step-num {
  color: #fff;
  background: $signal;
  border-color: $signal;
}

.step.done .step-num {
  color: $signal;
  background: $wash;
  border-color: $signal;
}

.step-label {
  font-size: 0.8125rem;
  font-weight: 650;
  color: $ink-2;
}

.step.active .step-label { color: $ink; font-weight: 700; }

.step-line {
  width: 32px;
  height: 1.5px;
  background: $line;
}

// ---- dropzone --------------------------------------------------------
.dropzone {
  border: 1.5px dashed $line;
  border-radius: 16px;
  padding: 40px 20px;
  text-align: center;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, background 0.15s ease;
  margin-bottom: 16px;

  &:hover, &.drag { border-color: $signal; background: $wash; }
}

.dropzone-icon {
  width: 44px;
  height: 44px;
  margin: 0 auto 12px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $card;
  border: 1px solid $line;
  color: $signal;
}

.dropzone p { font-size: 0.875rem; color: $ink; font-weight: 650; margin-bottom: 2px; }
.dropzone-hint { font-size: 0.78125rem; color: $ink-3 !important; font-weight: 500 !important; }
.dropzone-formats { font-family: $mono; font-size: 0.6875rem !important; color: $ink-3 !important; margin-top: 8px !important; font-weight: 500 !important; }

.file-info {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  margin-bottom: 16px;

  .file-name { font-weight: 700; color: $ink; font-size: 0.8125rem; }
  .file-size { font-family: $mono; font-size: 0.71875rem; color: $ink-3; }
}

// ---- detected columns / chips ------------------------------------------
.detected-columns {
  margin-top: 20px;

  h4 {
    @extend %micro;
    font-size: 0.6875rem;
    color: $ink-2;
    margin-bottom: 10px;
  }
}

.column-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 4px 10px;

  span { color: $ink-3; font-weight: 500; text-transform: none; letter-spacing: 0; }
}

// ---- field mapping --------------------------------------------------------
.mapping-row {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 12px 0;
  border-bottom: 1px solid $line-2;

  &:last-child { border-bottom: none; }
}

.mapping-target {
  flex: 0 0 220px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.84375rem;
  color: $ink;
}

.required { color: $danger; margin-right: 3px; }

.mapping-hint {
  display: block;
  font-family: $sans;
  font-weight: 500;
  font-size: 0.71875rem;
  color: $ink-3;
  margin-top: 2px;
}

.mapping-arrow { color: $ink-3; flex-shrink: 0; }
.mapping-source { flex: 1; }

.ai-assist {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 20px 0;
}

.ai-hint { font-size: 0.75rem; color: $ink-3; }

// ---- json preview / summary --------------------------------------------
.json-preview {
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;
  padding: 16px;
  margin-bottom: 20px;
  max-height: 340px;
  overflow: auto;

  pre {
    font-family: $mono;
    font-size: 0.75rem;
    line-height: 1.6;
    color: $ink;
    white-space: pre-wrap;
    word-break: break-word;
  }
}

.transform-summary {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 14px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;

  .summary-label { @extend %micro; font-size: 0.5625rem; color: $ink-3; }
  .summary-value { font-family: $mono; font-size: 1.0625rem; font-weight: 700; color: $ink; }
  .summary-value--danger { color: $danger; }
  .summary-value--ok { color: $ok; }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 0.8125rem;
}

// ---- toast --------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%;
  bottom: 28px;
  transform: translateX(-50%);
  z-index: 200;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 11px 18px;
  border-radius: 11px;
  background: #14161B;
  color: #fff;
  font-size: 0.8125rem;
  font-weight: 650;
  box-shadow: $lift;
  animation: cm-toast-in 0.18s ease;

  &--ok::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $ok; flex-shrink: 0; }
  &--error::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: #FF6B6B; flex-shrink: 0; }
}

@media (max-width: 900px) {
  .cards-row { grid-template-columns: 1fr; }
  .metric-templates { grid-template-columns: 1fr; }
  .transform-summary { grid-template-columns: repeat(2, 1fr); }
}

@media (max-width: 640px) {
  .cm__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .pg-body-scroll { padding: 16px 18px 22px; }
  .rule-fields { flex-wrap: wrap; }
  .mapping-row { flex-wrap: wrap; }
  .mapping-target { flex: 1 1 100%; }
}























//Mockdata.ts
export type EvalType = 'model' | 'agent' | 'rag';

export const EVAL_TYPES: { key: EvalType; label: string }[] = [
  { key: 'model', label: 'Model' },
  { key: 'agent', label: 'Agent' },
  { key: 'rag', label: 'RAG' },
];

// ---- Dashboard --------------------------------------------------------
export const SAVED_METRICS = [
  { id: 'm1', name: 'Keyword Match', type: 'visual' as const, created: '2024-01-15' },
  { id: 'm2', name: 'Custom ROUGE', type: 'code' as const, created: '2024-01-14' },
];

export const SAVED_DATASETS = [
  { id: 'd1', name: 'QA Dataset', type: 'model' as EvalType, rows: 150 },
  { id: 'd2', name: 'RAG Test Set', type: 'rag' as EvalType, rows: 75 },
];

// ---- Create Metric — visual builder templates --------------------------
export const METRIC_TEMPLATES: Record<EvalType, { id: string; name: string; desc: string; code: string }[]> = {
  model: [
    {
      id: 'exact_match',
      name: 'Exact Match',
      desc: 'Checks if actual_output exactly matches expected_output',
      code: 'score = 1.0 if actual_output == expected_output else 0.0',
    },
    {
      id: 'contains_answer',
      name: 'Contains Answer',
      desc: 'Checks if expected_output appears within actual_output',
      code: 'score = 1.0 if expected_output in actual_output else 0.0',
    },
    {
      id: 'keyword_match',
      name: 'Keyword Match',
      desc: 'Calculates percentage of expected keywords found in output',
      code: 'score = matched_keywords / total_keywords',
    },
  ],
  rag: [
    {
      id: 'faithfulness',
      name: 'Faithfulness',
      desc: 'Checks if actual_output is grounded in retrieval_context',
      code: 'score = claims_in_context / total_claims',
    },
    {
      id: 'context_relevancy',
      name: 'Context Relevancy',
      desc: 'Measures how relevant the retrieval_context is to input',
      code: 'score = relevant_sentences / total_sentences',
    },
    {
      id: 'answer_relevancy',
      name: 'Answer Relevancy',
      desc: 'Checks if actual_output answers the input question',
      code: 'score = semantic_similarity(output, expected)',
    },
  ],
  agent: [
    {
      id: 'tool_correctness',
      name: 'Tool Correctness',
      desc: 'Checks if tools_called matches expected_tools',
      code: 'score = matched_tools / expected_tools',
    },
    {
      id: 'task_completion',
      name: 'Task Completion',
      desc: 'Evaluates if the agent completed the requested task',
      code: 'score = LLM_judge(input, output, tools_called)',
    },
    {
      id: 'tool_params',
      name: 'Parameter Accuracy',
      desc: 'Checks if tool parameters match expected values',
      code: 'score = matched_params / total_params',
    },
  ],
};

export const RULE_FIELDS = ['input', 'actual_output', 'expected_output', 'context'];

export const RULE_OPERATORS: { value: string; label: string }[] = [
  { value: 'contains', label: 'contains' },
  { value: 'not_contains', label: 'not contains' },
  { value: 'equals', label: 'equals' },
  { value: 'starts_with', label: 'starts with' },
  { value: 'ends_with', label: 'ends with' },
  { value: 'length_gt', label: 'length >' },
  { value: 'length_lt', label: 'length <' },
  { value: 'regex', label: 'regex match' },
];

export const CODE_TEMPLATE = `from deepeval.metrics import BaseMetric
from deepeval.test_case import LLMTestCase

class CustomMetric(BaseMetric):
    """
    LLMTestCase fields:
    - test_case.input: the prompt/question (str)
    - test_case.actual_output: model's response (str)
    - test_case.expected_output: expected answer (str)
    - test_case.retrieval_context: context docs (list[str])
    - test_case.tools_called: tools used (list[ToolCall])
    - test_case.expected_tools: expected tools (list[ToolCall])
    """

    def __init__(self, threshold: float = 0.5):
        self.threshold = threshold

    def measure(self, test_case: LLMTestCase) -> float:
        # Example: Check if expected answer is in output
        if test_case.expected_output.lower() in test_case.actual_output.lower():
            self.score = 1.0
        else:
            self.score = 0.0
        self.success = self.score >= self.threshold
        return self.score

    async def a_measure(self, test_case: LLMTestCase) -> float:
        return self.measure(test_case)

    def is_successful(self) -> bool:
        return self.success

    @property
    def __name__(self):
        return "Custom Metric"
`;

export const VALIDATION_DATASETS = [
  { value: 'qa-sample', label: 'QA Dataset (150 rows)' },
  { value: 'rag-sample', label: 'RAG Test Set (75 rows)' },
];

export const VALIDATION_ROWS = [
  { input: 'What is the capital of France?', output: 'Paris is the capital of France.', expected: 'Paris', score: 1.0 },
  { input: 'Who wrote Romeo and Juliet?', output: 'It was written by William Shakespeare.', expected: 'William Shakespeare', score: 1.0 },
  { input: 'What is the chemical symbol for water?', output: 'The formula is H2O.', expected: 'H2O', score: 1.0 },
  { input: 'What year did World War II end?', output: 'The war ended in 1946.', expected: '1945', score: 0.0 },
  { input: 'What is the largest planet?', output: 'Jupiter is the largest planet in our solar system.', expected: 'Jupiter', score: 1.0 },
];

// ---- Upload Dataset — sample source files (first rows) ------------------
export const SAMPLE_SOURCE_DATA: Record<EvalType, Record<string, string>[]> = {
  model: [
    { question: 'What is the capital of France?', answer: 'Paris' },
    { question: 'Who wrote Romeo and Juliet?', answer: 'William Shakespeare' },
    { question: 'What is the chemical symbol for water?', answer: 'H2O' },
    { question: 'What year did World War II end?', answer: '1945' },
    { question: 'What is the largest planet in our solar system?', answer: 'Jupiter' },
  ],
  agent: [
    { task: "What's the weather in San Francisco?", tools_json: 'get_weather({"location":"San Francisco"})', final_answer: 'The weather in San Francisco is sunny with a high of 68°F' },
    { task: 'Search for the latest news about AI', tools_json: 'web_search({"query":"latest AI news"})', final_answer: 'Here are the latest AI news articles...' },
    { task: 'Send an email to john@example.com about the meeting', tools_json: 'send_email({"to":"john@example.com"})', final_answer: 'Email sent successfully to john@example.com' },
    { task: 'Calculate 25% of 200', tools_json: 'calculator({"expression":"0.25 * 200"})', final_answer: '25% of 200 is 50' },
    { task: 'Book a flight from NYC to LA for tomorrow', tools_json: 'book_flight({"from":"NYC","to":"LA"})', final_answer: 'Flight booked from NYC to LA' },
  ],
  rag: [
    { question: "What is the company's return policy?", context: 'Our return policy allows returns within 30 days of purchase with original receipt.', answer: '30 days with receipt' },
    { question: 'How do I reset my password?', context: "Click the 'Forgot Password' link on the login page.", answer: 'Click forgot password on login page' },
    { question: 'What are the shipping options?', context: 'We offer Standard (5-7 days) and Express (1-2 days) shipping.', answer: 'Standard (5-7 days) and Express (1-2 days)' },
    { question: 'How do I contact customer support?', context: 'Email support@company.com or call 1-800-123-4567.', answer: 'Email support@company.com or call 1-800-123-4567' },
    { question: 'What payment methods do you accept?', context: 'We accept credit cards, PayPal, and Apple Pay.', answer: 'Credit cards, PayPal, and Apple Pay' },
  ],
};

export const MAPPING_TARGETS: Record<EvalType, { key: string; label: string; hint: string; required: boolean }[]> = {
  model: [
    { key: 'input', label: 'input', hint: 'The question or prompt', required: true },
    { key: 'expected_output', label: 'expected_output', hint: 'The correct answer', required: true },
    { key: 'context', label: 'context', hint: 'Additional context (optional)', required: false },
  ],
  agent: [
    { key: 'input', label: 'input', hint: 'The task description', required: true },
    { key: 'expected_tools', label: 'expected_tools', hint: 'Expected tool calls (JSON)', required: true },
    { key: 'expected_output', label: 'expected_output', hint: 'Expected final answer (optional)', required: false },
  ],
  rag: [
    { key: 'input', label: 'input', hint: 'The question', required: true },
    { key: 'retrieval_context', label: 'retrieval_context', hint: 'Retrieved context documents', required: true },
    { key: 'expected_output', label: 'expected_output', hint: 'Expected answer', required: true },
  ],
};

// naive best-guess auto-mapping, keyed by eval type
export const AUTO_MAP_GUESS: Record<EvalType, Record<string, string>> = {
  model: { input: 'question', expected_output: 'answer', context: '' },
  agent: { input: 'task', expected_tools: 'tools_json', expected_output: 'final_answer' },
  rag: { input: 'question', retrieval_context: 'context', expected_output: 'answer' },
};



















//Usetoast.tsx
import { useCallback, useEffect, useState } from 'react';
import styles from './CustomMetrics.module.scss';

type ToastState = { message: string; type: 'ok' | 'error' | 'info' } | null;

export function useToast() {
  const [toast, setToast] = useState<ToastState>(null);

  useEffect(() => {
    if (!toast) return;
    const t = setTimeout(() => setToast(null), 2600);
    return () => clearTimeout(t);
  }, [toast]);

  const showToast = useCallback((message: string, type: 'ok' | 'error' | 'info' = 'info') => {
    setToast({ message, type });
  }, []);

  const ToastEl = toast ? (
    <div className={`${styles.toast} ${styles[`toast--${toast.type}`] || ''}`}>{toast.message}</div>
  ) : null;

  return { showToast, ToastEl };
}




















//Sidebar.tsx
import { useState } from 'react';
import { Link, NavLink, useLocation } from 'react-router-dom';
import {
  Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, FileText, LogOut,
  Gauge, ChevronDown, LayoutDashboard, PenSquare, UploadCloud,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { logout } from '../../store/slices/authSlice';
import ThemeToggle from '../common/ThemeToggle';
import styles from './Sidebar.module.scss';

const navItems = [
  { to: '/app/dashboard', icon: <Home size={18} />, label: 'Dashboard' },
  { to: '/app/providers', icon: <Link2 size={18} />, label: 'Providers' },
  { to: '/app/models', icon: <Cpu size={18} />, label: 'Models' },
  { to: '/app/datasets', icon: <BookOpen size={18} />, label: 'Datasets' },
];

const workflowItems = [
  { to: '/app/run-evaluation', icon: <Play size={18} />, label: 'New Evaluation' },
  { to: '/app/history', icon: <FlaskConical size={18} />, label: 'History' },
  { to: '/app/comparison', icon: <GitCompare size={18} />, label: 'Comparison' },
  { to: '/app/reports', icon: <FileText size={18} />, label: 'Reports' },
];

const customMetricsSubItems = [
  { to: '/app/custom-metrics/dashboard', icon: <LayoutDashboard size={15} />, label: 'Dashboard' },
  { to: '/app/custom-metrics/create', icon: <PenSquare size={15} />, label: 'Create Metric' },
  { to: '/app/custom-metrics/upload', icon: <UploadCloud size={15} />, label: 'Upload Dataset' },
];

export default function Sidebar() {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.auth.user);
  const location = useLocation();

  const isOnCustomMetrics = location.pathname.startsWith('/app/custom-metrics');
  const [customMetricsOpen, setCustomMetricsOpen] = useState(isOnCustomMetrics);

  const navLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${isActive ? styles.active : ''}`;

  const subNavLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${styles['nav-item--sub']} ${isActive ? styles.active : ''}`;

  return (
    <div className={styles.sidebar}>
      <Link to="/" className={styles['sidebar__logo']}>
        <div className={styles['sidebar__mark']}>&#9670;</div>
        SemcoEval
      </Link>
      <nav className={styles['sidebar__nav']}>
        {navItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}

        <div className={styles['sidebar__section']}>Workflow</div>
        {workflowItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}

        <button
          type="button"
          className={`${styles['nav-item']} ${styles['nav-item--expandable']} ${isOnCustomMetrics ? styles.active : ''}`}
          onClick={() => setCustomMetricsOpen((o) => !o)}
          aria-expanded={customMetricsOpen}
        >
          <Gauge size={18} />
          Custom Metrics
          <ChevronDown
            size={14}
            className={`${styles['nav-item__chevron']} ${customMetricsOpen ? styles['nav-item__chevron--open'] : ''}`}
          />
        </button>

        <div className={`${styles['nav-submenu']} ${customMetricsOpen ? styles['nav-submenu--open'] : ''}`}>
          <div className={styles['nav-submenu__inner']}>
            {customMetricsSubItems.map((item) => (
              <NavLink key={item.to} to={item.to} className={subNavLinkClass}>
                {item.icon}
                {item.label}
              </NavLink>
            ))}
          </div>
        </div>
      </nav>
      <div className={styles['sidebar__foot']}>
        <div className={styles['sidebar__theme-row']}>
          <span>Theme</span>
          <ThemeToggle />
        </div>
        <div className={styles['sidebar__user']}>
          <div className={styles['sidebar__avatar']}>
            {(user?.name || user?.email || '?').slice(0, 1).toUpperCase()}
          </div>
          <div className={styles['sidebar__user-info']}>
            <div className={styles['sidebar__user-name']}>{user?.name || 'Account'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email}</div>
          </div>
          <button
            type="button"
            className={styles['sidebar__logout']}
            title="Log out"
            onClick={() => dispatch(logout())}
          >
            <LogOut size={15} />
          </button>
        </div>
      </div>
    </div>
  );
}





















//Sidebar.module.scss
// ---- Add to Sidebar.module.scss ----------------------------------------
// Paste inside the file, after the existing `.nav-item { ... }` block.

.nav-item--expandable {
  justify-content: flex-start;
  position: relative;
}

.nav-item__chevron {
  margin-left: auto;
  color: $ink-3;
  transition: transform 0.18s ease, color 0.18s ease;
  flex-shrink: 0;

  &--open { transform: rotate(180deg); }
}

.nav-item--expandable.active .nav-item__chevron { color: $signal-active; }

.nav-submenu {
  display: grid;
  grid-template-rows: 0fr;
  overflow: hidden;
  transition: grid-template-rows 0.18s ease;

  &--open { grid-template-rows: 1fr; }
}

.nav-submenu__inner {
  min-height: 0;
  display: flex;
  flex-direction: column;
  gap: 1px;
  padding-left: 14px;
  margin-top: 2px;
  border-left: 1.5px solid $line;
}

.nav-item--sub {
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  padding: 8px 12px;
  gap: 10px;

  svg { color: $ink-3; }

  &.active {
    box-shadow: none;
    background: $wash;
  }
}
