//app.js
/**
 * SemcoEval - Interactive Prototype
 * Full E2E wireframe with all features functional
 */

const AppState = {
  currentView: 'dashboard',
  wizardStep: 1,
  evaluation: {
    name: '',
    type: null,
    providers: [],
    models: [],
    dataset: null,
    metrics: []
  },
  compareModels: [],
  uploadedFile: null
};

// Community datasets
const COMMUNITY_DATASETS = [
  {
    id: 'comm-chatbot-arena',
    name: 'LMSYS Chatbot Arena Eval',
    category: 'General',
    questions: 3200,
    language: 'English',
    difficulty: 'Varied',
    version: 'v2.3',
    maintainer: 'LMSYS Community',
    description: 'Large-scale human preference evaluation across conversational AI models with ELO ratings.',
    featured: true
  },
  {
    id: 'comm-agentbench',
    name: 'AgentBench v2.1',
    category: 'Agents',
    questions: 890,
    language: 'English / Code',
    difficulty: 'Expert',
    version: 'v2.1',
    maintainer: 'THU-CoAI',
    description: 'Comprehensive benchmark for evaluating LLM-as-Agent across 8 distinct environments.',
    featured: true
  },
  {
    id: 'comm-truthfulqa',
    name: 'TruthfulQA Corporate Defense',
    category: 'Safety',
    questions: 817,
    language: 'English',
    difficulty: 'Medium',
    version: '2.0-Corp',
    maintainer: 'AI Safety Foundation',
    description: 'Tests model truthfulness and resistance to generating false or misleading information in business contexts.',
    featured: false
  }
];

// Init
document.addEventListener('DOMContentLoaded', () => {
  lucide.createIcons();
  showLoginPage();
});

// ============ LOGIN ============
function showLoginPage() {
  document.getElementById('login-page').style.display = 'flex';
  document.getElementById('app-container').style.display = 'none';
  lucide.createIcons();
}

function handleLogin(e) {
  e.preventDefault();
  const btn = e.target.querySelector('button[type="submit"]');
  btn.innerHTML = '<span class="spinner"></span>';
  btn.disabled = true;

  setTimeout(() => {
    document.getElementById('login-page').style.display = 'none';
    document.getElementById('app-container').style.display = 'flex';
    lucide.createIcons();
    renderDashboard();
  }, 600);
}

// ============ NAVIGATION ============
function navigateTo(page) {
  document.querySelectorAll('.nav-item').forEach(item => {
    item.classList.remove('active');
    if (item.dataset.page === page) item.classList.add('active');
  });

  document.querySelectorAll('.view-section').forEach(v => v.classList.remove('active'));
  const target = document.getElementById(`view-${page}`);
  if (target) {
    target.classList.add('active');
    AppState.currentView = page;
  }

  switch(page) {
    case 'dashboard': renderDashboard(); break;
    case 'run-evaluation': initWizard(); break;
    case 'history': renderHistory(); break;
    case 'models': renderModelsCatalog(); break;
    case 'providers': renderProvidersPage(); break;
    case 'datasets': renderDatasetsLibrary(); break;
    case 'reports': renderReports(); break;
  }

  lucide.createIcons();
}

// ============ DASHBOARD ============
function renderDashboard() {
  document.getElementById('dashboard-stats').innerHTML = `
    <div class="stat-card">
      <div class="stat-value">${SEMCO_DATA.models.length}</div>
      <div class="stat-label">Models</div>
    </div>
    <div class="stat-card">
      <div class="stat-value">${SEMCO_DATA.providers.filter(p => p.status === 'connected').length}</div>
      <div class="stat-label">Providers</div>
    </div>
    <div class="stat-card">
      <div class="stat-value">${SEMCO_DATA.testSuites.length}</div>
      <div class="stat-label">Test Suites</div>
    </div>
    <div class="stat-card">
      <div class="stat-value">${SEMCO_DATA.recentEvaluations.length}</div>
      <div class="stat-label">Evaluations</div>
    </div>
  `;

  document.getElementById('recent-evals-list').innerHTML = SEMCO_DATA.recentEvaluations.map(ev => `
    <div class="recent-eval-item" onclick="viewEvaluationResult('${ev.id}')">
      <div class="eval-info">
        <div class="eval-name">${ev.name}</div>
        <div class="eval-meta">
          <span class="eval-type-badge">${ev.type.split('(')[0].trim()}</span>
          <span class="eval-date">${ev.date}</span>
        </div>
      </div>
      <div class="eval-stats">
        <div class="eval-stat">
          <span class="eval-stat-label">Top</span>
          <span class="eval-stat-value">${ev.topModel.split(' - ')[0]}</span>
        </div>
        <div class="eval-stat">
          <span class="eval-stat-label">Score</span>
          <span class="eval-stat-value highlight">${ev.topScore}</span>
        </div>
      </div>
      <i data-lucide="chevron-right" class="eval-arrow"></i>
    </div>
  `).join('');

  const latest = SEMCO_DATA.recentEvaluations[0];
  document.getElementById('dashboard-leaderboard').innerHTML = `
    <table class="data-table">
      <thead>
        <tr>
          <th style="width:50px">Rank</th>
          <th>Model</th>
          <th>Provider</th>
          <th>Score</th>
          <th>Accuracy</th>
          <th>Speed</th>
          <th>Cost</th>
        </tr>
      </thead>
      <tbody>
        ${latest.results.map(r => `
          <tr>
            <td><div class="rank-badge rank-${r.rank}">${r.rank}</div></td>
            <td><strong>${r.model}</strong></td>
            <td>${r.provider}</td>
            <td><span class="score-highlight">${r.score}</span></td>
            <td>${r.accuracy}</td>
            <td>${r.time}</td>
            <td>${r.cost}</td>
          </tr>
        `).join('')}
      </tbody>
    </table>
  `;

  lucide.createIcons();
}

// ============ WIZARD ============
function initWizard() {
  AppState.wizardStep = 1;
  AppState.evaluation = { name: '', type: null, providers: [], models: [], dataset: null, metrics: [] };
  updateWizardUI();
  renderEvalTypes();
  renderProviders();
  renderModels();
  renderDatasets();
  renderMetrics();
}

function updateWizardUI() {
  document.querySelectorAll('.step-indicator').forEach(el => {
    const step = parseInt(el.dataset.step);
    el.classList.remove('active', 'completed');
    if (step === AppState.wizardStep) el.classList.add('active');
    else if (step < AppState.wizardStep) el.classList.add('completed');
  });

  document.querySelectorAll('.step-pane').forEach(el => {
    el.classList.toggle('active', parseInt(el.dataset.step) === AppState.wizardStep);
  });

  lucide.createIcons();
}

function nextStep() {
  if (!validateStep(AppState.wizardStep)) return;
  if (AppState.wizardStep < 7) {
    AppState.wizardStep++;
    updateWizardUI();
    if (AppState.wizardStep === 7) updateReview();
  }
}

function prevStep() {
  if (AppState.wizardStep > 1) {
    AppState.wizardStep--;
    updateWizardUI();
  }
}

function validateStep(step) {
  switch(step) {
    case 1:
      const name = document.getElementById('eval-name').value.trim();
      if (!name) { showToast('Enter evaluation name', 'warning'); return false; }
      AppState.evaluation.name = name;
      return true;
    case 2:
      if (!AppState.evaluation.type) { showToast('Select evaluation type', 'warning'); return false; }
      return true;
    case 3:
      if (!AppState.evaluation.providers.length) { showToast('Select at least one provider', 'warning'); return false; }
      return true;
    case 4:
      if (!AppState.evaluation.models.length) { showToast('Select at least one model', 'warning'); return false; }
      return true;
    case 5:
      if (!AppState.evaluation.dataset) { showToast('Select a test suite', 'warning'); return false; }
      return true;
    case 6:
      if (!AppState.evaluation.metrics.length) { showToast('Select at least one metric', 'warning'); return false; }
      return true;
    default: return true;
  }
}

function setEvalName(name) {
  document.getElementById('eval-name').value = name;
  AppState.evaluation.name = name;
}

function renderEvalTypes() {
  document.getElementById('eval-type-grid').innerHTML = SEMCO_DATA.evalTypes.map(t => `
    <div class="eval-type-card ${AppState.evaluation.type === t.id ? 'selected' : ''}" onclick="selectEvalType('${t.id}')">
      <div class="eval-type-icon">
        <i data-lucide="${t.id === 'model' ? 'message-square' : t.id === 'agent' ? 'bot' : 'search'}"></i>
      </div>
      <div class="eval-type-content">
        <h3>${t.title}</h3>
        <p>${t.desc}</p>
      </div>
      <span class="badge badge-primary">${t.badge}</span>
      <div class="check-indicator"><i data-lucide="check"></i></div>
    </div>
  `).join('');
  lucide.createIcons();
}

function selectEvalType(id) {
  // Reset metrics when type changes so type-appropriate defaults are selected
  if (AppState.evaluation.type !== id) {
    AppState.evaluation.metrics = [];
  }
  AppState.evaluation.type = id;
  renderEvalTypes();
}

// FIX #1: Providers - Make unconnected cards clickable for inline connection
function renderProviders() {
  document.getElementById('provider-grid').innerHTML = SEMCO_DATA.providers.map(p => {
    const connected = p.status === 'connected';
    const selected = AppState.evaluation.providers.includes(p.id);
    // Show fetched count if available, otherwise general count
    const modelCount = p.fetchedModels || p.modelCount;
    return `
      <div class="provider-card ${selected ? 'selected' : ''} ${!connected ? 'not-connected' : ''}"
           onclick="${connected ? `toggleProvider('${p.id}')` : `connectProviderInline('${p.id}')`}">
        <div class="provider-logo">${p.logo}</div>
        <div class="provider-info">
          <h4>${p.name}</h4>
          <p>${connected ? `${modelCount} models available` : `${p.modelCount}+ models`}</p>
        </div>
        <div class="provider-status ${p.status}">
          ${connected
            ? '<i data-lucide="check-circle"></i> Connected'
            : '<i data-lucide="plus-circle"></i> Click to connect'}
        </div>
        <div class="check-indicator"><i data-lucide="check"></i></div>
      </div>
    `;
  }).join('');
  lucide.createIcons();
}

function toggleProvider(id) {
  const idx = AppState.evaluation.providers.indexOf(id);
  if (idx === -1) AppState.evaluation.providers.push(id);
  else AppState.evaluation.providers.splice(idx, 1);
  renderProviders();
  renderModels();
}

// Inline connection within wizard - doesn't break flow
function connectProviderInline(id) {
  const p = SEMCO_DATA.providers.find(x => x.id === id);
  showModal(`
    <div class="modal-header">
      <h3>Connect ${p.name}</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <p style="margin-bottom:16px;color:var(--text-secondary);">Enter your API key to connect ${p.name} and add it to this evaluation.</p>
      <div class="form-group">
        <label class="form-label">API Key</label>
        <input type="password" id="inline-api-key" class="form-control" placeholder="Enter API key">
        <p class="form-help">Find this in your ${p.name} dashboard.</p>
      </div>
      <button class="btn btn-primary btn-lg" style="width:100%" onclick="saveProviderInline('${id}')">Connect & Select</button>
    </div>
  `);
}

function saveProviderInline(id) {
  const apiKey = document.getElementById('inline-api-key')?.value;
  if (!apiKey) {
    showToast('Enter API key', 'warning');
    return;
  }

  const p = SEMCO_DATA.providers.find(x => x.id === id);

  // Show fetching state in modal
  document.querySelector('.modal-body').innerHTML = `
    <div class="fetching-models">
      <div class="fetch-spinner"></div>
      <h4>Connecting to ${p.name}...</h4>
      <p>Fetching available models and capabilities</p>
      <div class="fetch-steps">
        <div class="fetch-step active"><i data-lucide="check-circle"></i> Validating API key</div>
        <div class="fetch-step"><i data-lucide="loader"></i> Calling /v1/models</div>
        <div class="fetch-step"><i data-lucide="circle"></i> Detecting capabilities</div>
      </div>
    </div>
  `;
  lucide.createIcons();

  // Simulate API call with steps
  setTimeout(() => {
    document.querySelectorAll('.fetch-step')[1].classList.add('active');
    document.querySelectorAll('.fetch-step')[1].querySelector('i').setAttribute('data-lucide', 'check-circle');
    lucide.createIcons();
  }, 600);

  setTimeout(() => {
    document.querySelectorAll('.fetch-step')[2].classList.add('active');
    document.querySelectorAll('.fetch-step')[2].querySelector('i').setAttribute('data-lucide', 'check-circle');
    lucide.createIcons();
  }, 1000);

  // Complete connection
  setTimeout(() => {
    if (p) {
      p.status = 'connected';
      p.apiKey = 'sk-****-' + apiKey.slice(-4);
      // Simulate fetched model count
      p.fetchedModels = SEMCO_DATA.models.filter(m => m.providerId === id).length;
    }

    // Auto-select the newly connected provider
    if (!AppState.evaluation.providers.includes(id)) {
      AppState.evaluation.providers.push(id);
    }

    closeModal();
    showToast(`${p.name} connected · ${p.fetchedModels} models available`, 'success');
    renderProviders();
    renderModels();
  }, 1400);
}

function renderModels() {
  const selected = AppState.evaluation.providers;
  let models = SEMCO_DATA.models;
  if (selected.length) models = models.filter(m => selected.includes(m.providerId));

  // Show empty state if no providers selected
  if (selected.length === 0) {
    document.getElementById('models-grid').innerHTML = `
      <div class="models-empty-state">
        <i data-lucide="arrow-up"></i>
        <p>Select a provider above to see available models</p>
      </div>
    `;
    lucide.createIcons();
    updateSelectedCount();
    return;
  }

  document.getElementById('models-grid').innerHTML = `
    <div class="models-api-note">
      <i data-lucide="zap"></i>
      <span>Models and capabilities auto-detected from provider API</span>
    </div>
  ` + models.map(m => {
    const isSelected = AppState.evaluation.models.includes(m.id);
    const series = getModelSeriesTag(m);
    const isOpenSource = m.provider.toLowerCase().includes('together') || m.provider.toLowerCase().includes('ollama') || m.name.toLowerCase().includes('llama') || m.name.toLowerCase().includes('mistral') || m.name.toLowerCase().includes('qwen');
    return `
      <div class="model-card ${isSelected ? 'selected' : ''}"
           data-model-id="${m.id}"
           data-series="${series}"
           data-open-source="${isOpenSource}"
           onclick="toggleModel('${m.id}')">
        <div class="model-header">
          <h4>${m.name}</h4>
          <span class="provider-tag">${m.provider.split('(')[0].trim()}</span>
        </div>
        <p class="model-desc">${m.description.substring(0, 90)}...</p>
        <div class="model-capabilities">
          ${m.capabilities.slice(0, 3).map(c => `<span class="cap-tag">${c}</span>`).join('')}
          ${m.capabilities.length > 3 ? `<span class="cap-tag cap-more">+${m.capabilities.length - 3}</span>` : ''}
        </div>
        <div class="model-specs">
          <div class="spec"><span class="spec-label">max_tokens</span><span class="spec-value">${m.contextWindow}</span></div>
          <div class="spec"><span class="spec-label">$/1M tokens</span><span class="spec-value">${m.pricing}</span></div>
          <div class="spec"><span class="spec-label">latency</span><span class="spec-value">${m.speedRating.split('(')[0]}</span></div>
        </div>
        ${isSelected ? `<button class="configure-btn" onclick="event.stopPropagation(); showModelConfig('${m.id}')"><i data-lucide="settings"></i></button>` : ''}
        <div class="check-indicator"><i data-lucide="check"></i></div>
      </div>
    `;
  }).join('');

  updateSelectedCount();
  lucide.createIcons();
}

// Model capability configuration modal
function showModelConfig(modelId) {
  const m = SEMCO_DATA.models.find(x => x.id === modelId);
  if (!m) return;

  const hasToolCalling = m.capabilities.includes('Tool Calling');
  const hasVision = m.capabilities.includes('Vision');
  const hasReasoning = m.capabilities.includes('Reasoning') || m.capabilities.includes('Deep Reasoning');

  showModal(`
    <div class="modal-header">
      <h3>Configure ${m.name.split(' ')[0]}</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="api-detected-section">
        <div class="api-detected-header">
          <i data-lucide="radio"></i>
          <span>Auto-detected from /v1/models</span>
        </div>
        <div class="api-detected-grid">
          <div class="api-field">
            <span class="api-key">max_tokens</span>
            <span class="api-value">${m.contextWindow}</span>
          </div>
          <div class="api-field">
            <span class="api-key">supports_vision</span>
            <span class="api-value">${hasVision ? 'true' : 'false'}</span>
          </div>
          <div class="api-field">
            <span class="api-key">supports_tools</span>
            <span class="api-value">${hasToolCalling ? 'true' : 'false'}</span>
          </div>
          <div class="api-field">
            <span class="api-key">supports_reasoning</span>
            <span class="api-value">${hasReasoning ? 'true' : 'false'}</span>
          </div>
        </div>
      </div>

      <div class="config-section">
        <h4>Override Defaults (Optional)</h4>

        ${hasToolCalling ? `
        <div class="config-row">
          <div class="config-info">
            <span class="config-label">Tool Execution</span>
            <span class="config-desc">Allow model to call functions and APIs</span>
          </div>
          <div class="config-options">
            <label class="config-option selected"><input type="radio" name="tool-mode" checked> Auto</label>
            <label class="config-option"><input type="radio" name="tool-mode"> Off</label>
            <label class="config-option"><input type="radio" name="tool-mode"> Strict</label>
          </div>
        </div>
        ` : ''}

        ${hasReasoning ? `
        <div class="config-row">
          <div class="config-info">
            <span class="config-label">Reasoning Mode</span>
            <span class="config-desc">Chain-of-thought depth</span>
          </div>
          <div class="config-options">
            <label class="config-option selected"><input type="radio" name="reasoning" checked> Default</label>
            <label class="config-option"><input type="radio" name="reasoning"> Extended</label>
          </div>
        </div>
        ` : ''}

        ${hasVision ? `
        <div class="config-row">
          <div class="config-info">
            <span class="config-label">Image Detail</span>
            <span class="config-desc">Resolution for vision tasks</span>
          </div>
          <div class="config-options">
            <label class="config-option selected"><input type="radio" name="vision" checked> Auto</label>
            <label class="config-option"><input type="radio" name="vision"> High</label>
          </div>
        </div>
        ` : ''}

        <div class="config-row">
          <div class="config-info">
            <span class="config-label">Output Format</span>
            <span class="config-desc">Response structure</span>
          </div>
          <div class="config-options">
            <label class="config-option selected"><input type="radio" name="format" checked> Auto</label>
            <label class="config-option"><input type="radio" name="format"> JSON</label>
          </div>
        </div>
      </div>

      <div class="config-footer">
        <span class="config-note"><i data-lucide="info"></i> Overrides apply to this evaluation only</span>
        <button class="btn btn-primary" onclick="closeModal(); showToast('Configuration saved', 'success');">Save</button>
      </div>
    </div>
  `);
}

function toggleModel(id) {
  const idx = AppState.evaluation.models.indexOf(id);
  if (idx === -1) AppState.evaluation.models.push(id);
  else AppState.evaluation.models.splice(idx, 1);
  renderModels();
}

// ========== COMPREHENSIVE MODEL FILTERS ==========
const ModelFilters = {
  search: '',
  modality: [],
  context: [],
  price: [],
  series: [],
  capability: [],
  speed: [],
  other: [],
  intelligence: 0,
  coding_index: 0,
  agentic: 0
};

function filterModels() {
  const q = document.getElementById('model-search')?.value.toLowerCase() || '';
  ModelFilters.search = q;
  applyFilters();
}

function toggleFilterSection(el) {
  el.parentElement.classList.toggle('collapsed');
}

function toggleFilterSidebar() {
  document.getElementById('filter-sidebar')?.classList.toggle('collapsed');
}

function resetAllFilters() {
  // Reset all checkboxes
  document.querySelectorAll('.filter-chip input').forEach(cb => cb.checked = false);
  // Reset sliders
  document.querySelectorAll('.filter-slider').forEach(s => {
    s.value = 0;
    s.nextElementSibling.textContent = '0%';
  });
  // Reset filter state
  Object.keys(ModelFilters).forEach(k => {
    ModelFilters[k] = Array.isArray(ModelFilters[k]) ? [] : (typeof ModelFilters[k] === 'number' ? 0 : '');
  });
  applyFilters();
  showToast('Filters cleared', 'success');
}

function applyFilters() {
  // Collect active filters from checkboxes
  ModelFilters.modality = getCheckedValues('modality');
  ModelFilters.context = getCheckedValues('context');
  ModelFilters.price = getCheckedValues('price');
  ModelFilters.series = getCheckedValues('series');
  ModelFilters.capability = getCheckedValues('capability');
  ModelFilters.speed = getCheckedValues('speed');
  ModelFilters.other = getCheckedValues('other');

  // Collect slider values
  document.querySelectorAll('.filter-slider').forEach(s => {
    const key = s.getAttribute('data-filter');
    ModelFilters[key] = parseInt(s.value);
    s.nextElementSibling.textContent = s.value + '%';
  });

  // Filter model cards
  document.querySelectorAll('.model-card').forEach(card => {
    const txt = card.textContent.toLowerCase();
    const modelId = card.getAttribute('data-model-id');
    const model = SEMCO_DATA.models.find(m => m.id === modelId);
    if (!model) return;

    let visible = true;

    // Search filter
    if (ModelFilters.search && !txt.includes(ModelFilters.search)) {
      visible = false;
    }

    // Series filter
    if (visible && ModelFilters.series.length > 0) {
      const modelSeries = getModelSeries(model);
      if (!ModelFilters.series.some(s => modelSeries.includes(s))) {
        visible = false;
      }
    }

    // Capability filter
    if (visible && ModelFilters.capability.length > 0) {
      const caps = model.capabilities.map(c => c.toLowerCase());
      if (!ModelFilters.capability.every(f => {
        if (f === 'tool_calling') return caps.some(c => c.includes('tool'));
        if (f === 'vision') return caps.includes('vision');
        if (f === 'reasoning') return caps.some(c => c.includes('reasoning'));
        if (f === 'coding') return caps.includes('coding');
        if (f === 'json_mode') return caps.some(c => c.includes('json'));
        return true;
      })) {
        visible = false;
      }
    }

    // Context length filter
    if (visible && ModelFilters.context.length > 0) {
      const ctx = parseContextLength(model.contextWindow);
      if (!ModelFilters.context.some(f => matchContextFilter(ctx, f))) {
        visible = false;
      }
    }

    // Price filter
    if (visible && ModelFilters.price.length > 0) {
      const price = parsePrice(model.pricing);
      if (!ModelFilters.price.some(f => matchPriceFilter(price, f))) {
        visible = false;
      }
    }

    // Speed filter
    if (visible && ModelFilters.speed.length > 0) {
      const speed = parseSpeed(model.speedRating);
      if (!ModelFilters.speed.some(f => matchSpeedFilter(speed, f))) {
        visible = false;
      }
    }

    // Other filters (open source, new, etc.)
    if (visible && ModelFilters.other.length > 0) {
      const isOpenSource = card.getAttribute('data-open-source') === 'true';
      if (ModelFilters.other.includes('open_source') && !isOpenSource) {
        visible = false;
      }
    }

    card.style.display = visible ? '' : 'none';
  });

  updateActiveFilterTags();
}

function getCheckedValues(filterName) {
  return Array.from(document.querySelectorAll(`[data-filter="${filterName}"]:checked`))
    .map(cb => cb.value);
}

function getModelSeries(model) {
  const name = model.name.toLowerCase();
  if (name.includes('gpt') || name.includes('o1')) return 'gpt';
  if (name.includes('claude')) return 'claude';
  if (name.includes('gemini')) return 'gemini';
  if (name.includes('llama') || name.includes('hermes')) return 'llama';
  if (name.includes('mistral')) return 'mistral';
  if (name.includes('deepseek')) return 'deepseek';
  if (name.includes('qwen')) return 'qwen';
  return 'other';
}

function getModelSeriesTag(model) {
  return getModelSeries(model);
}

function parseContextLength(ctx) {
  const match = ctx.match(/(\d+)([km])/i);
  if (!match) return 0;
  const num = parseInt(match[1]);
  const unit = match[2].toLowerCase();
  return unit === 'm' ? num * 1000 : num;
}

function matchContextFilter(ctx, filter) {
  if (filter === '4k') return ctx <= 8;
  if (filter === '32k') return ctx > 8 && ctx <= 64;
  if (filter === '128k') return ctx > 64 && ctx <= 150;
  if (filter === '200k') return ctx > 150 && ctx < 1000;
  if (filter === '1m') return ctx >= 1000;
  return true;
}

function parsePrice(pricing) {
  const match = pricing.match(/\$?([\d.]+)/);
  return match ? parseFloat(match[1]) : 0;
}

function matchPriceFilter(price, filter) {
  if (filter === 'free') return price === 0;
  if (filter === 'low') return price > 0 && price < 1;
  if (filter === 'mid') return price >= 1 && price <= 5;
  if (filter === 'high') return price > 5;
  return true;
}

function parseSpeed(speedRating) {
  const match = speedRating.match(/(\d+)/);
  return match ? parseInt(match[1]) : 0;
}

function matchSpeedFilter(speed, filter) {
  if (filter === 'ultra') return speed >= 80;
  if (filter === 'fast') return speed >= 40 && speed < 80;
  if (filter === 'medium') return speed >= 20 && speed < 40;
  return true;
}

