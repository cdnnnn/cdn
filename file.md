// ═══════════════════════════════════════════════
// pages/UploadInfer/WorkspacePanel.module.scss
// LectureAI · Step-3 Workspace Result panel
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Keyframes ────────────────────────────────────────────
@keyframes fadein {
    from {
        opacity: 0;
        transform: translateY(2px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(10px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes float {

    0%,
    100% {
        transform: translateY(0);
    }

    50% {
        transform: translateY(-8px);
    }
}

@keyframes wsSpin {
    to {
        transform: rotate(360deg);
    }
}

@keyframes chipFadeIn {
    from {
        opacity: 0;
        transform: scale(0.8) translateY(-6px);
    }

    to {
        opacity: 1;
        transform: scale(1) translateY(0);
    }
}

@keyframes explanationSlideIn {
    from {
        opacity: 0;
        transform: translateY(-6px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes iconPop {
    0% {
        transform: scale(0);
        opacity: 0;
    }

    60% {
        transform: scale(1.2);
    }

    100% {
        transform: scale(1);
        opacity: 1;
    }
}

@keyframes completeFadeIn {
    from {
        opacity: 0;
        transform: scale(0.95);
    }

    to {
        opacity: 1;
        transform: scale(1);
    }
}

@keyframes iconBounce {

    0%,
    100% {
        transform: scale(1);
    }

    50% {
        transform: scale(1.1);
    }
}

// ── Panel shell ──────────────────────────────────────────
.wspanel {
    width: 360px;
    flex-shrink: 0;
    border-left: none;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    background: var(--bg0);
    transition: width 0.25s ease;
    position: relative;

    // Gradient border — only visible when step-2 panel is open beside it
    &::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        width: 1px;
        background: linear-gradient(180deg,
                rgba(139, 92, 246, 0.0) 0%,
                rgba(139, 92, 246, 0.7) 20%,
                rgba(56, 196, 186, 0.8) 55%,
                rgba(240, 160, 48, 0.7) 85%,
                rgba(240, 160, 48, 0.0) 100%);
        pointer-events: none;
        opacity: 0;
        transition: opacity 0.32s ease;
    }
}

// Border fades in only when step-2 is visible next to workspace
.wspanelWithStep2::before {
    opacity: 1;
}

.wspanelExpanded {
    width: auto;
    flex: 1;
}

// ── Header — unchanged from original ────────────────────
.wspanelHead {
    padding: 12px 18px 9px;
    border-bottom: 1px solid var(--bdr);
    background: var(--bg1);
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
}

.headLeft {
    flex: 1;
    min-width: 0;
}

.wsftitle {
    font-size: 16px;
    font-weight: 700;
    color: var(--t0);
    letter-spacing: -0.2px;
    @include m.truncate;
    line-height: 1.3;
}

.wsftitleEmpty {
    font-size: 18px;
    font-weight: 300;
    color: var(--t2);
    opacity: 0.4;
}

.wsmeta {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-top: 5px;
    flex-wrap: wrap;
}

.wsmetaId {
    display: inline-flex;
    align-items: center;
    font-size: 10px;
    font-weight: 700;
    font-family: var(--font-mono);
    color: #a78bfa;
    background: rgba(167, 139, 250, 0.12);
    border: 1px solid rgba(167, 139, 250, 0.3);
    border-radius: 4px;
    padding: 1px 6px;
    flex-shrink: 0;
    letter-spacing: 0.02em;
}

.wsmetaSep {
    color: var(--t2);
    opacity: 0.4;
    font-size: 13px;
}

.wsmetaDate {
    font-size: 13px;
    color: var(--t2);
    font-family: var(--font-mono);
}

.wsmetaHint {
    font-size: 13px;
    color: var(--t2);
    font-family: var(--font-mono);
    opacity: 0.6;
    font-style: italic;
}

// ── Tab bar ─────────────────────────────────────────────
.wsptabsWrap {
    position: relative;
    display: flex;
    align-items: center;
    border-bottom: 1px solid var(--bdr);
    background: var(--bg1);
    flex-shrink: 0;
}

.wsptabs {
    display: flex;
    padding: 0 16px;
    flex: 1;
    overflow-x: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;

    &::-webkit-scrollbar {
        display: none;
    }
}

.tabFade {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 28px;
    pointer-events: none;
    z-index: 1;
}

.tabFadeLeft {
    left: 28px;
    background: linear-gradient(90deg, var(--bg1), transparent);
}

.tabFadeRight {
    right: 28px;
    background: linear-gradient(270deg, var(--bg1), transparent);
}

.tabScrollBtn {
    flex-shrink: 0;
    align-self: stretch;
    width: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    border: none;
    background: var(--bg1);
    color: var(--t2);
    cursor: pointer;
    z-index: 2;
    transition: color 0.12s;

    svg {
        width: 13px;
        height: 13px;
    }

    &:hover {
        color: var(--t0);
    }
}

.tab {
    padding: 0 14px;
    height: 36px;
    display: flex;
    align-items: center;
    font-size: 14px;
    color: var(--t2);
    cursor: pointer;
    border: none;
    border-bottom: 2px solid transparent;
    background: transparent;
    font-family: var(--font-ui);
    transition: all 0.12s;
    user-select: none;
    white-space: nowrap;
    flex-shrink: 0;

    &:hover {
        color: var(--t1);
    }

    &.active {
        color: var(--blue);
        border-bottom-color: var(--blue);
        font-weight: 500;
    }
}

// ── Tab body ─────────────────────────────────────────────
.wsbody {
    flex: 1;
    overflow-y: auto;
    padding: 16px 18px;
    background: var(--bg0);
    @include m.scrollbar;
    animation: fadein 0.15s ease;
    display: flex;
    flex-direction: column;
}

// ── Empty state ──────────────────────────────────────────
.wsEmpty {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    text-align: center;

    svg {
        width: 44px;
        height: 44px;
        color: var(--t2);
        opacity: 0.25;
        flex-shrink: 0;
        animation: float 3s ease-in-out infinite;
    }
}

.wsEmptyTitle {
    font-size: 16px;
    font-weight: 600;
    color: var(--t1);
}

.wsEmptyDesc {
    font-size: 14px;
    color: var(--t2);
    @include m.mono;
    line-height: 1.6;
    max-width: 200px;
}

// ── Loading spinner ──────────────────────────────────────
.wsSpinner {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    color: var(--t2);
    font-size: 14px;
    @include m.mono;
}

.spinner {
    width: 28px;
    height: 28px;
    border: 2.5px solid var(--bdr2);
    border-top-color: var(--blue);
    border-radius: 50%;
    animation: wsSpin 0.7s linear infinite;
    flex-shrink: 0;
}

// ── Tab content wrapper ──────────────────────────────────
.tabContent {
    display: flex;
    flex-direction: column;
    gap: 10px;
    animation: fadein 0.15s ease;
    flex: 1;
    min-height: 0;
}

.tabEmpty {
    font-size: 14px;
    color: var(--t2);
    @include m.mono;
    padding: 24px 0;
    text-align: center;
}

// ── Summary — marked library output ─────────────────────
// Font sizes and spacing match Angular .markdown-content-inline exactly
.summaryMd {
    font-size: 14px;
    color: var(--t1);
    line-height: 1.7;

    // ── Paragraphs ──────────────────────────────────────
    // Angular: margin: 8px 0, line-height: 1.7
    p {
        margin: 8px 0;
        line-height: 1.7;

        &:first-child {
            margin-top: 0;
        }

        &:last-child {
            margin-bottom: 0;
        }
    }

    // ── Headings ─────────────────────────────────────────
    // Angular h1: 18px, margin 12px 0 8px 0, padding-bottom 6px
    h1 {
        font-size: 18px;
        font-weight: 700;
        color: var(--blue);
        margin: 12px 0 8px;
        padding-bottom: 6px;
        border-bottom: 2px solid rgba(139, 92, 246, 0.3);
        line-height: 1.3;
        position: relative;

        &:first-child {
            margin-top: 0;
        }

        &::before {
            content: '';
            position: absolute;
            bottom: -2px;
            left: 0;
            width: 60px;
            height: 2px;
            background: var(--blue);
        }
    }

    // Angular h2: 16px, margin 10px 0 6px 0, padding-left 12px
    h2 {
        font-size: 16px;
        font-weight: 700;
        color: #9f7aea;
        margin: 10px 0 6px;
        padding-left: 12px;
        line-height: 1.3;
        position: relative;

        &:first-child {
            margin-top: 0;
        }

        &::before {
            content: '';
            position: absolute;
            left: 0;
            top: 50%;
            transform: translateY(-50%);
            width: 4px;
            height: 18px;
            background: linear-gradient(180deg, var(--blue), #a78bfa);
            border-radius: 2px;
        }
    }

    // Angular h3: 14px, margin 8px 0 4px 0
    h3 {
        font-size: 14px;
        font-weight: 700;
        color: #a78bfa;
        margin: 8px 0 4px;
        line-height: 1.3;

        &:first-child {
            margin-top: 0;
        }
    }

    // Angular h4/h5/h6: 13px, margin 6px 0 4px 0
    h4,
    h5,
    h6 {
        font-size: 14px;
        font-weight: 600;
        color: #b794f4;
        margin: 6px 0 4px;
        line-height: 1.3;

        &:first-child {
            margin-top: 0;
        }
    }

    // ── Lists ────────────────────────────────────────────
    // Angular: margin 8px 0, padding-left 20px
    ul,
    ol {
        margin: 8px 0;
        padding-left: 20px;

        &:last-child {
            margin-bottom: 0;
        }
    }

    // Angular li: margin 4px 0, line-height 1.6, padding-left 8px
    li {
        margin: 4px 0;
        line-height: 1.6;
        padding-left: 8px;

        ul,
        ol {
            margin-top: 4px;
            margin-bottom: 4px;
        }
    }

    ul li::marker {
        color: var(--blue);
        font-weight: bold;
    }

    ol li::marker {
        color: var(--blue);
        font-weight: bold;
    }

    // ── Inline formatting ────────────────────────────────
    // Angular strong: color primary, background gradient, padding 2px 4px
    strong {
        font-weight: 700;
        color: var(--blue);
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.1),
                rgba(167, 139, 250, 0.05));
        padding: 2px 4px;
        border-radius: 3px;
    }

    em {
        font-style: italic;
        color: #a78bfa;
    }

    // ── HR ───────────────────────────────────────────────
    // Angular: height 2px, gradient, margin 16px 0, opacity 0.3
    hr {
        border: none;
        height: 2px;
        background: linear-gradient(90deg, transparent, var(--blue), transparent);
        margin: 16px 0;
        opacity: 0.3;
    }

    // ── Blockquote ───────────────────────────────────────
    // Angular: border-left 3px, padding 12px 12px 12px 16px, margin 12px 0
    blockquote {
        border-left: 3px solid var(--blue);
        margin: 12px 0;
        padding: 12px 12px 12px 16px;
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.08),
                rgba(167, 139, 250, 0.05));
        border-radius: 0 6px 6px 0;
        font-style: italic;
        opacity: 0.9;
        position: relative;

        &::before {
            content: '"';
            position: absolute;
            top: 8px;
            left: 8px;
            font-size: 32px;
            color: var(--blue);
            opacity: 0.3;
            font-family: Georgia, serif;
            line-height: 0;
        }

        // marked wraps blockquote text in <p>
        p {
            margin: 0;
            line-height: 1.6;
        }
    }

    // ── Inline code ──────────────────────────────────────
    // Angular: padding 3px 8px, font-size 0.9em (~11.7px), border rgba purple
    code {
        font-family: var(--font-mono);
        font-size: 0.9em;
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.15),
                rgba(167, 139, 250, 0.1));
        border: 1px solid rgba(139, 92, 246, 0.3);
        border-radius: 4px;
        padding: 3px 8px;
        color: var(--blue);
        font-weight: 500;
    }

    // ── Code block ───────────────────────────────────────
    // Angular: padding 14px, margin 12px 0, border rgba purple
    pre {
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.08),
                rgba(167, 139, 250, 0.05));
        border: 1px solid rgba(139, 92, 246, 0.25);
        border-radius: 8px;
        padding: 14px;
        overflow-x: auto;
        margin: 12px 0;
        position: relative;

        &:last-child {
            margin-bottom: 0;
        }

        // Angular pre::before — purple left accent bar
        &::before {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            width: 4px;
            height: 100%;
            background: linear-gradient(180deg, var(--blue), #a78bfa);
            border-radius: 8px 0 0 8px;
        }

        code {
            background: transparent;
            border: none;
            padding: 0;
            font-size: 0.9em;
            color: var(--t1);
            font-weight: normal;
        }
    }

    // ── Tables ───────────────────────────────────────────
    table {
        width: 100%;
        border-collapse: collapse;
        margin: 12px 0;
        border-radius: 6px;
        overflow: hidden;
    }

    th,
    td {
        padding: 10px 12px;
        border: 1px solid rgba(139, 92, 246, 0.2);
        text-align: left;
    }

    th {
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.15),
                rgba(167, 139, 250, 0.1));
        font-weight: 700;
        color: var(--blue);
        border-bottom: 2px solid rgba(139, 92, 246, 0.3);
    }

    tr:hover {
        background: rgba(139, 92, 246, 0.03);
    }

    // ── Links ────────────────────────────────────────────
    a {
        color: var(--blue);
        text-decoration: none;
        font-weight: 500;
        border-bottom: 1px solid transparent;
        transition: border-color 0.2s, color 0.2s;

        &:hover {
            color: #9f7aea;
            border-bottom-color: var(--blue);
        }
    }

    // First child: no top margin
    >*:first-child {
        margin-top: 0;
    }

    // Last child: no bottom margin
    >*:last-child {
        margin-bottom: 0;
    }
}

