import { useState, useRef, useEffect } from "react";

const SYSTEM_PROMPT = `Eres AgroIA, una inteligencia artificial especializada en agronomía y veterinaria para el campo argentino. Fuiste creada por estudiantes de una escuela de nivel medio profesional en producción agropecuaria y agroalimentaria de Argentina.

Tu rol es asistir a productores agropecuarios, estudiantes y profesionales del agro con:
- **Agronomía**: cultivos, suelos, fertilización, riego, rendimientos, plagas y enfermedades vegetales
- **Veterinaria**: sanidad animal, enfermedades, vacunación, nutrición animal, reproducción
- **Producción**: bovinos, ovinos, porcinos, avicultura, tambos, feedlots
- **Economía agropecuaria**: costos, rentabilidad, visión a futuro del campo
- **Agroalimentos**: procesamiento, calidad, normativas SENASA

Responde siempre en español rioplatense, con vocabulario técnico pero accesible. Cuando sea relevante, menciona condiciones específicas de Argentina (regiones, clima, normativas SENASA/INTA, etc.). Sé preciso, profesional y útil.`;

const suggestedQuestions = [
  "¿Cuál es el rendimiento promedio de soja en la Pampa Húmeda?",
  "¿Cómo prevenir la fiebre aftosa en bovinos?",
  "¿Qué fertilizante usar para maíz en suelos arenosos?",
  "¿Cuáles son las perspectivas del precio del trigo para 2025?",
  "¿Cómo manejar la sanidad en un tambo?",
  "¿Qué es la rotación de cultivos y por qué es importante?",
];

const GrainIcon = () => (
  <svg width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5">
    <path d="M12 2C8 2 5 6 5 10c0 5 7 12 7 12s7-7 7-12c0-4-3-8-7-8z"/>
    <path d="M12 6v8M9 9l3-3 3 3"/>
  </svg>
);

const SendIcon = () => (
  <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
    <path d="M22 2L11 13M22 2l-7 20-4-9-9-4 20-7z"/>
  </svg>
);

const UserIcon = () => (
  <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
    <path d="M12 12c2.7 0 5-2.3 5-5s-2.3-5-5-5-5 2.3-5 5 2.3 5 5 5zm0 2c-3.3 0-10 1.7-10 5v2h20v-2c0-3.3-6.7-5-10-5z"/>
  </svg>
);

const LeafIcon = () => (
  <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.8">
    <path d="M2 22c0 0 4-8 10-10S22 2 22 2s-4 8-10 10S2 22 2 22z"/>
    <path d="M2 22l8-8"/>
  </svg>
);

function TypingDots() {
  return (
    <div style={{ display: "flex", gap: "5px", alignItems: "center", padding: "4px 0" }}>
      {[0, 1, 2].map(i => (
        <div key={i} style={{
          width: 8, height: 8, borderRadius: "50%",
          background: "#4a7c3f",
          animation: `bounce 1.2s ease-in-out ${i * 0.2}s infinite`
        }} />
      ))}
    </div>
  );
}

function Message({ msg }) {
  const isUser = msg.role === "user";
  return (
    <div style={{
      display: "flex",
      justifyContent: isUser ? "flex-end" : "flex-start",
      gap: 10,
      marginBottom: 18,
      animation: "fadeUp 0.3s ease",
    }}>
      {!isUser && (
        <div style={{
          width: 36, height: 36, borderRadius: "50%",
          background: "linear-gradient(135deg, #4a7c3f, #2d5a27)",
          display: "flex", alignItems: "center", justifyContent: "center",
          flexShrink: 0, boxShadow: "0 2px 8px rgba(74,124,63,0.3)",
          color: "#e8f5e1"
        }}>
          <LeafIcon />
        </div>
      )}
      <div style={{
        maxWidth: "75%",
        padding: "12px 16px",
        borderRadius: isUser ? "18px 18px 4px 18px" : "18px 18px 18px 4px",
        background: isUser
          ? "linear-gradient(135deg, #4a7c3f, #2d5a27)"
          : "rgba(255,255,255,0.85)",
        color: isUser ? "#fff" : "#1a2e14",
        fontSize: 14.5,
        lineHeight: 1.65,
        boxShadow: isUser
          ? "0 4px 14px rgba(74,124,63,0.35)"
          : "0 2px 10px rgba(0,0,0,0.08)",
        backdropFilter: "blur(8px)",
        border: isUser ? "none" : "1px solid rgba(74,124,63,0.12)",
        whiteSpace: "pre-wrap",
        wordBreak: "break-word",
      }}>
        {msg.content}
      </div>
      {isUser && (
        <div style={{
          width: 36, height: 36, borderRadius: "50%",
          background: "linear-gradient(135deg, #8db87a, #5a9448)",
          display: "flex", alignItems: "center", justifyContent: "center",
          flexShrink: 0, color: "#fff"
        }}>
          <UserIcon />
        </div>
      )}
    </div>
  );
}

