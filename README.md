import { useState, useEffect, useRef, useCallback } from "react";

// ─── NEWS2 Scoring Engine ──────────────────────────────────────────────────
const NEWS2_THRESHOLDS = {
  spo2_scale1: [
    { range: [0, 91], score: 3 },
    { range: [92, 93], score: 2 },
    { range: [94, 95], score: 1 },
    { range: [96, 100], score: 0 },
  ],
  respRate: [
    { range: [0, 8], score: 3 },
    { range: [9, 11], score: 1 },
    { range: [12, 20], score: 0 },
    { range: [21, 24], score: 2 },
    { range: [25, 99], score: 3 },
  ],
  systolicBP: [
    { range: [0, 90], score: 3 },
    { range: [91, 100], score: 2 },
    { range: [101, 110], score: 1 },
    { range: [111, 219], score: 0 },
    { range: [220, 999], score: 3 },
  ],
  heartRate: [
    { range: [0, 40], score: 3 },
    { range: [41, 50], score: 1 },
    { range: [51, 90], score: 0 },
    { range: [91, 110], score: 1 },
    { range: [111, 130], score: 2 },
    { range: [131, 999], score: 3 },
  ],
  temperature: [
    { range: [0, 35.0], score: 3 },
    { range: [35.1, 36.0], score: 1 },
    { range: [36.1, 38.0], score: 0 },
    { range: [38.1, 39.0], score: 1 },
    { range: [39.1, 99], score: 2 },
  ],
};

function scoreParam(value, thresholds) {
  for (const t of thresholds) {
    if (value >= t.range[0] && value <= t.range[1]) return t.score;
  }
  return 0;
}

function calcNEWS2(vitals) {
  const s1 = scoreParam(vitals.spo2, NEWS2_THRESHOLDS.spo2_scale1);
  const s2 = scoreParam(vitals.respRate, NEWS2_THRESHOLDS.respRate);
  const s3 = scoreParam(vitals.systolicBP, NEWS2_THRESHOLDS.systolicBP);
  const s4 = scoreParam(vitals.heartRate, NEWS2_THRESHOLDS.heartRate);
  const s5 = scoreParam(vitals.temperature, NEWS2_THRESHOLDS.temperature);
  const s6 = vitals.avpu !== "A" ? 3 : 0; // AVPU consciousness score
  const total = s1 + s2 + s3 + s4 + s5 + s6;

  return {
    total,
    components: { spo2: s1, respRate: s2, systolicBP: s3, heartRate: s4, temperature: s5, avpu: s6 },
    risk: total >= 7 ? "HIGH" : total >= 5 ? "MEDIUM" : total >= 1 ? "LOW" : "NORMAL",
    trend: null,
  };
}

// ─── Patient Simulation Engine ─────────────────────────────────────────────
const PATIENT_PROFILES = {
  stable: {
    name: "Rahul M. — Bed 4A",
    age: 58,
    diagnosis: "Post-op Cholecystectomy",
    baseline: { spo2: 98, respRate: 14, systolicBP: 125, heartRate: 72, temperature: 36.8, avpu: "A" },
    drift: { spo2: 0, respRate: 0, systolicBP: 0, heartRate: 0, temperature: 0 },
    noise: { spo2: 0.5, respRate: 0.8, systolicBP: 3, heartRate: 3, temperature: 0.1 },
  },
  deteriorating: {
    name: "Priya S. — Bed 7C",
    age: 72,
    diagnosis: "Community Acquired Pneumonia",
    baseline: { spo2: 96, respRate: 18, systolicBP: 118, heartRate: 88, temperature: 37.9, avpu: "A" },
    drift: { spo2: -0.04, respRate: 0.06, systolicBP: -0.08, heartRate: 0.1, temperature: 0.015 },
    noise: { spo2: 0.6, respRate: 1, systolicBP: 4, heartRate: 4, temperature: 0.15 },
  },
  critical: {
    name: "Dev R. — Bed 2B",
    age: 65,
    diagnosis: "Sepsis — Urinary Source",
    baseline: { spo2: 93, respRate: 24, systolicBP: 95, heartRate: 118, temperature: 38.8, avpu: "A" },
    drift: { spo2: -0.08, respRate: 0.12, systolicBP: -0.15, heartRate: 0.2, temperature: 0.02 },
    noise: { spo2: 0.8, respRate: 1.2, systolicBP: 5, heartRate: 6, temperature: 0.2 },
  },
};

function generateVital(base, drift, noise, tick) {
  const driftedBase = base + drift * tick;
  return driftedBase + (Math.random() - 0.5) * 2 * noise;
}