// ── Keywords — gradient chips ────────────────────────────
.keywordGrid {
    display: flex;
    flex-wrap: wrap;
    gap: 7px;
}

.keywordPill {
    display: inline-flex;
    align-items: center;
    padding: 6px 12px;
    border-radius: 8px;
    font-size: 13px;
    font-weight: 600;
    @include m.mono;
    color: #fff;
    // background set inline via style prop (gradient)
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.15);
    animation: chipFadeIn 0.25s ease-out;
    position: relative;
    overflow: hidden;
    transition: transform 0.2s, box-shadow 0.2s;
    cursor: default;

    // shine on hover
    &::before {
        content: '';
        position: absolute;
        top: 0;
        left: -100%;
        width: 100%;
        height: 100%;
        background: linear-gradient(90deg,
                transparent,
                rgba(255, 255, 255, 0.25),
                transparent);
        transition: left 0.42s ease;
    }

    &:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);

        &::before {
            left: 100%;
        }
    }
}

// ── Assessment — progress bar ────────────────────────────
.assessProgress {
    display: flex;
    align-items: center;
    gap: 10px;
}

.assessProgressTrack {
    flex: 1;
    height: 4px;
    background: var(--bg3);
    border-radius: 99px;
    overflow: hidden;
}

.assessProgressFill {
    height: 100%;
    background: linear-gradient(90deg, var(--blue), #a78bfa);
    border-radius: 99px;
    transition: width 0.3s ease;
}

.assessProgressLabel {
    font-size: 13px;
    font-family: var(--font-mono);
    font-weight: 700;
    color: var(--blue);
    white-space: nowrap;
    flex-shrink: 0;
}

// ── Assessment — question card ───────────────────────────
.faqCard {
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 12px 14px;
    display: flex;
    flex-direction: column;
    gap: 9px;
    animation: fadeInUp 0.2s ease-out;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:hover {
        border-color: var(--blue-bdr);
        box-shadow: 0 2px 8px rgba(139, 92, 246, 0.07);
    }
}

.faqQ {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.5;
    display: flex;
    gap: 8px;
    align-items: flex-start;
}

.faqNum {
    font-size: 13px;
    font-weight: 700;
    color: var(--blue);
    background: var(--blue-dim);
    border: 1px solid var(--blue-bdr);
    border-radius: 4px;
    padding: 1px 6px;
    flex-shrink: 0;
    margin-top: 2px;
    @include m.mono;
}

.faqOptions {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

// ── Option button — base ─────────────────────────────────
.faqOpt {
    display: flex;
    align-items: center;
    gap: 9px;
    font-size: 13px;
    color: var(--t1);
    padding: 7px 10px;
    border-radius: 6px;
    border: 2px solid var(--bdr);
    background: var(--bg0);
    cursor: pointer;
    text-align: left;
    font-family: var(--font-ui);
    width: 100%;
    transition: border-color 0.15s, background 0.15s, transform 0.15s;

    &:hover:not(:disabled) {
        border-color: var(--blue-bdr);
        background: var(--blue-dim);
        transform: translateX(2px);
    }

    &:disabled {
        cursor: default;
    }
}

// Circle key badge
.faqOptKey {
    width: 22px;
    height: 22px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    background: var(--bg3);
    border: 1.5px solid var(--bdr2);
    font-size: 13px;
    font-weight: 700;
    color: var(--t1);
    @include m.mono;
    transition: background 0.15s, border-color 0.15s, color 0.15s;
}

.faqOptVal {
    flex: 1;
    line-height: 1.45;
}

// ── Option states ────────────────────────────────────────
.faqOptSelected {
    border-color: var(--blue-bdr);
    background: var(--blue-dim);

    .faqOptKey {
        background: var(--blue);
        border-color: var(--blue);
        color: #fff;
    }
}

.faqOptCorrect {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    .faqOptKey {
        background: var(--green);
        border-color: var(--green);
        color: #fff;
    }

    .faqOptVal {
        color: var(--green);
        font-weight: 600;
    }
}

.faqOptCorrectAlt {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    .faqOptKey {
        background: var(--green);
        border-color: var(--green);
        color: #fff;
    }

    .faqOptVal {
        color: var(--green);
        font-weight: 600;
    }
}

.faqOptWrong {
    border-color: var(--red-bdr);
    background: var(--red-dim);

    .faqOptKey {
        background: var(--red);
        border-color: var(--red);
        color: #fff;
    }

    .faqOptVal {
        color: var(--red);
        font-weight: 600;
    }
}

// ── Check / X icons ──────────────────────────────────────
.faqCheckIcon,
.faqXIcon {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
    margin-left: auto;
    animation: iconPop 0.25s ease-out;
}

.faqCheckIcon {
    color: var(--green);
}

.faqXIcon {
    color: var(--red);
}

// ── Explanation box ──────────────────────────────────────
.faqExplain {
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-left: 3px solid var(--blue-bdr);
    border-radius: 0 5px 5px 0;
    padding: 8px 10px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    animation: explanationSlideIn 0.2s ease-out;
}

.faqExplainLabel {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 13px;
    font-weight: 600;
    color: var(--blue);
    text-transform: uppercase;
    letter-spacing: 0.06em;
    @include m.mono;

    svg {
        width: 12px;
        height: 12px;
        flex-shrink: 0;
    }
}

.faqExplainText {
    font-size: 13px;
    color: var(--t2);
    line-height: 1.6;
    @include m.mono;
    margin: 0;
}

// ── Next / Finish button ─────────────────────────────────
.assessNextBtn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 5px;
    padding: 6px 14px; // matches .btnSm padding scale
    border-radius: var(--r);
    border: none;
    background: var(--blue);
    color: #fff;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    align-self: flex-end;
    animation: fadein 0.15s ease;
    box-shadow: 0 2px 6px rgba(91, 164, 239, 0.28);
    transition: opacity 0.12s, transform 0.15s, box-shadow 0.15s;

    svg {
        width: 12px;
        height: 12px;
    }

    &:hover {
        opacity: 0.9;
        transform: translateY(-1px);
        box-shadow: 0 4px 10px rgba(91, 164, 239, 0.35);
    }
}

.assessDoneBtn {
    background: var(--green);
    box-shadow: 0 2px 6px rgba(74, 222, 128, 0.28);

    &:hover {
        box-shadow: 0 4px 10px rgba(74, 222, 128, 0.35);
    }
}

// ── Complete screen ──────────────────────────────────────
.assessComplete {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 28px 16px;
    text-align: center;
    animation: completeFadeIn 0.3s ease-out;
}

.assessCompleteIcon {
    width: 56px;
    height: 56px;
    background: var(--green-dim);
    border: 2px solid var(--green-bdr);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    animation: iconBounce 0.5s ease-out;

    svg {
        width: 32px;
        height: 32px;
        color: var(--green);
    }
}

.assessCompleteTitle {
    font-size: 16px;
    font-weight: 700;
    color: var(--t0);
}

.assessCompleteDesc {
    font-size: 14px;
    color: var(--t2);
    @include m.mono;
}

.assessRestartBtn {
    margin-top: 4px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 20px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t1);
    font-size: 14px;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.12s;

    &:hover {
        background: var(--bg3);
        color: var(--t0);
        border-color: var(--bdr3);
    }
}