export default function AgroIA() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const [started, setStarted] = useState(false);
  const messagesEndRef = useRef(null);
  const textareaRef = useRef(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, loading]);

  const sendMessage = async (text) => {
    const userText = text || input.trim();
    if (!userText || loading) return;
    setInput("");
    setStarted(true);
    const newMessages = [...messages, { role: "user", content: userText }];
    setMessages(newMessages);
    setLoading(true);

    try {
      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          system: SYSTEM_PROMPT,
          messages: newMessages,
        }),
      });
      const data = await response.json();
      const reply = data.content?.map(b => b.text || "").join("") || "No pude procesar la respuesta.";
      setMessages([...newMessages, { role: "assistant", content: reply }]);
    } catch (err) {
      setMessages([...newMessages, { role: "assistant", content: "⚠️ Hubo un error al conectar con la IA. Verificá tu conexión e intentá de nuevo." }]);
    }
    setLoading(false);
  };

  const handleKey = (e) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  };

  return (
    <div style={{
      minHeight: "100vh",
      background: "linear-gradient(160deg, #0d1f0a 0%, #1a3312 40%, #0f2208 100%)",
      fontFamily: "'Georgia', serif",
      display: "flex",
      flexDirection: "column",
      position: "relative",
      overflow: "hidden",
    }}>
      <style>{`
        @keyframes bounce { 0%,80%,100%{transform:translateY(0)} 40%{transform:translateY(-8px)} }
        @keyframes fadeUp { from{opacity:0;transform:translateY(10px)} to{opacity:1;transform:translateY(0)} }
        @keyframes float { 0%,100%{transform:translateY(0)} 50%{transform:translateY(-6px)} }
        @keyframes pulse { 0%,100%{opacity:0.3} 50%{opacity:0.6} }
        textarea:focus { outline: none; }
        textarea { resize: none; }
        ::-webkit-scrollbar { width: 6px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: rgba(74,124,63,0.4); border-radius: 3px; }
        .suggest-btn:hover { background: rgba(74,124,63,0.25) !important; border-color: rgba(74,124,63,0.6) !important; }
        .send-btn:hover { transform: scale(1.05); box-shadow: 0 4px 18px rgba(74,124,63,0.5) !important; }
      `}</style>

      {/* Background decoration */}
      <div style={{ position: "absolute", inset: 0, overflow: "hidden", pointerEvents: "none" }}>
        {[...Array(6)].map((_, i) => (
          <div key={i} style={{
            position: "absolute",
            width: [180, 120, 200, 90, 150, 110][i],
            height: [180, 120, 200, 90, 150, 110][i],
            borderRadius: "50%",
            background: "radial-gradient(circle, rgba(74,124,63,0.08), transparent)",
            top: ["10%", "70%", "30%", "85%", "50%", "15%"][i],
            left: ["5%", "80%", "60%", "20%", "40%", "75%"][i],
            animation: `pulse ${3 + i * 0.5}s ease-in-out infinite`,
          }} />
        ))}
      </div>

      {/* Header */}
      <header style={{
        padding: "18px 24px",
        display: "flex",
        alignItems: "center",
        gap: 14,
        borderBottom: "1px solid rgba(74,124,63,0.2)",
        backdropFilter: "blur(12px)",
        background: "rgba(13,31,10,0.6)",
        position: "relative", zIndex: 10,
      }}>
        <div style={{
          width: 46, height: 46, borderRadius: 14,
          background: "linear-gradient(135deg, #4a7c3f, #2d5a27)",
          display: "flex", alignItems: "center", justifyContent: "center",
          color: "#c8e6b8", boxShadow: "0 4px 16px rgba(74,124,63,0.4)",
          animation: "float 3s ease-in-out infinite",
        }}>
          <GrainIcon />
        </div>
        <div>
          <h1 style={{ margin: 0, fontSize: 22, fontWeight: 700, color: "#c8e6b8", letterSpacing: "-0.3px" }}>
            AgroIA
          </h1>
          <p style={{ margin: 0, fontSize: 12, color: "rgba(200,230,184,0.55)", fontFamily: "sans-serif" }}>
            Inteligencia Artificial Agropecuaria · Campo Argentino
          </p>
        </div>
        <div style={{ marginLeft: "auto", display: "flex", alignItems: "center", gap: 8 }}>
          <div style={{ width: 8, height: 8, borderRadius: "50%", background: "#4fc848", boxShadow: "0 0 8px #4fc848" }} />
          <span style={{ fontSize: 12, color: "rgba(200,230,184,0.5)", fontFamily: "sans-serif" }}>En línea</span>
        </div>
      </header>

      {/* Main */}
      <main style={{ flex: 1, display: "flex", flexDirection: "column", maxWidth: 820, width: "100%", margin: "0 auto", padding: "0 16px", position: "relative", zIndex: 5 }}>
        {!started ? (
          <div style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", padding: "40px 0 20px", animation: "fadeUp 0.5s ease" }}>
            <div style={{
              width: 80, height: 80, borderRadius: 24,
              background: "linear-gradient(135deg, #4a7c3f22, #2d5a2711)",
              border: "1px solid rgba(74,124,63,0.3)",
              display: "flex", alignItems: "center", justifyContent: "center",
              color: "#8db87a", marginBottom: 20,
              boxShadow: "0 0 40px rgba(74,124,63,0.15)",
            }}>
              <svg width="42" height="42" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.2">
                <path d="M12 2C6 2 2 8 2 12c0 6 10 14 10 14s10-8 10-14c0-4-4-10-10-10z"/>
                <path d="M12 7v10M8 11l4-4 4 4"/>
              </svg>
            </div>
            <h2 style={{ color: "#c8e6b8", fontSize: 26, textAlign: "center", margin: "0 0 8px", fontWeight: 600 }}>
              ¿En qué puedo ayudarte hoy?
            </h2>
            <p style={{ color: "rgba(200,230,184,0.5)", fontSize: 14, textAlign: "center", margin: "0 0 36px", fontFamily: "sans-serif", maxWidth: 480, lineHeight: 1.6 }}>
              Consultame sobre cultivos, sanidad animal, producción, suelos, veterinaria y más. Especializado en el agro argentino.
            </p>
            <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(220px, 1fr))", gap: 10, width: "100%" }}>
              {suggestedQuestions.map((q, i) => (
                <button key={i} className="suggest-btn" onClick={() => sendMessage(q)} style={{
                  padding: "11px 14px",
                  background: "rgba(74,124,63,0.1)",
                  border: "1px solid rgba(74,124,63,0.25)",
                  borderRadius: 12,
                  color: "rgba(200,230,184,0.8)",
                  fontSize: 13,
                  cursor: "pointer",
                  textAlign: "left",
                  fontFamily: "sans-serif",
                  lineHeight: 1.4,
                  transition: "all 0.2s",
                }}>
                  {q}
                </button>
              ))}
            </div>
          </div>
        ) : (
          <div style={{ flex: 1, overflowY: "auto", padding: "24px 0 8px" }}>
            {messages.map((msg, i) => <Message key={i} msg={msg} />)}
            {loading && (
              <div style={{ display: "flex", gap: 10, marginBottom: 18 }}>
                <div style={{
                  width: 36, height: 36, borderRadius: "50%",
                  background: "linear-gradient(135deg, #4a7c3f, #2d5a27)",
                  display: "flex", alignItems: "center", justifyContent: "center",
                  flexShrink: 0, color: "#e8f5e1"
                }}><LeafIcon /></div>
                <div style={{
                  padding: "12px 16px",
                  background: "rgba(255,255,255,0.85)",
                  borderRadius: "18px 18px 18px 4px",
                  border: "1px solid rgba(74,124,63,0.12)",
                }}>
                  <TypingDots />
                </div>
              </div>
            )}
            <div ref={messagesEndRef} />
          </div>
        )}

        {/* Input */}
        <div style={{
          padding: "14px 0 20px",
          position: "sticky", bottom: 0,
        }}>
          <div style={{
            display: "flex",
            gap: 10,
            background: "rgba(255,255,255,0.07)",
            backdropFilter: "blur(16px)",
            border: "1px solid rgba(74,124,63,0.3)",
            borderRadius: 18,
            padding: "10px 10px 10px 18px",
            boxShadow: "0 8px 32px rgba(0,0,0,0.3)",
          }}>
            <textarea
              ref={textareaRef}
              value={input}
              onChange={e => setInput(e.target.value)}
              onKeyDown={handleKey}
              rows={1}
              placeholder="Consultá sobre cultivos, sanidad animal, producción..."
              style={{
                flex: 1, border: "none", background: "transparent",
                color: "#c8e6b8", fontSize: 14.5, fontFamily: "sans-serif",
                lineHeight: 1.5, paddingTop: 4, caretColor: "#8db87a",
                maxHeight: 120,
              }}
            />
            <button
              className="send-btn"
              onClick={() => sendMessage()}
              disabled={!input.trim() || loading}
              style={{
                width: 42, height: 42, borderRadius: 12, border: "none",
                background: input.trim() && !loading
                  ? "linear-gradient(135deg, #4a7c3f, #2d5a27)"
                  : "rgba(74,124,63,0.2)",
                color: input.trim() && !loading ? "#fff" : "rgba(200,230,184,0.3)",
                cursor: input.trim() && !loading ? "pointer" : "not-allowed",
                display: "flex", alignItems: "center", justifyContent: "center",
                flexShrink: 0, transition: "all 0.2s",
                boxShadow: input.trim() ? "0 4px 14px rgba(74,124,63,0.35)" : "none",
              }}
            >
              <SendIcon />
            </button>
          </div>
          <p style={{ textAlign: "center", margin: "8px 0 0", fontSize: 11, color: "rgba(200,230,184,0.25)", fontFamily: "sans-serif" }}>
            AgroIA · Proyecto Escolar · Escuela Agropecuaria Argentina
          </p>
        </div>
      </main>
    </div>
  );
}# agroia
Es una nueva IA que revolucionara el Agro en Argentina. Sacate tus dudas sobre cualquier problema que tengas.