// ─── Sub-components ────────────────────────────────────────────────────────
const RISK_CONFIG = {
  NORMAL:  { bg: "#0a2a1a", border: "#1a5c35", text: "#4ade80", badge: "#14532d", badgeText: "#86efac", dot: "#22c55e" },
  LOW:     { bg: "#1a2a0a", border: "#4a6a1a", text: "#a3e635", badge: "#3a5a0a", badgeText: "#bef264", dot: "#84cc16" },
  MEDIUM:  { bg: "#2a1a00", border: "#7a4a00", text: "#fbbf24", badge: "#6a3a00", badgeText: "#fde68a", dot: "#f59e0b" },
  HIGH:    { bg: "#2a0a0a", border: "#7a1a1a", text: "#f87171", badge: "#6a0a0a", badgeText: "#fca5a5", dot: "#ef4444" },
};

function VitalCard({ label, value, unit, score, icon, normal }) {
  const isAbnormal = score > 0;
  return (
    <div style={{
      background: isAbnormal ? "rgba(239,68,68,0.08)" : "rgba(255,255,255,0.04)",
      border: `1px solid ${isAbnormal ? "rgba(239,68,68,0.3)" : "rgba(255,255,255,0.1)"}`,
      borderRadius: 12,
      padding: "14px 16px",
      transition: "all 0.4s ease",
    }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 8 }}>
        <span style={{ fontSize: 11, color: "#94a3b8", letterSpacing: "0.05em", textTransform: "uppercase" }}>{label}</span>
        <span style={{
          fontSize: 10, fontWeight: 700, padding: "2px 7px", borderRadius: 20,
          background: isAbnormal ? "rgba(239,68,68,0.2)" : "rgba(34,197,94,0.15)",
          color: isAbnormal ? "#fca5a5" : "#86efac",
        }}>+{score}</span>
      </div>
      <div style={{ display: "flex", alignItems: "baseline", gap: 4 }}>
        <span style={{ fontSize: 28, fontWeight: 800, color: isAbnormal ? "#f87171" : "#f1f5f9", fontFamily: "monospace", letterSpacing: "-0.02em" }}>
          {typeof value === "number" ? (Number.isInteger(value) ? value : value.toFixed(1)) : value}
        </span>
        <span style={{ fontSize: 12, color: "#64748b" }}>{unit}</span>
      </div>
      <div style={{ fontSize: 10, color: "#475569", marginTop: 4 }}>Normal: {normal}</div>
    </div>
  );
}

function MiniSparkline({ data, color }) {
  if (!data || data.length < 2) return null;
  const min = Math.min(...data);
  const max = Math.max(...data);
  const range = max - min || 1;
  const w = 120, h = 32;
  const pts = data.map((v, i) => {
    const x = (i / (data.length - 1)) * w;
    const y = h - ((v - min) / range) * h;
    return `${x},${y}`;
  }).join(" ");

  return (
    <svg width={w} height={h} style={{ display: "block" }}>
      <polyline points={pts} fill="none" stroke={color} strokeWidth={1.5} strokeLinejoin="round" />
      <circle cx={parseFloat(pts.split(" ").at(-1).split(",")[0])} cy={parseFloat(pts.split(" ").at(-1).split(",")[1])} r={3} fill={color} />
    </svg>
  );
}