// (tabToolbar and editIconBtn replaced by actionBtn/dropdown system below)

// ── Edit mode layout ─────────────────────────────────────
.editMode {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
}

.editLayout {
    display: flex;
    gap: 12px;
    flex: 1;
    min-height: 0;
    position: relative;
    padding-bottom: 52px; // room for fixed footer
}

.editCol {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.editColHeader {
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.editColLabel {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.07em;
    color: var(--t2);
}

.editColTag {
    font-size: 10px;
    font-weight: 500;
    color: var(--t2);
    opacity: 0.55;
    font-style: italic;
    text-transform: none;
    letter-spacing: 0;
}

.editTextarea {
    flex: 1;
    resize: none;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    color: var(--t1);
    font-size: 13px;
    font-family: var(--font-mono);
    line-height: 1.6;
    padding: 10px 12px;
    outline: none;
    min-height: 280px;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px rgba(91, 164, 239, 0.12);
    }
}

.editPreview {
    flex: 1;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 10px 12px;
    overflow-y: auto;
    min-height: 280px;
}

// ── Fixed footer inside edit layout ──────────────────────
.editFooter {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 48px;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 8px;
    border-top: 1px solid var(--bdr);
    background: var(--bg1);
    padding: 0 4px;
    border-radius: 0 0 var(--r) var(--r);
}

.cancelBtn {
    padding: 6px 14px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t1);
    font-size: 13px;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.12s;

    &:hover:not(:disabled) {
        background: var(--bg3);
        border-color: var(--bdr3);
    }

    &:disabled {
        opacity: 0.4;
        cursor: default;
    }
}

.saveBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 16px;
    border-radius: var(--r);
    border: none;
    background: var(--blue);
    color: #fff;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: opacity 0.12s;
    box-shadow: 0 2px 6px rgba(91, 164, 239, 0.28);

    &:hover:not(:disabled) {
        opacity: 0.88;
    }

    &:disabled {
        opacity: 0.55;
        cursor: default;
    }
}

// ── Inline spinner for save button ───────────────────────
.inlineSpinner {
    width: 11px;
    height: 11px;
    border: 1.5px solid rgba(255, 255, 255, 0.35);
    border-top-color: #fff;
    border-radius: 50%;
    animation: wsSpin 0.65s linear infinite;
    flex-shrink: 0;
}

// ── Tab toolbar — icon-only buttons ──────────────────────
.tabToolbar {
    display: flex;
    align-items: center;
    gap: 4px;
    justify-content: flex-end;
    margin-bottom: 6px;
}

.actionBtn {
    width: 28px;
    height: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 7px;
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t2);
    cursor: pointer;
    padding: 0;
    transition: all 0.15s;
    flex-shrink: 0;

    svg {
        width: 13px;
        height: 13px;
    }

    &:hover {
        background: var(--blue-dim);
        border-color: var(--blue-bdr);
        color: var(--blue);
    }
}

.actionBtnActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
}

.successIcon {
    color: var(--green) !important;
}

// ── Dropdown ──────────────────────────────────────────────
.dropdownWrap {
    position: relative;
}

.dropdown {
    position: absolute;
    top: calc(100% + 5px);
    right: 0;
    min-width: 168px;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    box-shadow: 0 6px 20px rgba(0, 0, 0, 0.18);
    z-index: 50;
    overflow: hidden;
    animation: ddFade 0.12s ease;
}

@keyframes ddFade {
    from {
        opacity: 0;
        transform: translateY(-4px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.dropdownLabel {
    font-size: 10px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.07em;
    color: var(--t2);
    padding: 7px 11px 4px;
    opacity: 0.6;
}

.dropdownItem {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 100%;
    text-align: left;
    padding: 7px 11px;
    font-size: 13px;
    color: var(--t1);
    background: transparent;
    border: none;
    cursor: pointer;
    font-family: var(--font-ui);
    transition: background 0.1s, color 0.1s;

    &:hover {
        background: var(--blue-dim);
        color: var(--blue);
    }
}

.dropdownItemLabel {
    flex: 1;
    min-width: 0;
}

// Per-format colored glyph in copy/download menus
.fmtIcon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 6px;
    flex-shrink: 0;
    transition: transform 0.12s ease;

    svg {
        width: 13px;
        height: 13px;
    }
}

.dropdownItem:hover .fmtIcon {
    transform: scale(1.06);
}

// ── Keyword pill — new chip animation ────────────────────
.keywordPillNew {
    animation: chipFadeIn 0.25s ease-out;
}

// ── Assessment header row (mode tabs + toolbar) ───────────
.assessHeader {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 10px;
    gap: 8px;
}

// ── Mode toggle tabs ──────────────────────────────────────
.assessModeTabs {
    display: flex;
    align-items: center;
    gap: 3px;
    background: var(--bg2);
    border: 1px solid var(--bdr);
    border-radius: 8px;
    padding: 3px;
}

.assessModeTab {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 10px;
    border-radius: 6px;
    border: none;
    background: transparent;
    color: var(--t2);
    font-size: 12px;
    font-weight: 500;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.15s;

    svg {
        width: 12px;
        height: 12px;
        flex-shrink: 0;
    }

    &:hover {
        color: var(--t1);
    }
}

.assessModeTabActive {
    background: var(--bg0);
    color: var(--blue);
    font-weight: 600;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12);

    svg {
        stroke: var(--blue);
    }
}

// ── View All list ─────────────────────────────────────────
.viewAllList {
    display: flex;
    flex-direction: column;
    gap: 14px;
    animation: fadein 0.15s ease;
}

.viewAllCard {
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;

    &:hover {
        border-color: var(--blue-bdr);
        box-shadow: 0 2px 8px rgba(139, 92, 246, 0.07);
    }
}

.viewAllQ {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.5;
    display: flex;
    gap: 8px;
    align-items: flex-start;
}

// Neutral option style for View All (non-correct options)
.viewAllOptNeutral {
    cursor: default !important;
    opacity: 0.65;

    &:hover {
        transform: none !important;
        border-color: var(--bdr) !important;
        background: var(--bg0) !important;
    }
}

// Correct key badge in View All mode
.faqOptKeyCorrect {
    background: var(--green) !important;
    border-color: var(--green) !important;
    color: #fff !important;
}

// ── Large screen overrides (> 1900px) ────────────────────
@media (min-width: 1920px) {
    .wspanel {
        width: 400px;
    }

    .wsftitle {
        font-size: 18px;
    }

    .wsftitleEmpty {
        font-size: 20px;
    }

    .wsmetaId {
        font-size: 11px;
    }

    .wsmetaDate {
        font-size: 14px;
    }

    .wsmetaHint {
        font-size: 14px;
    }

    .tab {
        font-size: 15px;
    }

    .wsEmptyTitle {
        font-size: 18px;
    }

    .wsEmptyDesc {
        font-size: 15px;
    }

    .wsSpinner {
        font-size: 15px;
    }

    .tabEmpty {
        font-size: 15px;
    }

    .summaryMd {
        font-size: 15px;

        h1 {
            font-size: 20px;
        }

        h2 {
            font-size: 17px;
        }

        h3 {
            font-size: 15px;
        }

        h4,
        h5,
        h6 {
            font-size: 15px;
        }
    }

    .keywordPill {
        font-size: 14px;
    }

    .assessProgressLabel {
        font-size: 14px;
    }

    .faqQ {
        font-size: 15px;
    }

    .faqNum {
        font-size: 14px;
    }

    .faqOpt {
        font-size: 14px;
    }

    .faqOptKey {
        font-size: 14px;
    }

    .faqExplainLabel {
        font-size: 14px;
    }

    .faqExplainText {
        font-size: 14px;
    }

    .assessNextBtn {
        font-size: 14px;
    }

    .assessCompleteTitle {
        font-size: 18px;
    }

    .assessCompleteDesc {
        font-size: 15px;
    }

    .assessRestartBtn {
        font-size: 15px;
    }

    .dropdownItem {
        font-size: 14px;
    }

    .dropdownLabel {
        font-size: 11px;
    }

    .assessModeTab {
        font-size: 13px;
    }

    .viewAllQ {
        font-size: 15px;
    }

    .editColLabel {
        font-size: 12px;
    }

    .editColTag {
        font-size: 11px;
    }

    .editTextarea {
        font-size: 14px;
    }

    .cancelBtn {
        font-size: 14px;
    }

    .saveBtn {
        font-size: 14px;
    }
}
// ═══════════════════════════════════════════════
// New content types — Short Answer, True/False,
// Timestamped Summary, Keyword Insights
// ═══════════════════════════════════════════════

// ── Short Answer ──────────────────────────────────
.revealAllBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border-radius: 6px;
    border: 1px solid var(--bdr2);
    background: var(--bg0);
    color: var(--t1);
    font-size: 12.5px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.13s;

    svg { width: 14px; height: 14px; }

    &:hover {
        border-color: var(--blue-bdr);
        color: var(--blue);
        background: var(--blue-dim);
    }
}