function updateActiveFilterTags() {
  const container = document.getElementById('active-filters');
  if (!container) return;

  const tags = [];

  // Add tags for each active filter
  ModelFilters.series.forEach(s => tags.push({ label: s.toUpperCase(), filter: 'series', value: s }));
  ModelFilters.capability.forEach(c => tags.push({ label: c.replace('_', ' '), filter: 'capability', value: c }));
  ModelFilters.context.forEach(c => tags.push({ label: c.toUpperCase(), filter: 'context', value: c }));
  ModelFilters.price.forEach(p => tags.push({ label: p === 'free' ? 'FREE' : p, filter: 'price', value: p }));

  container.innerHTML = tags.slice(0, 5).map(t => `
    <span class="active-filter-tag">
      ${t.label}
      <button onclick="removeFilter('${t.filter}', '${t.value}')"><i data-lucide="x"></i></button>
    </span>
  `).join('') + (tags.length > 5 ? `<span class="active-filter-tag">+${tags.length - 5} more</span>` : '');

  lucide.createIcons();
}

function removeFilter(filterType, value) {
  const cb = document.querySelector(`[data-filter="${filterType}"][value="${value}"]`);
  if (cb) {
    cb.checked = false;
    applyFilters();
  }
}

function clearModelSelection() {
  AppState.evaluation.models = [];
  renderModels();
}

function updateSelectedCount() {
  const n = AppState.evaluation.models.length;
  document.getElementById('selected-count').textContent = n;
  document.getElementById('selected-models-bar').style.display = n ? 'flex' : 'none';
}

// Get required capabilities for a dataset
function getDatasetRequirements(dataset) {
  const cat = dataset.category;
  if (cat === 'Agents') return ['Tool Calling'];
  if (cat === 'Coding') return ['Coding'];
  if (cat === 'RAG') return ['Document Analysis', 'Long Context'];
  return []; // General/universal
}

// Check if selected models support dataset requirements
function checkCapabilityMatch(dataset) {
  const required = getDatasetRequirements(dataset);
  if (required.length === 0) return { compatible: true, missing: [] };

  const selectedModels = AppState.evaluation.models.map(id => SEMCO_DATA.models.find(m => m.id === id)).filter(Boolean);
  if (selectedModels.length === 0) return { compatible: true, missing: [] };

  const incompatibleModels = selectedModels.filter(m => {
    return !required.some(req => m.capabilities.some(cap => cap.toLowerCase().includes(req.toLowerCase())));
  });

  return {
    compatible: incompatibleModels.length === 0,
    missing: incompatibleModels,
    required: required
  };
}

// Get capability tag for display
function getCapabilityTag(dataset) {
  const cat = dataset.category;
  if (cat === 'Agents') return { label: 'Requires: Tool Calling', color: 'agent' };
  if (cat === 'Coding') return { label: 'Requires: Coding', color: 'coding' };
  if (cat === 'RAG') return { label: 'Requires: RAG', color: 'rag' };
  if (cat === 'Finance' || cat === 'Healthcare') return { label: 'Requires: Domain Knowledge', color: 'domain' };
  return { label: 'Universal', color: 'universal' };
}

function renderDatasets() {
  // Get selected models capabilities for smart sorting
  const selectedModels = AppState.evaluation.models.map(id => SEMCO_DATA.models.find(m => m.id === id)).filter(Boolean);
  const selectedCapabilities = selectedModels.flatMap(m => m.capabilities);
  const hasToolCalling = selectedCapabilities.some(c => c.includes('Tool'));
  const hasCoding = selectedCapabilities.some(c => c.includes('Coding'));
  const hasRAG = selectedCapabilities.some(c => c.includes('Document') || c.includes('Long Context'));

  // Sort datasets - recommended first based on selected model capabilities
  let sortedSuites = [...SEMCO_DATA.testSuites].sort((a, b) => {
    const aMatch = checkCapabilityMatch(a);
    const bMatch = checkCapabilityMatch(b);
    if (aMatch.compatible && !bMatch.compatible) return -1;
    if (!aMatch.compatible && bMatch.compatible) return 1;
    if (a.featured && !b.featured) return -1;
    if (!a.featured && b.featured) return 1;
    return 0;
  });

  document.getElementById('datasets-grid').innerHTML = sortedSuites.map(d => {
    const selected = AppState.evaluation.dataset === d.id;
    const capTag = getCapabilityTag(d);
    const match = checkCapabilityMatch(d);

    return `
      <div class="dataset-card ${selected ? 'selected' : ''} ${!match.compatible ? 'incompatible' : ''}"
           onclick="${match.compatible ? `selectDataset('${d.id}')` : `showIncompatibilityWarning('${d.id}')`}">
        ${d.featured && match.compatible ? '<span class="badge badge-success">Recommended</span>' : ''}
        ${!match.compatible ? '<span class="badge badge-warning">Mismatch</span>' : ''}
        <div class="dataset-header">
          <span class="dataset-category">${d.category}</span>
          <h4>${d.name}</h4>
        </div>
        <p class="dataset-desc">${d.description}</p>
        <div class="capability-requirement cap-${capTag.color}">
          <i data-lucide="${capTag.color === 'universal' ? 'check-circle' : 'zap'}"></i>
          ${capTag.label}
        </div>
        <div class="dataset-meta">
          <div class="meta-item"><i data-lucide="help-circle"></i>${d.questions} questions</div>
          <div class="meta-item"><i data-lucide="globe"></i>${d.language}</div>
          <div class="meta-item"><i data-lucide="bar-chart-2"></i>${d.difficulty}</div>
        </div>
        <div class="check-indicator"><i data-lucide="check"></i></div>
      </div>
    `;
  }).join('');
  lucide.createIcons();
}

function showIncompatibilityWarning(datasetId) {
  const dataset = SEMCO_DATA.testSuites.find(d => d.id === datasetId);
  if (!dataset) return;

  const match = checkCapabilityMatch(dataset);
  const capTag = getCapabilityTag(dataset);

  // Find compatible models to recommend
  const compatibleModels = SEMCO_DATA.models.filter(m => {
    return match.required.some(req => m.capabilities.some(cap => cap.toLowerCase().includes(req.toLowerCase())));
  }).slice(0, 3);

  showModal(`
    <div class="modal-header warning-header">
      <h3><i data-lucide="alert-triangle"></i> Capability Mismatch</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="warning-message">
        <p><strong>"${dataset.name}"</strong> requires <span class="cap-highlight">${match.required.join(', ')}</span>, but ${match.missing.length} selected model${match.missing.length > 1 ? 's do' : ' does'} not support this capability.</p>
      </div>

      <div class="incompatible-models">
        <h4>Incompatible Models:</h4>
        <ul>
          ${match.missing.map(m => `<li><i data-lucide="x-circle"></i> ${m.name}</li>`).join('')}
        </ul>
      </div>

      <div class="compatible-suggestion">
        <h4>Recommended Models for This Test:</h4>
        <div class="suggestion-models">
          ${compatibleModels.map(m => `
            <div class="suggestion-model" onclick="addSuggestedModel('${m.id}', '${datasetId}')">
              <div class="suggestion-model-info">
                <strong>${m.name}</strong>
                <span>${m.provider.split('(')[0].trim()}</span>
              </div>
              <div class="suggestion-caps">
                ${m.capabilities.slice(0, 2).map(c => `<span class="cap-tag">${c}</span>`).join('')}
              </div>
              <button class="btn btn-sm btn-primary">Add</button>
            </div>
          `).join('')}
        </div>
      </div>

      <div class="warning-actions">
        <button class="btn btn-secondary" onclick="closeModal()">Cancel</button>
        <button class="btn btn-outline" onclick="forceSelectDataset('${datasetId}')">Use Anyway (May Fail)</button>
      </div>
    </div>
  `);
}

function addSuggestedModel(modelId, datasetId) {
  if (!AppState.evaluation.models.includes(modelId)) {
    AppState.evaluation.models.push(modelId);
  }
  closeModal();
  showToast('Model added', 'success');
  renderModels();

  // Now select the dataset since we have a compatible model
  setTimeout(() => {
    selectDataset(datasetId);
  }, 300);
}

function forceSelectDataset(datasetId) {
  closeModal();
  AppState.evaluation.dataset = datasetId;
  renderDatasets();
  renderCommunityDatasets();
  showToast('Dataset selected (compatibility warning ignored)', 'warning');
}

function selectDataset(id) {
  AppState.evaluation.dataset = id;
  renderDatasets();
  // Also update community if selected from there
  renderCommunityDatasets();
}

function switchDatasetTab(tab) {
  document.querySelectorAll('.dataset-tab').forEach(t => t.classList.remove('active'));
  event.target.classList.add('active');
  document.querySelectorAll('.dataset-content').forEach(c => c.style.display = 'none');
  document.getElementById(`dataset-${tab}`).style.display = 'block';

  if (tab === 'community') {
    renderCommunityDatasets();
  }
}

function filterDatasets(cat) {
  document.querySelectorAll('.category-chip').forEach(c => c.classList.remove('active'));
  event.target.classList.add('active');
  document.querySelectorAll('.dataset-card').forEach(card => {
    card.style.display = cat === 'all' || card.querySelector('.dataset-category').textContent === cat ? '' : 'none';
  });
}

// FIX #4: Community Benchmarks - Load real community suites
function renderCommunityDatasets() {
  const container = document.getElementById('dataset-community');
  container.innerHTML = `
    <div class="datasets-grid">
      ${COMMUNITY_DATASETS.map(d => {
        const selected = AppState.evaluation.dataset === d.id;
        return `
          <div class="dataset-card ${selected ? 'selected' : ''}" onclick="selectDataset('${d.id}')">
            ${d.featured ? '<span class="badge badge-primary">Popular</span>' : ''}
            <div class="dataset-header">
              <span class="dataset-category">${d.category}</span>
              <h4>${d.name}</h4>
            </div>
            <p class="dataset-desc">${d.description}</p>
            <div class="dataset-meta">
              <div class="meta-item"><i data-lucide="help-circle"></i>${d.questions} questions</div>
              <div class="meta-item"><i data-lucide="globe"></i>${d.language}</div>
              <div class="meta-item"><i data-lucide="bar-chart-2"></i>${d.difficulty}</div>
            </div>
            <div class="check-indicator"><i data-lucide="check"></i></div>
          </div>
        `;
      }).join('')}
    </div>
  `;
  lucide.createIcons();
}

function loadCommunitySuites() {
  renderCommunityDatasets();
}

// FIX: File upload with column mapping preview
function handleFileUpload(input) {
  const file = input.files[0];
  if (!file) return;

  AppState.uploadedFile = file;
  showColumnMappingModal(file);
}

function showColumnMappingModal(file) {
  const sampleRows = SEMCO_DATA.sampleImportRows;
  const columns = Object.keys(sampleRows[0]);

  showModal(`
    <div class="modal-header">
      <h3>Map Your Columns</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <p style="margin-bottom:16px;color:var(--text-secondary);">
        File: <strong>${file.name}</strong> — Map your columns to the required fields.
      </p>

      <div class="column-mapping-grid">
        <div class="mapping-row">
          <div class="mapping-label">Question / Input</div>
          <select class="form-control mapping-select">
            <option value="col1" selected>Column 1 (Question)</option>
            <option value="col2">Column 2 (Expected Answer)</option>
            <option value="col3">Column 3 (Source)</option>
          </select>
        </div>
        <div class="mapping-row">
          <div class="mapping-label">Expected Answer</div>
          <select class="form-control mapping-select">
            <option value="col1">Column 1 (Question)</option>
            <option value="col2" selected>Column 2 (Expected Answer)</option>
            <option value="col3">Column 3 (Source)</option>
          </select>
        </div>
        <div class="mapping-row">
          <div class="mapping-label">Source / Context (Optional)</div>
          <select class="form-control mapping-select">
            <option value="">— None —</option>
            <option value="col1">Column 1 (Question)</option>
            <option value="col2">Column 2 (Expected Answer)</option>
            <option value="col3" selected>Column 3 (Source)</option>
          </select>
        </div>
      </div>

      <div class="preview-section">
        <h4>Preview (first 3 rows)</h4>
        <div class="preview-table-wrap">
          <table class="preview-table">
            <thead>
              <tr>
                <th>Question</th>
                <th>Expected Answer</th>
                <th>Source</th>
              </tr>
            </thead>
            <tbody>
              ${sampleRows.map(row => `
                <tr>
                  <td>${row.col1}</td>
                  <td><code>${row.col2}</code></td>
                  <td>${row.col3}</td>
                </tr>
              `).join('')}
            </tbody>
          </table>
        </div>
      </div>

      <div class="mapping-actions">
        <button class="btn btn-secondary" onclick="closeModal()">Cancel</button>
        <button class="btn btn-primary" onclick="confirmColumnMapping()">
          <i data-lucide="check"></i> Use This Dataset
        </button>
      </div>
    </div>
  `);
}

// Inline HuggingFace import from wizard
function importHuggingFaceInline() {
  const datasetId = document.getElementById('wizard-hf-id')?.value;
  if (!datasetId) {
    showToast('Enter dataset ID', 'warning');
    return;
  }

  showToast('Importing from Hugging Face...', 'info');

  setTimeout(() => {
    const customDataset = {
      id: 'hf-' + Date.now(),
      name: datasetId.split('/')[1] || datasetId,
      category: 'External',
      questions: 500,
      language: 'English',
      difficulty: 'Varied',
      version: 'HF Import',
      maintainer: datasetId.split('/')[0] || 'Hugging Face',
      description: `Imported from Hugging Face: ${datasetId}`,
      featured: false
    };

    SEMCO_DATA.testSuites.unshift(customDataset);
    AppState.evaluation.dataset = customDataset.id;

    showToast('Dataset imported successfully', 'success');

    // Switch to official tab to show it
    document.querySelectorAll('.dataset-tab').forEach(t => t.classList.remove('active'));
    document.querySelector('.dataset-tab').classList.add('active');
    document.querySelectorAll('.dataset-content').forEach(c => c.style.display = 'none');
    document.getElementById('dataset-official').style.display = 'block';

    renderDatasets();
  }, 800);
}

function confirmColumnMapping() {
  // Create custom dataset entry
  const customDataset = {
    id: 'custom-upload-' + Date.now(),
    name: AppState.uploadedFile?.name || 'Custom Dataset',
    category: 'Custom',
    questions: 3, // Demo
    language: 'English',
    difficulty: 'Custom',
    version: 'Uploaded',
    maintainer: 'You',
    description: 'Custom uploaded test suite with your own questions and expected answers.',
    featured: false
  };

  // Add to test suites temporarily
  SEMCO_DATA.testSuites.unshift(customDataset);

  // Select it
  AppState.evaluation.dataset = customDataset.id;

  closeModal();
  showToast('Dataset imported successfully', 'success');

  // Switch to official tab to show it
  document.querySelectorAll('.dataset-tab').forEach(t => t.classList.remove('active'));
  document.querySelector('.dataset-tab').classList.add('active');
  document.querySelectorAll('.dataset-content').forEach(c => c.style.display = 'none');
  document.getElementById('dataset-official').style.display = 'block';

  renderDatasets();
}

function renderMetrics() {
  const evalType = AppState.evaluation.type || 'model';
  const metrics = SEMCO_DATA.metrics;

  // Get universal metrics + type-specific metrics
  const universalMetrics = metrics.universal || [];
  const typeMetrics = metrics[evalType] || metrics.model || [];

  // Build the metrics grid with sections
  let html = `
    <div class="metrics-section">
      <h4 class="metrics-section-title"><i data-lucide="gauge"></i> Core Metrics</h4>
      <p class="metrics-section-desc">Essential measurements for any evaluation</p>
      <div class="metrics-cards">
        ${universalMetrics.map(m => renderMetricCard(m)).join('')}
      </div>
    </div>

    <div class="metrics-section">
      <h4 class="metrics-section-title"><i data-lucide="${getTypeIcon(evalType)}"></i> ${getTypeName(evalType)} Metrics</h4>
      <p class="metrics-section-desc">${getTypeDescription(evalType)}</p>
      <div class="metrics-cards">
        ${typeMetrics.map(m => renderMetricCard(m)).join('')}
      </div>
    </div>
  `;

  document.getElementById('metrics-grid').innerHTML = html;
  updateMetricsCount();
  lucide.createIcons();
}

function updateMetricsCount() {
  const countEl = document.getElementById('metrics-count');
  if (countEl) {
    countEl.textContent = AppState.evaluation.metrics.length;
  }
}

function renderMetricCard(m) {
  const checked = m.defaultChecked || AppState.evaluation.metrics.includes(m.id);
  if (m.defaultChecked && !AppState.evaluation.metrics.includes(m.id)) {
    AppState.evaluation.metrics.push(m.id);
  }
  return `
    <div class="metric-card ${checked ? 'selected' : ''}" onclick="toggleMetric('${m.id}')">
      <div class="metric-icon"><i data-lucide="${m.icon || 'activity'}"></i></div>
      <div class="metric-content">
        <h4>${m.name}</h4>
        <p>${m.tooltip}</p>
      </div>
      <div class="metric-checkbox"><input type="checkbox" ${checked ? 'checked' : ''}></div>
    </div>
  `;
}

function getTypeIcon(type) {
  if (type === 'agent') return 'bot';
  if (type === 'rag') return 'database';
  return 'message-square';
}

function getTypeName(type) {
  if (type === 'agent') return 'Agent & Automation';
  if (type === 'rag') return 'RAG & Retrieval';
  return 'Chat & Reasoning';
}

function getTypeDescription(type) {
  if (type === 'agent') return 'Metrics for autonomous workflows, tool calling, and multi-step task execution';
  if (type === 'rag') return 'Metrics for document retrieval accuracy, faithfulness, and hallucination detection';
  return 'Metrics for conversational quality, reasoning, and instruction following';
}

function toggleMetric(id) {
  const idx = AppState.evaluation.metrics.indexOf(id);
  if (idx === -1) AppState.evaluation.metrics.push(id);
  else AppState.evaluation.metrics.splice(idx, 1);
  renderMetrics();
  updateMetricsCount();
}

function toggleAdvancedMetrics() {
  showModal(`
    <div class="modal-header">
      <h3>Advanced Metric Settings</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="advanced-metric-section">
        <h4>Scoring Weights</h4>
        <p class="form-help">Adjust how much each metric contributes to the overall score.</p>
        <div class="weight-sliders">
          ${AppState.evaluation.metrics.map(id => {
            const allMetrics = [...(SEMCO_DATA.metrics.universal || []), ...(SEMCO_DATA.metrics[AppState.evaluation.type] || SEMCO_DATA.metrics.model || [])];
            const m = allMetrics.find(x => x.id === id);
            if (!m) return '';
            return `
              <div class="weight-row">
                <span>${m.name}</span>
                <input type="range" min="0" max="100" value="50">
                <span class="weight-val">50%</span>
              </div>
            `;
          }).join('')}
        </div>
      </div>

      <div class="advanced-metric-section">
        <h4>Thresholds</h4>
        <p class="form-help">Set minimum acceptable scores for pass/fail reporting.</p>
        <div class="form-group">
          <label class="form-label">Minimum Accuracy</label>
          <input type="number" class="form-control" value="80" style="width:100px;"> %
        </div>
        <div class="form-group">
          <label class="form-label">Maximum Latency</label>
          <input type="number" class="form-control" value="5" style="width:100px;"> seconds
        </div>
      </div>

      <div class="modal-actions">
        <button class="btn btn-secondary" onclick="closeModal()">Cancel</button>
        <button class="btn btn-primary" onclick="closeModal(); showToast('Settings saved', 'success');">Save</button>
      </div>
    </div>
  `);
}

function updateReview() {
  document.getElementById('review-name').textContent = AppState.evaluation.name;
  const type = SEMCO_DATA.evalTypes.find(t => t.id === AppState.evaluation.type);
  document.getElementById('review-type').textContent = type?.title || '-';
  const models = AppState.evaluation.models.map(id => SEMCO_DATA.models.find(m => m.id === id)?.name).filter(Boolean);
  document.getElementById('review-models').textContent = models.join(', ') || '-';

  // Check both official and community datasets
  let ds = SEMCO_DATA.testSuites.find(d => d.id === AppState.evaluation.dataset);
  if (!ds) ds = COMMUNITY_DATASETS.find(d => d.id === AppState.evaluation.dataset);

  document.getElementById('review-dataset').textContent = ds?.name || '-';
  document.getElementById('review-questions').textContent = ds ? `${ds.questions} questions` : '-';
  document.getElementById('review-cost').textContent = `~$${(AppState.evaluation.models.length * 0.12).toFixed(2)}`;
  document.getElementById('review-duration').textContent = `~${Math.max(2, AppState.evaluation.models.length * 1.5).toFixed(0)} min`;
}

// ============ RUNNING ============
function startEvaluation() {
  navigateTo('running');
  const models = AppState.evaluation.models.map(id => SEMCO_DATA.models.find(m => m.id === id));

  document.getElementById('progress-models-list').innerHTML = models.map((m, i) => `
    <div class="progress-model-item ${i === 0 ? 'active' : ''}" id="pm-${i}">
      <div class="model-status">${i === 0 ? '<i data-lucide="loader-2" class="spin-icon"></i>' : '<i data-lucide="circle"></i>'}</div>
      <span class="model-name">${m.name}</span>
      <span class="model-progress-status">${i === 0 ? 'Running...' : 'Waiting'}</span>
    </div>
  `).join('');

  lucide.createIcons();
  runSimulation(models);
}

