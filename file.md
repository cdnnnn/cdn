//AppHeader.jsx
import { Link } from 'react-router-dom';
import { useState } from 'react';
import shared from '../../styles/shared.module.scss';
import styles from './AppHeader.module.scss';

export default function AppHeader({ onSearch }) {
  const [query, setQuery] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    onSearch?.(query.trim());
  }

  return (
    <header className={styles.appHead}>
      <div className={styles.inner}>
        <Link className={shared.mark} to="/">
          <span className={shared.markGlyph} aria-hidden="true" />
          Sidenote
        </Link>

        <form className={styles.search} role="search" onSubmit={handleSubmit}>
          <svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="#7D8D84" strokeWidth="2" aria-hidden="true">
            <circle cx="11" cy="11" r="7" /><path d="m20 20-3.6-3.6" />
          </svg>
          <input
            type="search"
            placeholder="Search videos, or a phrase said inside one"
            aria-label="Search videos"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
          />
          <button type="submit">Search</button>
        </form>

        <span className={styles.avatar} title="Your account">AR</span>
      </div>
    </header>
  );
}


















//AppHeader.module.scss
.appHead {
  position: sticky;
  top: 0;
  z-index: 50;
  background: var(--surface);
  border-bottom: 1px solid var(--line);
}

.inner {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 24px;
  height: 66px;
  max-width: 1440px;
  margin-inline: auto;
  padding-inline: 24px;

  @media (max-width: 860px) {
    grid-template-columns: auto 1fr;
    grid-template-rows: auto auto;
    height: auto;
    padding-block: 12px;
    gap: 12px 16px;
  }
}

.search {
  display: flex;
  align-items: center;
  gap: 10px;
  max-width: 560px;
  width: 100%;
  justify-self: center;
  background: var(--paper);
  border: 1px solid var(--line);
  border-radius: var(--r-pill);
  padding: 0 6px 0 18px;
  transition: border-color .18s ease, box-shadow .18s ease, background .18s ease;

  &:focus-within {
    background: var(--surface);
    border-color: var(--brand);
    box-shadow: 0 0 0 4px rgba(39, 67, 196, .13);
  }

  input {
    flex: 1;
    border: 0;
    background: none;
    outline: none;
    padding: 12px 0;
    font-size: 15px;

    &::placeholder { color: var(--ink-3); }
  }

  button {
    border: 0;
    background: var(--brand);
    color: #fff;
    border-radius: var(--r-pill);
    padding: 8px 18px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;

    &:hover { background: var(--brand-600); }
  }

  @media (max-width: 860px) {
    grid-column: 1 / -1;
    grid-row: 2;
    max-width: none;
  }
}

.avatar {
  width: 36px; height: 36px;
  border-radius: 50%;
  background: var(--tint);
  color: var(--brand-900);
  display: grid;
  place-items: center;
  font-size: 13px;
  font-weight: 600;
  border: 1px solid var(--line);

  @media (max-width: 860px) {
    justify-self: end;
  }
}






















//SiteHeader.jsx
import { Link } from 'react-router-dom';
import shared from '../../styles/shared.module.scss';
import styles from './SiteHeader.module.scss';

export default function SiteHeader() {
  return (
    <header className={styles.siteHead}>
      <div className={`${shared.wrap} ${styles.inner}`}>
        <Link className={shared.mark} to="/">
          <span className={shared.markGlyph} aria-hidden="true" />
          Sidenote
        </Link>

        <nav className={styles.siteNav}>
          <a href="#how">How it works</a>
          <a href="#asks">What you can ask</a>
          <Link to="/app">Library</Link>
        </nav>

        <Link className={`${shared.btn} ${shared.btnPrimary} ${shared.btnSm}`} to="/app">
          Start watching
        </Link>
      </div>
    </header>
  );
}













//SiteHeader.module.scss
.siteHead {
  position: sticky;
  top: 0;
  z-index: 40;
  background: rgba(244, 245, 248, .86);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid var(--line);
}

.inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 68px;
}

.siteNav {
  display: flex;
  gap: 28px;
  font-size: 15px;
  color: var(--ink-2);

  a:hover { color: var(--brand); }

  @media (max-width: 860px) {
    display: none;
  }
}





















//VideoCard.jsx
import shared from '../../styles/shared.module.scss';
import styles from './VideoCard.module.scss';

/**
 * A video thumbnail. `variant="grid"` renders the browse-page card,
 * `variant="upnext"` renders the compact row used in the watch view.
 */
export default function VideoCard({ video, variant = 'grid', onClick }) {
  const thumbStyle = { '--thumb-a': video.a, '--thumb-b': video.b };

  if (variant === 'upnext') {
    return (
      <article className={styles.upItem} onClick={onClick}>
        <div className={shared.thumb} style={thumbStyle}>
          <span className={shared.dur}>{video.dur}</span>
        </div>
        <div>
          <h4 className={styles.upTitle}>{video.title}</h4>
          <p className={styles.upMeta}>{video.chan}</p>
          <p className={styles.upMeta}>{video.stats}</p>
        </div>
      </article>
    );
  }

  return (
    <article className={styles.card} tabIndex={0} onClick={onClick}
      onKeyDown={(e) => { if (e.key === 'Enter') onClick?.(); }}>
      <div className={shared.thumb} style={thumbStyle}>
        <span className={shared.badgeAi}>Ask enabled</span>
        <span className={shared.dur}>{video.dur}</span>
      </div>
      <h3 className={styles.cardTitle}>{video.title}</h3>
      <p className={styles.meta}>{video.chan}</p>
      <p className={styles.meta}>{video.stats}</p>
    </article>
  );
}


