.revealBtn {
    margin-top: 10px;
    padding: 7px 14px;
    border-radius: 6px;
    border: 1px dashed var(--bdr2);
    background: transparent;
    color: var(--t2);
    font-size: 12.5px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.13s;

    &:hover {
        border-color: var(--blue-bdr);
        border-style: solid;
        color: var(--blue);
        background: var(--blue-dim);
    }
}

.shortAnswerBox {
    margin-top: 10px;
    padding: 12px 14px;
    background: rgba(79, 172, 254, 0.08);
    border-left: 3px solid #4facfe;
    border-radius: 0 8px 8px 0;
}

.shortAnswerLabel {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: #4facfe;
    margin-bottom: 4px;
    @include m.mono;
}

.shortAnswerText {
    font-size: 13.5px;
    color: var(--t0);
    line-height: 1.55;
    margin: 0;
}

// ── True / False ──────────────────────────────────
.tfOptions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
    margin-top: 12px;
}

.tfOpt {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    font-size: 14px;
    font-weight: 700;
    color: var(--t1);
    padding: 16px 10px;
    border-radius: 8px;
    border: 2px solid var(--bdr);
    background: var(--bg0);
    cursor: pointer;
    text-align: center;
    font-family: var(--font-ui);
    transition: border-color 0.15s, background 0.15s, transform 0.15s;

    &:hover:not(:disabled) {
        border-color: var(--blue-bdr);
        background: var(--blue-dim);
        transform: translateY(-1px);
    }

    &:disabled {
        cursor: default;
    }
}