function runSimulation(models) {
  let progress = 0, current = 0, elapsed = 0;
  const total = models.length;
  const fill = document.getElementById('progress-fill');
  const pct = document.getElementById('progress-percent');
  const time = document.getElementById('elapsed-time');
  const done = document.getElementById('completed-count');
  const logs = document.getElementById('eval-logs');

  const iv = setInterval(() => {
    progress += Math.random() * 3 + 1;
    elapsed++;

    if (progress >= 100) {
      progress = 100;
      clearInterval(iv);
      setTimeout(showResults, 400);
    }

    fill.style.width = `${progress}%`;
    pct.textContent = `${Math.round(progress)}%`;
    const m = Math.floor(elapsed / 60), s = elapsed % 60;
    time.textContent = `${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;

    const newIdx = Math.min(Math.floor(progress / 100 * total), total - 1);
    if (newIdx !== current) {
      const prev = document.getElementById(`pm-${current}`);
      if (prev) {
        prev.classList.remove('active');
        prev.classList.add('completed');
        prev.querySelector('.model-status').innerHTML = '<i data-lucide="check-circle"></i>';
        prev.querySelector('.model-progress-status').textContent = 'Done';
      }
      const next = document.getElementById(`pm-${newIdx}`);
      if (next) {
        next.classList.add('active');
        next.querySelector('.model-status').innerHTML = '<i data-lucide="loader-2" class="spin-icon"></i>';
        next.querySelector('.model-progress-status').textContent = 'Running...';
      }
      current = newIdx;
      document.getElementById('current-model').textContent = models[current]?.name || models[0].name;
      logs.textContent += `\n[${new Date().toLocaleTimeString()}] Testing ${models[current]?.name}...`;
    }

    done.textContent = `${Math.min(current + 1, total)}/${total}`;
    lucide.createIcons();
  }, 300);
}

function showResults() {
  navigateTo('results');
  AppState.compareModels = [];
  const res = SEMCO_DATA.recentEvaluations[0].results;

  document.getElementById('results-subtitle').textContent = `${AppState.evaluation.name || 'Evaluation'} — just now`;
  renderResultsTable(res);

  document.getElementById('winner-model').textContent = res[0].model;
  document.getElementById('winner-score').textContent = res[0].score;
  lucide.createIcons();
}

// FIX #3: Add compare functionality to results
function renderResultsTable(results) {
  document.getElementById('results-tbody').innerHTML = results.map(r => `
    <tr>
      <td><div class="rank-badge rank-${r.rank}">${r.rank}</div></td>
      <td>
        <label class="compare-checkbox">
          <input type="checkbox" onchange="toggleCompare('${r.model}', this.checked)">
          <strong>${r.model}</strong>
        </label>
      </td>
      <td>${r.provider}</td>
      <td><span class="score-highlight">${r.score}</span></td>
      <td>${r.accuracy}</td>
      <td>${r.time}</td>
      <td>${r.cost}</td>
      <td>${r.incorrectRate || '-'}</td>
      <td>${r.toolSuccess || '-'}</td>
      <td><span class="status-badge ${r.status.toLowerCase()}">${r.status}</span></td>
    </tr>
  `).join('');

  updateCompareBar();
}

function toggleCompare(modelName, checked) {
  if (checked) {
    if (!AppState.compareModels.includes(modelName)) {
      AppState.compareModels.push(modelName);
    }
  } else {
    AppState.compareModels = AppState.compareModels.filter(m => m !== modelName);
  }
  updateCompareBar();
}

function updateCompareBar() {
  let bar = document.getElementById('compare-bar');

  if (AppState.compareModels.length >= 2) {
    if (!bar) {
      bar = document.createElement('div');
      bar.id = 'compare-bar';
      bar.className = 'compare-floating-bar';
      document.body.appendChild(bar);
    }
    bar.innerHTML = `
      <span><strong>${AppState.compareModels.length}</strong> models selected</span>
      <button class="btn btn-primary" onclick="showCompareModal()">
        <i data-lucide="columns"></i> Compare Side-by-Side
      </button>
    `;
    bar.style.display = 'flex';
    lucide.createIcons();
  } else if (bar) {
    bar.style.display = 'none';
  }
}

function showCompareModal() {
  const res = SEMCO_DATA.recentEvaluations[0].results;
  const selected = res.filter(r => AppState.compareModels.includes(r.model));

  if (selected.length < 2) {
    showToast('Select at least 2 models to compare', 'warning');
    return;
  }

  const m1 = selected[0];
  const m2 = selected[1];

  // Calculate comparisons
  const speedDiff = (parseFloat(m2.time) / parseFloat(m1.time)).toFixed(1);
  const costDiff = (parseFloat(m2.cost.replace('$', '')) - parseFloat(m1.cost.replace('$', ''))).toFixed(2);
  const accuracyDiff = (parseFloat(m1.accuracy) - parseFloat(m2.accuracy)).toFixed(1);

  showModal(`
    <div class="modal-header">
      <h3>Model Comparison</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="compare-header">
        <div class="compare-model">
          <div class="compare-rank rank-${m1.rank}">#${m1.rank}</div>
          <h4>${m1.model}</h4>
          <span>${m1.provider}</span>
        </div>
        <div class="compare-vs">VS</div>
        <div class="compare-model">
          <div class="compare-rank rank-${m2.rank}">#${m2.rank}</div>
          <h4>${m2.model}</h4>
          <span>${m2.provider}</span>
        </div>
      </div>

      <div class="compare-metrics">
        <div class="compare-row">
          <span class="metric-name">Overall Score</span>
          <span class="metric-value winner">${m1.score}</span>
          <span class="metric-value">${m2.score}</span>
        </div>
        <div class="compare-row">
          <span class="metric-name">Accuracy</span>
          <span class="metric-value ${parseFloat(m1.accuracy) >= parseFloat(m2.accuracy) ? 'winner' : ''}">${m1.accuracy}</span>
          <span class="metric-value ${parseFloat(m2.accuracy) > parseFloat(m1.accuracy) ? 'winner' : ''}">${m2.accuracy}</span>
        </div>
        <div class="compare-row">
          <span class="metric-name">Response Time</span>
          <span class="metric-value ${parseFloat(m1.time) <= parseFloat(m2.time) ? 'winner' : ''}">${m1.time}</span>
          <span class="metric-value ${parseFloat(m2.time) < parseFloat(m1.time) ? 'winner' : ''}">${m2.time}</span>
        </div>
        <div class="compare-row">
          <span class="metric-name">Cost</span>
          <span class="metric-value ${parseFloat(m1.cost.replace('$','')) <= parseFloat(m2.cost.replace('$','')) ? 'winner' : ''}">${m1.cost}</span>
          <span class="metric-value ${parseFloat(m2.cost.replace('$','')) < parseFloat(m1.cost.replace('$','')) ? 'winner' : ''}">${m2.cost}</span>
        </div>
        <div class="compare-row">
          <span class="metric-name">Tool Success</span>
          <span class="metric-value">${m1.toolSuccess || '-'}</span>
          <span class="metric-value">${m2.toolSuccess || '-'}</span>
        </div>
        <div class="compare-row">
          <span class="metric-name">Error Rate</span>
          <span class="metric-value">${m1.incorrectRate || '-'}</span>
          <span class="metric-value">${m2.incorrectRate || '-'}</span>
        </div>
      </div>

      <div class="compare-summary">
        <h4>Trade-off Summary</h4>
        <ul>
          <li><strong>Speed:</strong> ${m1.model.split(' ')[0]} is ${speedDiff}x faster</li>
          <li><strong>Cost:</strong> ${parseFloat(costDiff) > 0 ? m1.model.split(' ')[0] + ' saves $' + Math.abs(costDiff) : m2.model.split(' ')[0] + ' saves $' + Math.abs(costDiff)}</li>
          <li><strong>Accuracy:</strong> ${parseFloat(accuracyDiff) > 0 ? m1.model.split(' ')[0] + ' is ' + accuracyDiff + '% more accurate' : 'Nearly identical'}</li>
        </ul>
      </div>
    </div>
  `);
}

function runAgain() { navigateTo('run-evaluation'); }

function exportResults(fmt) {
  showToast(`Exporting ${fmt.toUpperCase()}...`, 'info');
  setTimeout(() => showToast('Export complete', 'success'), 800);
}

function shareResults() {
  showModal(`
    <div class="modal-header">
      <h3>Share Report</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">Link</label>
        <div class="share-link-box">
          <input type="text" class="form-control" value="https://semcoeval.internal/r/eval-9041" readonly>
          <button class="btn btn-primary" onclick="copyLink()">Copy</button>
        </div>
      </div>
      <div class="form-group">
        <label class="form-label">Email</label>
        <input type="email" class="form-control" placeholder="colleague@semcolabs.com">
      </div>
      <button class="btn btn-primary btn-lg" style="width:100%;margin-top:16px;">Send</button>
    </div>
  `);
}

// ============ HISTORY ============
function renderHistory() {
  document.getElementById('history-list').innerHTML = SEMCO_DATA.recentEvaluations.map(ev => `
    <div class="history-item" onclick="viewEvaluationResult('${ev.id}')">
      <div class="history-icon">
        <i data-lucide="${ev.type.includes('Agent') ? 'bot' : ev.type.includes('RAG') ? 'search' : 'message-square'}"></i>
      </div>
      <div class="history-content">
        <h4>${ev.name}</h4>
        <div class="history-meta">
          <span class="history-type">${ev.type.split('(')[0].trim()}</span>
          <span>${ev.date}</span>
        </div>
      </div>
      <div class="history-results">
        <div class="history-stat"><span class="stat-label">Winner</span><span class="stat-value">${ev.topModel.split(' - ')[0]}</span></div>
        <div class="history-stat"><span class="stat-label">Score</span><span class="stat-value highlight">${ev.topScore}</span></div>
        <div class="history-stat"><span class="stat-label">Models</span><span class="stat-value">${ev.modelsTested}</span></div>
      </div>
      <div class="history-actions">
        <button class="btn btn-sm btn-secondary" onclick="event.stopPropagation(); duplicateEval('${ev.id}')"><i data-lucide="copy"></i></button>
        <button class="btn btn-sm btn-secondary" onclick="event.stopPropagation(); deleteEval('${ev.id}')"><i data-lucide="trash-2"></i></button>
      </div>
    </div>
  `).join('');
  lucide.createIcons();
}

function viewEvaluationResult(id) {
  const ev = SEMCO_DATA.recentEvaluations.find(e => e.id === id);
  if (!ev) return;
  AppState.evaluation.name = ev.name;
  AppState.compareModels = [];
  navigateTo('results');
  document.getElementById('results-subtitle').textContent = `${ev.name} — ${ev.date}`;
  renderResultsTable(ev.results);
  document.getElementById('winner-model').textContent = ev.results[0].model;
  document.getElementById('winner-score').textContent = ev.results[0].score;
  lucide.createIcons();
}

function duplicateEval(id) {
  showToast('Duplicated', 'success');
  setTimeout(() => navigateTo('run-evaluation'), 300);
}

function deleteEval(id) {
  if (confirm('Delete this evaluation?')) {
    showToast('Deleted', 'success');
    renderHistory();
  }
}

// ============ MODELS CATALOG ============
function renderModelsCatalog() {
  AppState.compareModels = [];
  document.getElementById('models-catalog-grid').innerHTML = SEMCO_DATA.models.map(m => `
    <div class="model-catalog-card">
      <div class="model-catalog-header">
        <div class="model-provider-badge">${m.provider.split('(')[0].trim()}</div>
        <span class="model-version">${m.version}</span>
      </div>
      <h3>${m.name}</h3>
      <p>${m.description}</p>
      <div class="model-capabilities-list">
        ${m.capabilities.map(c => `<span class="cap-pill">${c}</span>`).join('')}
      </div>
      <div class="model-specs-grid">
        <div class="spec-item"><span class="spec-label">Context</span><span class="spec-value">${m.contextWindow}</span></div>
        <div class="spec-item"><span class="spec-label">Price</span><span class="spec-value">${m.pricing}</span></div>
        <div class="spec-item"><span class="spec-label">Speed</span><span class="spec-value">${m.speedRating}</span></div>
        <div class="spec-item"><span class="spec-label">Accuracy</span><span class="spec-value">${m.accuracyScore}%</span></div>
      </div>
      <div class="model-catalog-actions">
        <button class="btn btn-outline btn-sm" onclick="showModelDetails('${m.id}')">Details</button>
        <button class="btn btn-primary btn-sm" onclick="testModel('${m.id}')">Test</button>
      </div>
    </div>
  `).join('');
  lucide.createIcons();
}

function showModelDetails(id) {
  const m = SEMCO_DATA.models.find(x => x.id === id);
  if (!m) return;
  showModal(`
    <div class="modal-header">
      <h3>${m.name}</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <p class="model-detail-desc">${m.description}</p>
      <div class="model-detail-grid">
        <div class="detail-item"><span class="label">Provider</span><span class="value">${m.provider}</span></div>
        <div class="detail-item"><span class="label">Version</span><span class="value">${m.version}</span></div>
        <div class="detail-item"><span class="label">Context</span><span class="value">${m.contextWindow}</span></div>
        <div class="detail-item"><span class="label">Price</span><span class="value">${m.pricing}</span></div>
        <div class="detail-item"><span class="label">Speed</span><span class="value">${m.speedRating}</span></div>
        <div class="detail-item"><span class="label">Accuracy</span><span class="value">${m.accuracyScore}%</span></div>
      </div>
      <h4 style="margin:16px 0 10px;">Capabilities</h4>
      <div class="capabilities-list">${m.capabilities.map(c => `<span class="cap-pill">${c}</span>`).join('')}</div>
      <button class="btn btn-primary btn-lg" style="width:100%;margin-top:20px;" onclick="testModel('${m.id}'); closeModal();">
        <i data-lucide="play"></i> Test This Model
      </button>
    </div>
  `);
}

function testModel(id) {
  AppState.evaluation.models = [id];
  navigateTo('run-evaluation');
  showToast('Model added', 'success');
}

// ============ PROVIDERS ============
function renderProvidersPage() {
  document.getElementById('providers-page-grid').innerHTML = SEMCO_DATA.providers.map(p => `
    <div class="provider-page-card ${p.status}">
      <div class="provider-page-header">
        <div class="provider-logo-lg">${p.logo}</div>
        <div class="provider-status-badge ${p.status}">${p.status === 'connected' ? 'Connected' : 'Not connected'}</div>
      </div>
      <h3>${p.name}</h3>
      <p>${p.desc}</p>
      <div class="provider-stats">
        <div class="provider-stat"><span class="stat-value">${p.modelCount}</span><span class="stat-label">Models</span></div>
      </div>
      ${p.status === 'connected' ? `
        <div class="api-key-display">
          <span class="key-label">API Key</span>
          <code>${p.apiKey}</code>
        </div>
        <div class="provider-actions">
          <button class="btn btn-secondary btn-sm" onclick="configureProvider('${p.id}')">Configure</button>
          <button class="btn btn-outline btn-sm" onclick="disconnectProvider('${p.id}')">Disconnect</button>
        </div>
      ` : `<button class="btn btn-primary" onclick="connectProvider('${p.id}')"><i data-lucide="plug-zap"></i> Connect</button>`}
    </div>
  `).join('');
  lucide.createIcons();
}

function showAddProviderModal() {
  showModal(`
    <div class="modal-header">
      <h3>Add Provider</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <p>Select a provider:</p>
      <div class="add-provider-list">
        ${SEMCO_DATA.providers.filter(p => p.status !== 'connected').map(p => `
          <div class="add-provider-item" onclick="connectProvider('${p.id}')">
            <div class="provider-logo">${p.logo}</div>
            <div><strong>${p.name}</strong><p>${p.modelCount} models</p></div>
          </div>
        `).join('')}
      </div>
    </div>
  `);
}

function connectProvider(id) {
  closeModal();
  const p = SEMCO_DATA.providers.find(x => x.id === id);
  showModal(`
    <div class="modal-header">
      <h3>Connect ${p.name}</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">API Key</label>
        <input type="password" id="provider-api-key" class="form-control" placeholder="Enter API key">
        <p class="form-help">Find this in your ${p.name} dashboard.</p>
      </div>
      <button class="btn btn-primary btn-lg" style="width:100%" onclick="saveProvider('${id}')">Connect</button>
    </div>
  `);
}

function saveProvider(id) {
  const apiKey = document.getElementById('provider-api-key')?.value;
  if (!apiKey) {
    showToast('Enter API key', 'warning');
    return;
  }

  closeModal();
  const p = SEMCO_DATA.providers.find(x => x.id === id);
  if (p) { p.status = 'connected'; p.apiKey = 'sk-****-' + apiKey.slice(-4); }
  showToast('Provider connected', 'success');
  renderProvidersPage();
}

function configureProvider(id) {
  const p = SEMCO_DATA.providers.find(x => x.id === id);
  showModal(`
    <div class="modal-header">
      <h3>Configure ${p.name}</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">API Key</label>
        <input type="password" class="form-control" value="${p.apiKey}">
      </div>
      <div class="form-group">
        <label class="form-label">Rate Limit (req/min)</label>
        <input type="number" class="form-control" value="60">
      </div>
      <button class="btn btn-primary btn-lg" style="width:100%" onclick="closeModal(); showToast('Saved', 'success');">Save</button>
    </div>
  `);
}

function disconnectProvider(id) {
  if (!confirm('Disconnect this provider?')) return;
  const p = SEMCO_DATA.providers.find(x => x.id === id);
  if (p) { p.status = 'not_connected'; p.apiKey = ''; }
  showToast('Disconnected', 'success');
  renderProvidersPage();
}

// ============ DATASETS ============
function renderDatasetsLibrary() {
  document.getElementById('datasets-library-grid').innerHTML = SEMCO_DATA.testSuites.map(d => `
    <div class="dataset-library-card">
      ${d.featured ? '<div class="featured-badge"><i data-lucide="star"></i> Featured</div>' : ''}
      <div class="dataset-category-tag">${d.category}</div>
      <h3>${d.name}</h3>
      <p>${d.description}</p>
      <div class="dataset-details">
        <div class="detail-row"><span class="detail-label">Questions</span><span class="detail-value">${d.questions}</span></div>
        <div class="detail-row"><span class="detail-label">Language</span><span class="detail-value">${d.language}</span></div>
        <div class="detail-row"><span class="detail-label">Difficulty</span><span class="detail-value">${d.difficulty}</span></div>
        <div class="detail-row"><span class="detail-label">Version</span><span class="detail-value">${d.version}</span></div>
        <div class="detail-row"><span class="detail-label">Maintainer</span><span class="detail-value">${d.maintainer}</span></div>
      </div>
      <div class="dataset-actions"><button class="btn btn-primary" onclick="useDataset('${d.id}')"><i data-lucide="play"></i> Use</button></div>
    </div>
  `).join('');
  lucide.createIcons();
}

function showUploadDatasetModal() {
  showModal(`
    <div class="modal-header">
      <h3>Upload Test Suite</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="upload-zone" onclick="document.getElementById('modal-file-input').click()">
        <input type="file" id="modal-file-input" style="display:none" accept=".csv,.json,.jsonl" onchange="handleFileUploadModal(this)">
        <i data-lucide="upload-cloud" style="width:40px;height:40px;color:var(--primary);margin-bottom:12px;"></i>
        <h4>Drop file here</h4>
        <p>CSV, JSON, JSONL</p>
      </div>
      <div class="divider-text"><span>or import</span></div>
      <div class="import-options">
        <button class="import-btn" onclick="showHuggingFaceImport()"><img src="https://huggingface.co/front/assets/huggingface_logo-noborder.svg" style="width:20px;height:20px;"> HuggingFace</button>
        <button class="import-btn" onclick="showGitHubImport()"><i data-lucide="github"></i> GitHub</button>
      </div>
    </div>
  `);
}

function handleFileUploadModal(input) {
  const file = input.files[0];
  if (!file) return;
  closeModal();
  AppState.uploadedFile = file;
  showColumnMappingModal(file);
}

function showHuggingFaceImport() {
  closeModal();
  showModal(`
    <div class="modal-header">
      <h3>Import from Hugging Face</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">Dataset ID</label>
        <input type="text" id="hf-dataset-id" class="form-control" placeholder="e.g., openai/humaneval">
        <p class="form-help">Enter the Hugging Face dataset path (owner/dataset-name)</p>
      </div>
      <div class="form-group">
        <label class="form-label">Split (Optional)</label>
        <select class="form-control">
          <option value="test">test</option>
          <option value="train">train</option>
          <option value="validation">validation</option>
        </select>
      </div>
      <button class="btn btn-primary btn-lg" style="width:100%" onclick="importHuggingFace()">Import Dataset</button>
    </div>
  `);
}

function importHuggingFace() {
  const datasetId = document.getElementById('hf-dataset-id')?.value;
  if (!datasetId) {
    showToast('Enter dataset ID', 'warning');
    return;
  }

  closeModal();
  showToast('Importing from Hugging Face...', 'info');

  setTimeout(() => {
    const customDataset = {
      id: 'hf-' + Date.now(),
      name: datasetId.split('/')[1] || datasetId,
      category: 'External',
      questions: 500,
      language: 'English',
      difficulty: 'Varied',
      version: 'HF Import',
      maintainer: datasetId.split('/')[0] || 'Hugging Face',
      description: `Imported from Hugging Face: ${datasetId}`,
      featured: false
    };

    SEMCO_DATA.testSuites.unshift(customDataset);
    AppState.evaluation.dataset = customDataset.id;

    showToast('Dataset imported successfully', 'success');
    renderDatasetsLibrary();
  }, 1000);
}

function showGitHubImport() {
  closeModal();
  showModal(`
    <div class="modal-header">
      <h3>Import from GitHub</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">Repository URL</label>
        <input type="text" id="gh-repo-url" class="form-control" placeholder="e.g., github.com/owner/repo/path/to/data.json">
        <p class="form-help">Direct link to JSON or CSV file in a public repository</p>
      </div>
      <button class="btn btn-primary btn-lg" style="width:100%" onclick="importGitHub()">Import</button>
    </div>
  `);
}

function importGitHub() {
  const url = document.getElementById('gh-repo-url')?.value;
  if (!url) {
    showToast('Enter repository URL', 'warning');
    return;
  }

  closeModal();
  showToast('Importing from GitHub...', 'info');

  setTimeout(() => {
    const customDataset = {
      id: 'gh-' + Date.now(),
      name: 'GitHub Dataset',
      category: 'External',
      questions: 250,
      language: 'English',
      difficulty: 'Varied',
      version: 'GitHub Import',
      maintainer: 'External',
      description: `Imported from GitHub`,
      featured: false
    };

    SEMCO_DATA.testSuites.unshift(customDataset);
    showToast('Dataset imported successfully', 'success');
    renderDatasetsLibrary();
  }, 1000);
}

function useDataset(id) {
  AppState.evaluation.dataset = id;
  navigateTo('run-evaluation');
  AppState.wizardStep = 5;
  updateWizardUI();
  showToast('Dataset selected', 'success');
}

// ============ REPORTS ============
function renderReports() {
  document.getElementById('reports-list').innerHTML = SEMCO_DATA.reports.map(r => `
    <div class="report-card">
      <div class="report-header">
        <div class="report-date">${r.date}</div>
        <div class="report-actions">
          <button class="btn btn-sm btn-secondary"><i data-lucide="download"></i> ${r.downloadSize}</button>
          <button class="btn btn-sm btn-secondary"><i data-lucide="share-2"></i></button>
        </div>
      </div>
      <h3>${r.title}</h3>
      <p class="report-summary">${r.summary}</p>
      <div class="report-verdict">
        <i data-lucide="lightbulb"></i>
        <div><strong>Recommendation</strong><p>${r.verdict}</p></div>
      </div>
      <div class="report-footer">
        <div class="top-model"><span class="label">Top:</span><span class="value">${r.topModel}</span></div>
        <div class="metrics-tested">${r.metricsTested.map(m => `<span class="metric-tag">${m}</span>`).join('')}</div>
      </div>
    </div>
  `).join('');
  lucide.createIcons();
}

// ============ HELPERS ============
function showModal(html) {
  document.getElementById('modal-content').innerHTML = html;
  document.getElementById('modal-overlay').classList.add('active');
  lucide.createIcons();
}

function closeModal() {
  document.getElementById('modal-overlay').classList.remove('active');
}

document.getElementById('modal-overlay')?.addEventListener('click', e => {
  if (e.target.id === 'modal-overlay') closeModal();
});

function showToast(msg, type = 'info') {
  document.querySelectorAll('.toast').forEach(t => t.remove());
  const t = document.createElement('div');
  t.className = `toast toast-${type}`;
  t.innerHTML = `<i data-lucide="${type === 'success' ? 'check-circle' : type === 'warning' ? 'alert-circle' : 'info'}"></i><span>${msg}</span>`;
  document.body.appendChild(t);
  lucide.createIcons();
  setTimeout(() => t.classList.add('show'), 10);
  setTimeout(() => { t.classList.remove('show'); setTimeout(() => t.remove(), 200); }, 2500);
}

function showNotifications() {
  showModal(`
    <div class="modal-header">
      <h3>Notifications</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="notification-item unread">
        <div class="notif-icon"><i data-lucide="check-circle"></i></div>
        <div class="notif-content">
          <strong>Evaluation complete</strong>
          <p>Agent Tool Calling Test finished.</p>
          <span class="notif-time">10 min ago</span>
        </div>
      </div>
      <div class="notification-item">
        <div class="notif-icon"><i data-lucide="box"></i></div>
        <div class="notif-content">
          <strong>New model</strong>
          <p>Claude 3.5 Sonnet v2 available.</p>
          <span class="notif-time">2 hours ago</span>
        </div>
      </div>
    </div>
  `);
}

function showUserMenu() {
  showModal(`
    <div class="modal-header">
      <h3>Account</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="user-profile-card">
        <div class="avatar avatar-lg">AM</div>
        <div>
          <h4>Alex Mercer</h4>
          <p>AI Engineering Lead</p>
          <p class="org">Semco Enterprise Labs</p>
        </div>
      </div>
      <div class="menu-items">
        <button class="menu-item" onclick="closeModal(); navigateTo('settings');"><i data-lucide="settings"></i> Settings</button>
        <button class="menu-item" onclick="closeModal(); showHelp();"><i data-lucide="circle-help"></i> Help</button>
        <button class="menu-item" onclick="logout();"><i data-lucide="log-out"></i> Sign out</button>
      </div>
    </div>
  `);
}

function showHelp() {
  showModal(`
    <div class="modal-header">
      <h3>Help</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="help-section">
        <h4>Getting Started</h4>
        <ol>
          <li>Connect your AI providers</li>
          <li>Click "New Evaluation"</li>
          <li>Select models to compare</li>
          <li>Choose a test suite</li>
          <li>View results</li>
        </ol>
      </div>
      <div class="help-links">
        <a href="#" class="help-link"><i data-lucide="book-open"></i> Documentation</a>
        <a href="#" class="help-link"><i data-lucide="message-circle"></i> Contact Support</a>
      </div>
    </div>
  `);
}

// ========== SETTINGS PAGE ==========
function switchSettingsTab(tabId, btn) {
  // Update nav buttons
  document.querySelectorAll('.settings-nav-item').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');

  // Switch tab content
  document.querySelectorAll('.settings-tab').forEach(t => t.classList.remove('active'));
  document.getElementById(`settings-${tabId}`).classList.add('active');

  // Render API keys if switching to that tab
  if (tabId === 'apikeys') {
    renderApiKeysList();
  }

  lucide.createIcons();
}

function saveWorkspaceSettings() {
  const orgName = document.getElementById('org-name').value;
  const evalType = document.getElementById('default-eval-type').value;
  const timezone = document.getElementById('timezone').value;

  // Update app state (simulated)
  SEMCO_DATA.currentUser.organization = orgName;

  showToast('Settings saved', 'success');
}

function renderApiKeysList() {
  const container = document.getElementById('api-keys-list');
  container.innerHTML = SEMCO_DATA.providers.map(p => `
    <div class="api-key-row">
      <div class="api-key-provider">
        <div class="provider-logo-sm">${p.logo}</div>
        <div class="api-key-info">
          <strong>${p.name}</strong>
          <span class="api-key-masked">${p.status === 'connected' ? p.apiKey : 'Not connected'}</span>
        </div>
      </div>
      <div class="api-key-actions">
        ${p.status === 'connected' ? `
          <span class="status-badge connected"><i data-lucide="check-circle"></i> Connected</span>
          <button class="btn btn-sm btn-ghost" onclick="editApiKey('${p.id}')"><i data-lucide="edit-2"></i></button>
          <button class="btn btn-sm btn-ghost" onclick="disconnectProvider('${p.id}')"><i data-lucide="trash-2"></i></button>
        ` : `
          <button class="btn btn-sm btn-primary" onclick="connectProviderInline('${p.id}')">Connect</button>
        `}
      </div>
    </div>
  `).join('');
  lucide.createIcons();
}

function editApiKey(providerId) {
  const p = SEMCO_DATA.providers.find(x => x.id === providerId);
  showModal(`
    <div class="modal-header">
      <h3>Update ${p.name} API Key</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">Current Key</label>
        <input type="text" class="form-control" value="${p.apiKey}" disabled>
      </div>
      <div class="form-group">
        <label class="form-label">New API Key</label>
        <input type="password" id="new-api-key" class="form-control" placeholder="Enter new key">
      </div>
      <div class="modal-actions">
        <button class="btn btn-secondary" onclick="closeModal()">Cancel</button>
        <button class="btn btn-primary" onclick="updateApiKey('${providerId}')">Update Key</button>
      </div>
    </div>
  `);
}

function updateApiKey(providerId) {
  const newKey = document.getElementById('new-api-key').value;
  if (!newKey) {
    showToast('Enter new API key', 'warning');
    return;
  }
  const p = SEMCO_DATA.providers.find(x => x.id === providerId);
  p.apiKey = 'sk-****-' + newKey.slice(-4);
  closeModal();
  showToast(`${p.name} API key updated`, 'success');
  renderApiKeysList();
}

function disconnectProvider(providerId) {
  const p = SEMCO_DATA.providers.find(x => x.id === providerId);
  showModal(`
    <div class="modal-header">
      <h3>Disconnect ${p.name}?</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <p style="color:var(--text-secondary);margin-bottom:20px;">This will remove the API key and disable ${p.name} models for future evaluations.</p>
      <div class="modal-actions">
        <button class="btn btn-secondary" onclick="closeModal()">Cancel</button>
        <button class="btn btn-danger" onclick="confirmDisconnect('${providerId}')">Disconnect</button>
      </div>
    </div>
  `);
}

function confirmDisconnect(providerId) {
  const p = SEMCO_DATA.providers.find(x => x.id === providerId);
  p.status = 'not_connected';
  p.apiKey = '';
  closeModal();
  showToast(`${p.name} disconnected`, 'success');
  renderApiKeysList();
}

function showAddProviderModal() {
  const unconnected = SEMCO_DATA.providers.filter(p => p.status !== 'connected');
  showModal(`
    <div class="modal-header">
      <h3>Add Provider</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <p style="color:var(--text-secondary);margin-bottom:16px;">Select a provider to connect:</p>
      <div class="provider-select-list">
        ${unconnected.length > 0 ? unconnected.map(p => `
          <div class="provider-select-item" onclick="connectProviderInline('${p.id}')">
            <div class="provider-logo-sm">${p.logo}</div>
            <div>
              <strong>${p.name}</strong>
              <span>${p.modelCount}+ models</span>
            </div>
            <i data-lucide="chevron-right"></i>
          </div>
        `).join('') : '<p style="color:var(--text-tertiary);text-align:center;padding:20px;">All providers connected</p>'}
      </div>
    </div>
  `);
  lucide.createIcons();
}

function showInviteModal() {
  showModal(`
    <div class="modal-header">
      <h3>Invite Team Member</h3>
      <button class="modal-close" onclick="closeModal()"><i data-lucide="x"></i></button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">Email Address</label>
        <input type="email" id="invite-email" class="form-control" placeholder="colleague@semco.ai">
      </div>
      <div class="form-group">
        <label class="form-label">Role</label>
        <select id="invite-role" class="form-control">
          <option value="member">Member - Can run evaluations</option>
          <option value="admin">Admin - Full access</option>
        </select>
      </div>
      <div class="modal-actions">
        <button class="btn btn-secondary" onclick="closeModal()">Cancel</button>
        <button class="btn btn-primary" onclick="sendInvite()">Send Invite</button>
      </div>
    </div>
  `);
}

function sendInvite() {
  const email = document.getElementById('invite-email').value;
  if (!email) {
    showToast('Enter email address', 'warning');
    return;
  }
  closeModal();
  showToast(`Invitation sent to ${email}`, 'success');
}

function saveNotificationSetting(setting, enabled) {
  showToast(`${enabled ? 'Enabled' : 'Disabled'} ${setting.replace('_', ' ')}`, 'success');
}

function logout() {
  closeModal();
  showLoginPage();
}

function copyLink() {
  navigator.clipboard.writeText('https://semcoeval.internal/r/eval-9041');
  showToast('Copied', 'success');
}

document.addEventListener('keydown', e => {
  if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
    e.preventDefault();
    document.querySelector('.search-box input')?.focus();
  }
  if (e.key === 'Escape') closeModal();
});































//data.js
/**
 * SemcoEval - Interactive Prototype Data Engine
 *
 * This data simulates what would be fetched from provider APIs:
 * - Models & capabilities auto-detected via /v1/models endpoint
 * - Vision, Tool Calling, max_tokens all come from API response
 * - No manual configuration required - everything is automatic
 *
 * In production: User connects API key → System calls /v1/models →
 * Capabilities (vision, tool_use, context_length) are auto-populated
 */

const SEMCO_DATA = {
  currentUser: {
    name: "Alex Mercer",
    role: "Head of AI Engineering & Strategy",
    organization: "Semco Enterprise Labs",
    avatar: "AM"
  },
  
  evalTypes: [
    {
      id: "model",
      title: "General Chat & Text (AI Model)",
      desc: "Evaluate base model knowledge, summarization quality, and conversation tone across standardized test suites.",
      badge: "Fast Evaluation"
    },
    {
      id: "agent",
      title: "Autonomous Workflow (Agent Evaluation)",
      desc: "Test autonomous agents like Hermes-Agent on multi-step tool execution, function calling, and programmatic workflow accuracy.",
      badge: "Recommended for Automation"
    },
    {
      id: "rag",
      title: "Document Search & Answering (Knowledge / RAG)",
      desc: "Measure how accurately AI models retrieve information from corporate documents without generating incorrect answers.",
      badge: "High Precision"
    }
  ],

  providers: [
    { id: "openai", name: "OpenAI", status: "connected", modelCount: 8, logo: "O", desc: "Industry benchmark provider offering GPT-4o, o1-reasoning, and embedded functions.", apiKey: "sk-proj-*****************8f91" },
    { id: "together", name: "Together AI (Nous / Hermes)", status: "connected", modelCount: 14, logo: "T", desc: "High-performance infrastructure hosting Hermes-Agent, Llama 3.3, and DeepSeek weights.", apiKey: "tog-api-*****************9a0c" },
    { id: "anthropic", name: "Anthropic", status: "connected", modelCount: 5, logo: "A", desc: "Safety-first reasoning models including Claude 3.5 Sonnet and Opus optimized for long context.", apiKey: "ant-key-*****************4b22" },
    { id: "groq", name: "Groq Cloud", status: "connected", modelCount: 6, logo: "G", desc: "Ultra-low response time LPU inference engine for instant response generation.", apiKey: "gq-live-*****************7d11" },
    { id: "openrouter", name: "OpenRouter", status: "connected", modelCount: 45, logo: "R", desc: "Unified routing engine providing failover access to open-source and proprietary models.", apiKey: "or-prod-*****************6f88" },
    { id: "gemini", name: "Google Gemini", status: "connected", modelCount: 6, logo: "G", desc: "Massive 2M context window models featuring native multi-modal reasoning & code agents.", apiKey: "AIzaSy*****************9kLq" },
    { id: "azure", name: "Azure OpenAI Service", status: "not_connected", modelCount: 8, logo: "Az", desc: "Enterprise tenant hosting with zero data-retention SLAs and HIPAA compliant endpoints.", apiKey: "" },
    { id: "ollama", name: "Ollama Local (On-Prem)", status: "not_connected", modelCount: 12, logo: "OL", desc: "Locally running quantized open-source models inside secure corporate network parameters.", apiKey: "" },
    { id: "fireworks", name: "Fireworks AI", status: "not_connected", modelCount: 10, logo: "F", desc: "Lightning fast fine-tuned agent hosting engineered for high-concurrency tool use.", apiKey: "" }
  ],

  models: [
    {
      id: "m-hermes3",
      name: "Hermes 3 - 70B Agent",
      provider: "Together AI (Nous / Hermes)",
      providerId: "together",
      capabilities: ["Tool Calling", "Autonomous Agent", "Reasoning", "JSON Mode"],
      contextWindow: "128k tokens",
      pricing: "$0.70 / 1M tokens",
      speedRating: "Ultra Fast (85 t/s)",
      accuracyScore: 94.8,
      agentScore: 97.5,
      version: "v3.0-Llama3.3-fine-tuned",
      description: "State-of-the-art open source autonomous agent model engineered by Nous Research. Exceptional at multi-step tool calling, API integration, and workflow orchestration without human intervention.",
      selectedByDefault: true,
      category: "agent"
    },
    {
      id: "m-claude35",
      name: "Claude 3.5 Sonnet Agent v2",
      provider: "Anthropic",
      providerId: "anthropic",
      capabilities: ["Tool Calling", "Deep Reasoning", "Vision", "Coding"],
      contextWindow: "200k tokens",
      pricing: "$3.00 / 1M tokens",
      speedRating: "Fast (55 t/s)",
      accuracyScore: 95.4,
      agentScore: 96.2,
      version: "2024-10-22-v2",
      description: "Industry leading commercial model for complex logic, multi-document synthesis, and dependable autonomous task execution.",
      selectedByDefault: true,
      category: "agent"
    },
    {
      id: "m-gpt4o",
      name: "GPT-4o Autonomous Agent",
      provider: "OpenAI",
      providerId: "openai",
      capabilities: ["Tool Calling", "Multimodal Vision", "Reasoning", "Fast Inference"],
      contextWindow: "128k tokens",
      pricing: "$2.50 / 1M tokens",
      speedRating: "Fast (65 t/s)",
      accuracyScore: 94.2,
      agentScore: 95.0,
      version: "2024-11-20-o",
      description: "Flagship general purpose and conversational agent optimized for interactive web workflows and dynamic function execution.",
      selectedByDefault: true,
      category: "agent"
    },
    {
      id: "m-hermes2pro",
      name: "Hermes 2 Pro - Mistral 7B Agent",
      provider: "OpenRouter",
      providerId: "openrouter",
      capabilities: ["Tool Calling", "Structured JSON", "Edge Deployment"],
      contextWindow: "32k tokens",
      pricing: "$0.18 / 1M tokens",
      speedRating: "Instant (140 t/s)",
      accuracyScore: 88.5,
      agentScore: 92.4,
      version: "v2.5-Pro",
      description: "Cost-effective, highly efficient micro-agent engineered specifically for embedded structured output and fast API dispatching.",
      selectedByDefault: false,
      category: "agent"
    },
    {
      id: "m-deepseekr1",
      name: "DeepSeek R1 Open-Reasoning",
      provider: "Together AI (Nous / Hermes)",
      providerId: "together",
      capabilities: ["Deep Math", "Advanced Logic", "Code Generation"],
      contextWindow: "64k tokens",
      pricing: "$0.55 / 1M tokens",
      speedRating: "Medium (42 t/s)",
      accuracyScore: 96.1,
      agentScore: 91.8,
      version: "R1-Distill-70B",
      description: "Breakthrough reasoning architecture featuring chain-of-thought analysis, surpassing proprietary models in computational complexity.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-llama33",
      name: "Llama 3.3 70B Instruct",
      provider: "Groq Cloud",
      providerId: "groq",
      capabilities: ["Instant Response", "General Chat", "Tool Calling"],
      contextWindow: "128k tokens",
      pricing: "$0.59 / 1M tokens",
      speedRating: "Instant (280 t/s)",
      accuracyScore: 91.5,
      agentScore: 89.4,
      version: "3.3-70b-versatile",
      description: "Powered by custom LPU hardware for real-time human conversation speeds with state-of-the-art open weights accuracy.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-gemini15",
      name: "Gemini 1.5 Pro Long-Context",
      provider: "Google Gemini",
      providerId: "gemini",
      capabilities: ["2M Tokens", "Multi-modal Video", "Document Analysis"],
      contextWindow: "2,000,000 tokens",
      pricing: "$2.50 / 1M tokens",
      speedRating: "Fast (50 t/s)",
      accuracyScore: 93.7,
      agentScore: 91.0,
      version: "1.5-pro-002",
      description: "Unmatched context ingestion capacity capable of analyzing entire software repositories and hours of video in a single evaluation pass.",
      selectedByDefault: false,
      category: "rag"
    },
    {
      id: "m-gemini20",
      name: "Gemini 2.0 Flash",
      provider: "Google Gemini",
      providerId: "gemini",
      capabilities: ["Vision", "Tool Calling", "Fast Inference", "Streaming"],
      contextWindow: "1,000,000 tokens",
      pricing: "$0.10 / 1M tokens",
      speedRating: "Ultra Fast (120 t/s)",
      accuracyScore: 92.4,
      agentScore: 94.3,
      version: "2.0-flash-exp",
      description: "Next generation multimodal model with native tool use, real-time streaming, and extremely low latency for interactive applications.",
      selectedByDefault: false,
      category: "agent"
    },
    {
      id: "m-claude-opus",
      name: "Claude 3.5 Opus",
      provider: "Anthropic",
      providerId: "anthropic",
      capabilities: ["Deep Reasoning", "Long Context", "Complex Analysis", "Vision"],
      contextWindow: "200k tokens",
      pricing: "$15.00 / 1M tokens",
      speedRating: "Medium (25 t/s)",
      accuracyScore: 97.2,
      agentScore: 94.8,
      version: "3.5-opus-20241022",
      description: "Most capable Claude model for complex reasoning, research synthesis, and nuanced analysis requiring deep understanding.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-qwen25",
      name: "Qwen 2.5 72B Instruct",
      provider: "Together AI (Nous / Hermes)",
      providerId: "together",
      capabilities: ["Coding", "Math", "Tool Calling", "Multilingual"],
      contextWindow: "128k tokens",
      pricing: "$0.90 / 1M tokens",
      speedRating: "Fast (65 t/s)",
      accuracyScore: 93.8,
      agentScore: 92.1,
      version: "2.5-72b-instruct",
      description: "State-of-the-art open source model from Alibaba excelling at coding, mathematical reasoning, and multilingual tasks.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-mistral-large",
      name: "Mistral Large 2",
      provider: "OpenRouter",
      providerId: "openrouter",
      capabilities: ["Reasoning", "Coding", "Tool Calling", "JSON Mode"],
      contextWindow: "128k tokens",
      pricing: "$2.00 / 1M tokens",
      speedRating: "Fast (55 t/s)",
      accuracyScore: 92.6,
      agentScore: 91.4,
      version: "mistral-large-2407",
      description: "European frontier model with strong multilingual capabilities and excellent performance on reasoning benchmarks.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-gpt4-mini",
      name: "GPT-4o Mini",
      provider: "OpenAI",
      providerId: "openai",
      capabilities: ["Vision", "Tool Calling", "Fast Inference", "Streaming"],
      contextWindow: "128k tokens",
      pricing: "$0.15 / 1M tokens",
      speedRating: "Ultra Fast (90 t/s)",
      accuracyScore: 88.5,
      agentScore: 87.2,
      version: "gpt-4o-mini-2024-07-18",
      description: "Cost-effective GPT-4 class model ideal for high-volume applications requiring vision and tool calling capabilities.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-deepseek-coder",
      name: "DeepSeek Coder V2",
      provider: "Together AI (Nous / Hermes)",
      providerId: "together",
      capabilities: ["Coding", "Code Review", "Bug Detection", "Refactoring"],
      contextWindow: "128k tokens",
      pricing: "$0.28 / 1M tokens",
      speedRating: "Fast (72 t/s)",
      accuracyScore: 91.4,
      agentScore: 88.9,
      version: "deepseek-coder-v2-236b",
      description: "Specialized coding model trained on extensive code repositories, excelling at code generation and software engineering tasks.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-llama-vision",
      name: "Llama 3.2 90B Vision",
      provider: "Together AI (Nous / Hermes)",
      providerId: "together",
      capabilities: ["Vision", "Image Analysis", "Document OCR", "Chart Reading"],
      contextWindow: "128k tokens",
      pricing: "$1.20 / 1M tokens",
      speedRating: "Medium (35 t/s)",
      accuracyScore: 90.2,
      agentScore: 85.6,
      version: "llama-3.2-90b-vision",
      description: "Open source multimodal model capable of understanding images, documents, charts, and visual content alongside text.",
      selectedByDefault: false,
      category: "model"
    },
    {
      id: "m-o1-preview",
      name: "o1 Preview",
      provider: "OpenAI",
      providerId: "openai",
      capabilities: ["Deep Reasoning", "Math", "Science", "Complex Logic"],
      contextWindow: "128k tokens",
      pricing: "$15.00 / 1M tokens",
      speedRating: "Slow (8 t/s)",
      accuracyScore: 98.1,
      agentScore: 94.0,
      version: "o1-preview-2024-09-12",
      description: "Advanced reasoning model using chain-of-thought processing for complex mathematical, scientific, and logical problems.",
      selectedByDefault: false,
      category: "model"
    }
  ],

  testSuites: [
    {
      id: "ts-hermes-agent",
      name: "Hermes Autonomous Tool & Workflow Suite",
      category: "Agents",
      questions: 420,
      language: "English / JSON",
      task: "Multi-step API Execution & Self-Correction",
      difficulty: "Expert",
      version: "v3.2 Official",
      maintainer: "Nous Research / Semco Labs",
      description: "Evaluates multi-step tool calling, nested JSON schema generation, error recovery, and complex workflow execution without human interference.",
      recommendedFor: ["agent"],
      featured: true
    },
    {
      id: "ts-swe-bench",
      name: "SWE-bench Verified Software Engineer Suite",
      category: "Coding",
      questions: 500,
      language: "Python / JS / TS",
      task: "Autonomous GitHub Issue Resolution",
      difficulty: "Expert",
      version: "2026.2 Verified",
      maintainer: "Open Source AI Labs",
      description: "Measures AI model ability to independently read a codebase, diagnose bug reports, run local unit tests, and commit working patches.",
      recommendedFor: ["agent", "model"],
      featured: true
    },
    {
      id: "ts-mmlu",
      name: "MMLU-Pro General Knowledge & Reasoning Test Suite",
      category: "General",
      questions: 1400,
      language: "Multilingual (24 Languages)",
      task: "Academic & Professional Problem Solving",
      difficulty: "High",
      version: "Pro-v2",
      maintainer: "Stanford CRFM",
      description: "Comprehensive multi-domain intelligence benchmark covering 57 disciplines including law, quantum physics, corporate accounting, and ethics.",
      recommendedFor: ["model"],
      featured: false
    },
    {
      id: "ts-ragas",
      name: "Ragas Document Factual Recall & Faithfulness",
      category: "RAG",
      questions: 350,
      language: "English",
      task: "Context Precision & Hallucination Defense",
      difficulty: "Medium",
      version: "1.4-Production",
      maintainer: "Ragas Ecosystem",
      description: "Evaluates retrieval accuracy and ensures answers derive strictly from verified company documents without generating incorrect or fabricated claims.",
      recommendedFor: ["rag"],
      featured: true
    },
    {
      id: "ts-finance",
      name: "Corporate Finance & Audit Math Test Suite",
      category: "Finance",
      questions: 280,
      language: "English / Numeric",
      task: "Exact Mathematical Deduction & SEC Regulation",
      difficulty: "Advanced",
      version: "2026-Q1",
      maintainer: "Semco Enterprise Labs",
      description: "Rigorous test suite validating exact calculation precision, spreadsheet formula interpretation, and regulatory compliance audit reasoning.",
      recommendedFor: ["model", "agent"],
      featured: false
    },
    {
      id: "ts-healthcare",
      name: "Clinical Diagnostic Safety & Medical Care Benchmark",
      category: "Healthcare",
      questions: 310,
      language: "English / Medical",
      task: "Diagnostic Logic & Patient Guardrails",
      difficulty: "Expert",
      version: "Med-2.1",
      maintainer: "HealthAI Foundation",
      description: "Assesses diagnostic recommendation accuracy and strict safety compliance to prevent dangerous patient guidance in automated health triage.",
      recommendedFor: ["model"],
      featured: false
    }
  ],

  metrics: {
    // Universal metrics (all evaluation types)
    universal: [
      { id: "accuracy", name: "Accuracy", tooltip: "Percentage of correct answers or successful task completions.", defaultChecked: true, icon: "target" },
      { id: "latency", name: "Response Time", tooltip: "Average time to generate a complete response (seconds).", defaultChecked: true, icon: "clock" },
      { id: "cost", name: "Cost Efficiency", tooltip: "Cost per 1,000 API calls at provider pricing.", defaultChecked: true, icon: "dollar-sign" },
      { id: "safety", name: "Safety Score", tooltip: "Resistance to jailbreaks, prompt injection, and harmful outputs.", defaultChecked: false, icon: "shield" }
    ],

    // LLM/Chat specific metrics
    model: [
      { id: "fluency", name: "Fluency & Coherence", tooltip: "How natural and well-structured the generated text is.", defaultChecked: true, icon: "message-square" },
      { id: "instruction_following", name: "Instruction Following", tooltip: "How well the model adheres to specific instructions and constraints.", defaultChecked: true, icon: "check-square" },
      { id: "reasoning", name: "Reasoning Quality", tooltip: "Logical consistency, chain-of-thought clarity, and problem decomposition.", defaultChecked: true, icon: "brain" },
      { id: "factuality", name: "Factual Accuracy", tooltip: "Correctness of factual claims based on world knowledge.", defaultChecked: false, icon: "book-open" },
      { id: "helpfulness", name: "Helpfulness", tooltip: "How useful and relevant the response is to the user's query.", defaultChecked: false, icon: "thumbs-up" },
      { id: "creativity", name: "Creativity", tooltip: "Originality and diversity in generated content.", defaultChecked: false, icon: "sparkles" }
    ],

    // Agent/Autonomous workflow metrics
    agent: [
      { id: "tool_success", name: "Tool Calling Success", tooltip: "Percentage of function/API calls with correct syntax and parameters.", defaultChecked: true, icon: "wrench" },
      { id: "task_completion", name: "Task Completion Rate", tooltip: "Percentage of multi-step tasks completed successfully end-to-end.", defaultChecked: true, icon: "check-circle" },
      { id: "action_accuracy", name: "Action Sequencing", tooltip: "Correct ordering and logic of multi-step action plans.", defaultChecked: true, icon: "list-ordered" },
      { id: "error_recovery", name: "Error Recovery", tooltip: "Ability to detect failures and self-correct without human intervention.", defaultChecked: true, icon: "refresh-cw" },
      { id: "planning", name: "Planning Quality", tooltip: "Quality of task decomposition and strategic planning.", defaultChecked: false, icon: "map" },
      { id: "autonomy", name: "Autonomy Score", tooltip: "Degree of independence in completing complex workflows.", defaultChecked: false, icon: "bot" },
      { id: "api_efficiency", name: "API Call Efficiency", tooltip: "Minimizing unnecessary API calls while achieving goals.", defaultChecked: false, icon: "zap" }
    ],

    // RAG specific metrics
    rag: [
      { id: "faithfulness", name: "Faithfulness", tooltip: "Does the answer accurately reflect the retrieved context without adding unsupported claims?", defaultChecked: true, icon: "file-check" },
      { id: "answer_relevance", name: "Answer Relevance", tooltip: "How well the generated answer addresses the original question.", defaultChecked: true, icon: "crosshair" },
      { id: "context_precision", name: "Context Precision", tooltip: "Are the retrieved documents relevant to the question? (Precision)", defaultChecked: true, icon: "filter" },
      { id: "context_recall", name: "Context Recall", tooltip: "Did retrieval capture all the relevant information? (Recall)", defaultChecked: true, icon: "search" },
      { id: "groundedness", name: "Groundedness", tooltip: "Is every claim in the answer supported by the source documents?", defaultChecked: true, icon: "anchor" },
      { id: "hallucination", name: "Hallucination Rate", tooltip: "Percentage of fabricated facts not present in retrieved context.", defaultChecked: true, icon: "alert-triangle" },
      { id: "citation_accuracy", name: "Citation Accuracy", tooltip: "Correctness of source citations and references.", defaultChecked: false, icon: "quote" },
      { id: "retrieval_latency", name: "Retrieval Latency", tooltip: "Time to retrieve relevant documents from the knowledge base.", defaultChecked: false, icon: "database" }
    ]
  },

  recentEvaluations: [
    {
      id: "eval-9041",
      name: "Hermes-Agent vs Claude 3.5 Tool Calling Duel",
      type: "Autonomous Workflow (Agent)",
      testSuite: "Hermes Autonomous Tool & Workflow Suite",
      date: "10 mins ago",
      modelsTested: 3,
      topModel: "Hermes 3 - 70B Agent",
      topScore: "97.5%",
      avgResponseTime: "0.82s",
      costSpent: "$0.14",
      status: "Completed",
      results: [
        { rank: 1, model: "Hermes 3 - 70B Agent", provider: "Together AI (Nous)", score: "97.5%", accuracy: "96.2%", time: "0.75s", cost: "$0.08", incorrectRate: "0.4%", toolSuccess: "99.1%", status: "Winner" },
        { rank: 2, model: "Claude 3.5 Sonnet Agent v2", provider: "Anthropic", score: "96.2%", accuracy: "97.0%", time: "1.12s", cost: "$0.42", incorrectRate: "0.2%", toolSuccess: "98.5%", status: "Runner-up" },
        { rank: 3, model: "GPT-4o Autonomous Agent", provider: "OpenAI", score: "94.0%", accuracy: "93.8%", time: "0.95s", cost: "$0.35", incorrectRate: "1.1%", toolSuccess: "95.0%", status: "Third" }
      ]
    },
    {
      id: "eval-8820",
      name: "Q3 Financial Audit & SEC Accounting Verification",
      type: "General Chat & Text (AI Model)",
      testSuite: "Corporate Finance & Audit Math Test Suite",
      date: "Yesterday",
      modelsTested: 4,
      topModel: "DeepSeek R1 Open-Reasoning",
      topScore: "96.8%",
      avgResponseTime: "1.45s",
      costSpent: "$0.29",
      status: "Completed",
      results: [
        { rank: 1, model: "DeepSeek R1 Open-Reasoning", provider: "Together AI", score: "96.8%", accuracy: "98.2%", time: "1.85s", cost: "$0.09", incorrectRate: "0.1%", status: "Winner" },
        { rank: 2, model: "Claude 3.5 Sonnet Agent v2", provider: "Anthropic", score: "95.1%", accuracy: "96.0%", time: "1.10s", cost: "$0.45", incorrectRate: "0.3%", status: "Runner-up" },
        { rank: 3, model: "GPT-4o Autonomous Agent", provider: "OpenAI", score: "92.4%", accuracy: "91.5%", time: "0.88s", cost: "$0.38", incorrectRate: "1.5%", status: "Third" }
      ]
    },
    {
      id: "eval-8715",
      name: "Customer Support Knowledge Retrieval (Ragas Test)",
      type: "Document Search & Answering (RAG)",
      testSuite: "Ragas Document Factual Recall & Faithfulness",
      date: "3 days ago",
      modelsTested: 3,
      topModel: "Gemini 1.5 Pro Long-Context",
      topScore: "95.3%",
      avgResponseTime: "1.02s",
      costSpent: "$0.22",
      status: "Completed",
      results: [
        { rank: 1, model: "Gemini 1.5 Pro Long-Context", provider: "Google Gemini", score: "95.3%", accuracy: "96.5%", time: "1.20s", cost: "$0.31", incorrectRate: "0.2%", status: "Winner" },
        { rank: 2, model: "Llama 3.3 70B Instruct", provider: "Groq Cloud", score: "93.1%", accuracy: "91.2%", time: "0.28s", cost: "$0.07", incorrectRate: "1.2%", status: "Best Value" }
      ]
    }
  ],

  reports: [
    {
      id: "rep-1",
      title: "Executive Recommendation: Autonomous Agent Standardization",
      date: "July 26, 2026",
      summary: "After subjecting 6 candidate models to the Hermes Autonomous Tool & Workflow Suite, Hermes 3 - 70B Agent and Claude 3.5 Sonnet emerged as top performers. Hermes 3 offers a 76% cost reduction and 35% speed gain while matching proprietary tool execution accuracy (99.1%).",
      topModel: "Hermes 3 - 70B Agent",
      verdict: "Adopt Hermes 3 for high-volume automated tools; preserve Claude 3.5 for highly ambiguous policy edge-cases.",
      metricsTested: ["Tool Calling Success", "Response Time", "Task Accuracy", "Cost per 1,000 Tasks"],
      downloadSize: "2.4 MB PDF"
    },
    {
      id: "rep-2",
      title: "Quarterly RAG Factual Accuracy Assessment",
      date: "July 22, 2026",
      summary: "Evaluation of our customer-facing knowledge assistant across 350 ground-truth QA pairs. Gemini 1.5 Pro demonstrated lowest incorrect answer rates (0.2%), while Groq hosted Llama 3.3 provided the fastest user interaction speed (280 t/s).",
      topModel: "Gemini 1.5 Pro Long-Context",
      verdict: "Implement Groq Llama 3.3 as frontline responder with automated fall-back to Gemini 1.5 Pro for queries over 30,000 characters.",
      metricsTested: ["Incorrect Answer Rate", "Response Time", "Retrieval Precision"],
      downloadSize: "1.8 MB PDF"
    }
  ],

  sampleImportRows: [
    { col1: "What function retrieves current user banking ledger?", col2: "get_account_ledger({user_id: '881', include_pending: true})", col3: "Banking documentation API v2.4 section 4" },
    { col2: "execute_refund_transaction({txn_id: 'TX992', reason: 'damaged_item'})", col1: "Customer demands refund for damaged shipment TX992", col3: "Merchant return guidelines page 12" },
    { col1: "Schedule recurring cron for daily compliance scan at midnight", col2: "create_scheduler({cron: '0 0 * * *', job: 'security_audit'})", col3: "DevOps Automated Operations manual chapter 3" }
  ]
};























//index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SemcoEval</title>
  <link rel="stylesheet" href="styles.css">
  <script src="https://unpkg.com/lucide@latest"></script>
</head>
<body>

<!-- LOGIN -->
<div id="login-page" class="login-container">
  <div class="login-left">
    <div class="login-brand">
      <div class="brand-logo">S</div>
      <span class="brand-title">SemcoEval</span>
    </div>

    <div class="login-content">
      <h1 class="login-title">Welcome back</h1>
      <p class="login-subtitle">Sign in to your account to continue</p>

      <form class="login-form" onsubmit="handleLogin(event)">
        <div class="form-group">
          <label class="form-label">Email</label>
          <input type="email" class="form-control" placeholder="you@semcolabs.com" value="alex@semcolabs.com">
        </div>
        <div class="form-group">
          <label class="form-label">Password</label>
          <input type="password" class="form-control" placeholder="Enter password" value="password123">
        </div>
        <div class="login-options">
          <label class="checkbox-label">
            <input type="checkbox" checked> Keep me signed in
          </label>
        </div>
        <button type="submit" class="btn btn-primary btn-lg login-btn">Sign In</button>
      </form>
    </div>

    <div class="login-footer">
      <span>Semco Enterprise Labs</span>
    </div>
  </div>

  <div class="login-right">
    <div class="login-visual">
      <div class="visual-content">
        <div class="visual-badge">AI Evaluation Platform</div>
        <h2>Find the best AI model for your use case</h2>
        <p>Compare models across providers using standardized benchmarks. Get actionable insights without writing code.</p>
        <div class="visual-features">
          <div class="visual-feature">
            <i data-lucide="check"></i>
            <span>Test any model from OpenAI, Anthropic, Google & more</span>
          </div>
          <div class="visual-feature">
            <i data-lucide="check"></i>
            <span>Industry-standard benchmarks & custom datasets</span>
          </div>
          <div class="visual-feature">
            <i data-lucide="check"></i>
            <span>Clear results with actionable recommendations</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- APP -->
<div id="app-container" class="app-container" style="display: none;">

  <!-- Sidebar -->
  <aside class="sidebar">
    <div class="sidebar-top">
      <div class="brand-section">
        <div class="brand-logo">S</div>
        <span class="brand-title">SemcoEval</span>
      </div>

      <ul class="nav-links">
        <li class="nav-item highlight-btn" onclick="navigateTo('run-evaluation')">
          <i data-lucide="play"></i>
          New Evaluation
        </li>
        <li class="nav-item active" data-page="dashboard" onclick="navigateTo('dashboard')">
          <i data-lucide="layout-grid"></i>
          Dashboard
        </li>
        <li class="nav-item" data-page="history" onclick="navigateTo('history')">
          <i data-lucide="clock"></i>
          History
        </li>
        <li class="nav-item" data-page="models" onclick="navigateTo('models')">
          <i data-lucide="box"></i>
          Models
        </li>
        <li class="nav-item" data-page="providers" onclick="navigateTo('providers')">
          <i data-lucide="plug-zap"></i>
          Providers
        </li>
        <li class="nav-item" data-page="datasets" onclick="navigateTo('datasets')">
          <i data-lucide="database"></i>
          Test Suites
        </li>
        <li class="nav-item" data-page="reports" onclick="navigateTo('reports')">
          <i data-lucide="file-bar-chart"></i>
          Reports
        </li>
      </ul>
    </div>

    <div class="sidebar-footer">
      <div class="nav-item" data-page="settings" onclick="navigateTo('settings')">
        <i data-lucide="settings"></i>
        Settings
      </div>
      <div class="nav-item" onclick="showHelp()">
        <i data-lucide="circle-help"></i>
        Help
      </div>
    </div>
  </aside>

  <!-- Main -->
  <main class="main-wrapper">

    <header class="topbar">
      <div class="search-box">
        <i data-lucide="search"></i>
        <input type="text" placeholder="Search...">
        <kbd>⌘K</kbd>
      </div>

      <div class="topbar-actions">
        <button class="notification-btn" onclick="showNotifications()">
          <i data-lucide="bell"></i>
          <span class="notif-dot"></span>
        </button>
        <div class="user-profile" onclick="showUserMenu()">
          <div class="avatar">AM</div>
          <div>
            <div class="name">Alex Mercer</div>
            <div class="role">AI Engineering</div>
          </div>
          <i data-lucide="chevron-down"></i>
        </div>
      </div>
    </header>

    <div class="content-area">

      <!-- DASHBOARD -->
      <section id="view-dashboard" class="view-section active">
        <div class="hero-banner">
          <div>
            <h1>Welcome back, Alex</h1>
            <p>Compare AI models, run standardized tests, and make data-driven decisions.</p>
          </div>
          <button class="hero-btn" onclick="navigateTo('run-evaluation')">
            <i data-lucide="play"></i>
            New Evaluation
          </button>
        </div>

        <div class="stats-grid" id="dashboard-stats"></div>

        <div class="dashboard-grid">
          <div class="card">
            <div class="card-header-row">
              <h3 class="card-title">Recent Evaluations</h3>
              <button class="btn btn-sm btn-secondary" onclick="navigateTo('history')">View All</button>
            </div>
            <div class="recent-evals-list" id="recent-evals-list"></div>
          </div>

          <div class="card">
            <h3 class="card-title">Quick Actions</h3>
            <div class="quick-actions-grid">
              <div class="quick-action-card" onclick="navigateTo('run-evaluation')">
                <div class="qa-icon" style="background:#EFF2FF;color:#1428A0"><i data-lucide="play"></i></div>
                <div>
                  <div class="qa-title">Run Evaluation</div>
                  <div class="qa-desc">Test models on benchmarks</div>
                </div>
              </div>
              <div class="quick-action-card" onclick="navigateTo('providers')">
                <div class="qa-icon" style="background:#DAFBE1;color:#1A7F37"><i data-lucide="plug-zap"></i></div>
                <div>
                  <div class="qa-title">Add Provider</div>
                  <div class="qa-desc">Connect API keys</div>
                </div>
              </div>
              <div class="quick-action-card" onclick="navigateTo('datasets')">
                <div class="qa-icon" style="background:#FFF8C5;color:#BF8700"><i data-lucide="upload"></i></div>
                <div>
                  <div class="qa-title">Upload Dataset</div>
                  <div class="qa-desc">Custom test questions</div>
                </div>
              </div>
              <div class="quick-action-card" onclick="navigateTo('models')">
                <div class="qa-icon" style="background:#F3E8FF;color:#7C3AED"><i data-lucide="search"></i></div>
                <div>
                  <div class="qa-title">Browse Models</div>
                  <div class="qa-desc">Explore 100+ models</div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="card">
          <div class="card-header-row">
            <div>
              <h3 class="card-title">Latest Results</h3>
              <p class="card-subtitle">From your most recent evaluation</p>
            </div>
            <span class="badge badge-success">Complete</span>
          </div>
          <div class="table-container" id="dashboard-leaderboard"></div>
        </div>
      </section>

      <!-- RUN EVALUATION -->
      <section id="view-run-evaluation" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">New Evaluation</h1>
            <p class="page-subtitle">Compare AI models with standardized tests</p>
          </div>
          <button class="btn btn-secondary" onclick="navigateTo('dashboard')">
            <i data-lucide="x"></i> Cancel
          </button>
        </div>

        <div class="wizard-container">
          <div class="wizard-header">
            <div class="wizard-steps" id="wizard-steps">
              <div class="step-indicator active" data-step="1"><span class="step-num">1</span> Name</div>
              <div class="step-indicator" data-step="2"><span class="step-num">2</span> Type</div>
              <div class="step-indicator" data-step="3"><span class="step-num">3</span> Providers</div>
              <div class="step-indicator" data-step="4"><span class="step-num">4</span> Models</div>
              <div class="step-indicator" data-step="5"><span class="step-num">5</span> Test Suite</div>
              <div class="step-indicator" data-step="6"><span class="step-num">6</span> Metrics</div>
              <div class="step-indicator" data-step="7"><span class="step-num">7</span> Review</div>
            </div>
          </div>

          <!-- Step 1 -->
          <div class="step-pane active" data-step="1">
            <div class="step-content-card">
              <h2 class="step-title">Name your evaluation</h2>
              <p class="step-desc">Give it a memorable name so you can find it later.</p>
              <div class="form-group" style="max-width:480px;margin-top:28px;">
                <label class="form-label">Evaluation Name</label>
                <input type="text" id="eval-name" class="form-control form-control-lg" placeholder="e.g., Q3 Customer Support Bot Test">
              </div>
              <div class="suggested-names">
                <span class="suggested-label">Try:</span>
                <button class="suggestion-chip" onclick="setEvalName('Agent Tool Calling Test')">Agent Tool Calling Test</button>
                <button class="suggestion-chip" onclick="setEvalName('Support Bot Comparison')">Support Bot Comparison</button>
                <button class="suggestion-chip" onclick="setEvalName('Code Generation Test')">Code Generation Test</button>
              </div>
            </div>
            <div class="wizard-nav">
              <div></div>
              <button class="btn btn-primary btn-lg" onclick="nextStep()">Continue <i data-lucide="arrow-right"></i></button>
            </div>
          </div>

          <!-- Step 2 -->
          <div class="step-pane" data-step="2">
            <div class="step-content-card">
              <h2 class="step-title">What are you testing?</h2>
              <p class="step-desc">Different AI types need different evaluation methods.</p>
              <div class="eval-type-grid" id="eval-type-grid"></div>
            </div>
            <div class="wizard-nav">
              <button class="btn btn-secondary btn-lg" onclick="prevStep()"><i data-lucide="arrow-left"></i> Back</button>
              <button class="btn btn-primary btn-lg" onclick="nextStep()">Continue <i data-lucide="arrow-right"></i></button>
            </div>
          </div>

          <!-- Step 3 -->
          <div class="step-pane" data-step="3">
            <div class="step-content-card">
              <h2 class="step-title">Select providers</h2>
              <p class="step-desc">Choose which AI providers to include. Only connected providers are available.</p>
              <div class="provider-grid" id="provider-grid"></div>
              <div class="add-provider-hint">
                <i data-lucide="info"></i>
                <span>Need another provider? <a href="#" onclick="navigateTo('providers'); return false;">Add it in Settings</a></span>
              </div>
            </div>
            <div class="wizard-nav">
              <button class="btn btn-secondary btn-lg" onclick="prevStep()"><i data-lucide="arrow-left"></i> Back</button>
              <button class="btn btn-primary btn-lg" onclick="nextStep()">Continue <i data-lucide="arrow-right"></i></button>
            </div>
          </div>

          <!-- Step 4 -->
          <div class="step-pane" data-step="4">
            <div class="step-content-card step-content-wide">
              <h2 class="step-title">Choose models</h2>
              <p class="step-desc">Select the models you want to compare. Use filters to find the right models.</p>

              <div class="models-layout">
                <!-- Filter Sidebar -->
                <div class="filter-sidebar" id="filter-sidebar">
                  <div class="filter-header">
                    <span>Filters</span>
                    <button class="btn-link" onclick="resetAllFilters()">Reset all</button>
                  </div>

                  <!-- Modalities -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Modalities</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="modality" value="text"> <i data-lucide="type"></i> Text</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="modality" value="image"> <i data-lucide="image"></i> Image</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="modality" value="audio"> <i data-lucide="mic"></i> Audio</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="modality" value="video"> <i data-lucide="video"></i> Video</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="modality" value="file"> <i data-lucide="file"></i> File/PDF</label>
                    </div>
                  </div>

                  <!-- Context Length -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Context Length</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="context" value="4k"> 4K</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="context" value="32k"> 32K</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="context" value="128k"> 128K</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="context" value="200k"> 200K+</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="context" value="1m"> 1M+</label>
                    </div>
                  </div>

                  <!-- Pricing -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Pricing (per 1M tokens)</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="price" value="free"> FREE</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="price" value="low"> &lt; $1</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="price" value="mid"> $1 - $5</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="price" value="high"> $5+</label>
                    </div>
                  </div>

                  <!-- Model Series -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Model Series</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="gpt"> GPT</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="claude"> Claude</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="gemini"> Gemini</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="llama"> Llama</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="mistral"> Mistral</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="deepseek"> DeepSeek</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="series" value="qwen"> Qwen</label>
                    </div>
                  </div>

                  <!-- Capabilities -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Capabilities</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="capability" value="tool_calling"> <i data-lucide="wrench"></i> Tool Calling</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="capability" value="vision"> <i data-lucide="eye"></i> Vision</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="capability" value="reasoning"> <i data-lucide="brain"></i> Reasoning</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="capability" value="coding"> <i data-lucide="code"></i> Coding</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="capability" value="json_mode"> <i data-lucide="braces"></i> JSON Mode</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="capability" value="streaming"> <i data-lucide="radio"></i> Streaming</label>
                    </div>
                  </div>

                  <!-- Speed -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Speed (tokens/sec)</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="speed" value="ultra"> Ultra Fast (80+)</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="speed" value="fast"> Fast (40-80)</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="speed" value="medium"> Medium (20-40)</label>
                    </div>
                  </div>

                  <!-- Benchmark Scores -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Benchmark Index</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <div class="filter-slider-row">
                        <span>Intelligence</span>
                        <input type="range" min="0" max="100" value="0" class="filter-slider" onchange="applyFilters()" data-filter="intelligence">
                        <span class="filter-slider-val">0%</span>
                      </div>
                      <div class="filter-slider-row">
                        <span>Coding</span>
                        <input type="range" min="0" max="100" value="0" class="filter-slider" onchange="applyFilters()" data-filter="coding_index">
                        <span class="filter-slider-val">0%</span>
                      </div>
                      <div class="filter-slider-row">
                        <span>Agentic</span>
                        <input type="range" min="0" max="100" value="0" class="filter-slider" onchange="applyFilters()" data-filter="agentic">
                        <span class="filter-slider-val">0%</span>
                      </div>
                    </div>
                  </div>

                  <!-- Other Options -->
                  <div class="filter-section">
                    <div class="filter-title" onclick="toggleFilterSection(this)">
                      <span>Other</span>
                      <i data-lucide="chevron-down"></i>
                    </div>
                    <div class="filter-options">
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="other" value="new"> <i data-lucide="sparkles"></i> New Models</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="other" value="open_source"> <i data-lucide="unlock"></i> Open Source</label>
                      <label class="filter-chip"><input type="checkbox" onchange="applyFilters()" data-filter="other" value="zdr"> <i data-lucide="shield"></i> Zero Data Retention</label>
                    </div>
                  </div>
                </div>

                <!-- Models Grid -->
                <div class="models-main">
                  <div class="model-search-bar">
                    <i data-lucide="search"></i>
                    <input type="text" id="model-search" placeholder="Search models..." oninput="filterModels()">
                    <div class="active-filters" id="active-filters"></div>
                    <button class="btn btn-sm btn-secondary" onclick="toggleFilterSidebar()">
                      <i data-lucide="sliders-horizontal"></i> Filters
                    </button>
                  </div>
                  <div class="models-grid" id="models-grid"></div>
                  <div class="selected-models-bar" id="selected-models-bar">
                    <span><strong id="selected-count">0</strong> models selected</span>
                    <button class="btn btn-sm btn-secondary" onclick="clearModelSelection()">Clear</button>
                  </div>
                </div>
              </div>
            </div>
            <div class="wizard-nav">
              <button class="btn btn-secondary btn-lg" onclick="prevStep()"><i data-lucide="arrow-left"></i> Back</button>
              <button class="btn btn-primary btn-lg" onclick="nextStep()">Continue <i data-lucide="arrow-right"></i></button>
            </div>
          </div>

          <!-- Step 5 -->
          <div class="step-pane" data-step="5">
            <div class="step-content-card">
              <h2 class="step-title">Pick a test suite</h2>
              <p class="step-desc">Test suites contain questions that measure AI capabilities.</p>

              <div class="dataset-tabs">
                <button class="dataset-tab active" onclick="switchDatasetTab('official')">Benchmarks</button>
                <button class="dataset-tab" onclick="switchDatasetTab('community')">Community</button>
                <button class="dataset-tab" onclick="switchDatasetTab('private')">Upload</button>
              </div>

              <div class="dataset-content" id="dataset-official">
                <div class="dataset-category-filters">
                  <button class="category-chip active" onclick="filterDatasets('all')">All</button>
                  <button class="category-chip" onclick="filterDatasets('Agents')">Agents</button>
                  <button class="category-chip" onclick="filterDatasets('Coding')">Coding</button>
                  <button class="category-chip" onclick="filterDatasets('General')">General</button>
                  <button class="category-chip" onclick="filterDatasets('RAG')">RAG</button>
                  <button class="category-chip" onclick="filterDatasets('Finance')">Finance</button>
                  <button class="category-chip" onclick="filterDatasets('Healthcare')">Healthcare</button>
                </div>
                <div class="datasets-grid" id="datasets-grid"></div>
              </div>

              <div class="dataset-content" id="dataset-community" style="display:none;">
                <div class="empty-state">
                  <i data-lucide="users"></i>
                  <h3>Community Test Suites</h3>
                  <p>Browse tests created by the community</p>
                  <button class="btn btn-primary">Browse</button>
                </div>
              </div>

              <div class="dataset-content" id="dataset-private" style="display:none;">
                <div class="upload-zone" onclick="document.getElementById('file-upload').click()">
                  <input type="file" id="file-upload" style="display:none" accept=".csv,.json,.jsonl" onchange="handleFileUpload(this)">
                  <i data-lucide="upload-cloud"></i>
                  <h3>Upload Test Data</h3>
                  <p>Drag & drop or click to browse</p>
                  <div class="upload-formats">
                    <span class="format-chip">CSV</span>
                    <span class="format-chip">JSON</span>
                    <span class="format-chip">JSONL</span>
                    <span class="format-chip">HuggingFace</span>
                  </div>
                </div>
                <div class="hf-import">
                  <label class="form-label">Import from Hugging Face</label>
                  <div class="hf-input-row">
                    <input type="text" id="wizard-hf-id" class="form-control" placeholder="e.g., openai/humaneval">
                    <button class="btn btn-primary" onclick="importHuggingFaceInline()">Import</button>
                  </div>
                </div>
              </div>
            </div>
            <div class="wizard-nav">
              <button class="btn btn-secondary btn-lg" onclick="prevStep()"><i data-lucide="arrow-left"></i> Back</button>
              <button class="btn btn-primary btn-lg" onclick="nextStep()">Continue <i data-lucide="arrow-right"></i></button>
            </div>
          </div>

          <!-- Step 6 -->
          <div class="step-pane" data-step="6">
            <div class="step-content-card step-content-wide">
              <div class="step-header-row">
                <div>
                  <h2 class="step-title">What to measure?</h2>
                  <p class="step-desc">Select the metrics that matter for your use case. Metrics are tailored to your evaluation type.</p>
                </div>
                <div class="metrics-summary">
                  <span id="metrics-count">0</span> selected
                </div>
              </div>
              <div class="metrics-grid" id="metrics-grid"></div>
              <div class="advanced-toggle">
                <button class="btn btn-sm btn-secondary" onclick="toggleAdvancedMetrics()">
                  <i data-lucide="sliders-horizontal"></i> Advanced Settings
                </button>
              </div>
            </div>
            <div class="wizard-nav">
              <button class="btn btn-secondary btn-lg" onclick="prevStep()"><i data-lucide="arrow-left"></i> Back</button>
              <button class="btn btn-primary btn-lg" onclick="nextStep()">Continue <i data-lucide="arrow-right"></i></button>
            </div>
          </div>

          <!-- Step 7 -->
          <div class="step-pane" data-step="7">
            <div class="step-content-card">
              <h2 class="step-title">Review & Run</h2>
              <p class="step-desc">Confirm your settings before starting.</p>

              <div class="review-summary">
                <div class="review-item">
                  <span class="review-label">Name</span>
                  <span class="review-value" id="review-name">-</span>
                </div>
                <div class="review-item">
                  <span class="review-label">Type</span>
                  <span class="review-value" id="review-type">-</span>
                </div>
                <div class="review-item">
                  <span class="review-label">Models</span>
                  <span class="review-value" id="review-models">-</span>
                </div>
                <div class="review-item">
                  <span class="review-label">Test Suite</span>
                  <span class="review-value" id="review-dataset">-</span>
                </div>
                <div class="review-item">
                  <span class="review-label">Questions</span>
                  <span class="review-value" id="review-questions">-</span>
                </div>
                <div class="review-divider"></div>
                <div class="review-item highlight">
                  <span class="review-label">Est. Cost</span>
                  <span class="review-value" id="review-cost">~$0.35</span>
                </div>
                <div class="review-item highlight">
                  <span class="review-label">Est. Time</span>
                  <span class="review-value" id="review-duration">~3 min</span>
                </div>
              </div>

              <div class="review-notice">
                <i data-lucide="info"></i>
                <p>Costs are estimates. Actual costs depend on provider pricing.</p>
              </div>
            </div>
            <div class="wizard-nav">
              <button class="btn btn-secondary btn-lg" onclick="prevStep()"><i data-lucide="arrow-left"></i> Back</button>
              <button class="btn btn-primary btn-lg run-eval-btn" onclick="startEvaluation()">
                <i data-lucide="play"></i> Start Evaluation
              </button>
            </div>
          </div>
        </div>
      </section>

      <!-- RUNNING -->
      <section id="view-running" class="view-section">
        <div class="progress-container">
          <div class="progress-icon">
            <i data-lucide="loader-2" class="spin-icon"></i>
          </div>
          <h2 class="progress-title">Evaluation Running</h2>
          <p class="progress-subtitle" id="progress-subtitle">Testing models...</p>

          <div class="progress-bar-track">
            <div class="progress-bar-fill" id="progress-fill"></div>
          </div>
          <div class="progress-percent" id="progress-percent">0%</div>

          <div class="status-pills">
            <div class="status-pill">
              <i data-lucide="box"></i>
              <span id="current-model">Hermes 3</span>
            </div>
            <div class="status-pill">
              <i data-lucide="clock"></i>
              <span id="elapsed-time">00:00</span>
            </div>
            <div class="status-pill">
              <i data-lucide="check-circle"></i>
              <span id="completed-count">0/3</span>
            </div>
          </div>

          <div class="progress-models-list" id="progress-models-list"></div>

          <details class="advanced-logs">
            <summary>View logs</summary>
            <pre id="eval-logs">Starting evaluation...</pre>
          </details>
        </div>
      </section>

      <!-- RESULTS -->
      <section id="view-results" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">Results</h1>
            <p class="page-subtitle" id="results-subtitle">Evaluation complete</p>
          </div>
          <div class="results-actions">
            <button class="btn btn-secondary" onclick="exportResults('csv')"><i data-lucide="download"></i> CSV</button>
            <button class="btn btn-secondary" onclick="exportResults('pdf')"><i data-lucide="file-text"></i> PDF</button>
            <button class="btn btn-primary" onclick="shareResults()"><i data-lucide="share-2"></i> Share</button>
          </div>
        </div>

        <div class="results-summary-cards">
          <div class="summary-card winner">
            <div class="summary-icon"><i data-lucide="trophy"></i></div>
            <div class="summary-content">
              <div class="summary-label">Winner</div>
              <div class="summary-value" id="winner-model">Hermes 3</div>
              <div class="summary-score" id="winner-score">97.5%</div>
            </div>
          </div>
          <div class="summary-card">
            <div class="summary-icon"><i data-lucide="zap"></i></div>
            <div class="summary-content">
              <div class="summary-label">Fastest</div>
              <div class="summary-value" id="fastest-model">Hermes 3</div>
              <div class="summary-score" id="fastest-time">0.75s</div>
            </div>
          </div>
          <div class="summary-card">
            <div class="summary-icon"><i data-lucide="wallet"></i></div>
            <div class="summary-content">
              <div class="summary-label">Best Value</div>
              <div class="summary-value" id="cheapest-model">Hermes 3</div>
              <div class="summary-score" id="cheapest-cost">$0.08</div>
            </div>
          </div>
        </div>

        <div class="card">
          <div class="card-header-row">
            <h3 class="card-title">Leaderboard</h3>
            <select class="form-control form-control-sm" id="sort-metric" style="width:180px;">
              <option value="score">Sort: Overall Score</option>
              <option value="accuracy">Sort: Accuracy</option>
              <option value="time">Sort: Speed</option>
              <option value="cost">Sort: Cost</option>
            </select>
          </div>
          <div class="table-container">
            <table class="data-table" id="results-table">
              <thead>
                <tr>
                  <th style="width:50px">Rank</th>
                  <th>Model</th>
                  <th>Provider</th>
                  <th>Score</th>
                  <th>Accuracy</th>
                  <th>Speed</th>
                  <th>Cost</th>
                  <th>Errors</th>
                  <th>Tools</th>
                  <th></th>
                </tr>
              </thead>
              <tbody id="results-tbody"></tbody>
            </table>
          </div>
        </div>

        <div class="results-recommendations card">
          <h3 class="card-title"><i data-lucide="lightbulb"></i> Recommendation</h3>
          <div class="recommendation-content" id="recommendation-content">
            <p><strong>For production workloads:</strong> Use <em>Hermes 3 - 70B Agent</em>. It offers the best balance of accuracy (96.2%) and cost ($0.08), with excellent tool execution (99.1%).</p>
            <p><strong>For complex tasks:</strong> Consider <em>Claude 3.5 Sonnet</em> for edge cases requiring deeper reasoning. Higher cost but marginally better accuracy.</p>
            <p><strong>Cost opportunity:</strong> Switching from GPT-4o to Hermes 3 reduces costs by ~77% with comparable accuracy.</p>
          </div>
        </div>

        <div class="results-nav">
          <button class="btn btn-secondary btn-lg" onclick="navigateTo('dashboard')"><i data-lucide="home"></i> Dashboard</button>
          <button class="btn btn-primary btn-lg" onclick="runAgain()"><i data-lucide="refresh-cw"></i> New Evaluation</button>
        </div>
      </section>

      <!-- HISTORY -->
      <section id="view-history" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">History</h1>
            <p class="page-subtitle">Past evaluations</p>
          </div>
          <button class="btn btn-primary" onclick="navigateTo('run-evaluation')">
            <i data-lucide="play"></i> New Evaluation
          </button>
        </div>

        <div class="history-filters">
          <div class="search-box" style="width:280px;">
            <i data-lucide="search"></i>
            <input type="text" placeholder="Search...">
          </div>
          <select class="form-control form-control-sm" style="width:140px;">
            <option>All Types</option>
            <option>AI Model</option>
            <option>Agent</option>
            <option>RAG</option>
          </select>
          <select class="form-control form-control-sm" style="width:140px;">
            <option>Last 30 days</option>
            <option>Last 7 days</option>
            <option>All time</option>
          </select>
        </div>

        <div class="history-list" id="history-list"></div>
      </section>

      <!-- MODELS -->
      <section id="view-models" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">Models</h1>
            <p class="page-subtitle">Browse available AI models</p>
          </div>
        </div>

        <div class="model-catalog-filters">
          <div class="search-box" style="width:300px;">
            <i data-lucide="search"></i>
            <input type="text" placeholder="Search models...">
          </div>
          <div class="filter-chips">
            <button class="filter-chip active">All</button>
            <button class="filter-chip">Tool Calling</button>
            <button class="filter-chip">Vision</button>
            <button class="filter-chip">Coding</button>
            <button class="filter-chip">Reasoning</button>
          </div>
        </div>

        <div class="models-catalog-grid" id="models-catalog-grid"></div>
      </section>

      <!-- PROVIDERS -->
      <section id="view-providers" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">Providers</h1>
            <p class="page-subtitle">Manage AI service connections</p>
          </div>
          <button class="btn btn-primary" onclick="showAddProviderModal()">
            <i data-lucide="plus"></i> Add Provider
          </button>
        </div>

        <div class="providers-grid" id="providers-page-grid"></div>
      </section>

      <!-- DATASETS -->
      <section id="view-datasets" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">Test Suites</h1>
            <p class="page-subtitle">Benchmark datasets and custom tests</p>
          </div>
          <button class="btn btn-primary" onclick="showUploadDatasetModal()">
            <i data-lucide="upload"></i> Upload
          </button>
        </div>

        <div class="dataset-category-tabs">
          <button class="category-tab active">All</button>
          <button class="category-tab">Agents</button>
          <button class="category-tab">Coding</button>
          <button class="category-tab">General</button>
          <button class="category-tab">RAG</button>
          <button class="category-tab">Finance</button>
          <button class="category-tab">Healthcare</button>
        </div>

        <div class="datasets-library-grid" id="datasets-library-grid"></div>
      </section>

      <!-- REPORTS -->
      <section id="view-reports" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">Reports</h1>
            <p class="page-subtitle">Generated analysis and recommendations</p>
          </div>
        </div>

        <div class="reports-list" id="reports-list"></div>
      </section>

      <!-- SETTINGS -->
      <section id="view-settings" class="view-section">
        <div class="page-header">
          <div>
            <h1 class="page-title">Settings</h1>
            <p class="page-subtitle">Workspace configuration</p>
          </div>
        </div>

        <div class="settings-layout">
          <div class="settings-nav">
            <button class="settings-nav-item active" onclick="switchSettingsTab('workspace', this)">Workspace</button>
            <button class="settings-nav-item" onclick="switchSettingsTab('apikeys', this)">API Keys</button>
            <button class="settings-nav-item" onclick="switchSettingsTab('team', this)">Team</button>
            <button class="settings-nav-item" onclick="switchSettingsTab('notifications', this)">Notifications</button>
          </div>

          <div class="settings-content">
            <!-- Workspace Tab -->
            <div id="settings-workspace" class="settings-tab active">
              <div class="card">
                <h3 class="card-title">Workspace</h3>
                <p class="card-subtitle">Organization settings</p>

                <div class="form-group" style="margin-top:20px;">
                  <label class="form-label">Organization Name</label>
                  <input type="text" id="org-name" class="form-control" value="Semco Enterprise Labs" style="max-width:360px;">
                </div>
                <div class="form-group">
                  <label class="form-label">Default Evaluation Type</label>
                  <select id="default-eval-type" class="form-control" style="max-width:360px;">
                    <option value="agent">Agent (Autonomous Workflow)</option>
                    <option value="model">AI Model (General Chat)</option>
                    <option value="rag">RAG (Document Search)</option>
                  </select>
                </div>
                <div class="form-group">
                  <label class="form-label">Timezone</label>
                  <select id="timezone" class="form-control" style="max-width:360px;">
                    <option value="utc">UTC</option>
                    <option value="est" selected>Eastern Time (ET)</option>
                    <option value="pst">Pacific Time (PT)</option>
                    <option value="ist">India Standard Time (IST)</option>
                  </select>
                </div>

                <button class="btn btn-primary" style="margin-top:12px;" onclick="saveWorkspaceSettings()">Save Changes</button>
              </div>
            </div>

            <!-- API Keys Tab -->
            <div id="settings-apikeys" class="settings-tab">
              <div class="card">
                <h3 class="card-title">API Keys</h3>
                <p class="card-subtitle">Manage provider connections</p>

                <div class="api-keys-list" id="api-keys-list"></div>

                <button class="btn btn-secondary" style="margin-top:16px;" onclick="showAddProviderModal()">
                  <i data-lucide="plus"></i> Add Provider
                </button>
              </div>
            </div>

            <!-- Team Tab -->
            <div id="settings-team" class="settings-tab">
              <div class="card">
                <h3 class="card-title">Team Members</h3>
                <p class="card-subtitle">Manage who has access to this workspace</p>

                <div class="team-list" style="margin-top:20px;">
                  <div class="team-member">
                    <div class="team-avatar">AM</div>
                    <div class="team-info">
                      <strong>Alex Mercer</strong>
                      <span>alex.mercer@semco.ai</span>
                    </div>
                    <span class="badge badge-primary">Owner</span>
                  </div>
                  <div class="team-member">
                    <div class="team-avatar">SJ</div>
                    <div class="team-info">
                      <strong>Sarah Johnson</strong>
                      <span>sarah.j@semco.ai</span>
                    </div>
                    <span class="badge">Admin</span>
                  </div>
                  <div class="team-member">
                    <div class="team-avatar">MK</div>
                    <div class="team-info">
                      <strong>Mike Kim</strong>
                      <span>m.kim@semco.ai</span>
                    </div>
                    <span class="badge">Member</span>
                  </div>
                </div>

                <button class="btn btn-secondary" style="margin-top:16px;" onclick="showInviteModal()">
                  <i data-lucide="user-plus"></i> Invite Member
                </button>
              </div>
            </div>

            <!-- Notifications Tab -->
            <div id="settings-notifications" class="settings-tab">
              <div class="card">
                <h3 class="card-title">Notifications</h3>
                <p class="card-subtitle">Configure how you receive updates</p>

                <div class="notification-settings" style="margin-top:20px;">
                  <div class="notification-row">
                    <div class="notification-info">
                      <strong>Evaluation Complete</strong>
                      <span>Get notified when an evaluation finishes</span>
                    </div>
                    <label class="toggle">
                      <input type="checkbox" checked onchange="saveNotificationSetting('eval_complete', this.checked)">
                      <span class="toggle-slider"></span>
                    </label>
                  </div>
                  <div class="notification-row">
                    <div class="notification-info">
                      <strong>Evaluation Failed</strong>
                      <span>Alert when an evaluation encounters an error</span>
                    </div>
                    <label class="toggle">
                      <input type="checkbox" checked onchange="saveNotificationSetting('eval_failed', this.checked)">
                      <span class="toggle-slider"></span>
                    </label>
                  </div>
                  <div class="notification-row">
                    <div class="notification-info">
                      <strong>New Model Available</strong>
                      <span>Updates when providers release new models</span>
                    </div>
                    <label class="toggle">
                      <input type="checkbox" onchange="saveNotificationSetting('new_model', this.checked)">
                      <span class="toggle-slider"></span>
                    </label>
                  </div>
                  <div class="notification-row">
                    <div class="notification-info">
                      <strong>Weekly Summary</strong>
                      <span>Receive a digest of evaluation results</span>
                    </div>
                    <label class="toggle">
                      <input type="checkbox" checked onchange="saveNotificationSetting('weekly_summary', this.checked)">
                      <span class="toggle-slider"></span>
                    </label>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

    </div>
  </main>
</div>

<!-- Modal -->
<div class="modal-overlay" id="modal-overlay">
  <div class="modal-content" id="modal-content"></div>
</div>

<script src="data.js"></script>
<script src="app.js"></script>
</body>
</html>





























//styles.css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;450;500;600;700&display=swap');

:root {
  --primary: #1428A0;
  --primary-hover: #0D1D78;
  --primary-light: #F0F3FF;
  --primary-subtle: #E8ECFC;

  --bg-main: #FFFFFF;
  --bg-page: #FAFBFC;
  --bg-elevated: #FFFFFF;
  --bg-subtle: #F6F8FA;
  --bg-inset: #F0F2F5;

  --border-default: #E1E4E8;
  --border-subtle: #EBEEF1;
  --border-strong: #D0D4DA;

  --text-primary: #1A1D21;
  --text-secondary: #57606A;
  --text-tertiary: #8B949E;
  --text-placeholder: #A8B1BB;

  --success: #1A7F37;
  --success-subtle: #DAFBE1;
  --warning: #BF8700;
  --warning-subtle: #FFF8C5;
  --danger: #CF222E;
  --danger-subtle: #FFEBE9;

  --shadow-xs: 0 1px 2px rgba(27,31,36,0.04);
  --shadow-sm: 0 1px 3px rgba(27,31,36,0.06), 0 1px 2px rgba(27,31,36,0.04);
  --shadow-md: 0 3px 8px rgba(27,31,36,0.08), 0 1px 3px rgba(27,31,36,0.04);
  --shadow-lg: 0 8px 24px rgba(27,31,36,0.12), 0 2px 6px rgba(27,31,36,0.04);
  --shadow-xl: 0 12px 40px rgba(27,31,36,0.16);

  --radius-sm: 6px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
}

*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  font-size: 15px;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: var(--bg-page);
  color: var(--text-primary);
  line-height: 1.5;
}

/* ========== LOGIN ========== */
.login-container {
  display: flex;
  min-height: 100vh;
}

.login-left {
  flex: 0 0 480px;
  padding: 48px 56px;
  display: flex;
  flex-direction: column;
  background: var(--bg-main);
}

.login-brand {
  display: flex;
  align-items: center;
  gap: 10px;
}

.brand-logo {
  width: 36px;
  height: 36px;
  background: var(--primary);
  color: white;
  font-weight: 700;
  font-size: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
}

.brand-title {
  font-size: 19px;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.3px;
}

.login-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  max-width: 340px;
}

.login-title {
  font-size: 28px;
  font-weight: 600;
  letter-spacing: -0.5px;
  color: var(--text-primary);
  margin-bottom: 8px;
  line-height: 1.2;
}

.login-subtitle {
  font-size: 15px;
  color: var(--text-secondary);
  margin-bottom: 32px;
  line-height: 1.5;
}

.login-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.form-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-primary);
}

.form-control {
  height: 42px;
  padding: 0 14px;
  font-size: 14px;
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  background: var(--bg-main);
  color: var(--text-primary);
  transition: border-color 0.15s, box-shadow 0.15s;
}

.form-control:focus {
  outline: none;
  border-color: var(--primary);
  box-shadow: 0 0 0 3px rgba(20,40,160,0.1);
}

.form-control::placeholder {
  color: var(--text-placeholder);
}

.login-options {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 13px;
  margin-top: 4px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 8px;
  color: var(--text-secondary);
  cursor: pointer;
}

.checkbox-label input {
  width: 16px;
  height: 16px;
  accent-color: var(--primary);
}

.forgot-link {
  color: var(--primary);
  text-decoration: none;
  font-weight: 500;
}

.forgot-link:hover {
  text-decoration: underline;
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  height: 42px;
  padding: 0 20px;
  font-size: 14px;
  font-weight: 500;
  border-radius: var(--radius-md);
  border: none;
  cursor: pointer;
  transition: all 0.15s;
}

.btn-primary {
  background: var(--primary);
  color: white;
}

.btn-primary:hover {
  background: var(--primary-hover);
}

.btn-secondary {
  background: var(--bg-subtle);
  color: var(--text-primary);
  border: 1px solid var(--border-default);
}

.btn-secondary:hover {
  background: var(--bg-inset);
  border-color: var(--border-strong);
}

.btn-lg {
  height: 46px;
  padding: 0 24px;
  font-size: 15px;
  font-weight: 600;
}

.btn-sm {
  height: 34px;
  padding: 0 14px;
  font-size: 13px;
}

.login-btn {
  width: 100%;
  margin-top: 8px;
}

.login-divider {
  display: flex;
  align-items: center;
  gap: 16px;
  margin: 8px 0;
  color: var(--text-tertiary);
  font-size: 12px;
}

.login-divider::before,
.login-divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: var(--border-subtle);
}

.google-btn {
  width: 100%;
}

.google-btn svg {
  width: 18px;
  height: 18px;
}

.login-footer-text {
  margin-top: 28px;
  text-align: center;
  font-size: 13px;
  color: var(--text-secondary);
}

.login-footer-text a {
  color: var(--primary);
  font-weight: 500;
  text-decoration: none;
}

.login-footer-text a:hover {
  text-decoration: underline;
}

.login-footer {
  padding-top: 32px;
  font-size: 12px;
  color: var(--text-tertiary);
}

.spinner {
  width: 18px;
  height: 18px;
  border: 2px solid rgba(255,255,255,0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
  display: inline-block;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.spin-icon {
  animation: spin 1s linear infinite;
}

.login-right {
  flex: 1;
  background: var(--primary);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 64px;
}

.login-visual {
  max-width: 440px;
}

.visual-content {
  color: white;
}

.visual-badge {
  display: inline-block;
  padding: 6px 12px;
  background: rgba(255,255,255,0.15);
  border-radius: 20px;
  font-size: 12px;
  font-weight: 500;
  margin-bottom: 20px;
}

.visual-content h2 {
  font-size: 32px;
  font-weight: 600;
  line-height: 1.25;
  margin-bottom: 16px;
  letter-spacing: -0.5px;
}

.visual-content > p {
  font-size: 15px;
  line-height: 1.6;
  opacity: 0.85;
  margin-bottom: 32px;
}

.visual-features {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.visual-feature {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  font-size: 14px;
  opacity: 0.9;
}

.visual-feature svg {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
  margin-top: 2px;
  opacity: 0.8;
}

/* ========== APP LAYOUT ========== */
.app-container {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  width: 240px;
  background: var(--bg-main);
  border-right: 1px solid var(--border-subtle);
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  z-index: 100;
}

.sidebar-top {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.brand-section {
  padding: 20px 20px 16px;
  border-bottom: 1px solid var(--border-subtle);
}

.nav-links {
  list-style: none;
  padding: 12px;
  flex: 1;
  overflow-y: auto;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  margin-bottom: 2px;
  border-radius: var(--radius-sm);
  color: var(--text-secondary);
  font-size: 14px;
  font-weight: 450;
  cursor: pointer;
  transition: all 0.12s;
  text-decoration: none;
}

.nav-item svg {
  width: 18px;
  height: 18px;
  stroke-width: 1.75;
}

.nav-item:hover {
  background: var(--bg-subtle);
  color: var(--text-primary);
}

.nav-item.active {
  background: var(--primary-light);
  color: var(--primary);
  font-weight: 500;
}

.nav-item.highlight-btn {
  background: var(--primary);
  color: white;
  font-weight: 500;
  margin-bottom: 12px;
  justify-content: center;
}

.nav-item.highlight-btn:hover {
  background: var(--primary-hover);
}

.sidebar-footer {
  padding: 12px;
  border-top: 1px solid var(--border-subtle);
}

.main-wrapper {
  flex: 1;
  margin-left: 240px;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.topbar {
  height: 60px;
  background: var(--bg-main);
  border-bottom: 1px solid var(--border-subtle);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 28px;
  position: sticky;
  top: 0;
  z-index: 50;
}

.search-box {
  display: flex;
  align-items: center;
  gap: 10px;
  height: 38px;
  padding: 0 14px;
  background: var(--bg-subtle);
  border: 1px solid transparent;
  border-radius: var(--radius-md);
  width: 320px;
  transition: all 0.15s;
}

.search-box:focus-within {
  background: white;
  border-color: var(--border-default);
  box-shadow: var(--shadow-sm);
}

.search-box svg {
  width: 16px;
  height: 16px;
  color: var(--text-tertiary);
  stroke-width: 2;
}

.search-box input {
  flex: 1;
  border: none;
  background: transparent;
  font-size: 13px;
  color: var(--text-primary);
  outline: none;
}

.search-box input::placeholder {
  color: var(--text-placeholder);
}

.search-box kbd {
  font-size: 11px;
  padding: 2px 6px;
  background: var(--bg-inset);
  border-radius: 4px;
  color: var(--text-tertiary);
  font-family: inherit;
}

.topbar-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.notification-btn {
  width: 38px;
  height: 38px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: 1px solid transparent;
  border-radius: var(--radius-md);
  cursor: pointer;
  color: var(--text-secondary);
  position: relative;
  transition: all 0.12s;
}

.notification-btn:hover {
  background: var(--bg-subtle);
  border-color: var(--border-subtle);
}

.notification-btn svg {
  width: 20px;
  height: 20px;
  stroke-width: 1.75;
}

.notif-dot {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 7px;
  height: 7px;
  background: var(--primary);
  border-radius: 50%;
  border: 2px solid white;
}

.user-profile {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 10px 6px 6px;
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.12s;
  margin-left: 4px;
}

.user-profile:hover {
  background: var(--bg-subtle);
}

.avatar {
  width: 32px;
  height: 32px;
  background: var(--primary);
  color: white;
  font-size: 12px;
  font-weight: 600;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
}

.user-profile > div {
  text-align: left;
}

.user-profile .name {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-primary);
}

.user-profile .role {
  font-size: 11px;
  color: var(--text-tertiary);
}

.user-profile svg {
  width: 16px;
  height: 16px;
  color: var(--text-tertiary);
}

.content-area {
  flex: 1;
  padding: 28px 32px 48px;
  max-width: 1200px;
}

.view-section {
  display: none;
}

.view-section.active {
  display: block;
  animation: fadeUp 0.2s ease;
}

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: translateY(0); }
}

/* ========== TYPOGRAPHY ========== */
.page-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
}

.page-title {
  font-size: 22px;
  font-weight: 600;
  color: var(--text-primary);
  letter-spacing: -0.3px;
}

.page-subtitle {
  font-size: 14px;
  color: var(--text-secondary);
  margin-top: 4px;
}

/* ========== CARDS ========== */
.card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 24px;
  margin-bottom: 20px;
}

.card-header-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.card-title {
  font-size: 15px;
  font-weight: 600;
  color: var(--text-primary);
}

.card-subtitle {
  font-size: 13px;
  color: var(--text-secondary);
  margin-top: 2px;
}

/* ========== DASHBOARD ========== */
.hero-banner {
  background: var(--primary);
  border-radius: var(--radius-lg);
  padding: 28px 32px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.hero-banner h1 {
  font-size: 20px;
  font-weight: 600;
  color: white;
  margin-bottom: 6px;
}

.hero-banner p {
  font-size: 14px;
  color: rgba(255,255,255,0.85);
  max-width: 480px;
  line-height: 1.5;
}

.hero-btn {
  height: 44px;
  padding: 0 24px;
  background: white;
  color: var(--primary);
  font-size: 14px;
  font-weight: 600;
  border: none;
  border-radius: var(--radius-md);
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: transform 0.12s, box-shadow 0.12s;
}

.hero-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.hero-btn svg {
  width: 18px;
  height: 18px;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.stat-card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 20px;
}

.stat-value {
  font-size: 28px;
  font-weight: 600;
  color: var(--text-primary);
  letter-spacing: -0.5px;
}

.stat-label {
  font-size: 13px;
  color: var(--text-secondary);
  margin-top: 4px;
}

.dashboard-grid {
  display: grid;
  grid-template-columns: 1.3fr 0.7fr;
  gap: 20px;
  margin-bottom: 20px;
}

.recent-evals-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.recent-eval-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 14px 16px;
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.12s;
}

.recent-eval-item:hover {
  background: var(--bg-inset);
}

.eval-info {
  flex: 1;
  min-width: 0;
}

.eval-name {
  font-size: 14px;
  font-weight: 500;
  color: var(--text-primary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.eval-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 4px;
}

.eval-type-badge {
  font-size: 11px;
  font-weight: 500;
  padding: 2px 8px;
  background: var(--primary-light);
  color: var(--primary);
  border-radius: 4px;
}

.eval-date {
  font-size: 12px;
  color: var(--text-tertiary);
}

.eval-stats {
  display: flex;
  gap: 20px;
}

.eval-stat {
  text-align: right;
}

.eval-stat-label {
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-tertiary);
}

.eval-stat-value {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-primary);
}

.eval-stat-value.highlight {
  color: var(--primary);
}

.eval-arrow {
  width: 16px;
  height: 16px;
  color: var(--text-tertiary);
}

.quick-actions-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.quick-action-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px;
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.12s;
}

.quick-action-card:hover {
  background: var(--bg-inset);
}

.qa-icon {
  width: 40px;
  height: 40px;
  border-radius: var(--radius-sm);
  display: flex;
  align-items: center;
  justify-content: center;
}

.qa-icon svg {
  width: 20px;
  height: 20px;
  stroke-width: 1.75;
}

.qa-title {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-primary);
}

.qa-desc {
  font-size: 12px;
  color: var(--text-tertiary);
  margin-top: 2px;
}

/* ========== TABLE ========== */
.table-container {
  overflow-x: auto;
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
}

.data-table {
  width: 100%;
  border-collapse: collapse;
}

.data-table th {
  text-align: left;
  padding: 12px 16px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-tertiary);
  background: var(--bg-subtle);
  border-bottom: 1px solid var(--border-subtle);
}

.data-table td {
  padding: 14px 16px;
  font-size: 13px;
  color: var(--text-primary);
  border-bottom: 1px solid var(--border-subtle);
}

.data-table tbody tr:last-child td {
  border-bottom: none;
}

.data-table tbody tr:hover {
  background: var(--bg-subtle);
}

.rank-badge {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 600;
}

.rank-1 {
  background: #FEF3C7;
  color: #B45309;
}

.rank-2 {
  background: var(--bg-inset);
  color: var(--text-secondary);
}

.rank-3 {
  background: #FFEDD5;
  color: #C2410C;
}

.score-highlight {
  font-weight: 600;
  color: var(--primary);
}

.badge {
  display: inline-flex;
  align-items: center;
  padding: 3px 8px;
  font-size: 11px;
  font-weight: 600;
  border-radius: 4px;
}

.badge-primary {
  background: var(--primary-light);
  color: var(--primary);
}

.badge-success {
  background: var(--success-subtle);
  color: var(--success);
}

/* ========== WIZARD ========== */
.wizard-container {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 28px;
}

.wizard-header {
  padding-bottom: 20px;
  border-bottom: 1px solid var(--border-subtle);
  margin-bottom: 24px;
}

.wizard-steps {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.step-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 14px;
  background: var(--bg-subtle);
  border: 1px solid var(--border-subtle);
  border-radius: 20px;
  font-size: 13px;
  font-weight: 450;
  color: var(--text-secondary);
  cursor: pointer;
  transition: all 0.12s;
}

.step-num {
  width: 20px;
  height: 20px;
  background: var(--border-default);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 600;
  color: var(--text-secondary);
}

.step-indicator.active {
  background: var(--primary);
  border-color: var(--primary);
  color: white;
}

.step-indicator.active .step-num {
  background: rgba(255,255,255,0.25);
  color: white;
}

.step-indicator.completed {
  background: var(--success-subtle);
  border-color: var(--success-subtle);
  color: var(--success);
}

.step-indicator.completed .step-num {
  background: var(--success);
  color: white;
}

.step-pane {
  display: none;
  min-height: 400px;
}

.step-pane.active {
  display: block;
}

.step-content-card {
  padding: 16px 0 24px;
}

.step-title {
  font-size: 22px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 8px;
  letter-spacing: -0.3px;
}

.step-desc {
  font-size: 14px;
  color: var(--text-secondary);
  max-width: 560px;
}

.form-control-lg {
  height: 50px;
  padding: 0 18px;
  font-size: 16px;
}

.suggested-names {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 16px;
  flex-wrap: wrap;
}

.suggested-label {
  font-size: 12px;
  color: var(--text-tertiary);
}

.suggestion-chip {
  padding: 8px 14px;
  background: var(--bg-subtle);
  border: 1px solid var(--border-default);
  border-radius: 20px;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.12s;
}

.suggestion-chip:hover {
  background: var(--primary-light);
  border-color: var(--primary);
  color: var(--primary);
}

.wizard-nav {
  display: flex;
  justify-content: space-between;
  padding-top: 20px;
  border-top: 1px solid var(--border-subtle);
  margin-top: 20px;
}

.wizard-nav .btn svg {
  width: 18px;
  height: 18px;
}

/* Eval Type Selection */
.eval-type-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-top: 28px;
}

.eval-type-card {
  background: var(--bg-main);
  border: 2px solid var(--border-default);
  border-radius: var(--radius-lg);
  padding: 24px;
  cursor: pointer;
  transition: all 0.15s;
  position: relative;
}

.eval-type-card:hover {
  border-color: var(--border-strong);
}

.eval-type-card.selected {
  border-color: var(--primary);
  background: var(--primary-light);
}

.eval-type-card .badge {
  position: absolute;
  top: 14px;
  right: 14px;
}

.eval-type-icon {
  width: 52px;
  height: 52px;
  background: var(--primary-subtle);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 16px;
}

.eval-type-icon svg {
  width: 26px;
  height: 26px;
  color: var(--primary);
  stroke-width: 1.5;
}

.eval-type-content h3 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 6px;
}

.eval-type-content p {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.45;
}

.check-indicator {
  position: absolute;
  bottom: 14px;
  right: 14px;
  width: 24px;
  height: 24px;
  background: var(--primary);
  border-radius: 50%;
  display: none;
  align-items: center;
  justify-content: center;
  color: white;
}

.check-indicator svg {
  width: 14px;
  height: 14px;
}

.selected .check-indicator {
  display: flex;
}

/* Provider Selection */
.provider-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  margin-top: 24px;
}

.provider-card {
  background: var(--bg-main);
  border: 2px solid var(--border-default);
  border-radius: var(--radius-lg);
  padding: 18px;
  cursor: pointer;
  transition: all 0.12s;
  position: relative;
}

.provider-card:hover:not(.disabled) {
  border-color: var(--border-strong);
}

.provider-card.selected {
  border-color: var(--primary);
  background: var(--primary-light);
}

.provider-card.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.provider-logo {
  width: 42px;
  height: 42px;
  background: var(--bg-subtle);
  border-radius: var(--radius-sm);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 16px;
  color: var(--text-secondary);
  margin-bottom: 12px;
}

.provider-info h4 {
  font-size: 14px;
  font-weight: 500;
}

.provider-info p {
  font-size: 12px;
  color: var(--text-tertiary);
  margin-top: 2px;
}

.provider-status {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid var(--border-subtle);
}

.provider-status svg {
  width: 13px;
  height: 13px;
}

.provider-status.connected {
  color: var(--success);
}

.provider-status.not_connected {
  color: var(--text-tertiary);
}

.add-provider-hint {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 16px;
  padding: 12px 14px;
  background: #F0F6FF;
  border-radius: var(--radius-sm);
  font-size: 13px;
  color: #2563EB;
}

.add-provider-hint svg {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
}

.add-provider-hint a {
  color: var(--primary);
  font-weight: 500;
}

/* Model Selection */
.model-search-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 24px;
  margin-bottom: 16px;
  padding: 10px 16px;
  background: var(--bg-subtle);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
}

.model-search-bar:focus-within {
  background: white;
  border-color: var(--border-default);
}

.model-search-bar svg {
  width: 18px;
  height: 18px;
  color: var(--text-tertiary);
}

.model-search-bar input {
  flex: 1;
  border: none;
  background: transparent;
  font-size: 14px;
  outline: none;
}

.model-filters select {
  height: 34px;
  padding: 0 12px;
  border: 1px solid var(--border-default);
  border-radius: var(--radius-sm);
  background: white;
  font-size: 13px;
  cursor: pointer;
}

.models-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 14px;
  max-height: 420px;
  overflow-y: auto;
  padding-right: 4px;
}

.model-card {
  background: var(--bg-main);
  border: 2px solid var(--border-default);
  border-radius: var(--radius-lg);
  padding: 18px;
  cursor: pointer;
  transition: all 0.12s;
  position: relative;
  padding-top: 20px;
}

.model-card:hover {
  border-color: var(--border-strong);
}

.model-card.selected {
  border-color: var(--primary);
  background: var(--primary-light);
}

.model-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 8px;
}

.model-header h4 {
  font-size: 14px;
  font-weight: 500;
  line-height: 1.3;
}

.provider-tag {
  font-size: 10px;
  font-weight: 500;
  padding: 3px 7px;
  background: var(--bg-inset);
  border-radius: 4px;
  color: var(--text-tertiary);
  white-space: nowrap;
}

.model-desc {
  font-size: 12px;
  color: var(--text-secondary);
  line-height: 1.4;
  margin-bottom: 10px;
}

.model-capabilities {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  margin-bottom: 12px;
}

.cap-tag {
  font-size: 10px;
  font-weight: 500;
  padding: 3px 8px;
  background: var(--primary-subtle);
  color: var(--primary);
  border-radius: 4px;
}

.model-specs {
  display: flex;
  gap: 12px;
  padding-top: 12px;
  border-top: 1px solid var(--border-subtle);
}

.spec {
  flex: 1;
}

.spec-label {
  display: block;
  font-size: 9px;
  text-transform: uppercase;
  letter-spacing: 0.4px;
  color: var(--text-tertiary);
  margin-bottom: 2px;
}

.spec-value {
  font-size: 11px;
  font-weight: 500;
  color: var(--text-primary);
}

.selected-models-bar {
  display: none;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: var(--primary-light);
  border: 1px solid var(--primary);
  border-radius: var(--radius-md);
  margin-top: 14px;
  font-size: 13px;
}

/* Dataset Selection */
.dataset-tabs {
  display: inline-flex;
  gap: 2px;
  margin-top: 24px;
  margin-bottom: 16px;
  background: var(--bg-inset);
  padding: 3px;
  border-radius: var(--radius-md);
}

.dataset-tab {
  padding: 9px 18px;
  border: none;
  background: transparent;
  border-radius: var(--radius-sm);
  font-size: 13px;
  font-weight: 450;
  color: var(--text-secondary);
  cursor: pointer;
  transition: all 0.12s;
}

.dataset-tab.active {
  background: white;
  color: var(--text-primary);
  box-shadow: var(--shadow-xs);
}

.dataset-category-filters {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.category-chip {
  padding: 7px 14px;
  border: 1px solid var(--border-default);
  background: white;
  border-radius: 18px;
  font-size: 13px;
  font-weight: 450;
  cursor: pointer;
  transition: all 0.12s;
}

.category-chip:hover {
  border-color: var(--primary);
}

.category-chip.active {
  background: var(--primary);
  border-color: var(--primary);
  color: white;
}

.datasets-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 14px;
  max-height: 380px;
  overflow-y: auto;
}

.dataset-card {
  background: var(--bg-main);
  border: 2px solid var(--border-default);
  border-radius: var(--radius-lg);
  padding: 18px;
  cursor: pointer;
  transition: all 0.12s;
  position: relative;
}

.dataset-card:hover {
  border-color: var(--border-strong);
}

.dataset-card.selected {
  border-color: var(--primary);
  background: var(--primary-light);
}

.dataset-card .badge {
  position: absolute;
  top: 12px;
  right: 12px;
}

.dataset-category {
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--primary);
  margin-bottom: 6px;
}

.dataset-header h4 {
  font-size: 14px;
  font-weight: 500;
  padding-right: 60px;
}

.dataset-desc {
  font-size: 12px;
  color: var(--text-secondary);
  line-height: 1.4;
  margin: 8px 0 12px;
}

.dataset-meta {
  display: flex;
  gap: 14px;
  padding-top: 12px;
  border-top: 1px solid var(--border-subtle);
}

.meta-item {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  color: var(--text-tertiary);
}

.meta-item svg {
  width: 13px;
  height: 13px;
}

.upload-zone {
  border: 2px dashed var(--border-default);
  border-radius: var(--radius-lg);
  padding: 40px;
  text-align: center;
  cursor: pointer;
  transition: all 0.15s;
}

.upload-zone:hover {
  border-color: var(--primary);
  background: var(--primary-light);
}

.upload-zone svg {
  width: 40px;
  height: 40px;
  color: var(--primary);
  margin-bottom: 12px;
  stroke-width: 1.25;
}

.upload-zone h3 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 4px;
}

.upload-zone p {
  font-size: 13px;
  color: var(--text-secondary);
}

.upload-formats {
  display: flex;
  justify-content: center;
  gap: 8px;
  margin-top: 14px;
}

.format-chip {
  padding: 4px 10px;
  background: var(--bg-subtle);
  border-radius: 4px;
  font-size: 11px;
  font-weight: 500;
  color: var(--text-secondary);
}

.hf-import {
  margin-top: 20px;
}

.hf-input-row {
  display: flex;
  gap: 10px;
}

.hf-input-row input {
  flex: 1;
}

.empty-state {
  text-align: center;
  padding: 48px 32px;
}

.empty-state svg {
  width: 40px;
  height: 40px;
  color: var(--text-tertiary);
  margin-bottom: 12px;
  stroke-width: 1.25;
}

.empty-state h3 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 4px;
}

.empty-state p {
  font-size: 13px;
  color: var(--text-secondary);
  margin-bottom: 16px;
}

/* Metrics */
.metrics-grid {
  margin-top: 20px;
}

.metrics-section {
  margin-bottom: 24px;
}

.metrics-section:last-child {
  margin-bottom: 0;
}

.metrics-section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 4px;
}

.metrics-section-title svg {
  width: 16px;
  height: 16px;
  color: var(--primary);
}

.metrics-section-desc {
  font-size: 12px;
  color: var(--text-tertiary);
  margin-bottom: 12px;
}

.metrics-cards {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
}

.metric-card {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 14px;
  background: var(--bg-main);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.12s;
}

.metric-card:hover {
  border-color: var(--border-strong);
  background: var(--bg-inset);
}

.metric-card.selected {
  border-color: var(--primary);
  background: var(--primary-light);
}

.metric-icon {
  width: 32px;
  height: 32px;
  background: var(--bg-inset);
  border-radius: var(--radius-sm);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.metric-icon svg {
  width: 16px;
  height: 16px;
  color: var(--text-secondary);
}

.metric-card.selected .metric-icon {
  background: var(--primary);
}

.metric-card.selected .metric-icon svg {
  color: white;
}

.metric-checkbox {
  flex-shrink: 0;
}

.metric-checkbox input {
  width: 16px;
  height: 16px;
  accent-color: var(--primary);
}

.metric-content {
  flex: 1;
  min-width: 0;
}

.metric-content h4 {
  font-size: 13px;
  font-weight: 500;
  margin-bottom: 2px;
}

.metric-content p {
  font-size: 11px;
  color: var(--text-tertiary);
  line-height: 1.4;
}

.metric-card.selected .metric-content p {
  color: var(--text-secondary);
}

/* Advanced metrics modal */
.advanced-metric-section {
  margin-bottom: 24px;
}

.advanced-metric-section h4 {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 4px;
}

.weight-sliders {
  margin-top: 12px;
}

.weight-row {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.weight-row span:first-child {
  flex: 1;
  font-size: 13px;
}

.weight-row input[type="range"] {
  width: 120px;
}

.weight-val {
  font-size: 12px;
  color: var(--text-secondary);
  min-width: 40px;
  text-align: right;
}

.step-header-row {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 8px;
}

.metrics-summary {
  background: var(--primary-light);
  color: var(--primary);
  padding: 8px 16px;
  border-radius: var(--radius-md);
  font-size: 13px;
  font-weight: 500;
}

.metrics-summary span {
  font-weight: 700;
  font-size: 18px;
}

.metric-tooltip {
  color: var(--text-tertiary);
}

.metric-tooltip svg {
  width: 15px;
  height: 15px;
}

.advanced-toggle {
  margin-top: 16px;
}

/* Review */
.review-summary {
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  padding: 20px 24px;
  margin-top: 24px;
}

.review-item {
  display: flex;
  justify-content: space-between;
  padding: 11px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.review-item:last-child {
  border-bottom: none;
}

.review-label {
  font-size: 13px;
  color: var(--text-secondary);
}

.review-value {
  font-size: 13px;
  font-weight: 500;
  text-align: right;
  max-width: 55%;
}

.review-divider {
  height: 1px;
  background: var(--border-default);
  margin: 6px 0;
}

.review-item.highlight .review-value {
  font-size: 15px;
  font-weight: 600;
  color: var(--primary);
}

.review-notice {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  margin-top: 16px;
  padding: 12px 14px;
  background: #F0F6FF;
  border-radius: var(--radius-sm);
  font-size: 12px;
  color: #2563EB;
}

.review-notice svg {
  width: 15px;
  height: 15px;
  flex-shrink: 0;
  margin-top: 1px;
}

.run-eval-btn {
  background: var(--success);
  padding: 0 28px;
}

.run-eval-btn:hover {
  background: #158A42;
}

/* ========== PROGRESS ========== */
.progress-container {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 48px 40px;
  text-align: center;
  max-width: 640px;
  margin: 32px auto;
}

.progress-icon {
  width: 64px;
  height: 64px;
  background: var(--primary-light);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 20px;
}

.progress-icon svg {
  width: 32px;
  height: 32px;
  color: var(--primary);
}

.spin-icon {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.progress-title {
  font-size: 20px;
  font-weight: 600;
  margin-bottom: 6px;
}

.progress-subtitle {
  font-size: 14px;
  color: var(--text-secondary);
  margin-bottom: 28px;
}

.progress-bar-track {
  width: 100%;
  height: 10px;
  background: var(--bg-inset);
  border-radius: 5px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  background: var(--primary);
  border-radius: 5px;
  transition: width 0.3s ease;
}

.progress-percent {
  font-size: 22px;
  font-weight: 600;
  color: var(--primary);
  margin-top: 14px;
}

.status-pills {
  display: flex;
  justify-content: center;
  gap: 12px;
  margin-top: 24px;
  flex-wrap: wrap;
}

.status-pill {
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 9px 15px;
  background: var(--bg-subtle);
  border-radius: 20px;
  font-size: 13px;
  font-weight: 450;
}

.status-pill svg {
  width: 15px;
  height: 15px;
  color: var(--text-tertiary);
}

.progress-models-list {
  margin-top: 28px;
  text-align: left;
}

.progress-model-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 11px 14px;
  background: var(--bg-subtle);
  border-radius: var(--radius-sm);
  margin-bottom: 6px;
  border: 1px solid transparent;
}

.progress-model-item.active {
  background: var(--primary-light);
  border-color: var(--primary);
}

.progress-model-item.completed {
  background: var(--success-subtle);
  border-color: var(--success-subtle);
}

.model-status {
  width: 22px;
  height: 22px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.model-status svg {
  width: 18px;
  height: 18px;
  color: var(--text-tertiary);
}

.progress-model-item.active .model-status svg {
  color: var(--primary);
}

.progress-model-item.completed .model-status svg {
  color: var(--success);
}

.model-name {
  flex: 1;
  font-size: 13px;
  font-weight: 450;
}

.model-progress-status {
  font-size: 12px;
  color: var(--text-tertiary);
}

.advanced-logs {
  margin-top: 28px;
  text-align: left;
}

.advanced-logs summary {
  cursor: pointer;
  font-size: 12px;
  color: var(--text-tertiary);
  padding: 8px 0;
}

.advanced-logs pre {
  background: #1F2937;
  color: #E5E7EB;
  padding: 14px;
  border-radius: var(--radius-sm);
  font-size: 11px;
  line-height: 1.6;
  overflow-x: auto;
  max-height: 180px;
  margin-top: 8px;
}

/* ========== RESULTS ========== */
.results-actions {
  display: flex;
  gap: 10px;
}

.results-summary-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.summary-card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 20px;
  display: flex;
  gap: 14px;
}

.summary-card.winner {
  background: linear-gradient(135deg, #FFFBEB 0%, #FEF3C7 100%);
  border-color: #FDE68A;
}

.summary-icon {
  width: 44px;
  height: 44px;
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.summary-icon svg {
  width: 22px;
  height: 22px;
  color: var(--text-secondary);
  stroke-width: 1.5;
}

.summary-card.winner .summary-icon {
  background: rgba(180,83,9,0.12);
}

.summary-card.winner .summary-icon svg {
  color: #B45309;
}

.summary-label {
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-tertiary);
  margin-bottom: 4px;
}

.summary-value {
  font-size: 16px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 2px;
}

.summary-score {
  font-size: 13px;
  color: var(--text-secondary);
}

.status-badge {
  display: inline-flex;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
}

.status-badge.winner {
  background: #FEF3C7;
  color: #B45309;
}

.status-badge.runner-up {
  background: var(--bg-inset);
  color: var(--text-secondary);
}

.status-badge.third {
  background: #FFEDD5;
  color: #C2410C;
}

.results-recommendations {
  margin-top: 20px;
}

.results-recommendations .card-title {
  display: flex;
  align-items: center;
  gap: 8px;
}

.results-recommendations .card-title svg {
  width: 18px;
  height: 18px;
  color: #F59E0B;
}

.recommendation-content {
  font-size: 14px;
  line-height: 1.65;
  color: var(--text-secondary);
}

.recommendation-content p {
  margin-bottom: 10px;
}

.recommendation-content p:last-child {
  margin-bottom: 0;
}

.recommendation-content em {
  color: var(--primary);
  font-style: normal;
  font-weight: 500;
}

.recommendation-content strong {
  color: var(--text-primary);
}

.results-nav {
  display: flex;
  justify-content: center;
  gap: 14px;
  margin-top: 36px;
}

/* ========== HISTORY ========== */
.history-filters {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 18px 20px;
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  cursor: pointer;
  transition: all 0.12s;
}

.history-item:hover {
  border-color: var(--primary);
  box-shadow: var(--shadow-sm);
}

.history-icon {
  width: 44px;
  height: 44px;
  background: var(--primary-light);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.history-icon svg {
  width: 22px;
  height: 22px;
  color: var(--primary);
  stroke-width: 1.5;
}

.history-content {
  flex: 1;
  min-width: 0;
}

.history-content h4 {
  font-size: 14px;
  font-weight: 500;
  margin-bottom: 4px;
}

.history-meta {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 12px;
  color: var(--text-tertiary);
}

.history-type {
  padding: 2px 8px;
  background: var(--primary-light);
  color: var(--primary);
  border-radius: 4px;
  font-weight: 500;
}

.history-results {
  display: flex;
  gap: 28px;
}

.history-stat .stat-label {
  display: block;
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.4px;
  color: var(--text-tertiary);
  margin-bottom: 2px;
}

.history-stat .stat-value {
  font-size: 13px;
  font-weight: 500;
}

.history-stat .stat-value.highlight {
  color: var(--primary);
}

.history-actions {
  display: flex;
  gap: 6px;
}

.history-actions .btn svg {
  width: 16px;
  height: 16px;
}

/* ========== MODELS CATALOG ========== */
.model-catalog-filters {
  display: flex;
  gap: 16px;
  margin-bottom: 24px;
  align-items: center;
}

.filter-chips {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.filter-chip {
  padding: 7px 14px;
  border: 1px solid var(--border-default);
  background: white;
  border-radius: 18px;
  font-size: 13px;
  font-weight: 450;
  cursor: pointer;
  transition: all 0.12s;
}

.filter-chip:hover {
  border-color: var(--primary);
}

.filter-chip.active {
  background: var(--primary);
  border-color: var(--primary);
  color: white;
}

.models-catalog-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
}

.model-catalog-card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 22px;
  transition: all 0.12s;
}

.model-catalog-card:hover {
  border-color: var(--border-strong);
  box-shadow: var(--shadow-sm);
}

.model-catalog-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.model-provider-badge {
  background: var(--primary-light);
  color: var(--primary);
  padding: 4px 10px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 600;
}

.model-version {
  font-size: 11px;
  color: var(--text-tertiary);
}

.model-catalog-card h3 {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 6px;
}

.model-catalog-card > p {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.5;
  margin-bottom: 14px;
}

.model-capabilities-list {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  margin-bottom: 14px;
}

.cap-pill {
  padding: 4px 10px;
  background: var(--bg-subtle);
  border-radius: 12px;
  font-size: 11px;
  font-weight: 450;
  color: var(--text-secondary);
}

.model-specs-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  padding: 14px;
  background: var(--bg-subtle);
  border-radius: var(--radius-sm);
  margin-bottom: 14px;
}

.spec-item .spec-label {
  display: block;
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.4px;
  color: var(--text-tertiary);
  margin-bottom: 2px;
}

.spec-item .spec-value {
  font-size: 12px;
  font-weight: 500;
}

.model-catalog-actions {
  display: flex;
  gap: 8px;
}

.model-catalog-actions .btn {
  flex: 1;
}

.btn-outline {
  background: transparent;
  border: 1px solid var(--primary);
  color: var(--primary);
}

.btn-outline:hover {
  background: var(--primary-light);
}

/* ========== PROVIDERS PAGE ========== */
.providers-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}

.provider-page-card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 22px;
  transition: all 0.12s;
}

.provider-page-card:hover {
  box-shadow: var(--shadow-sm);
}

.provider-page-card.not_connected {
  opacity: 0.6;
}

.provider-page-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 14px;
}

.provider-logo-lg {
  width: 52px;
  height: 52px;
  background: var(--primary-light);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 20px;
  color: var(--primary);
}

.provider-status-badge {
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
}

.provider-status-badge.connected {
  background: var(--success-subtle);
  color: var(--success);
}

.provider-status-badge.not_connected {
  background: var(--bg-inset);
  color: var(--text-tertiary);
}

.provider-page-card h3 {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 6px;
}

.provider-page-card > p {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.5;
  margin-bottom: 14px;
}

.provider-stats {
  display: flex;
  gap: 20px;
  margin-bottom: 14px;
}

.provider-stat {
  text-align: center;
}

.provider-stat .stat-value {
  font-size: 22px;
  font-weight: 600;
  color: var(--primary);
}

.provider-stat .stat-label {
  font-size: 11px;
  color: var(--text-tertiary);
}

.api-key-display {
  background: var(--bg-subtle);
  padding: 10px 14px;
  border-radius: var(--radius-sm);
  margin-bottom: 14px;
}

.key-label {
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.4px;
  color: var(--text-tertiary);
  display: block;
  margin-bottom: 4px;
}

.api-key-display code {
  font-size: 12px;
  font-family: monospace;
  color: var(--text-secondary);
}

.provider-actions {
  display: flex;
  gap: 8px;
}

.provider-actions .btn {
  flex: 1;
}

/* ========== DATASETS LIBRARY ========== */
.dataset-category-tabs {
  display: flex;
  gap: 6px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.category-tab {
  padding: 9px 18px;
  border: 1px solid var(--border-default);
  background: white;
  border-radius: 18px;
  font-size: 13px;
  font-weight: 450;
  cursor: pointer;
  transition: all 0.12s;
}

.category-tab:hover {
  border-color: var(--primary);
}

.category-tab.active {
  background: var(--primary);
  border-color: var(--primary);
  color: white;
}

.datasets-library-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
}

.dataset-library-card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 22px;
  position: relative;
  transition: all 0.12s;
}

.dataset-library-card:hover {
  box-shadow: var(--shadow-sm);
}

.featured-badge {
  position: absolute;
  top: 14px;
  right: 14px;
  display: flex;
  align-items: center;
  gap: 4px;
  background: #FEF3C7;
  color: #B45309;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
}

.featured-badge svg {
  width: 12px;
  height: 12px;
}

.dataset-category-tag {
  display: inline-block;
  background: var(--primary-light);
  color: var(--primary);
  padding: 4px 10px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 600;
  margin-bottom: 10px;
}

.dataset-library-card h3 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 6px;
  padding-right: 70px;
}

.dataset-library-card > p {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.5;
  margin-bottom: 14px;
}

.dataset-details {
  background: var(--bg-subtle);
  padding: 14px;
  border-radius: var(--radius-sm);
  margin-bottom: 14px;
}

.detail-row {
  display: flex;
  justify-content: space-between;
  padding: 5px 0;
  font-size: 12px;
}

.detail-row:not(:last-child) {
  border-bottom: 1px solid var(--border-subtle);
}

.detail-label {
  color: var(--text-tertiary);
}

.detail-value {
  font-weight: 500;
  color: var(--text-primary);
}

.dataset-actions .btn {
  width: 100%;
}

/* ========== REPORTS ========== */
.reports-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.report-card {
  background: var(--bg-main);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 24px;
  transition: all 0.12s;
}

.report-card:hover {
  box-shadow: var(--shadow-sm);
}

.report-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.report-date {
  font-size: 12px;
  color: var(--text-tertiary);
}

.report-actions {
  display: flex;
  gap: 6px;
}

.report-card h3 {
  font-size: 17px;
  font-weight: 600;
  margin-bottom: 8px;
}

.report-summary {
  font-size: 14px;
  color: var(--text-secondary);
  line-height: 1.6;
  margin-bottom: 16px;
}

.report-verdict {
  display: flex;
  gap: 12px;
  background: var(--success-subtle);
  padding: 14px 18px;
  border-radius: var(--radius-md);
  margin-bottom: 16px;
}

.report-verdict svg {
  width: 18px;
  height: 18px;
  color: var(--success);
  flex-shrink: 0;
  margin-top: 2px;
}

.report-verdict strong {
  display: block;
  margin-bottom: 4px;
  color: var(--success);
  font-size: 13px;
}

.report-verdict p {
  font-size: 13px;
  color: var(--text-primary);
  line-height: 1.5;
}

.report-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 14px;
  border-top: 1px solid var(--border-subtle);
}

.top-model .label {
  font-size: 11px;
  color: var(--text-tertiary);
  margin-right: 6px;
}

.top-model .value {
  font-weight: 600;
  color: var(--primary);
  font-size: 13px;
}

.metrics-tested {
  display: flex;
  gap: 5px;
}

.metric-tag {
  background: var(--bg-subtle);
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 11px;
  color: var(--text-secondary);
}

/* ========== SETTINGS ========== */
.settings-layout {
  display: grid;
  grid-template-columns: 200px 1fr;
  gap: 28px;
}

.settings-nav {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.settings-nav-item {
  padding: 10px 14px;
  border: none;
  background: transparent;
  text-align: left;
  font-size: 13px;
  font-weight: 450;
  color: var(--text-secondary);
  cursor: pointer;
  border-radius: var(--radius-sm);
  transition: all 0.12s;
}

.settings-nav-item:hover {
  background: var(--bg-subtle);
  color: var(--text-primary);
}

.settings-nav-item.active {
  background: var(--primary-light);
  color: var(--primary);
  font-weight: 500;
}

/* ========== MODALS ========== */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.15s;
}

.modal-overlay.active {
  opacity: 1;
  pointer-events: all;
}

.modal-content {
  background: white;
  border-radius: var(--radius-xl);
  width: 480px;
  max-width: 90vw;
  padding: 28px;
  box-shadow: var(--shadow-xl);
  transform: translateY(8px) scale(0.98);
  transition: transform 0.15s;
}

.modal-overlay.active .modal-content {
  transform: translateY(0) scale(1);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.modal-header h3 {
  font-size: 17px;
  font-weight: 600;
}

.modal-close {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  border-radius: var(--radius-sm);
  cursor: pointer;
  color: var(--text-tertiary);
}

.modal-close:hover {
  background: var(--bg-subtle);
  color: var(--text-primary);
}

.modal-close svg {
  width: 18px;
  height: 18px;
}

.modal-body {
  font-size: 14px;
}

.share-link-box {
  display: flex;
  gap: 8px;
}

.share-link-box input {
  flex: 1;
}

.add-provider-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: 14px;
}

.add-provider-item {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px;
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.12s;
}

.add-provider-item:hover {
  background: var(--primary-light);
}

.add-provider-item strong {
  display: block;
  font-size: 14px;
  margin-bottom: 2px;
}

.add-provider-item p {
  font-size: 12px;
  color: var(--text-tertiary);
  margin: 0;
}

.form-help {
  font-size: 12px;
  color: var(--text-tertiary);
  margin-top: 6px;
}

.user-profile-card {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 18px;
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  margin-bottom: 18px;
}

.avatar-lg {
  width: 52px;
  height: 52px;
  font-size: 18px;
}

.user-profile-card h4 {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 2px;
}

.user-profile-card p {
  font-size: 13px;
  color: var(--text-secondary);
  margin: 0;
}

.user-profile-card .org {
  font-size: 12px;
  color: var(--text-tertiary);
}

.menu-items {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.menu-item {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 11px 14px;
  border: none;
  background: transparent;
  font-size: 14px;
  cursor: pointer;
  border-radius: var(--radius-sm);
  transition: all 0.12s;
  text-align: left;
}

.menu-item:hover {
  background: var(--bg-subtle);
}

.menu-item svg {
  width: 18px;
  height: 18px;
  color: var(--text-tertiary);
}

.notification-item {
  display: flex;
  gap: 12px;
  padding: 14px;
  border-radius: var(--radius-md);
  transition: all 0.12s;
}

.notification-item:hover {
  background: var(--bg-subtle);
}

.notification-item.unread {
  background: var(--primary-light);
}

.notif-icon {
  width: 36px;
  height: 36px;
  background: var(--bg-subtle);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.notif-icon svg {
  width: 18px;
  height: 18px;
  color: var(--primary);
}

.notif-content strong {
  display: block;
  font-size: 13px;
  margin-bottom: 3px;
}

.notif-content p {
  font-size: 12px;
  color: var(--text-secondary);
  margin: 0 0 4px 0;
  line-height: 1.4;
}

.notif-time {
  font-size: 11px;
  color: var(--text-tertiary);
}

.help-section {
  margin-bottom: 20px;
}

.help-section h4 {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 10px;
}

.help-section p {
  font-size: 13px;
  color: var(--text-secondary);
  margin-bottom: 10px;
}

.help-section ol {
  padding-left: 18px;
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.7;
}

.help-links {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.help-link {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 14px;
  background: var(--bg-subtle);
  border-radius: var(--radius-sm);
  color: var(--text-primary);
  text-decoration: none;
  font-size: 13px;
  font-weight: 450;
  transition: all 0.12s;
}

.help-link:hover {
  background: var(--primary-light);
  color: var(--primary);
}

.help-link svg {
  width: 18px;
  height: 18px;
  color: var(--text-tertiary);
}

.help-link:hover svg {
  color: var(--primary);
}

.divider-text {
  display: flex;
  align-items: center;
  gap: 14px;
  margin: 20px 0;
  color: var(--text-tertiary);
  font-size: 12px;
}

.divider-text::before,
.divider-text::after {
  content: '';
  flex: 1;
  height: 1px;
  background: var(--border-subtle);
}

.import-options {
  display: flex;
  gap: 10px;
}

.import-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 14px;
  background: var(--bg-subtle);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
}

.import-btn:hover {
  background: var(--primary-light);
  border-color: var(--primary);
}

.import-btn svg {
  width: 18px;
  height: 18px;
}

.model-detail-desc {
  font-size: 14px;
  color: var(--text-secondary);
  line-height: 1.6;
  margin-bottom: 20px;
}

.model-detail-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  margin-bottom: 18px;
}

.detail-item {
  padding: 12px;
  background: var(--bg-subtle);
  border-radius: var(--radius-sm);
}

.detail-item .label {
  display: block;
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.4px;
  color: var(--text-tertiary);
  margin-bottom: 4px;
}

.detail-item .value {
  font-size: 13px;
  font-weight: 500;
}

.capabilities-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

/* ========== TOAST ========== */
.toast {
  position: fixed;
  bottom: 20px;
  right: 20px;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 18px;
  background: white;
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-lg);
  font-size: 13px;
  font-weight: 450;
  z-index: 2000;
  transform: translateY(16px);
  opacity: 0;
  transition: all 0.2s ease;
}

.toast.show {
  transform: translateY(0);
  opacity: 1;
}

.toast svg {
  width: 18px;
  height: 18px;
}

.toast-success {
  border-left: 3px solid var(--success);
}

.toast-success svg {
  color: var(--success);
}

.toast-warning {
  border-left: 3px solid var(--warning);
}

.toast-warning svg {
  color: var(--warning);
}

.toast-info {
  border-left: 3px solid var(--primary);
}

.toast-info svg {
  color: var(--primary);
}

/* ========== COMPARE FEATURE ========== */
.compare-checkbox {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}

.compare-checkbox input {
  width: 16px;
  height: 16px;
  accent-color: var(--primary);
}

.compare-floating-bar {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 14px 20px;
  background: var(--text-primary);
  color: white;
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-xl);
  z-index: 500;
  animation: slideUp 0.2s ease;
}

@keyframes slideUp {
  from { transform: translateX(-50%) translateY(20px); opacity: 0; }
  to { transform: translateX(-50%) translateY(0); opacity: 1; }
}

.compare-floating-bar span {
  font-size: 14px;
}

.compare-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 24px;
  padding-bottom: 20px;
  border-bottom: 1px solid var(--border-subtle);
}

.compare-model {
  flex: 1;
  text-align: center;
}

.compare-rank {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  font-size: 12px;
  font-weight: 700;
  margin-bottom: 8px;
}

.compare-rank.rank-1 { background: #FEF3C7; color: #B45309; }
.compare-rank.rank-2 { background: var(--bg-inset); color: var(--text-secondary); }
.compare-rank.rank-3 { background: #FFEDD5; color: #C2410C; }

.compare-model h4 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 4px;
}

.compare-model span {
  font-size: 12px;
  color: var(--text-tertiary);
}

.compare-vs {
  font-size: 12px;
  font-weight: 700;
  color: var(--text-tertiary);
  background: var(--bg-subtle);
  padding: 8px 12px;
  border-radius: 20px;
}

.compare-metrics {
  margin-bottom: 20px;
}

.compare-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 12px;
  padding: 12px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.compare-row:last-child {
  border-bottom: none;
}

.metric-name {
  font-size: 13px;
  color: var(--text-secondary);
}

.metric-value {
  font-size: 14px;
  font-weight: 500;
  text-align: center;
}

.metric-value.winner {
  color: var(--success);
  font-weight: 600;
}

.compare-summary {
  background: var(--bg-subtle);
  padding: 16px;
  border-radius: var(--radius-md);
}

.compare-summary h4 {
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 10px;
}

.compare-summary ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.compare-summary li {
  font-size: 13px;
  color: var(--text-secondary);
  padding: 4px 0;
}

.compare-summary strong {
  color: var(--text-primary);
}

/* ========== COLUMN MAPPING ========== */
.column-mapping-grid {
  margin-bottom: 20px;
}

.mapping-row {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.mapping-row:last-child {
  border-bottom: none;
}

.mapping-label {
  flex: 0 0 160px;
  font-size: 13px;
  font-weight: 500;
}

.mapping-select {
  flex: 1;
}

.preview-section {
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid var(--border-subtle);
}

.preview-section h4 {
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 12px;
}

.preview-table-wrap {
  overflow-x: auto;
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
}

.preview-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
}

.preview-table th {
  background: var(--bg-subtle);
  padding: 10px 12px;
  text-align: left;
  font-weight: 600;
  border-bottom: 1px solid var(--border-subtle);
}

.preview-table td {
  padding: 10px 12px;
  border-bottom: 1px solid var(--border-subtle);
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.preview-table tr:last-child td {
  border-bottom: none;
}

.preview-table code {
  font-size: 11px;
  background: var(--bg-inset);
  padding: 2px 6px;
  border-radius: 3px;
}

.mapping-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid var(--border-subtle);
}

/* Not connected provider styling */
.provider-card.not-connected {
  border-style: dashed;
  cursor: pointer;
}

.provider-card.not-connected:hover {
  border-color: var(--primary);
  background: var(--primary-light);
}

.provider-card.not-connected .provider-status {
  color: var(--primary);
}

/* ========== MODEL CONFIGURATION ========== */
.configure-btn {
  position: absolute;
  top: 12px;
  right: 12px;
  width: 28px;
  height: 28px;
  border: none;
  background: var(--bg-subtle);
  border-radius: var(--radius-sm);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--text-tertiary);
  transition: all 0.12s;
  z-index: 5;
}

.configure-btn:hover {
  background: var(--primary-light);
  color: var(--primary);
}

.configure-btn svg {
  width: 14px;
  height: 14px;
}

.config-intro {
  color: var(--text-secondary);
  margin-bottom: 20px;
  font-size: 14px;
}

.api-detected-section {
  background: var(--bg-inset);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  padding: 14px;
  margin-bottom: 20px;
}

.api-detected-header {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--success);
  margin-bottom: 12px;
}

.api-detected-header svg {
  width: 14px;
  height: 14px;
}

.api-detected-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}

.api-field {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 10px;
  background: var(--bg-default);
  border-radius: var(--radius-sm);
  font-size: 12px;
}

.api-key {
  font-family: 'SF Mono', Consolas, monospace;
  color: var(--text-tertiary);
}

.api-value {
  font-family: 'SF Mono', Consolas, monospace;
  font-weight: 600;
  color: var(--text-primary);
}

.config-section h4 {
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-tertiary);
  margin-bottom: 12px;
}

.config-row {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 14px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.config-row:last-child {
  border-bottom: none;
}

.config-info {
  flex: 1;
}

.config-label {
  display: block;
  font-size: 14px;
  font-weight: 500;
  margin-bottom: 2px;
}

.config-desc {
  font-size: 12px;
  color: var(--text-tertiary);
}

.config-options {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.config-option {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: var(--bg-subtle);
  border-radius: 16px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.12s;
}

.config-option:hover {
  background: var(--bg-inset);
}

.config-option.selected,
.config-option:has(input:checked) {
  background: var(--primary-light);
  color: var(--primary);
}

.config-option input {
  display: none;
}

.config-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid var(--border-subtle);
}

.config-note {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: var(--text-tertiary);
}

.config-note svg {
  width: 14px;
  height: 14px;
}

/* ========== CAPABILITY MATCHING ========== */
.capability-requirement {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 5px 10px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 500;
  margin-bottom: 10px;
}

.capability-requirement svg {
  width: 12px;
  height: 12px;
}

.cap-universal {
  background: var(--success-subtle);
  color: var(--success);
}

.cap-agent {
  background: #E0E7FF;
  color: #3730A3;
}

.cap-coding {
  background: #DBEAFE;
  color: #1D4ED8;
}

.cap-rag {
  background: #FEF3C7;
  color: #B45309;
}

.cap-domain {
  background: #FCE7F3;
  color: #BE185D;
}

.dataset-card.incompatible {
  opacity: 0.7;
  border-style: dashed;
}

.dataset-card.incompatible:hover {
  border-color: var(--warning);
}

.badge-warning {
  background: var(--warning-subtle);
  color: var(--warning);
}

/* Incompatibility Warning Modal */
.warning-header {
  color: var(--warning);
}

.warning-header h3 {
  display: flex;
  align-items: center;
  gap: 10px;
}

.warning-header svg {
  width: 22px;
  height: 22px;
}

.warning-message {
  background: var(--warning-subtle);
  padding: 16px;
  border-radius: var(--radius-md);
  margin-bottom: 20px;
}

.warning-message p {
  font-size: 14px;
  color: var(--text-primary);
  margin: 0;
}

.cap-highlight {
  font-weight: 600;
  color: var(--warning);
}

.incompatible-models {
  margin-bottom: 20px;
}

.incompatible-models h4 {
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 10px;
}

.incompatible-models ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.incompatible-models li {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 0;
  font-size: 13px;
  color: var(--danger);
}

.incompatible-models li svg {
  width: 16px;
  height: 16px;
}

.compatible-suggestion h4 {
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 12px;
  color: var(--text-primary);
}

.suggestion-models {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.suggestion-model {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  background: var(--bg-subtle);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.12s;
}

.suggestion-model:hover {
  background: var(--primary-light);
}

.suggestion-model-info {
  flex: 1;
}

.suggestion-model-info strong {
  display: block;
  font-size: 13px;
  margin-bottom: 2px;
}

.suggestion-model-info span {
  font-size: 11px;
  color: var(--text-tertiary);
}

.suggestion-caps {
  display: flex;
  gap: 4px;
}

.suggestion-caps .cap-tag {
  font-size: 10px;
  padding: 2px 6px;
}

.warning-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid var(--border-subtle);
}

/* ========== FETCHING MODELS ANIMATION ========== */
.fetching-models {
  text-align: center;
  padding: 30px 20px;
}

.fetch-spinner {
  width: 48px;
  height: 48px;
  border: 3px solid var(--border-default);
  border-top-color: var(--primary);
  border-radius: 50%;
  margin: 0 auto 20px;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.fetching-models h4 {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 6px;
}

.fetching-models > p {
  color: var(--text-secondary);
  font-size: 13px;
  margin-bottom: 24px;
}

.fetch-steps {
  text-align: left;
  max-width: 240px;
  margin: 0 auto;
}

.fetch-step {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 0;
  font-size: 13px;
  color: var(--text-tertiary);
}

.fetch-step svg {
  width: 16px;
  height: 16px;
}

.fetch-step.active {
  color: var(--text-primary);
}

.fetch-step.active svg {
  color: var(--success);
}

/* ========== MODELS API NOTE ========== */
.models-api-note {
  grid-column: 1 / -1;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  background: var(--primary-light);
  border: 1px solid rgba(20, 40, 160, 0.1);
  border-radius: var(--radius-md);
  font-size: 12px;
  color: var(--primary);
  margin-bottom: 4px;
}

.models-api-note svg {
  width: 14px;
  height: 14px;
}

.models-empty-state {
  grid-column: 1 / -1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 60px 20px;
  color: var(--text-tertiary);
  text-align: center;
}

.models-empty-state svg {
  width: 32px;
  height: 32px;
  margin-bottom: 12px;
  opacity: 0.5;
}

.models-empty-state p {
  font-size: 14px;
}

.cap-more {
  background: var(--bg-inset);
  color: var(--text-tertiary);
}

/* ========== SETTINGS PAGE ========== */
.settings-tab {
  display: none;
}

.settings-tab.active {
  display: block;
}

.api-keys-list {
  margin-top: 20px;
}

.api-key-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.api-key-row:last-child {
  border-bottom: none;
}

.api-key-provider {
  display: flex;
  align-items: center;
  gap: 12px;
}

.provider-logo-sm {
  width: 36px;
  height: 36px;
  background: var(--bg-inset);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 14px;
  color: var(--text-secondary);
}

.api-key-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.api-key-info strong {
  font-size: 14px;
  font-weight: 500;
}

.api-key-masked {
  font-size: 12px;
  color: var(--text-tertiary);
  font-family: 'SF Mono', Consolas, monospace;
}

.api-key-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.status-badge {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 12px;
  background: var(--bg-inset);
  color: var(--text-tertiary);
}

.status-badge.connected {
  background: rgba(34, 197, 94, 0.1);
  color: var(--success);
}

.status-badge svg {
  width: 12px;
  height: 12px;
}

.btn-ghost {
  background: transparent;
  border: none;
  color: var(--text-tertiary);
  padding: 6px;
  border-radius: var(--radius-sm);
  cursor: pointer;
}

.btn-ghost:hover {
  background: var(--bg-inset);
  color: var(--text-primary);
}

.btn-ghost svg {
  width: 16px;
  height: 16px;
}

.btn-danger {
  background: var(--danger);
  color: white;
}

.btn-danger:hover {
  background: #dc2626;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}

.team-list {
  display: flex;
  flex-direction: column;
}

.team-member {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.team-member:last-child {
  border-bottom: none;
}

.team-avatar {
  width: 40px;
  height: 40px;
  background: var(--primary-light);
  color: var(--primary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 14px;
}

.team-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.team-info strong {
  font-size: 14px;
  font-weight: 500;
}

.team-info span {
  font-size: 12px;
  color: var(--text-tertiary);
}

.notification-settings {
  display: flex;
  flex-direction: column;
}

.notification-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 0;
  border-bottom: 1px solid var(--border-subtle);
}

.notification-row:last-child {
  border-bottom: none;
}

.notification-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.notification-info strong {
  font-size: 14px;
  font-weight: 500;
}

.notification-info span {
  font-size: 12px;
  color: var(--text-tertiary);
}

.toggle {
  position: relative;
  display: inline-block;
  width: 44px;
  height: 24px;
}

.toggle input {
  opacity: 0;
  width: 0;
  height: 0;
}

.toggle-slider {
  position: absolute;
  cursor: pointer;
  inset: 0;
  background: var(--border-default);
  border-radius: 24px;
  transition: 0.2s;
}

.toggle-slider::before {
  content: '';
  position: absolute;
  width: 18px;
  height: 18px;
  left: 3px;
  bottom: 3px;
  background: white;
  border-radius: 50%;
  transition: 0.2s;
}

.toggle input:checked + .toggle-slider {
  background: var(--primary);
}

.toggle input:checked + .toggle-slider::before {
  transform: translateX(20px);
}

.provider-select-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.provider-select-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.15s;
}

.provider-select-item:hover {
  border-color: var(--primary);
  background: var(--primary-light);
}

.provider-select-item div {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.provider-select-item strong {
  font-size: 14px;
}

.provider-select-item span {
  font-size: 12px;
  color: var(--text-tertiary);
}

.provider-select-item svg {
  width: 16px;
  height: 16px;
  color: var(--text-tertiary);
}

/* ========== MODEL FILTER SIDEBAR ========== */
.step-content-wide {
  max-width: 1200px;
}

.models-layout {
  display: flex;
  gap: 24px;
  margin-top: 20px;
}

.filter-sidebar {
  width: 260px;
  flex-shrink: 0;
  background: var(--bg-default);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-lg);
  padding: 16px;
  height: fit-content;
  max-height: calc(100vh - 300px);
  overflow-y: auto;
  position: sticky;
  top: 20px;
}

.filter-sidebar.collapsed {
  display: none;
}

.filter-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 12px;
  border-bottom: 1px solid var(--border-subtle);
  margin-bottom: 12px;
}

.filter-header span {
  font-weight: 600;
  font-size: 14px;
}

.btn-link {
  background: none;
  border: none;
  color: var(--primary);
  font-size: 12px;
  cursor: pointer;
  padding: 0;
}

.btn-link:hover {
  text-decoration: underline;
}

.filter-section {
  border-bottom: 1px solid var(--border-subtle);
  padding: 12px 0;
}

.filter-section:last-child {
  border-bottom: none;
}

.filter-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-secondary);
  cursor: pointer;
  padding: 4px 0;
}

.filter-title svg {
  width: 14px;
  height: 14px;
  transition: transform 0.2s;
}

.filter-section.collapsed .filter-title svg {
  transform: rotate(-90deg);
}

.filter-section.collapsed .filter-options {
  display: none;
}

.filter-options {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 10px;
}

.filter-chip {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 5px 10px;
  background: var(--bg-inset);
  border: 1px solid var(--border-default);
  border-radius: 16px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.15s;
  white-space: nowrap;
}

.filter-chip:hover {
  border-color: var(--border-strong);
}

.filter-chip:has(input:checked) {
  background: var(--primary-light);
  border-color: var(--primary);
  color: var(--primary);
}

.filter-chip input {
  display: none;
}

.filter-chip svg {
  width: 12px;
  height: 12px;
}

.filter-slider-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 0;
  width: 100%;
}

.filter-slider-row span:first-child {
  font-size: 12px;
  color: var(--text-secondary);
  min-width: 70px;
}

.filter-slider {
  flex: 1;
  height: 4px;
  -webkit-appearance: none;
  background: var(--border-default);
  border-radius: 2px;
  outline: none;
}

.filter-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 14px;
  height: 14px;
  background: var(--primary);
  border-radius: 50%;
  cursor: pointer;
}

.filter-slider-val {
  font-size: 11px;
  color: var(--text-tertiary);
  min-width: 28px;
  text-align: right;
}

.models-main {
  flex: 1;
  min-width: 0;
}

.active-filters {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  flex: 1;
}

.active-filter-tag {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  background: var(--primary-light);
  color: var(--primary);
  border-radius: 12px;
  font-size: 11px;
  font-weight: 500;
}

.active-filter-tag button {
  background: none;
  border: none;
  padding: 0;
  cursor: pointer;
  color: var(--primary);
  display: flex;
}

.active-filter-tag button svg {
  width: 12px;
  height: 12px;
}

@media (max-width: 900px) {
  .models-layout {
    flex-direction: column;
  }

  .filter-sidebar {
    width: 100%;
    position: relative;
    top: 0;
    max-height: none;
  }
}

/* ========== RESPONSIVE ========== */
@media (max-width: 1100px) {
  .login-right { display: none; }
  .login-left { flex: 1; max-width: none; align-items: center; }
  .login-content { max-width: 360px; }

  .stats-grid { grid-template-columns: repeat(2, 1fr); }
  .eval-type-grid { grid-template-columns: 1fr; }
  .provider-grid { grid-template-columns: repeat(2, 1fr); }
  .providers-grid { grid-template-columns: repeat(2, 1fr); }
  .results-summary-cards { grid-template-columns: 1fr; }
  .dashboard-grid { grid-template-columns: 1fr; }
}

@media (max-width: 768px) {
  .sidebar { display: none; }
  .main-wrapper { margin-left: 0; }
  .content-area { padding: 20px; }

  .models-grid { grid-template-columns: 1fr; }
  .datasets-grid { grid-template-columns: 1fr; }
  .metrics-grid { grid-template-columns: 1fr; }
  .models-catalog-grid { grid-template-columns: 1fr; }
  .datasets-library-grid { grid-template-columns: 1fr; }
  .providers-grid { grid-template-columns: 1fr; }

  .settings-layout { grid-template-columns: 1fr; }
  .settings-nav { flex-direction: row; flex-wrap: wrap; }

  .search-box { width: 100%; }
  .topbar { padding: 0 16px; }
}