//VideoCard.module.scss
.card {
  cursor: pointer;

  &:hover > div:first-child {
    transform: translateY(-3px);
    box-shadow: var(--shadow-m);
  }
}

.cardTitle {
  font-size: 15.5px;
  font-weight: 600;
  line-height: 1.35;
  letter-spacing: -.015em;
  margin: 12px 0 5px;

  .card:hover & { color: var(--brand); }
}

.meta {
  font-size: 13.5px;
  color: var(--ink-3);
  margin: 0;
}

.upItem {
  display: grid;
  grid-template-columns: 158px 1fr;
  gap: 14px;
  cursor: pointer;

  @media (max-width: 860px) {
    grid-template-columns: 128px 1fr;
  }
}

.upTitle {
  font-size: 14.5px;
  font-weight: 600;
  line-height: 1.35;
  margin: 0 0 4px;
  letter-spacing: -.015em;

  .upItem:hover & { color: var(--brand); }
}

.upMeta {
  margin: 0;
  font-size: 13px;
  color: var(--ink-3);
}


















//videos.js
export const VIDEOS = [
  { id: 1, title: "How ocean currents move heat around the planet", chan: "Deep Field", stats: "412K views · 3 weeks ago", dur: "11:47", a: "#1E3A8A", b: "#0B1638" },
  { id: 2, title: "The Kalman filter, explained without a single matrix", chan: "Signal Shop", stats: "188K views · 5 days ago", dur: "18:03", a: "#2743C4", b: "#141A4D" },
  { id: 3, title: "Why your rent went up: land value in six charts", chan: "Ground Floor", stats: "1.2M views · 2 months ago", dur: "14:20", a: "#6D3C9E", b: "#26143F" },
  { id: 4, title: "Bridge that shouldn't stand: the Firth of Forth", chan: "Load Path", stats: "602K views · 1 week ago", dur: "22:36", a: "#1C5F6B", b: "#0B2A33" },
  { id: 5, title: "Sourdough is a chemistry problem", chan: "Slow Rise", stats: "940K views · 4 months ago", dur: "09:12", a: "#A85A2B", b: "#3A1D10" },
  { id: 6, title: "What a typeface is actually doing to your reading speed", chan: "Counterform", stats: "77K views · 2 days ago", dur: "12:55", a: "#3F4A6B", b: "#171B2E" },
  { id: 7, title: "Sleep debt: what the science does and doesn't say", chan: "Night Shift", stats: "1.8M views · 6 months ago", dur: "16:41", a: "#3B2FA0", b: "#120F38" },
  { id: 8, title: "The bank run, from the teller's side of the counter", chan: "Ground Floor", stats: "355K views · 3 weeks ago", dur: "20:08", a: "#8C2F4A", b: "#300F1C" },
];

export const SUGGESTIONS = [
  "Summarise in 5 bullets",
  "Explain that simpler",
  "Where does she cover the poles?",
  "Is this still accurate?",
];

export const FILTERS = ["All", "Science", "Engineering", "Money", "History", "Design", "Health", "Watch later"];

























//Landing.jsx
import { Link } from 'react-router-dom';
import SiteHeader from '../../components/SiteHeader/SiteHeader';
import shared from '../../styles/shared.module.scss';
import styles from './Landing.module.scss';

const RAIL_STEPS = [
  {
    at: '00:00',
    title: 'Find something worth watching',
    body: 'Search by topic or scroll the shelf. Every video ships with a transcript, so search reaches the words spoken inside it, not just the title.',
  },
  {
    at: '02:31',
    title: 'Ask without pausing',
    body: 'The panel sits beside the player. Type a question, keep watching — the answer arrives in the margin while the video runs.',
  },
  {
    at: '04:12',
    title: 'Jump to the proof',
    body: 'Answers carry timestamps. Tap one and the player skips to the moment it came from, so nothing has to be taken on trust.',
  },
  {
    at: '11:47',
    title: 'Leave with the summary',
    body: 'When the video ends, the thread collapses into a set of notes and key moments you can save or send to someone else.',
  },
];

const ASK_CHIPS = [
  'Explain that last part again, simpler',
  'What did she mean by "thermohaline"?',
  'Give me the three main points',
  'Where does he cover the cost side?',
  'Is this still accurate in 2026?',
  'Turn this into study notes',
  'Skip me to the demo',
];