.tfOptLabel {
    letter-spacing: 0.02em;
}

// ── Timestamped Summary ───────────────────────────
.tsList {
    display: flex;
    flex-direction: column;
    margin-top: 4px;
}

.tsRow {
    display: flex;
    gap: 14px;
}

.tsRail {
    display: flex;
    flex-direction: column;
    align-items: center;
    flex-shrink: 0;
    padding-top: 6px;
}

.tsDot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: var(--blue);
    box-shadow: 0 0 0 3px var(--blue-dim);
    flex-shrink: 0;
}

.tsLine {
    width: 2px;
    flex: 1;
    background: var(--bdr2);
    margin-top: 2px;
}

.tsCard {
    flex: 1;
    padding-bottom: 20px;
}

.tsRange {
    font-size: 12px;
    font-weight: 700;
    color: var(--blue);
    margin-bottom: 4px;
    @include m.mono;
}

.tsArrow {
    color: var(--t2);
    margin: 0 2px;
}

.tsText {
    font-size: 13.5px;
    color: var(--t1);
    line-height: 1.6;
    margin: 0;
}

// ── Keyword Insights: sub-nav (scrollable, one line) ──
.kiSubNavWrap {
    position: relative;
    display: flex;
    align-items: center;
    margin-bottom: 16px;
    padding-bottom: 12px;
    border-bottom: 1px solid var(--bdr);

    .tabScrollBtn {
        background: var(--bg0);
    }

    .tabFadeLeft {
        background: linear-gradient(90deg, var(--bg0), transparent);
    }

    .tabFadeRight {
        background: linear-gradient(270deg, var(--bg0), transparent);
    }
}

.kiSubNav {
    display: flex;
    gap: 4px;
    flex: 1;
    overflow-x: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;

    &::-webkit-scrollbar {
        display: none;
    }
}

.kiSubTab {
    padding: 5px 11px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: var(--bg0);
    color: var(--t2);
    font-size: 12px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.13s;
    white-space: nowrap;
    flex-shrink: 0;

    &:hover {
        color: var(--t0);
        border-color: var(--bdr3, var(--bdr2));
    }
}

.kiSubTabActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
}

.kiBody {
    min-height: 200px;
    flex: 1;
    display: flex;
    flex-direction: column;
}

// ── Keyword Insights: node-link graph (React Flow mindmap) ──
.graphWrap {
    width: 100%;
    flex: 1;
    min-height: 360px;
    border-radius: 10px;
    overflow: hidden;
    border: 1px solid var(--bdr);
    background: var(--bg0);

    :global(.react-flow__attribution) {
        display: none;
    }

    :global(.react-flow__controls) {
        background: var(--bg1);
        border: 1px solid var(--bdr2);
        border-radius: 8px;
        overflow: hidden;
        box-shadow: none;
    }

    :global(.react-flow__controls-button) {
        background: var(--bg1);
        border-bottom: 1px solid var(--bdr2);
        color: var(--t1);

        &:hover {
            background: var(--bg3);
        }

        svg {
            fill: var(--t1);
        }
    }

    :global(.react-flow__minimap) {
        border-radius: 8px;
        border: 1px solid var(--bdr2);
        overflow: hidden;
    }

    :global(.react-flow__edge-path) {
        stroke: var(--bdr2);
    }

    :global(.react-flow__edge:hover) .react-flow__edge-path,
    :global(.react-flow__edge.selected) .react-flow__edge-path {
        stroke: var(--blue);
    }

    :global(.react-flow__edge-text) {
        fill: var(--t2);
    }
}

