import { useState, useRef, useEffect } from "react";

const SYSTEM_PROMPT = `You are a warm, grounded mental health companion. Your role is to offer a safe, non-judgmental space for people to reflect, process emotions, and feel heard.

Guidelines:
- Lead with empathy. Acknowledge feelings before offering anything else.
- Ask one thoughtful follow-up question at a time — never multiple at once.
- Never diagnose, prescribe, or replace professional therapy. If someone is in crisis, always recommend they contact a crisis line (988 in the US) or emergency services.
- Be concise — 2–4 sentences per response unless the person clearly wants more.
- Don't project emotions — ask instead of assume.
- You can gently suggest breathing, grounding, or journaling when appropriate.
- Speak like a warm human, not a clinical chatbot.

Crisis keywords to watch: "suicide", "kill myself", "end it all", "can't go on", "self-harm". If detected, warmly but clearly direct the person to crisis resources.`;

const MOODS = [
  { label: "Struggling", emoji: "😔", value: 1, color: "#7c6fa0" },
  { label: "Low", emoji: "😞", value: 2, color: "#8a7faf" },
  { label: "Okay", emoji: "😐", value: 3, color: "#6b9ab8" },
  { label: "Good", emoji: "🙂", value: 4, color: "#5ba08a" },
  { label: "Great", emoji: "😊", value: 5, color: "#4d9e6b" },
];

const PROMPTS = [
  "I've been feeling anxious lately…",
  "I need to talk through something",
  "I'm having trouble sleeping",
  "I feel disconnected from people",
  "Work has been overwhelming me",
  "I want to feel more like myself",
];

const BREATHING_STEPS = [
  { label: "Breathe in", duration: 4, color: "#7c9cbf" },
  { label: "Hold", duration: 4, color: "#9b89b8" },
  { label: "Breathe out", duration: 6, color: "#6baa8e" },
  { label: "Hold", duration: 2, color: "#b89b78" },
];