export default function Landing() {
  return (
    <>
      <SiteHeader />

      <main>
        {/* ================= HERO ================= */}
        <section className={styles.hero}>
          <div className={`${shared.wrap} ${styles.inner}`}>
            <div>
              <p className={shared.eyebrow}>Video + answers, side by side</p>
              <h1 className={styles.h1}>
                Watch the video.<br />
                Ask it <span className={shared.swipe}>anything.</span>
              </h1>
              <p className={styles.lede}>
                Search a library of explainers, hit play, and keep asking questions in the
                margin. Every answer points back to the second it came from, so you can
                check the tape yourself.
              </p>
              <div className={styles.heroActions}>
                <Link className={`${shared.btn} ${shared.btnPrimary}`} to="/app">Start watching</Link>
                <a className={`${shared.btn} ${shared.btnGhost}`} href="#how">See how it works</a>
              </div>
              <p className={styles.heroMeta}>Free to browse · No account needed to ask your first five questions</p>
            </div>

            {/* live-feel demo of the product's core mechanic */}
            <div className={styles.demo} aria-hidden="true">
              <div className={styles.demoVideo}><span className={styles.demoPlay} /></div>
              <div className={styles.demoThread}>
                <div className={styles.demoQ}>Wait — why does the current sink there?</div>
                <div className={styles.demoA}>
                  Cold, salty water is denser, so it drops at the poles and pulls the
                  surface water behind it. She draws it out at <span className={styles.stamp}>4:12</span>.
                </div>
              </div>
            </div>
          </div>
        </section>

        {/* ================= HOW IT WORKS ================= */}
        <section className={styles.rail} id="how">
          <div className={shared.wrap}>
            <h2 className={styles.railHeading}>One video, one running conversation</h2>
            <ol className={styles.railList}>
              {RAIL_STEPS.map((step) => (
                <li className={styles.railItem} key={step.at}>
                  <span className={styles.at}>{step.at}</span>
                  <h3>{step.title}</h3>
                  <p>{step.body}</p>
                </li>
              ))}
            </ol>
          </div>
        </section>

        {/* ================= QUESTION STRIP ================= */}
        <section className={styles.strip} id="asks">
          <div className={shared.wrap}>
            <p className={shared.eyebrow}>Things people actually ask</p>
            <h2 className={styles.stripHeading}>Written like you'd say it out loud</h2>
            <div className={styles.chips}>
              {ASK_CHIPS.map((chip) => (
                <span className={shared.chip} key={chip}>{chip}</span>
              ))}
            </div>
          </div>
        </section>

        {/* ================= CLOSER ================= */}
        <section className={styles.closer}>
          <div className={shared.wrap}>
            <h2 className={styles.closerHeading}>Stop rewinding to find that one bit.</h2>
            <p className={styles.closerBody}>Ask instead. Sidenote knows the whole video and remembers where everything was said.</p>
            <Link className={`${shared.btn} ${shared.btnPrimary} ${styles.closerBtn}`} to="/app">Open the library</Link>
          </div>
        </section>
      </main>

      <footer className={styles.siteFoot}>
        <div className={`${shared.wrap} ${styles.footInner}`}>
          <span>© 2026 Sidenote</span>
          <span>Privacy · Terms · Contact</span>
        </div>
      </footer>
    </>
  );
}






















//Landing.module.scss
.hero { padding: 84px 0 72px; }

.inner {
  display: grid;
  grid-template-columns: 1.05fr .95fr;
  gap: 64px;
  align-items: center;

  @media (max-width: 860px) {
    grid-template-columns: 1fr;
    gap: 40px;
  }
}

.h1 {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: clamp(40px, 5.2vw, 64px);
  line-height: 1.04;
  letter-spacing: -.042em;
  margin: 0 0 22px;
  color: var(--brand-900);
}

.lede {
  font-size: 18px;
  line-height: 1.6;
  color: var(--ink-2);
  max-width: 46ch;
  margin: 0 0 30px;
}

.heroActions { display: flex; gap: 12px; flex-wrap: wrap; }

.heroMeta {
  margin-top: 28px;
  font-family: var(--font-mono);
  font-size: 12.5px;
  color: var(--ink-3);
  letter-spacing: .01em;
}

/* hero demo card */
.demo {
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: var(--r-l);
  box-shadow: var(--shadow-m);
  padding: 14px;
}