.rfNode {
    border-radius: 50%;
    background: var(--blue-dim);
    border: 1.6px solid var(--blue);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: grab;
    position: relative;
    transition: background 0.15s;

    &:active {
        cursor: grabbing;
    }

    &:hover {
        background: var(--blue);

        .rfNodeLabel {
            color: var(--t0);
        }
    }
}

.rfNodeLabel {
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    margin-top: 4px;
    font-size: 10.5px;
    color: var(--t1);
    white-space: nowrap;
    font-family: var(--font-ui);
    pointer-events: none;
}

.rfHandle {
    width: 6px !important;
    height: 6px !important;
    min-width: 0 !important;
    background: transparent !important;
    border: none !important;
    opacity: 0;
}

// ── Keyword Insights: matrix / heatmap grid ───────
.matrixScroll {
    overflow-x: auto;
    @include m.scrollbar;
}

.matrixGrid {
    display: grid;
    gap: 2px;
    width: max-content;
    min-width: 100%;
}

.matrixCorner {
    background: transparent;
}

.matrixColHead {
    font-size: 10px;
    color: var(--t2);
    writing-mode: vertical-rl;
    transform: rotate(180deg);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-height: 90px;
    padding: 4px 2px;
    text-align: left;
    @include m.mono;
}

.matrixRowHead {
    font-size: 11.5px;
    color: var(--t1);
    padding: 4px 8px 4px 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 140px;
    display: flex;
    align-items: center;
}

.matrixCell {
    min-width: 28px;
    aspect-ratio: 1;
    border-radius: 3px;
    border: 1px solid var(--bdr);
}

// ── Keyword Insights: word cloud ──────────────────
.wordCloud {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: center;
    gap: 12px 18px;
    padding: 24px 12px;
}

.wcWord {
    font-weight: 700;
    font-family: var(--font-ui);
    line-height: 1;
    cursor: default;
    transition: transform 0.15s;

    &:hover {
        transform: scale(1.08);
    }
}

// ── Keyword Insights: clusters ────────────────────
.clusterGrid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 12px;
}

.clusterCard {
    padding: 12px 14px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 10px;
}

.clusterTitle {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--t2);
    margin-bottom: 8px;
    @include m.mono;
}

.clusterChips {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
}

.clusterChip {
    padding: 3px 9px;
    border-radius: 99px;
    background: var(--blue-dim);
    border: 1px solid var(--blue-bdr);
    color: var(--blue);
    font-size: 11.5px;
    font-weight: 600;
}

// ── Keyword Insights: frequency bars ──────────────
.freqList {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.freqRow {
    display: grid;
    grid-template-columns: 130px 1fr 130px;
    align-items: center;
    gap: 12px;
}

.freqLabel {
    font-size: 12.5px;
    color: var(--t1);
    font-weight: 600;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.freqBarTrack {
    height: 10px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 99px;
    overflow: hidden;
}

.freqBarFill {
    height: 100%;
    background: linear-gradient(90deg, var(--blue), #a78bfa);
    border-radius: 99px;
}

.freqMeta {
    display: flex;
    gap: 8px;
    align-items: center;
    font-size: 11px;
    color: var(--t1);
    justify-content: flex-end;
    @include m.mono;
}

.freqDim {
    color: var(--t2);
}

// ── Keyword Insights: importance/complexity scatter ──
.scatterWrap {
    width: 100%;
    display: flex;
    justify-content: center;
}

.scatterSvg {
    width: 100%;
    max-width: 580px;
    height: auto;
}

.scatterAxis {
    stroke: var(--bdr2);
    stroke-width: 1.4;
}

.scatterAxisLabel {
    font-size: 10.5px;
    fill: var(--t2);
    font-family: var(--font-ui);
}

.scatterDot {
    fill: var(--blue-dim);
    stroke: var(--blue);
    stroke-width: 1.6;
}

.scatterDotLabel {
    font-size: 9.5px;
    fill: var(--t1);
    font-family: var(--font-ui);
    pointer-events: none;
}

// ── Keyword Insights: glossary ────────────────────
.glossaryList {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.glossaryRow {
    padding: 10px 14px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 8px;
}

.glossaryTerm {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 13.5px;
    font-weight: 700;
    color: var(--t0);
    margin-bottom: 4px;
}

.glossaryTime {
    font-size: 10.5px;
    font-weight: 600;
    color: var(--t2);
    background: var(--bg1);
    border: 1px solid var(--bdr2);
    border-radius: 99px;
    padding: 1px 7px;
    @include m.mono;
}

.glossaryDef {
    font-size: 13px;
    color: var(--t1);
    line-height: 1.55;
}