export default function App() {
  const [view, setView] = useState("home"); // home | chat | mood | breathe | journal
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const [selectedMood, setSelectedMood] = useState(null);
  const [moodHistory, setMoodHistory] = useState([]);
  const [moodNote, setMoodNote] = useState("");
  const [breathStep, setBreathStep] = useState(0);
  const [breathProgress, setBreathProgress] = useState(0);
  const [breathActive, setBreathActive] = useState(false);
  const [journalEntry, setJournalEntry] = useState("");
  const [journalSaved, setJournalSaved] = useState(false);
  const [journalEntries, setJournalEntries] = useState([]);
  const [crisisWarning, setCrisisWarning] = useState(false);
  const messagesEndRef = useRef(null);
  const breathTimerRef = useRef(null);
  const breathProgressRef = useRef(null);

  const CRISIS_WORDS = ["suicide", "kill myself", "end it all", "can't go on", "self-harm", "hurt myself"];

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  // Breathing animation
  useEffect(() => {
    if (breathActive) {
      const step = BREATHING_STEPS[breathStep];
      const totalMs = step.duration * 1000;
      const interval = 50;
      let elapsed = 0;
      breathProgressRef.current = setInterval(() => {
        elapsed += interval;
        setBreathProgress(Math.min(elapsed / totalMs, 1));
        if (elapsed >= totalMs) {
          clearInterval(breathProgressRef.current);
          breathTimerRef.current = setTimeout(() => {
            setBreathStep((s) => (s + 1) % BREATHING_STEPS.length);
            setBreathProgress(0);
          }, 200);
        }
      }, interval);
    }
    return () => {
      clearInterval(breathProgressRef.current);
      clearTimeout(breathTimerRef.current);
    };
  }, [breathActive, breathStep]);

  const sendMessage = async (text) => {
    const userText = text || input.trim();
    if (!userText || loading) return;
    const lower = userText.toLowerCase();
    const hasCrisis = CRISIS_WORDS.some((w) => lower.includes(w));
    if (hasCrisis) setCrisisWarning(true);
    const newMessages = [...messages, { role: "user", content: userText }];
    setMessages(newMessages);
    setInput("");
    setLoading(true);
    try {
      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-6",
          max_tokens: 1000,
          system: SYSTEM_PROMPT,
          messages: newMessages,
        }),
      });
      const data = await response.json();
      const reply = data.content?.[0]?.text || "I'm here with you. Can you tell me more?";
      setMessages([...newMessages, { role: "assistant", content: reply }]);
    } catch {
      setMessages([...newMessages, { role: "assistant", content: "I'm having trouble connecting. Please try again in a moment." }]);
    }
    setLoading(false);
  };

  const saveMood = () => {
    if (!selectedMood) return;
    const entry = {
      mood: selectedMood,
      note: moodNote,
      time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }),
      date: new Date().toLocaleDateString([], { month: "short", day: "numeric" }),
    };
    setMoodHistory((prev) => [entry, ...prev].slice(0, 14));
    setSelectedMood(null);
    setMoodNote("");
  };

  const saveJournal = () => {
    if (!journalEntry.trim()) return;
    const entry = {
      text: journalEntry,
      date: new Date().toLocaleDateString([], { month: "short", day: "numeric" }),
      time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }),
    };
    setJournalEntries((prev) => [entry, ...prev].slice(0, 20));
    setJournalEntry("");
    setJournalSaved(true);
    setTimeout(() => setJournalSaved(false), 2500);
  };

  const styles = {
    app: {
      fontFamily: "'Inter', system-ui, sans-serif",
      background: "linear-gradient(160deg, #0f1117 0%, #161824 50%, #1a1628 100%)",
      minHeight: "100vh",
      color: "#e8e4f0",
      display: "flex",
      flexDirection: "column",
      maxWidth: 480,
      margin: "0 auto",
      position: "relative",
    },
    nav: {
      display: "flex",
      justifyContent: "space-around",
      padding: "12px 0 16px",
      borderTop: "1px solid rgba(255,255,255,0.06)",
      background: "rgba(15,17,23,0.92)",
      backdropFilter: "blur(12px)",
      position: "fixed",
      bottom: 0,
      left: "50%",
      transform: "translateX(-50%)",
      width: "100%",
      maxWidth: 480,
      zIndex: 10,
    },
    navBtn: (active) => ({
      display: "flex",
      flexDirection: "column",
      alignItems: "center",
      gap: 4,
      background: "none",
      border: "none",
      cursor: "pointer",
      padding: "6px 14px",
      borderRadius: 12,
      color: active ? "#a78bfa" : "rgba(255,255,255,0.35)",
      fontSize: 10,
      fontWeight: active ? 600 : 400,
      transition: "all 0.2s",
    }),
    header: {
      padding: "52px 24px 20px",
      borderBottom: "1px solid rgba(255,255,255,0.06)",
    },
  };

  const NavBar = () => (
    <nav style={styles.nav}>
      {[
        { id: "home", icon: "✦", label: "Home" },
        { id: "chat", icon: "◎", label: "Talk" },
        { id: "mood", icon: "◐", label: "Mood" },
        { id: "breathe", icon: "◯", label: "Breathe" },
        { id: "journal", icon: "◫", label: "Journal" },
      ].map((item) => (
        <button key={item.id} style={styles.navBtn(view === item.id)} onClick={() => setView(item.id)}>
          <span style={{ fontSize: 18 }}>{item.icon}</span>
          {item.label}
        </button>
      ))}
    </nav>
  );

  // ── HOME ──────────────────────────────────────────
  if (view === "home") return (
    <div style={styles.app}>
      <div style={{ padding: "60px 24px 100px", flex: 1 }}>
        <div style={{ marginBottom: 36 }}>
          <div style={{ fontSize: 11, letterSpacing: 3, color: "#7c6fa0", textTransform: "uppercase", marginBottom: 8 }}>MindBloom</div>
          <h1 style={{ fontSize: 32, fontWeight: 700, lineHeight: 1.2, margin: 0, color: "#f0ecff" }}>
            How are you<br />doing today?
          </h1>
          <p style={{ fontSize: 14, color: "rgba(255,255,255,0.4)", marginTop: 10, lineHeight: 1.6 }}>
            A quiet space to check in, breathe, and talk things through.
          </p>
        </div>
        {/* Quick mood */}
        <div style={{ background: "rgba(255,255,255,0.04)", borderRadius: 16, padding: 20, marginBottom: 20, border: "1px solid rgba(255,255,255,0.07)" }}>
          <div style={{ fontSize: 12, color: "rgba(255,255,255,0.4)", marginBottom: 14, letterSpacing: 1 }}>QUICK CHECK-IN</div>
          <div style={{ display: "flex", gap: 8, justifyContent: "space-between" }}>
            {MOODS.map((m) => (
              <button key={m.value} onClick={() => { setView("mood"); setSelectedMood(m); }}
                style={{ background: "rgba(255,255,255,0.05)", border: "1px solid rgba(255,255,255,0.08)", borderRadius: 12, padding: "10px 6px", cursor: "pointer", flex: 1, fontSize: 20, transition: "all 0.2s" }}>
                {m.emoji}
              </button>
            ))}
          </div>
        </div>
        {/* Prompt starters */}
        <div style={{ marginBottom: 20 }}>
          <div style={{ fontSize: 12, color: "rgba(255,255,255,0.4)", marginBottom: 14, letterSpacing: 1 }}>START A CONVERSATION</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
            {PROMPTS.slice(0, 4).map((p) => (
              <button key={p} onClick={() => { setView("chat"); setTimeout(() => sendMessage(p), 100); }}
                style={{ background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.07)", borderRadius: 12, padding: "12px 16px", cursor: "pointer", color: "rgba(255,255,255,0.7)", fontSize: 13, textAlign: "left", transition: "all 0.2s" }}>
                {p}
              </button>
            ))}
          </div>
        </div>
        {moodHistory.length > 0 && (
          <div style={{ background: "rgba(255,255,255,0.03)", borderRadius: 16, padding: 16, border: "1px solid rgba(255,255,255,0.06)" }}>
            <div style={{ fontSize: 12, color: "rgba(255,255,255,0.4)", marginBottom: 10, letterSpacing: 1 }}>RECENT MOOD</div>
            <div style={{ display: "flex", gap: 8 }}>
              {moodHistory.slice(0, 7).map((e, i) => (
                <div key={i} title={${e.mood.label} · ${e.date}}
                  style={{ width: 32, height: 32, borderRadius: 8, background: e.mood.color + "40", border: 1px solid ${e.mood.color}60, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16 }}>
                  {e.mood.emoji}
                </div>
              ))}
            </div>
          </div>
        )}
      </div>
      <NavBar />
    </div>
  );

  // ── CHAT ──────────────────────────────────────────
  if (view === "chat") return (
    <div style={{ ...styles.app }}>
      <div style={{ ...styles.header }}>
        <div style={{ fontSize: 11, letterSpacing: 3, color: "#7c6fa0", textTransform: "uppercase" }}>MindBloom</div>
        <div style={{ fontSize: 18, fontWeight: 600, marginTop: 4 }}>Talk it through</div>
      </div>
      {crisisWarning && (
        <div style={{ margin: "12px 16px 0", background: "rgba(180,60,60,0.15)", border: "1px solid rgba(220,80,80,0.3)", borderRadius: 12, padding: "12px 16px" }}>
          <div style={{ fontSize: 13, fontWeight: 600, color: "#f87171", marginBottom: 4 }}>You're not alone</div>
          <div style={{ fontSize: 12, color: "rgba(255,255,255,0.6)", lineHeight: 1.6 }}>
            If you're in crisis, please reach out to the <strong style={{ color: "#fca5a5" }}>988 Suicide & Crisis Lifeline</strong> — call or text <strong style={{ color: "#fca5a5" }}>988</strong> (US). You matter.
          </div>
          <button onClick={() => setCrisisWarning(false)} style={{ marginTop: 8, background: "none", border: "none", color: "rgba(255,255,255,0.35)", fontSize: 11, cursor: "pointer" }}>Dismiss</button>
        </div>
      )}
      <div style={{ flex: 1, overflowY: "auto", padding: "16px 16px 0", display: "flex", flexDirection: "column", gap: 12, paddingBottom: 160 }}>
        {messages.length === 0 && (
          <div style={{ textAlign: "center", padding: "40px 20px", color: "rgba(255,255,255,0.3)" }}>
            <div style={{ fontSize: 36, marginBottom: 12 }}>◎</div>
            <div style={{ fontSize: 14, lineHeight: 1.7 }}>This is a safe space.<br />Share what's on your mind.</div>
          </div>
        )}
        {messages.map((m, i) => (
          <div key={i} style={{ display: "flex", justifyContent: m.role === "user" ? "flex-end" : "flex-start" }}>
            <div style={{
              maxWidth: "82%", padding: "12px 16px", borderRadius: m.role === "user" ? "18px 18px 4px 18px" : "18px 18px 18px 4px",
              background: m.role === "user" ? "linear-gradient(135deg, #6d5acd, #4c3f9f)" : "rgba(255,255,255,0.07)",
              border: m.role === "assistant" ? "1px solid rgba(255,255,255,0.08)" : "none",
              fontSize: 14, lineHeight: 1.6, color: "#f0ecff",
            }}>
              {m.content}
            </div>
          </div>
        ))}
        {loading && (
          <div style={{ display: "flex" }}>
            <div style={{ background: "rgba(255,255,255,0.07)", border: "1px solid rgba(255,255,255,0.08)", borderRadius: "18px 18px 18px 4px", padding: "14px 18px" }}>
              <div style={{ display: "flex", gap: 5 }}>
                {[0, 1, 2].map((i) => (
                  <div key={i} style={{ width: 6, height: 6, borderRadius: "50%", background: "#7c6fa0", animation: pulse 1.2s ${i * 0.2}s infinite }} />
                ))}
              </div>
            </div>
          </div>
        )}
        <div ref={messagesEndRef} />
      </div>
      <div style={{ position: "fixed", bottom: 64, left: "50%", transform: "translateX(-50%)", width: "100%", maxWidth: 480, padding: "12px 16px", background: "rgba(15,17,23,0.95)", backdropFilter: "blur(12px)", borderTop: "1px solid rgba(255,255,255,0.06)" }}>
        <div style={{ display: "flex", gap: 10 }}>
          <input value={input} onChange={(e) => setInput(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && !e.shiftKey && sendMessage()}
            placeholder="What's on your mind?"
            style={{ flex: 1, background: "rgba(255,255,255,0.07)", border: "1px solid rgba(255,255,255,0.1)", borderRadius: 24, padding: "12px 18px", color: "#f0ecff", fontSize: 14, outline: "none" }} />
          <button onClick={() => sendMessage()} disabled={!input.trim() || loading}
            style={{ width: 46, height: 46, borderRadius: "50%", background: input.trim() ? "linear-gradient(135deg, #6d5acd, #4c3f9f)" : "rgba(255,255,255,0.08)", border: "none", cursor: input.trim() ? "pointer" : "default", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 18, transition: "all 0.2s" }}>
            ↑
          </button>
        </div>
      </div>
      <NavBar />
      <style>{@keyframes pulse { 0%,80%,100%{opacity:0.3;transform:scale(0.8)} 40%{opacity:1;transform:scale(1)} }}</style>
    </div>
  );

  // ── MOOD ──────────────────────────────────────────
  if (view === "mood") return (
    <div style={styles.app}>
      <div style={styles.header}>
        <div style={{ fontSize: 11, letterSpacing: 3, color: "#7c6fa0", textTransform: "uppercase" }}>MindBloom</div>
        <div style={{ fontSize: 18, fontWeight: 600, marginTop: 4 }}>Mood tracker</div>
      </div>
      <div style={{ flex: 1, padding: "24px 20px 100px", overflowY: "auto" }}>
        <div style={{ marginBottom: 28 }}>
          <div style={{ fontSize: 13, color: "rgba(255,255,255,0.5)", marginBottom: 16 }}>How are you feeling right now?</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
            {MOODS.map((m) => (
              <button key={m.value} onClick={() => setSelectedMood(m)}
                style={{ display: "flex", alignItems: "center", gap: 14, padding: "14px 18px", background: selectedMood?.value === m.value ? m.color + "30" : "rgba(255,255,255,0.04)", border: 1px solid ${selectedMood?.value === m.value ? m.color : "rgba(255,255,255,0.07)"}, borderRadius: 14, cursor: "pointer", transition: "all 0.2s", textAlign: "left" }}>
                <span style={{ fontSize: 24 }}>{m.emoji}</span>
                <span style={{ fontSize: 15, color: "#f0ecff", fontWeight: selectedMood?.value === m.value ? 600 : 400 }}>{m.label}</span>
                {selectedMood?.value === m.value && <span style={{ marginLeft: "auto", color: m.color, fontSize: 18 }}>✓</span>}
              </button>
            ))}
          </div>
        </div>
        {selectedMood && (
          <div style={{ marginBottom: 24 }}>
            <div style={{ fontSize: 13, color: "rgba(255,255,255,0.5)", marginBottom: 10 }}>Add a note (optional)</div>
            <textarea value={moodNote} onChange={(e) => setMoodNote(e.target.value)} placeholder="What's contributing to this feeling?"
              style={{ width: "100%", background: "rgba(255,255,255,0.05)", border: "1px solid rgba(255,255,255,0.1)", borderRadius: 14, padding: 16, color: "#f0ecff", fontSize: 13, lineHeight: 1.6, resize: "none", outline: "none", minHeight: 90, boxSizing: "border-box" }} />
            <button onClick={saveMood}
              style={{ width: "100%", marginTop: 12, background: "linear-gradient(135deg, #6d5acd, #4c3f9f)", border: "none", borderRadius: 14, padding: 16, color: "#fff", fontSize: 15, fontWeight: 600, cursor: "pointer" }}>
              Save check-in
            </button>
          </div>
        )}
        {moodHistory.length > 0 && (
          <div>
            <div style={{ fontSize: 12, color: "rgba(255,255,255,0.4)", letterSpacing: 1, marginBottom: 14 }}>RECENT CHECK-INS</div>
            <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
              {moodHistory.map((e, i) => (
                <div key={i} style={{ display: "flex", alignItems: "center", gap: 14, padding: "12px 16px", background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.06)", borderRadius: 12 }}>
                  <span style={{ fontSize: 22 }}>{e.mood.emoji}</span>
                  <div style={{ flex: 1 }}>
                    <div style={{ fontSize: 13, color: "#f0ecff", fontWeight: 500 }}>{e.mood.label}</div>
                    {e.note && <div style={{ fontSize: 12, color: "rgba(255,255,255,0.4)", marginTop: 2 }}>{e.note}</div>}
                  </div>
                  <div style={{ fontSize: 11, color: "rgba(255,255,255,0.3)", textAlign: "right" }}>
                    <div>{e.date}</div>
                    <div>{e.time}</div>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}
      </div>
      <NavBar />
    </div>
  );

  // ── BREATHE ───────────────────────────────────────
  if (view === "breathe") {
    const step = BREATHING_STEPS[breathStep];
    const radius = 90;
    const circumference = 2 * Math.PI * radius;
    const strokeDashoffset = circumference * (1 - breathProgress);
    const scale = breathStep === 0 ? 1 + breathProgress * 0.25 : breathStep === 2 ? 1.25 - breathProgress * 0.25 : 1.25;
    return (
      <div style={styles.app}>
        <div style={styles.header}>
          <div style={{ fontSize: 11, letterSpacing: 3, color: "#7c6fa0", textTransform: "uppercase" }}>MindBloom</div>
          <div style={{ fontSize: 18, fontWeight: 600, marginTop: 4 }}>Box breathing</div>
          <div style={{ fontSize: 13, color: "rgba(255,255,255,0.4)", marginTop: 4 }}>4-4-6-2 pattern for calm</div>
        </div>
        <div style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", padding: "0 24px 100px" }}>
          <div style={{ position: "relative", width: 240, height: 240, display: "flex", alignItems: "center", justifyContent: "center", marginBottom: 40 }}>
            <svg width={240} height={240} style={{ position: "absolute", top: 0, left: 0, transform: "rotate(-90deg)" }}>
              <circle cx={120} cy={120} r={radius} fill="none" stroke="rgba(255,255,255,0.06)" strokeWidth={3} />
              <circle cx={120} cy={120} r={radius} fill="none" stroke={step.color} strokeWidth={3}
                strokeDasharray={circumference} strokeDashoffset={strokeDashoffset}
                strokeLinecap="round" style={{ transition: "stroke-dashoffset 0.05s linear, stroke 0.3s" }} />
            </svg>
            <div style={{ width: 160, height: 160, borderRadius: "50%", background: radial-gradient(circle, ${step.color}25, ${step.color}08), border: 1px solid ${step.color}40, transform: scale(${scale}), transition: "transform 0.05s linear", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap: 6 }}>
              <div style={{ fontSize: 22, fontWeight: 700, color: "#f0ecff" }}>{step.label}</div>
              <div style={{ fontSize: 38, fontWeight: 300, color: step.color }}>{step.duration}</div>
            </div>
          </div>
          <div style={{ display: "flex", gap: 8, marginBottom: 40 }}>
            {BREATHING_STEPS.map((s, i) => (
              <div key={i} style={{ width: 8, height: 8, borderRadius: "50%", background: i === breathStep ? s.color : "rgba(255,255,255,0.15)", transition: "all 0.3s" }} />
            ))}
          </div>
          <button onClick={() => { setBreathActive((a) => !a); if (!breathActive) { setBreathStep(0); setBreathProgress(0); } }}
            style={{ background: breathActive ? "rgba(255,255,255,0.08)" : "linear-gradient(135deg, #6d5acd, #4c3f9f)", border: "1px solid rgba(255,255,255,0.15)", borderRadius: 50, padding: "16px 48px", color: "#fff", fontSize: 16, fontWeight: 600, cursor: "pointer" }}>
            {breathActive ? "Pause" : "Begin"}
          </button>
          <div style={{ marginTop: 32, fontSize: 13, color: "rgba(255,255,255,0.3)", lineHeight: 1.7, textAlign: "center", maxWidth: 280 }}>
            Box breathing activates your parasympathetic nervous system, reducing stress within minutes.
          </div>
        </div>
        <NavBar />
      </div>
   );
  }
  
  // ── JOURNAL ───────────────────────────────────────
  if (view === "journal") return (
    <div style={styles.app}>
      <div style={styles.header}>
        <div style={{ fontSize: 11, letterSpacing: 3, color: "#7c6fa0", textTransform: "uppercase" }}>MindBloom</div>
        <div style={{ fontSize: 18, fontWeight: 600, marginTop: 4 }}>Journal</div>
        <div style={{ fontSize: 13, color: "rgba(255,255,255,0.4)", marginTop: 4 }}>Private. Just for you.</div>
      </div>
      <div style={{ flex: 1, padding: "20px 20px 100px", overflowY: "auto" }}>
        <div style={{ marginBottom: 24 }}>
          <textarea value={journalEntry} onChange={(e) => setJournalEntry(e.target.value)}
            placeholder={"What's present for you today?\n\nThis space is yours."}
            style={{ width: "100%", minHeight: 180, background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.08)", borderRadius: 16, padding: 18, color: "#f0ecff", fontSize: 14, lineHeight: 1.8, resize: "none", outline: "none", boxSizing: "border-box", fontFamily: "inherit" }} />
          <button onClick={saveJournal} disabled={!journalEntry.trim()}
            style={{ width: "100%", marginTop: 12, background: journalEntry.trim() ? "linear-gradient(135deg, #6d5acd, #4c3f9f)" : "rgba(255,255,255,0.07)", border: "none", borderRadius: 14, padding: 16, color: journalEntry.trim() ? "#fff" : "rgba(255,255,255,0.3)", fontSize: 15, fontWeight: 600, cursor: journalEntry.trim() ? "pointer" : "default" }}>
            {journalSaved ? "✓ Saved" : "Save entry"}
          </button>
        </div>
        {journalEntries.length > 0 && (
          <div>
            <div style={{ fontSize: 12, color: "rgba(255,255,255,0.4)", letterSpacing: 1, marginBottom: 14 }}>PAST ENTRIES</div>
            <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
              {journalEntries.map((e, i) => (
                <div key={i} style={{ background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.07)", borderRadius: 14, padding: "14px 16px" }}>
                  <div style={{ fontSize: 11, color: "rgba(255,255,255,0.3)", marginBottom: 8 }}>{e.date} · {e.time}</div>
                  <div style={{ fontSize: 13, color: "rgba(255,255,255,0.7)", lineHeight: 1.7 }}>{e.text.length > 200 ? e.text.slice(0, 200) + "…" : e.text}</div>
                </div>
              ))}
            </div>
          </div>
        )}
      </div>
      <NavBar />
    </div>
  );
}