.demoVideo {
  aspect-ratio: 16 / 9;
  border-radius: var(--r-m);
  background:
    radial-gradient(120% 90% at 22% 12%, #3F5BE0 0%, transparent 58%),
    linear-gradient(140deg, #101A4A, #1B2A6B 55%, #0C1236);
  position: relative;
  overflow: hidden;

  &::after {
    content: "";
    position: absolute;
    inset: 0;
    background: repeating-linear-gradient(115deg,
      rgba(255,255,255,.05) 0 2px, transparent 2px 9px);
  }
}

.demoPlay {
  position: absolute;
  inset: 0;
  margin: auto;
  width: 54px; height: 54px;
  border-radius: 50%;
  background: rgba(255,255,255,.95);
  display: grid;
  place-items: center;
  z-index: 2;

  &::before {
    content: "";
    width: 0; height: 0;
    border-left: 14px solid var(--brand-900);
    border-top: 9px solid transparent;
    border-bottom: 9px solid transparent;
    transform: translateX(2px);
  }
}

.demoThread { padding: 16px 8px 6px; }

.demoQ, .demoA {
  font-size: 14.5px;
  line-height: 1.5;
  border-radius: var(--r-m);
  padding: 10px 14px;
  margin-bottom: 10px;
  max-width: 92%;
}

.demoQ {
  background: var(--brand);
  color: #fff;
  margin-left: auto;
  border-bottom-right-radius: 5px;
}

.demoA {
  background: var(--surface-2);
  border: 1px solid var(--line);
  color: var(--ink-2);
  border-bottom-left-radius: 5px;
}

.stamp {
  font-family: var(--font-mono);
  font-size: 12px;
  background: var(--accent);
  color: #3D2606;
  padding: 1px 7px;
  border-radius: 5px;
  white-space: nowrap;
}

/* ---------------- how it works rail ---------------- */

.rail { padding: 76px 0; background: var(--surface); border-block: 1px solid var(--line); }

.railHeading {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: clamp(28px, 3.2vw, 40px);
  letter-spacing: -.035em;
  line-height: 1.12;
  margin: 0 0 46px;
  color: var(--brand-900);
}

.railList {
  list-style: none;
  margin: 0;
  padding: 0 0 0 118px;
  border-left: 1px solid var(--line);
  position: relative;

  @media (max-width: 860px) {
    padding-left: 0;
    border-left: 0;
  }
}

.railItem {
  position: relative;
  padding: 0 0 42px;

  &:last-child { padding-bottom: 0; }

  h3 {
    font-size: 19px;
    font-weight: 600;
    letter-spacing: -.02em;
    margin: 0 0 6px;
  }

  p {
    margin: 0;
    color: var(--ink-2);
    max-width: 58ch;
  }
}

.at {
  position: absolute;
  left: -118px;
  top: 2px;
  font-family: var(--font-mono);
  font-size: 12.5px;
  color: var(--brand);
  letter-spacing: .03em;

  &::after {
    content: "";
    position: absolute;
    left: 62px; top: 9px;
    width: 24px;
    border-top: 1px dashed var(--line);
  }

  @media (max-width: 860px) {
    position: static;
    display: block;
    margin-bottom: 6px;

    &::after { display: none; }
  }
}

/* ---------------- question strip ---------------- */

.strip { padding: 76px 0; }

.stripHeading {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: clamp(28px, 3.2vw, 40px);
  letter-spacing: -.035em;
  line-height: 1.12;
  margin: 0 0 46px;
  color: var(--brand-900);
}

.chips { display: flex; flex-wrap: wrap; gap: 10px; }

/* ---------------- closer ---------------- */

.closer {
  background: var(--brand-900);
  color: #E8EBF9;
  padding: 84px 0;
  text-align: center;
}

.closerHeading {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: clamp(28px, 3.2vw, 40px);
  letter-spacing: -.035em;
  line-height: 1.12;
  color: #fff;
  margin: 0 0 14px;
}

.closerBody { margin: 0 auto 30px; max-width: 48ch; color: #AEB6DA; }

.closerBtn {
  background: var(--accent);
  color: #3D2606;

  &:hover { background: var(--accent-600); }
}

.siteFoot { }

.footInner {
  padding: 26px 0;
  font-size: 14px;
  color: var(--ink-3);
  display: flex;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
}





















//Library.jsx
import { useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import AppHeader from '../../components/AppHeader/AppHeader';
import VideoCard from '../../components/VideoCard/VideoCard';
import { VIDEOS, FILTERS } from '../../data/videos';
import shared from '../../styles/shared.module.scss';
import styles from './Library.module.scss';

export default function Library() {
  const navigate = useNavigate();
  const [activeFilter, setActiveFilter] = useState('All');
  const [query, setQuery] = useState('');

  const results = useMemo(() => {
    if (!query) return VIDEOS;
    const q = query.toLowerCase();
    return VIDEOS.filter((v) => (v.title + v.chan).toLowerCase().includes(q));
  }, [query]);

  const title = query
    ? `${results.length} result${results.length === 1 ? '' : 's'} for “${query}”`
    : 'Recommended for you';

  return (
    <>
      <AppHeader onSearch={setQuery} />

      <main className={styles.appWrap}>
        <div className={styles.filters}>
          {FILTERS.map((f) => (
            <button
              key={f}
              className={`${styles.filter} ${activeFilter === f ? styles.isOn : ''}`}
              onClick={() => setActiveFilter(f)}
            >
              {f}
            </button>
          ))}
        </div>

        <h1 className={shared.sectionTitle}>{title}</h1>

        {results.length ? (
          <div className={styles.grid}>
            {results.map((v) => (
              <VideoCard key={v.id} video={v} onClick={() => navigate(`/app/watch/${v.id}`)} />
            ))}
          </div>
        ) : (
          <p className={styles.empty}>Nothing matched that. Try a broader word — “currents”, “rent”, “sleep”.</p>
        )}
      </main>
    </>
  );
}
























//Library.module.scss
.appWrap {
  max-width: 1440px;
  margin-inline: auto;
  padding: 26px 24px 64px;

  @media (max-width: 520px) {
    padding-inline: 16px;
  }
}

.filters {
  display: flex;
  gap: 9px;
  overflow-x: auto;
  padding-bottom: 20px;
  scrollbar-width: none;

  &::-webkit-scrollbar { display: none; }
}

.filter {
  border: 1px solid var(--line);
  background: var(--surface);
  border-radius: var(--r-pill);
  padding: 7px 15px;
  font-size: 14px;
  white-space: nowrap;
  cursor: pointer;
  color: var(--ink-2);

  &:hover { border-color: var(--brand); color: var(--brand-900); }
}

.isOn {
  background: var(--brand-900);
  border-color: var(--brand-900);
  color: #fff;
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(272px, 1fr));
  gap: 28px 22px;

  @media (max-width: 520px) {
    grid-template-columns: 1fr;
  }
}

.empty {
  font-size: 13.5px;
  color: var(--ink-3);
}





















//Watch.jsx
import { useEffect, useRef, useState } from 'react';
import { useNavigate, useParams } from 'react-router-dom';
import AppHeader from '../../components/AppHeader/AppHeader';
import VideoCard from '../../components/VideoCard/VideoCard';
import { VIDEOS, SUGGESTIONS } from '../../data/videos';
import shared from '../../styles/shared.module.scss';
import styles from './Watch.module.scss';

function initials(chan) {
  return chan.split(' ').map((w) => w[0]).join('').slice(0, 2);
}

export default function Watch() {
  const { id } = useParams();
  const navigate = useNavigate();
  const video = VIDEOS.find((v) => v.id === Number(id)) ?? VIDEOS[0];

  const [thread, setThread] = useState([]);
  const [pending, setPending] = useState(false);
  const [input, setInput] = useState('');
  const threadRef = useRef(null);
  const textareaRef = useRef(null);
  const timerRef = useRef(null);

  // reset the conversation whenever the video changes
  useEffect(() => {
    setThread([
      { role: 'ai', text: `Hi — I've read the whole transcript of "${video.title}". Ask me anything while it plays, or start with one of the prompts below.` },
    ]);
    setPending(false);
    setInput('');
    window.scrollTo({ top: 0 });
    return () => clearTimeout(timerRef.current);
  }, [video.id]);

  useEffect(() => {
    if (threadRef.current) threadRef.current.scrollTop = threadRef.current.scrollHeight;
  }, [thread, pending]);

  function ask(question) {
    const q = question.trim();
    if (!q) return;
    setThread((t) => [...t, { role: 'you', text: q }]);
    setInput('');
    if (textareaRef.current) textareaRef.current.style.height = 'auto';
    setPending(true);

    timerRef.current = setTimeout(() => {
      setPending(false);
      setThread((t) => [...t, {
        role: 'ai',
        text: "She sets this up in the middle section: cold, salty water sinks at the poles and drags the surface flow behind it — that's the engine for the whole loop.",
        jump: '4:12',
      }]);
    }, 900);
  }

  function handleSubmit(e) {
    e.preventDefault();
    ask(input);
  }

  function handleTextareaChange(e) {
    setInput(e.target.value);
    const el = e.target;
    el.style.height = 'auto';
    el.style.height = Math.min(el.scrollHeight, 120) + 'px';
  }

  function handleTextareaKeyDown(e) {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      ask(input);
    }
  }

  const upNext = VIDEOS.filter((v) => v.id !== video.id).slice(0, 5);

  return (
    <>
      <AppHeader onSearch={(q) => navigate(`/app${q ? `?q=${encodeURIComponent(q)}` : ''}`)} />

      <main className={styles.appWrap}>
        <button className={styles.back} onClick={() => navigate('/app')}>
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" aria-hidden="true">
            <path d="M15 5l-7 7 7 7" />
          </svg>
          Back to library
        </button>

        <div className={styles.watch}>
          {/* player column */}
          <div>
            <div className={styles.player}>
              <div className={styles.playerBar}>
                <button className={styles.iconBtn} aria-label="Pause">
                  <svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
                    <rect x="6" y="4" width="4" height="16" rx="1" /><rect x="14" y="4" width="4" height="16" rx="1" />
                  </svg>
                </button>
                <span className={styles.time}>4:12</span>
                <div className={styles.scrub}><span /></div>
                <span className={styles.time}>{video.dur}</span>
                <button className={styles.iconBtn} aria-label="Full screen">
                  <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.4" aria-hidden="true">
                    <path d="M4 9V4h5M20 15v5h-5M15 4h5v5M9 20H4v-5" />
                  </svg>
                </button>
              </div>
            </div>

            <h1 className={styles.videoTitle}>{video.title}</h1>
            <div className={styles.videoMeta}>
              <span className={styles.channel}>
                <span className={styles.dot}>{initials(video.chan)}</span>
                <span>{video.chan}</span>
              </span>
              <span>{video.stats}</span>
              <span className={styles.spacer} />
              <button className={`${shared.btn} ${shared.btnQuiet} ${shared.btnSm}`}>Save</button>
              <button className={`${shared.btn} ${shared.btnQuiet} ${shared.btnSm}`}>Share</button>
            </div>

            <div className={styles.upNext}>
              <h2 className={shared.sectionTitle}>Up next</h2>
              <div className={styles.upList}>
                {upNext.map((v) => (
                  <VideoCard key={v.id} video={v} variant="upnext" onClick={() => navigate(`/app/watch/${v.id}`)} />
                ))}
              </div>
            </div>
          </div>

          {/* AI side panel */}
          <aside className={styles.notes}>
            <div className={styles.notesHead}>
              <h2>Sidenote</h2>
              <p>Ask about this video. Answers link back to the timestamp.</p>
            </div>

            <div className={styles.tabs}>
              <button className={styles.tabOn}>Ask</button>
              <button className={styles.tab}>Summary</button>
              <button className={styles.tab}>Key moments</button>
              <button className={styles.tab}>Transcript</button>
            </div>

            <div className={styles.thread} ref={threadRef}>
              {thread.map((m, i) => (
                <div key={i} className={m.role === 'you' ? styles.msgYou : styles.msgAi}>
                  {m.text}
                  {m.jump && (
                    <>
                      {' '}
                      <button className={styles.jump} type="button">Jump to {m.jump}</button>
                    </>
                  )}
                </div>
              ))}
              {pending && (
                <div className={styles.msgAi}>
                  <div className={styles.typing}><i /><i /><i /></div>
                </div>
              )}
            </div>

            <div className={styles.suggest}>
              {SUGGESTIONS.map((s) => (
                <button key={s} className={shared.chip} type="button" onClick={() => ask(s)}>{s}</button>
              ))}
            </div>

            <form className={styles.ask} onSubmit={handleSubmit}>
              <textarea
                ref={textareaRef}
                rows={1}
                placeholder="Ask about what you're watching…"
                aria-label="Ask about this video"
                value={input}
                onChange={handleTextareaChange}
                onKeyDown={handleTextareaKeyDown}
              />
              <button className={styles.send} type="submit" aria-label="Send question">
                <svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.2" aria-hidden="true">
                  <path d="M5 12h13M12 5l7 7-7 7" />
                </svg>
              </button>
            </form>
          </aside>
        </div>
      </main>
    </>
  );
}




















//Watch.module.scss
.appWrap {
  max-width: 1440px;
  margin-inline: auto;
  padding: 26px 24px 64px;

  @media (max-width: 520px) {
    padding-inline: 16px;
  }
}

.back {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  font-size: 14px;
  color: var(--ink-2);
  margin-bottom: 16px;
  cursor: pointer;
  background: none;
  border: 0;
  padding: 0;

  &:hover { color: var(--brand); }
}

.watch {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 396px;
  gap: 28px;
  align-items: start;

  @media (max-width: 1080px) {
    grid-template-columns: 1fr;
  }
}

/* ---------------- player ---------------- */

.player {
  aspect-ratio: 16 / 9;
  border-radius: var(--r-l);
  overflow: hidden;
  position: relative;
  background:
    radial-gradient(120% 90% at 25% 10%, #3F5BE0 0%, transparent 60%),
    linear-gradient(140deg, #101A4A, #1B2A6B 55%, #0A0E24);

  &::after {
    content: "";
    position: absolute;
    inset: 0;
    background: repeating-linear-gradient(115deg,
      rgba(255,255,255,.05) 0 2px, transparent 2px 9px);
  }
}

.playerBar {
  position: absolute;
  left: 0; right: 0; bottom: 0;
  z-index: 3;
  padding: 30px 18px 14px;
  background: linear-gradient(transparent, rgba(6, 9, 26, .74));
  display: flex;
  align-items: center;
  gap: 14px;
}

.scrub {
  flex: 1;
  height: 4px;
  border-radius: 3px;
  background: rgba(255, 255, 255, .28);
  position: relative;

  span {
    position: absolute;
    inset: 0 62% 0 0;
    background: var(--accent);
    border-radius: 3px;

    &::after {
      content: "";
      position: absolute;
      right: -5px; top: 50%;
      width: 11px; height: 11px;
      border-radius: 50%;
      background: var(--accent);
      transform: translateY(-50%);
    }
  }
}

.time {
  font-family: var(--font-mono);
  font-size: 12.5px;
  color: #E8EBF9;
}

.iconBtn {
  width: 34px; height: 34px;
  border-radius: 50%;
  border: 0;
  background: rgba(255, 255, 255, .15);
  color: #fff;
  display: grid;
  place-items: center;
  cursor: pointer;
  flex: none;

  &:hover { background: rgba(255, 255, 255, .28); }
}

/* ---------------- meta ---------------- */

.videoTitle {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: 26px;
  line-height: 1.2;
  letter-spacing: -.035em;
  color: var(--brand-900);
  margin: 20px 0 10px;

  @media (max-width: 520px) {
    font-size: 22px;
  }
}

.videoMeta {
  display: flex;
  align-items: center;
  gap: 14px;
  flex-wrap: wrap;
  font-size: 14px;
  color: var(--ink-3);
  padding-bottom: 18px;
  border-bottom: 1px solid var(--line);
}

.channel {
  display: flex;
  align-items: center;
  gap: 10px;
  color: var(--ink);
  font-weight: 600;
  font-size: 14.5px;
}

.dot {
  width: 34px; height: 34px;
  border-radius: 50%;
  background: var(--tint);
  color: var(--brand-900);
  display: grid;
  place-items: center;
  font-size: 13px;
}

.spacer { margin-left: auto; }

.upNext { margin-top: 30px; }

.upList { display: grid; gap: 16px; }

/* ---------------- Sidenote panel ---------------- */

.notes {
  position: sticky;
  top: 92px;
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: var(--r-l);
  box-shadow: var(--shadow-s);
  display: flex;
  flex-direction: column;
  height: calc(100vh - 122px);
  min-height: 540px;
  overflow: hidden;

  @media (max-width: 1080px) {
    position: static;
    height: auto;
    min-height: 0;
  }
}

.notesHead {
  padding: 16px 18px 12px;
  border-bottom: 1px solid var(--line);

  h2 {
    font-family: var(--font-display);
    font-weight: 700;
    font-size: 18px;
    letter-spacing: -.03em;
    color: var(--brand-900);
    margin: 0 0 3px;
  }

  p {
    margin: 0;
    font-size: 13px;
    color: var(--ink-3);
  }
}

.tabs {
  display: flex;
  gap: 4px;
  padding: 10px 12px;
  border-bottom: 1px solid var(--line);
  background: var(--surface-2);
}

.tab, .tabOn {
  border: 0;
  background: none;
  border-radius: var(--r-pill);
  padding: 6px 14px;
  font-size: 13.5px;
  color: var(--ink-2);
  cursor: pointer;

  &:hover { background: var(--tint); }
}

.tabOn {
  background: var(--brand-900);
  color: #fff;

  &:hover { background: var(--brand-900); }
}

.thread {
  flex: 1;
  overflow-y: auto;
  padding: 18px;
  display: flex;
  flex-direction: column;
  gap: 14px;
  max-height: 100%;

  &::-webkit-scrollbar { width: 8px; }
  &::-webkit-scrollbar-thumb { background: var(--line); border-radius: 8px; }

  @media (max-width: 1080px) {
    max-height: 420px;
  }
}

.msgYou, .msgAi {
  font-size: 14.5px;
  line-height: 1.55;
  border-radius: var(--r-m);
  padding: 11px 14px;
  max-width: 88%;
}

.msgYou {
  background: var(--brand);
  color: #fff;
  align-self: flex-end;
  border-bottom-right-radius: 5px;
}

.msgAi {
  background: var(--surface-2);
  border: 1px solid var(--line);
  color: var(--ink-2);
  align-self: flex-start;
  border-bottom-left-radius: 5px;
}

.jump {
  font-family: var(--font-mono);
  font-size: 12px;
  background: var(--accent);
  color: #3D2606;
  border: 0;
  padding: 2px 7px;
  border-radius: 5px;
  cursor: pointer;

  &:hover { background: var(--accent-600); }
}

.suggest {
  padding: 0 18px 12px;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.ask {
  border-top: 1px solid var(--line);
  padding: 12px;
  display: flex;
  gap: 10px;
  align-items: flex-end;
  background: var(--surface);

  textarea {
    flex: 1;
    resize: none;
    border: 1px solid var(--line);
    border-radius: var(--r-m);
    padding: 11px 14px;
    font-size: 14.5px;
    max-height: 120px;
    background: var(--paper);
    outline: none;
    transition: border-color .18s ease, box-shadow .18s ease, background .18s ease;

    &:focus {
      background: var(--surface);
      border-color: var(--brand);
      box-shadow: 0 0 0 4px rgba(39, 67, 196, .13);
    }
  }
}

.send {
  width: 42px; height: 42px;
  flex: none;
  border: 0;
  border-radius: 50%;
  background: var(--brand);
  color: #fff;
  cursor: pointer;
  display: grid;
  place-items: center;

  &:hover { background: var(--brand-600); }
}

.typing {
  display: flex;
  gap: 4px;
  padding: 4px 2px;

  i {
    width: 6px; height: 6px;
    border-radius: 50%;
    background: var(--ink-3);
    animation: blip 1s infinite ease-in-out;

    &:nth-child(2) { animation-delay: .15s; }
    &:nth-child(3) { animation-delay: .3s; }
  }
}

@keyframes blip {
  0%, 60%, 100% { opacity: .3; transform: translateY(0); }
  30% { opacity: 1; transform: translateY(-3px); }
}
























//global.scss
/* =========================================================
   Sidenote — global styles
   Design tokens live here as CSS custom properties so every
   CSS-Module file in the app can reference var(--brand) etc.
   ========================================================= */

:root {
  /* surfaces */
  --paper:      #F4F5F8;
  --surface:    #FFFFFF;
  --surface-2:  #F8F9FC;

  /* ink */
  --ink:        #141829;
  --ink-2:      #4C5570;
  --ink-3:      #848CA4;
  --line:       #E1E4EE;

  /* brand */
  --brand:      #2743C4;
  --brand-600:  #1F35A3;
  --brand-900:  #101A4A;
  --tint:       #EAEDFB;
  --accent:     #FFB454;
  --accent-600: #F5A23C;

  /* shape */
  --r-s: 8px;
  --r-m: 14px;
  --r-l: 22px;
  --r-pill: 999px;

  --shadow-s: 0 1px 2px rgba(16,26,74,.07);
  --shadow-m: 0 12px 32px -14px rgba(16,26,74,.30);

  /* type */
  --font-display: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-ui: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-mono: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;

  --wrap: 1180px;
}

* { box-sizing: border-box; }

html { scroll-behavior: smooth; }

body {
  margin: 0;
  background: var(--paper);
  color: var(--ink);
  font-family: var(--font-ui);
  font-size: 16px;
  line-height: 1.55;
  -webkit-font-smoothing: antialiased;
}

img, svg { display: block; max-width: 100%; }
a { color: inherit; text-decoration: none; }
button, input, textarea { font: inherit; color: inherit; }

:focus-visible {
  outline: 2px solid var(--brand);
  outline-offset: 3px;
  border-radius: 4px;
}

@media (prefers-reduced-motion: reduce) {
  * { animation-duration: .01ms !important; transition-duration: .01ms !important; }
  html { scroll-behavior: auto; }
}


















//shared.module.scss
/* =========================================================
   Shared, reusable pieces (buttons, wrap, chips, eyebrow…)
   Exposed as a CSS Module so components can do:
     import shared from '../../styles/shared.module.scss';
     <button className={shared.btn + ' ' + shared.btnPrimary}>
   ========================================================= */

.wrap {
  width: 100%;
  max-width: var(--wrap);
  margin-inline: auto;
  padding-inline: 24px;

  @media (max-width: 520px) {
    padding-inline: 16px;
  }
}

.eyebrow {
  font-family: var(--font-mono);
  font-size: 12px;
  letter-spacing: .14em;
  text-transform: uppercase;
  color: var(--brand);
  margin: 0 0 14px;
}

.swipe {
  position: relative;
  white-space: nowrap;

  &::after {
    content: "";
    position: absolute;
    left: -.12em; right: -.12em;
    bottom: .08em;
    height: .32em;
    background: var(--accent);
    border-radius: 2px;
    transform: skewX(-14deg);
    z-index: -1;
  }
}

.btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  border: 1px solid transparent;
  border-radius: var(--r-pill);
  padding: 11px 22px;
  font-size: 15px;
  font-weight: 600;
  letter-spacing: -.005em;
  cursor: pointer;
  transition: background .18s ease, color .18s ease,
              border-color .18s ease, transform .18s ease;

  &:active { transform: translateY(1px); }
}

.btnPrimary {
  background: var(--brand);
  color: #fff;
  box-shadow: var(--shadow-s);

  &:hover { background: var(--brand-600); }
}

.btnGhost {
  background: transparent;
  color: var(--brand-900);
  border-color: var(--line);

  &:hover { border-color: var(--brand); background: var(--tint); }
}

.btnQuiet {
  background: var(--tint);
  color: var(--brand-900);

  &:hover { background: #DDE2F8; }
}

.btnSm { padding: 8px 16px; font-size: 14px; }

.mark {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  font-family: var(--font-display);
  font-weight: 700;
  font-size: 20px;
  letter-spacing: -.035em;
  color: var(--brand-900);
}

.markGlyph {
  width: 26px; height: 26px;
  border-radius: 8px;
  background: var(--brand);
  position: relative;
  flex: none;

  &::before {
    content: "";
    position: absolute;
    inset: 0;
    margin: auto;
    width: 0; height: 0;
    border-left: 8px solid var(--accent);
    border-top: 5px solid transparent;
    border-bottom: 5px solid transparent;
    transform: translateX(1px);
  }
}

.chip {
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: var(--r-pill);
  padding: 9px 16px;
  font-size: 14.5px;
  color: var(--ink-2);
  cursor: pointer;
  transition: border-color .18s ease, color .18s ease, background .18s ease;

  &:hover { border-color: var(--brand); color: var(--brand-900); background: var(--tint); }
}

.sectionTitle {
  font-family: var(--font-display);
  font-weight: 700;
  font-size: 24px;
  letter-spacing: -.032em;
  color: var(--brand-900);
  margin: 8px 0 18px;
}

.thumb {
  aspect-ratio: 16 / 9;
  border-radius: var(--r-m);
  position: relative;
  overflow: hidden;
  background: linear-gradient(145deg, var(--thumb-a, #1B2A6B), var(--thumb-b, #0C1236));
  transition: transform .22s ease, box-shadow .22s ease;

  &::after {
    content: "";
    position: absolute;
    inset: 0;
    background: repeating-linear-gradient(115deg,
      rgba(255,255,255,.06) 0 2px, transparent 2px 10px);
  }
}

.dur {
  position: absolute;
  right: 8px; bottom: 8px;
  z-index: 2;
  font-family: var(--font-mono);
  font-size: 11.5px;
  background: rgba(10,14,34,.82);
  color: #fff;
  padding: 2px 6px;
  border-radius: 5px;
}

.badgeAi {
  position: absolute;
  left: 8px; top: 8px;
  z-index: 2;
  font-family: var(--font-mono);
  font-size: 10.5px;
  letter-spacing: .08em;
  text-transform: uppercase;
  background: var(--accent);
  color: #3D2606;
  padding: 3px 7px;
  border-radius: 5px;
}























//App.jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Landing from './pages/Landing/Landing';
import Library from './pages/Library/Library';
import Watch from './pages/Watch/Watch';

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Landing />} />
        <Route path="/app" element={<Library />} />
        <Route path="/app/watch/:id" element={<Watch />} />
      </Routes>
    </BrowserRouter>
  );
}













//package.json
{
  "name": "sidenote",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.26.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "sass": "^1.77.8",
    "vite": "^5.4.0"
  }
}