function ScoreGauge({ score }) {
  const max = 20;
  const pct = Math.min(score / max, 1);
  const color = score >= 7 ? "#ef4444" : score >= 5 ? "#f59e0b" : score >= 1 ? "#84cc16" : "#22c55e";
  const circumference = 2 * Math.PI * 52;
  const offset = circumference * (1 - pct);

  return (
    <div style={{ position: "relative", width: 140, height: 140, margin: "0 auto" }}>
      <svg width={140} height={140} style={{ transform: "rotate(-90deg)" }}>
        <circle cx={70} cy={70} r={52} fill="none" stroke="rgba(255,255,255,0.07)" strokeWidth={10} />
        <circle cx={70} cy={70} r={52} fill="none" stroke={color} strokeWidth={10}
          strokeDasharray={circumference} strokeDashoffset={offset}
          style={{ transition: "stroke-dashoffset 0.6s ease, stroke 0.4s" }}
          strokeLinecap="round" />
      </svg>
      <div style={{ position: "absolute", inset: 0, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center" }}>
        <span style={{ fontSize: 36, fontWeight: 900, color, fontFamily: "monospace", lineHeight: 1 }}>{score}</span>
        <span style={{ fontSize: 11, color: "#64748b", letterSpacing: "0.05em" }}>/ {max}</span>
      </div>
    </div>
  );
}

function AlertBanner({ risk, score, timestamp }) {
  if (risk === "NORMAL") return null;
  const c = RISK_CONFIG[risk];
  const messages = {
    LOW: "Low-level monitoring — check next scheduled round.",
    MEDIUM: "⚠ Escalation recommended — notify charge nurse.",
    HIGH: "🚨 URGENT RESPONSE REQUIRED — activate rapid response team.",
  };
  return (
    <div style={{
      background: c.bg, border: `1px solid ${c.border}`, borderRadius: 10,
      padding: "12px 16px", marginTop: 16,
      display: "flex", alignItems: "center", justifyContent: "space-between", gap: 12,
      animation: risk === "HIGH" ? "pulse 1.5s ease-in-out infinite" : "none",
    }}>
      <div>
        <span style={{ fontSize: 12, fontWeight: 700, color: c.text }}>{risk} RISK — NEWS2 Score {score}</span>
        <div style={{ fontSize: 11, color: "#94a3b8", marginTop: 2 }}>{messages[risk]}</div>
      </div>
      <span style={{ fontSize: 10, color: "#475569", whiteSpace: "nowrap" }}>{timestamp}</span>
    </div>
  );
}

function TrendArrow({ history }) {
  if (history.length < 4) return null;
  const recent = history.slice(-3).map(h => h.total);
  const prev = history.slice(-6, -3).map(h => h.total);
  const rAvg = recent.reduce((a, b) => a + b, 0) / recent.length;
  const pAvg = prev.reduce((a, b) => a + b, 0) / prev.length;
  const delta = rAvg - pAvg;
  if (Math.abs(delta) < 0.3) return <span style={{ color: "#64748b", fontSize: 14 }}>→</span>;
  if (delta > 0) return <span style={{ color: "#ef4444", fontSize: 14 }}>↑ +{delta.toFixed(1)}</span>;
  return <span style={{ color: "#22c55e", fontSize: 14 }}>↓ {delta.toFixed(1)}</span>;
}

// ─── Main App ──────────────────────────────────────────────────────────────
export default function EWSMonitor() {
  const [selectedPatient, setSelectedPatient] = useState("deteriorating");
  const [tick, setTick] = useState(0);
  const [running, setRunning] = useState(true);
  const [vitalsHistory, setVitalsHistory] = useState([]);
  const [scoreHistory, setScoreHistory] = useState([]);
  const [alertLog, setAlertLog] = useState([]);
  const [currentVitals, setCurrentVitals] = useState(null);
  const [currentScore, setCurrentScore] = useState(null);
  const intervalRef = useRef(null);
  const prevRiskRef = useRef("NORMAL");

  const profile = PATIENT_PROFILES[selectedPatient];

  const tick_ = useCallback((t) => {
    const p = PATIENT_PROFILES[selectedPatient];
    const raw = {
      spo2: Math.max(80, Math.min(100, generateVital(p.baseline.spo2, p.drift.spo2, p.noise.spo2, t))),
      respRate: Math.max(5, Math.min(40, generateVital(p.baseline.respRate, p.drift.respRate, p.noise.respRate, t))),
      systolicBP: Math.max(70, Math.min(230, generateVital(p.baseline.systolicBP, p.drift.systolicBP, p.noise.systolicBP, t))),
      heartRate: Math.max(35, Math.min(160, generateVital(p.baseline.heartRate, p.drift.heartRate, p.noise.heartRate, t))),
      temperature: Math.max(34, Math.min(41, generateVital(p.baseline.temperature, p.drift.temperature, p.noise.temperature, t))),
      avpu: t > 80 && p.drift.spo2 < -0.05 ? "V" : "A",
    };
    const score = calcNEWS2(raw);
    const ts = new Date().toLocaleTimeString("en-IN");

    setCurrentVitals(raw);
    setCurrentScore(score);
    setVitalsHistory(h => [...h.slice(-60), raw]);
    setScoreHistory(h => [...h.slice(-60), { ...score, ts }]);

    if (score.risk !== prevRiskRef.current && score.risk !== "NORMAL") {
      setAlertLog(l => [{
        time: ts, risk: score.risk, score: score.total,
        patient: PATIENT_PROFILES[selectedPatient].name,
      }, ...l.slice(0, 9)]);
    }
    prevRiskRef.current = score.risk;
  }, [selectedPatient]);

  useEffect(() => {
    setVitalsHistory([]);
    setScoreHistory([]);
    setAlertLog([]);
    setTick(0);
    prevRiskRef.current = "NORMAL";
  }, [selectedPatient]);

  useEffect(() => {
    if (!running) { clearInterval(intervalRef.current); return; }
    intervalRef.current = setInterval(() => {
      setTick(t => { tick_(t); return t + 1; });
    }, 1200);
    return () => clearInterval(intervalRef.current);
  }, [running, tick_]);

  if (!currentVitals || !currentScore) {
    return <div style={{ background: "#0f1117", color: "#f1f5f9", height: "100vh", display: "flex", alignItems: "center", justifyContent: "center", fontFamily: "system-ui" }}>Initialising sensors…</div>;
  }

  const rc = RISK_CONFIG[currentScore.risk];
  const sHistory = scoreHistory.map(s => s.total);

  return (
    <div style={{
      background: "#0b0d12", minHeight: "100vh", fontFamily: "'Inter', system-ui, sans-serif",
      color: "#f1f5f9", padding: "20px 16px", maxWidth: 900, margin: "0 auto",
    }}>
      <style>{`
        @keyframes pulse { 0%,100%{opacity:1} 50%{opacity:0.7} }
        @keyframes blink { 0%,100%{opacity:1} 50%{opacity:0} }
        * { box-sizing: border-box; }
        ::-webkit-scrollbar { width: 4px; } ::-webkit-scrollbar-thumb { background:#334155; border-radius:4px; }
      `}</style>

      {/* Header */}
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 24, flexWrap: "wrap", gap: 12 }}>
        <div>
          <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
            <div style={{ width: 8, height: 8, borderRadius: "50%", background: running ? "#22c55e" : "#64748b", animation: running ? "blink 1.2s infinite" : "none" }} />
            <span style={{ fontSize: 11, color: "#94a3b8", letterSpacing: "0.12em", textTransform: "uppercase" }}>Live Stream {running ? "Active" : "Paused"}</span>
          </div>
          <h1 style={{ margin: "4px 0 0", fontSize: 22, fontWeight: 900, letterSpacing: "-0.02em" }}>
            NEWS2 Early Warning Monitor
          </h1>
          <p style={{ margin: "2px 0 0", fontSize: 12, color: "#64748b" }}>Real-time ML-scored vital sign surveillance · General Ward</p>
        </div>
        <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
          {Object.entries(PATIENT_PROFILES).map(([key, p]) => (
            <button key={key} onClick={() => setSelectedPatient(key)} style={{
              padding: "7px 14px", borderRadius: 8, border: `1px solid ${selectedPatient === key ? rc.border : "rgba(255,255,255,0.1)"}`,
              background: selectedPatient === key ? rc.bg : "rgba(255,255,255,0.03)",
              color: selectedPatient === key ? rc.text : "#94a3b8",
              fontSize: 12, fontWeight: 600, cursor: "pointer", transition: "all 0.2s",
            }}>{p.name.split("—")[0].trim()}</button>
          ))}
        </div>
      </div>

      {/* Patient Info Bar */}
      <div style={{
        background: "rgba(255,255,255,0.03)", border: "1px solid rgba(255,255,255,0.08)",
        borderRadius: 12, padding: "14px 18px", marginBottom: 20,
        display: "flex", alignItems: "center", justifyContent: "space-between", flexWrap: "wrap", gap: 10,
      }}>
        <div>
          <div style={{ fontSize: 16, fontWeight: 700 }}>{profile.name}</div>
          <div style={{ fontSize: 12, color: "#64748b", marginTop: 2 }}>Age {profile.age} · {profile.diagnosis}</div>
        </div>
        <div style={{ display: "flex", gap: 20, flexWrap: "wrap" }}>
          {[
            ["Tick / Reading", `#${tick.toString().padStart(4,"0")}`],
            ["Trend", null],
            ["Last Alert", alertLog[0] ? `${alertLog[0].time}` : "—"],
          ].map(([label, value]) => (
            <div key={label} style={{ textAlign: "right" }}>
              <div style={{ fontSize: 10, color: "#475569", textTransform: "uppercase", letterSpacing: "0.06em" }}>{label}</div>
              <div style={{ fontSize: 13, fontWeight: 600, color: "#cbd5e1", marginTop: 2 }}>
                {label === "Trend" ? <TrendArrow history={scoreHistory} /> : value}
              </div>
            </div>
          ))}
        </div>
      </div>

      {/* Score + Vitals Grid */}
      <div style={{ display: "grid", gridTemplateColumns: "1fr 2fr", gap: 16, marginBottom: 16 }}>
        {/* Score Panel */}
        <div style={{
          background: rc.bg, border: `1px solid ${rc.border}`,
          borderRadius: 14, padding: 20, display: "flex", flexDirection: "column", alignItems: "center", gap: 12,
        }}>
          <span style={{ fontSize: 11, color: "#94a3b8", textTransform: "uppercase", letterSpacing: "0.1em" }}>NEWS2 Score</span>
          <ScoreGauge score={currentScore.total} />
          <div style={{
            background: rc.badge, color: rc.badgeText,
            fontSize: 13, fontWeight: 800, padding: "6px 20px", borderRadius: 20, letterSpacing: "0.08em",
          }}>{currentScore.risk} RISK</div>
          <div style={{ width: "100%", borderTop: "1px solid rgba(255,255,255,0.06)", paddingTop: 12 }}>
            <div style={{ fontSize: 10, color: "#475569", marginBottom: 6, textTransform: "uppercase", letterSpacing: "0.05em" }}>Score Trend</div>
            <MiniSparkline data={sHistory} color={rc.dot} />
          </div>
          <button onClick={() => setRunning(r => !r)} style={{
            width: "100%", padding: "9px", borderRadius: 8,
            background: running ? "rgba(239,68,68,0.15)" : "rgba(34,197,94,0.15)",
            border: `1px solid ${running ? "rgba(239,68,68,0.3)" : "rgba(34,197,94,0.3)"}`,
            color: running ? "#fca5a5" : "#86efac",
            fontSize: 12, fontWeight: 700, cursor: "pointer",
          }}>{running ? "⏸ Pause Stream" : "▶ Resume Stream"}</button>
        </div>

        {/* Vitals Grid */}
        <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10 }}>
          <VitalCard label="SpO₂" value={currentVitals.spo2} unit="%" score={currentScore.components.spo2} normal="96–100%" />
          <VitalCard label="Respiratory Rate" value={Math.round(currentVitals.respRate)} unit="br/min" score={currentScore.components.respRate} normal="12–20" />
          <VitalCard label="Systolic BP" value={Math.round(currentVitals.systolicBP)} unit="mmHg" score={currentScore.components.systolicBP} normal="111–219" />
          <VitalCard label="Heart Rate" value={Math.round(currentVitals.heartRate)} unit="bpm" score={currentScore.components.heartRate} normal="51–90" />
          <VitalCard label="Temperature" value={currentVitals.temperature} unit="°C" score={currentScore.components.temperature} normal="36.1–38.0" />
          <VitalCard label="AVPU / Consciousness" value={currentVitals.avpu} unit="" score={currentScore.components.avpu} normal="Alert (A)" />
        </div>
      </div>

      {/* Alert Banner */}
      <AlertBanner risk={currentScore.risk} score={currentScore.total} timestamp={new Date().toLocaleTimeString("en-IN")} />

      {/* Score History Chart */}
      <div style={{
        background: "rgba(255,255,255,0.025)", border: "1px solid rgba(255,255,255,0.07)",
        borderRadius: 14, padding: 16, marginTop: 16,
      }}>
        <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 12, alignItems: "center" }}>
          <span style={{ fontSize: 12, fontWeight: 600, color: "#94a3b8" }}>NEWS2 Score History — Last {sHistory.length} readings</span>
          <div style={{ display: "flex", gap: 16 }}>
            {[["≥7 HIGH","#ef4444"],["5–6 MED","#f59e0b"],["1–4 LOW","#84cc16"],["0 OK","#22c55e"]].map(([l, c]) => (
              <span key={l} style={{ fontSize: 10, color: c, display: "flex", alignItems: "center", gap: 4 }}>
                <span style={{ width: 8, height: 8, borderRadius: "50%", background: c, display: "inline-block" }} />{l}
              </span>
            ))}
          </div>
        </div>
        <svg width="100%" height={90} style={{ display: "block" }}>
          {/* Reference lines */}
          {[{y:7,c:"rgba(239,68,68,0.3)",label:"HIGH"},{y:5,c:"rgba(245,158,11,0.3)",label:"MED"}].map(({y,c,label}) => {
            const yPx = 90 - (y / 20) * 90;
            return <g key={y}><line x1="0" y1={yPx} x2="100%" y2={yPx} stroke={c} strokeDasharray="4,4" /><text x={4} y={yPx - 3} fontSize={9} fill={c}>{label} {y}</text></g>;
          })}
          {sHistory.length > 1 && (() => {
            const pts = sHistory.map((v, i) => {
              const x = (i / (sHistory.length - 1)) * 100;
              const y = 90 - (Math.min(v, 20) / 20) * 88;
              return [x, y, v];
            });
            const pathD = pts.map(([x, y], i) => `${i === 0 ? "M" : "L"} ${x}% ${y}`).join(" ");
            const lastV = sHistory.at(-1);
            const lastColor = lastV >=
