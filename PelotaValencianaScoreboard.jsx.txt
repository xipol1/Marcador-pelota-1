import React, { useReducer, useEffect } from 'react';

/* ============================================================
   PELOTA VALENCIANA — LLARGUES con RAYAS · v3
   - Setup: pueblo + hasta 5 jugadores por equipo + qui saca
   - Puntos directos también en RALLA 1
   - Botones +1 visuales en cada panel
   ============================================================ */

const TARGET_GAMES = 10;
const MAX_PLAYERS = 5;
const POINT_LABELS = ['0', '15', '30', 'VAL'];

const C = {
  bg: '#F2EBDC',
  paper: '#EDE3D0',
  paperDark: '#E2D6BC',
  ink: '#1A1814',
  inkSoft: '#5A5347',
  inkLight: '#8A8270',
  red: '#C8242C',
  redDark: '#8A1A21',
  redDeep: '#5A0F14',
  blue: '#1B3A5C',
  blueDark: '#0F2540',
  blueDeep: '#06182E',
  gold: '#C8941A',
  line: '#C9BCA0',
  lineSoft: '#D9CFB7',
  cream: '#FFF8E1',
};

const F_DISPLAY = "'Big Shoulders Display', Impact, system-ui, sans-serif";
const F_SERIF = "'Fraunces', Georgia, serif";
const F_MONO = "'JetBrains Mono', ui-monospace, monospace";

/* ============================================================ */

const emptyTeam = {
  points: 0,
  games: 0,
  pueblo: '',
  players: ['', '', '', '', ''],
  shownPlayers: 2, // cuántos inputs visibles inicialmente
};

const initialState = {
  teamA: { ...emptyTeam, players: ['', '', '', '', ''] },
  teamB: { ...emptyTeam, players: ['', '', '', '', ''] },
  server: null,
  fieldSwapped: false,
  rallas: { r1: false, r2: false },
  modo: 'SETUP',
  winner: null,
  flash: null,
  history: [],
};

const snap = (s) => ({
  teamA: { ...s.teamA, players: [...s.teamA.players] },
  teamB: { ...s.teamB, players: [...s.teamB.players] },
  server: s.server,
  fieldSwapped: s.fieldSwapped,
  rallas: { ...s.rallas },
  modo: s.modo,
  winner: s.winner,
});

function reducer(state, action) {
  const pushHistory = (next) => ({
    ...next,
    history: [...state.history, snap(state)].slice(-40),
  });

  switch (action.type) {
    case 'UPDATE_PUEBLO': {
      const key = action.team === 'A' ? 'teamA' : 'teamB';
      return {
        ...state,
        [key]: { ...state[key], pueblo: action.value },
      };
    }

    case 'UPDATE_PLAYER': {
      const key = action.team === 'A' ? 'teamA' : 'teamB';
      const players = [...state[key].players];
      players[action.index] = action.value;
      return {
        ...state,
        [key]: { ...state[key], players },
      };
    }

    case 'ADD_PLAYER_SLOT': {
      const key = action.team === 'A' ? 'teamA' : 'teamB';
      const current = state[key].shownPlayers;
      if (current >= MAX_PLAYERS) return state;
      return {
        ...state,
        [key]: { ...state[key], shownPlayers: current + 1 },
      };
    }

    case 'CHOOSE_SERVER': {
      if (state.modo !== 'SETUP') return state;
      // Preservamos pueblo y players, reseteamos puntuación
      return {
        ...state,
        teamA: { ...state.teamA, points: 0, games: 0 },
        teamB: { ...state.teamB, points: 0, games: 0 },
        server: action.team,
        fieldSwapped: false,
        rallas: { r1: false, r2: false },
        modo: 'IDLE',
        winner: null,
        flash: `Saca ${getTeamName(state, action.team)}`,
        history: [],
      };
    }

    case 'POINT': {
      if (state.modo === 'FINAL' || state.modo === 'SETUP') return state;
      if (!['IDLE', 'RALLA_1', 'RESOLVING_R1', 'RESOLVING_R2'].includes(state.modo)) {
        return state;
      }

      const team = action.team;
      const meKey = team === 'A' ? 'teamA' : 'teamB';
      const oppKey = team === 'A' ? 'teamB' : 'teamA';

      const me = { ...state[meKey] };
      const opp = { ...state[oppKey] };
      let next = { ...state };
      let gameWon = false;

      if (me.points >= 3) {
        me.games += 1;
        me.points = 0;
        opp.points = 0;
        gameWon = true;
        if (me.games >= TARGET_GAMES) {
          next.winner = team;
          next.modo = 'FINAL';
          next.flash = `¡${getTeamName(state, team)} guanya!`;
        } else {
          next.flash = `Joc per a ${getTeamName(state, team)}`;
        }
      } else {
        me.points += 1;
        if (me.points === 3) {
          next.flash = `VAL — ${getTeamName(state, team)}`;
        } else if (me.points === 2 && opp.points === 2) {
          next.flash = '30 iguales';
        }
      }

      next[meKey] = me;
      next[oppKey] = opp;

      if (gameWon) {
        next.rallas = { r1: false, r2: false };
        if (next.modo !== 'FINAL') next.modo = 'IDLE';
      } else {
        if (state.modo === 'RESOLVING_R1') {
          if (state.rallas.r2) {
            next.modo = 'RESOLVING_R2';
          } else {
            next.modo = 'IDLE';
            next.rallas = { r1: false, r2: false };
          }
        } else if (state.modo === 'RESOLVING_R2') {
          next.modo = 'IDLE';
          next.rallas = { r1: false, r2: false };
        }
      }

      return pushHistory(next);
    }

    case 'RALLA_1': {
      if (state.modo !== 'IDLE') return state;
      const hasVal = state.teamA.points === 3 || state.teamB.points === 3;
      let next = {
        ...state,
        rallas: { ...state.rallas, r1: true },
      };
      if (hasVal) {
        next.fieldSwapped = !state.fieldSwapped;
        next.server = state.server === 'A' ? 'B' : 'A';
        next.modo = 'RESOLVING_R1';
        next.flash = 'VAL + RALLA 1 · cambio campo i saque';
      } else {
        next.modo = 'RALLA_1';
        next.flash = 'Ralla 1 marcada';
      }
      return pushHistory(next);
    }

    case 'RALLA_2': {
      if (state.modo !== 'RALLA_1') return state;
      const next = {
        ...state,
        rallas: { ...state.rallas, r2: true },
        server: state.server === 'A' ? 'B' : 'A',
        modo: 'RESOLVING_R1',
        flash: 'Ralla 2 · cambio de saque',
      };
      return pushHistory(next);
    }

    case 'UNDO': {
      if (state.history.length === 0) return state;
      const prev = state.history[state.history.length - 1];
      return {
        ...prev,
        history: state.history.slice(0, -1),
        flash: '↶ deshecho',
      };
    }

    case 'RESET':
      // Vuelve a SETUP preservando pueblos y jugadores (rematch fácil)
      return {
        ...state,
        teamA: { ...state.teamA, points: 0, games: 0 },
        teamB: { ...state.teamB, points: 0, games: 0 },
        server: null,
        fieldSwapped: false,
        rallas: { r1: false, r2: false },
        modo: 'SETUP',
        winner: null,
        flash: null,
        history: [],
      };

    case 'CLEAR_FLASH':
      return { ...state, flash: null };

    default:
      return state;
  }
}

function pointLabel(team, opp) {
  if (team.points === 2 && opp.points === 2) return '30 IG';
  return POINT_LABELS[team.points] || '0';
}

function getTeamName(state, team) {
  const t = team === 'A' ? state.teamA : state.teamB;
  const fallback = team === 'A' ? 'ROIG' : 'BLAU';
  return (t.pueblo || '').trim() || fallback;
}

function getFilledPlayers(team) {
  return team.players.map((p) => p.trim()).filter(Boolean);
}

const Pelota = ({ size = 14, color = C.ink }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="none" style={{ display: 'inline-block' }}>
    <circle cx="12" cy="12" r="10" fill={color} />
    <path d="M3 9.5C7 11 17 11 21 9.5" stroke={C.cream} strokeWidth="0.8" opacity="0.85" />
    <path d="M3 14.5C7 13 17 13 21 14.5" stroke={C.cream} strokeWidth="0.8" opacity="0.85" />
  </svg>
);

/* ============================================================ */

export default function PelotaScoreboard() {
  const [state, dispatch] = useReducer(reducer, initialState);

  useEffect(() => {
    if (state.flash) {
      const t = setTimeout(() => dispatch({ type: 'CLEAR_FLASH' }), 2400);
      return () => clearTimeout(t);
    }
  }, [state.flash]);

  const aLabel = pointLabel(state.teamA, state.teamB);
  const bLabel = pointLabel(state.teamB, state.teamA);
  const aIsVal = state.teamA.points === 3;
  const bIsVal = state.teamB.points === 3;

  const canPoint = ['IDLE', 'RALLA_1', 'RESOLVING_R1', 'RESOLVING_R2'].includes(state.modo);
  const canRalla1 = state.modo === 'IDLE';
  const canRalla2 = state.modo === 'RALLA_1';
  const isResolving = state.modo === 'RESOLVING_R1' || state.modo === 'RESOLVING_R2';
  const canUndo = state.history.length > 0;
  const isFinal = state.modo === 'FINAL';
  const isSetup = state.modo === 'SETUP';

  const teamAName = getTeamName(state, 'A');
  const teamBName = getTeamName(state, 'B');

  const statusLabel = {
    IDLE: 'JOC',
    RALLA_1: 'RALLA 1 · joc continúa',
    RESOLVING_R1: 'RESOLENT RALLA 1',
    RESOLVING_R2: 'RESOLENT RALLA 2',
    FINAL: 'FINAL',
  }[state.modo];

  const scoreFontSize = (label) =>
    label.length > 2 ? 'clamp(2.6rem, 14vw, 4.6rem)' : 'clamp(4rem, 22vw, 7rem)';

  /* ---- TEAM PANEL ---- */
  const TeamPanel = ({ team, fallbackLabel, displayName, color, colorDark, colorDeep, score, isVal, games, isServer, position }) => {
    const active = canPoint && !isFinal;
    const showFallback = displayName === fallbackLabel;
    return (
      <div
        className="flex-1 flex flex-col grain"
        style={{
          background: C.paper,
          [position === 'top' ? 'borderBottom' : 'borderTop']: `1px solid ${C.line}`,
          minHeight: 0,
        }}
      >
        {/* Header bar */}
        <div className="px-4 pt-2.5 pb-1.5 flex items-center justify-between gap-2" style={{ flexShrink: 0 }}>
          <div className="flex items-center gap-2 min-w-0 flex-1">
            <div
              className="flex-shrink-0"
              style={{ width: 14, height: 14, background: color, borderRadius: 2 }}
            />
            <div className="min-w-0 flex-1">
              <div
                className="text-[11px] font-bold truncate"
                style={{
                  color: showFallback ? C.inkSoft : C.ink,
                  fontFamily: F_MONO,
                  letterSpacing: '0.22em',
                  textTransform: 'uppercase',
                }}
                title={displayName}
              >
                {displayName} <span style={{ color: C.inkLight, fontWeight: 600 }}>· {team}</span>
              </div>
            </div>
            {isServer && (
              <div
                className="flex items-center gap-1 px-1.5 py-[3px] rounded-sm flex-shrink-0"
                style={{ background: C.gold + '22' }}
              >
                <Pelota size={9} color={C.gold} />
                <span
                  className="text-[8.5px] font-bold"
                  style={{ color: C.gold, fontFamily: F_MONO, letterSpacing: '0.18em' }}
                >
                  SAQUE
                </span>
              </div>
            )}
          </div>
          <div className="text-right flex items-baseline gap-1 flex-shrink-0">
            <span
              className="text-[9px]"
              style={{ color: C.inkSoft, fontFamily: F_MONO, letterSpacing: '0.22em' }}
            >
              JOCS
            </span>
            <span
              style={{
                fontFamily: F_DISPLAY,
                fontWeight: 800,
                fontSize: '1.55rem',
                color: color,
                lineHeight: 1,
              }}
            >
              {games}
              <span style={{ color: C.inkLight, fontWeight: 600 }}>/{TARGET_GAMES}</span>
            </span>
          </div>
        </div>

        {/* Score */}
        <div className="flex-1 flex items-center justify-center min-h-0" style={{ padding: '4px 0' }}>
          <div
            className={isVal ? 'pulse-val' : ''}
            style={{
              fontFamily: F_DISPLAY,
              fontWeight: 900,
              fontSize: scoreFontSize(score),
              lineHeight: 0.85,
              color: isVal ? C.gold : color,
              letterSpacing: '-0.025em',
              textShadow: isVal ? `0 3px 0 ${color}33` : 'none',
            }}
          >
            {score}
          </div>
        </div>

        {/* +1 PUNT button */}
        <div className="px-3 pb-3" style={{ flexShrink: 0 }}>
          <button
            onClick={() => dispatch({ type: 'POINT', team })}
            disabled={!active}
            className="w-full punt-btn relative overflow-hidden"
            style={{
              background: active
                ? `linear-gradient(180deg, ${color} 0%, ${colorDark} 100%)`
                : C.paperDark,
              color: active ? C.cream : C.inkLight,
              opacity: active ? 1 : 0.5,
              border: `1px solid ${active ? colorDark : C.line}`,
              borderRadius: 8,
              padding: '12px 16px',
              boxShadow: active
                ? `0 4px 0 ${colorDeep}, inset 0 1px 0 ${C.cream}33`
                : 'none',
              fontFamily: F_DISPLAY,
              transition: 'all 0.08s ease',
              cursor: active ? 'pointer' : 'not-allowed',
            }}
          >
            <div className="flex items-center justify-center gap-3">
              <div
                style={{
                  fontWeight: 900,
                  fontSize: 'clamp(2rem, 9vw, 2.6rem)',
                  lineHeight: 0.9,
                  letterSpacing: '-0.02em',
                }}
              >
                +1
              </div>
              <div className="flex flex-col items-start" style={{ lineHeight: 1.05 }}>
                <span
                  style={{
                    fontWeight: 800,
                    fontSize: '1.3rem',
                    letterSpacing: '0.04em',
                    lineHeight: 1,
                  }}
                >
                  PUNT
                </span>
                <span
                  className="opacity-80 truncate"
                  style={{
                    fontFamily: F_MONO,
                    fontSize: '8.5px',
                    fontWeight: 700,
                    letterSpacing: '0.22em',
                    marginTop: 3,
                    maxWidth: 140,
                  }}
                >
                  {isResolving ? 'ASIGNAR' : displayName.slice(0, 14)}
                </span>
              </div>
            </div>
            {active && (
              <div
                className="absolute inset-0 pointer-events-none opacity-15"
                style={{
                  background: `repeating-linear-gradient(45deg, transparent 0 8px, ${C.cream}08 8px 9px)`,
                }}
              />
            )}
          </button>
        </div>
      </div>
    );
  };

  return (
    <div
      className="relative flex flex-col w-full select-none overflow-hidden"
      style={{
        background: C.bg,
        fontFamily: F_SERIF,
        color: C.ink,
        height: '100dvh',
        minHeight: '100vh',
      }}
    >
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Big+Shoulders+Display:wght@600;800;900&family=Fraunces:ital,wght@0,400;0,600;0,700;1,500&family=JetBrains+Mono:wght@400;600;700&display=swap');
        * { -webkit-tap-highlight-color: transparent; box-sizing: border-box; }
        button { font-family: inherit; }
        .grain {
          background-image:
            radial-gradient(rgba(26,24,20,0.025) 1px, transparent 1px),
            radial-gradient(rgba(26,24,20,0.018) 1px, transparent 1px);
          background-size: 5px 5px, 9px 9px;
          background-position: 0 0, 2px 2px;
        }
        .punt-btn:active:not(:disabled) {
          transform: translateY(2px);
          box-shadow: 0 1px 0 currentColor !important;
        }
        .flash-anim { animation: flashIn 0.25s ease-out; }
        @keyframes flashIn {
          from { opacity: 0; transform: translate(-50%, -8px); }
          to { opacity: 1; transform: translate(-50%, 0); }
        }
        .pulse-val { animation: pulseVal 1.5s ease-in-out infinite; transform-origin: center; }
        @keyframes pulseVal {
          0%, 100% { transform: scale(1); }
          50% { transform: scale(1.04); }
        }
        .ralla-pop { animation: rallaPop 0.3s cubic-bezier(.5,1.6,.5,1); }
        @keyframes rallaPop {
          from { transform: scale(0.6); }
          to { transform: scale(1); }
        }
        .final-in { animation: finalIn 0.5s ease-out; }
        @keyframes finalIn {
          from { opacity: 0; transform: scale(0.92); }
          to { opacity: 1; transform: scale(1); }
        }
        .setup-in { animation: setupIn 0.4s ease-out; }
        @keyframes setupIn {
          from { opacity: 0; transform: translateY(8px); }
          to { opacity: 1; transform: translateY(0); }
        }
        .status-anim { animation: statusIn 0.3s ease; }
        @keyframes statusIn {
          from { opacity: 0; letter-spacing: 0.5em; }
          to { opacity: 1; letter-spacing: 0.22em; }
        }
        .setup-btn:active { transform: translateY(3px); }
        .pelota-input {
          background: transparent;
          border: none;
          border-bottom: 1.5px solid ${C.line};
          padding: 7px 4px 6px;
          font-family: ${F_SERIF};
          font-size: 15px;
          color: ${C.ink};
          width: 100%;
          outline: none;
          transition: border-color 0.15s;
        }
        .pelota-input::placeholder {
          color: ${C.inkLight};
          font-style: italic;
          opacity: 0.7;
        }
        .pelota-input:focus { border-bottom-color: ${C.gold}; }
        .pelota-input.pueblo {
          font-family: ${F_DISPLAY};
          font-weight: 800;
          font-size: 22px;
          letter-spacing: -0.01em;
          text-transform: uppercase;
          padding: 4px 4px 4px;
        }
        .pelota-input.pueblo::placeholder {
          font-family: ${F_DISPLAY};
          font-weight: 600;
          letter-spacing: 0.02em;
          font-style: normal;
          text-transform: uppercase;
          opacity: 0.45;
        }
        /* Custom scrollbar for setup */
        .setup-scroll::-webkit-scrollbar { width: 4px; }
        .setup-scroll::-webkit-scrollbar-track { background: transparent; }
        .setup-scroll::-webkit-scrollbar-thumb { background: ${C.line}; border-radius: 2px; }
      `}</style>

      {/* HEADER */}
      <header
        className="px-4 py-2.5 flex items-center justify-between grain"
        style={{ borderBottom: `1px solid ${C.line}`, flexShrink: 0 }}
      >
        <div>
          <div
            className="text-[9px] font-bold"
            style={{ color: C.inkSoft, fontFamily: F_MONO, letterSpacing: '0.32em' }}
          >
            PILOTA VALENCIANA
          </div>
          <div
            className="text-xl leading-tight mt-0.5"
            style={{ fontFamily: F_SERIF, fontWeight: 700, fontStyle: 'italic' }}
          >
            Llargues{' '}
            <span style={{ color: C.gold, fontWeight: 400 }}>·</span>{' '}
            <span style={{ color: C.inkSoft, fontWeight: 400 }}>partida a {TARGET_GAMES}</span>
          </div>
        </div>
        <button
          onClick={() => {
            if (window.confirm('¿Iniciar nova partida?')) dispatch({ type: 'RESET' });
          }}
          className="px-3 py-2 text-[10px] font-bold transition-all"
          style={{
            background: C.paperDark,
            color: C.ink,
            fontFamily: F_MONO,
            border: `1px solid ${C.line}`,
            borderRadius: 3,
            letterSpacing: '0.2em',
          }}
        >
          NOVA
        </button>
      </header>

      {/* TEAM A */}
      <TeamPanel
        team="A"
        fallbackLabel="ROIG"
        displayName={teamAName}
        color={C.red}
        colorDark={C.redDark}
        colorDeep={C.redDeep}
        score={aLabel}
        isVal={aIsVal}
        games={state.teamA.games}
        isServer={state.server === 'A'}
        position="top"
      />

      {/* STATUS STRIP */}
      <div
        className="px-4 py-1.5 flex items-center justify-between grain"
        style={{
          background: isResolving
            ? `linear-gradient(90deg, ${C.gold}18, ${C.gold}08, ${C.gold}18)`
            : state.modo === 'RALLA_1'
            ? `linear-gradient(90deg, ${C.red}10, transparent, ${C.red}10)`
            : C.paperDark,
          borderTop: `1px solid ${C.line}`,
          borderBottom: `1px solid ${C.line}`,
          flexShrink: 0,
        }}
      >
        <div className="flex items-center gap-1.5">
          <div
            className={`flex items-center justify-center transition-all ${state.rallas.r1 ? 'ralla-pop' : ''}`}
            style={{
              width: 24,
              height: 24,
              borderRadius: '50%',
              background: state.rallas.r1 ? C.red : 'transparent',
              border: `1.5px solid ${state.rallas.r1 ? C.red : C.lineSoft}`,
              color: state.rallas.r1 ? C.cream : C.inkLight,
              fontFamily: F_MONO,
              fontSize: 9.5,
              fontWeight: 700,
            }}
          >
            R1
          </div>
          <div
            className={`flex items-center justify-center transition-all ${state.rallas.r2 ? 'ralla-pop' : ''}`}
            style={{
              width: 24,
              height: 24,
              borderRadius: '50%',
              background: state.rallas.r2 ? C.blue : 'transparent',
              border: `1.5px solid ${state.rallas.r2 ? C.blue : C.lineSoft}`,
              color: state.rallas.r2 ? C.cream : C.inkLight,
              fontFamily: F_MONO,
              fontSize: 9.5,
              fontWeight: 700,
            }}
          >
            R2
          </div>
        </div>

        <div className="flex-1 text-center px-2">
          <div
            key={state.modo}
            className="status-anim text-[11px] font-bold"
            style={{
              fontFamily: F_MONO,
              color: isResolving ? C.gold : state.modo === 'RALLA_1' ? C.red : C.ink,
              letterSpacing: '0.22em',
            }}
          >
            {statusLabel}
          </div>
          {state.teamA.points === 2 &&
            state.teamB.points === 2 &&
            (state.modo === 'IDLE' || state.modo === 'RALLA_1') && (
              <div
                className="text-[10px] italic"
                style={{ color: C.inkSoft, fontFamily: F_SERIF, marginTop: 1 }}
              >
                30 iguales
              </div>
            )}
        </div>

        <div
          className="text-[9px] font-bold flex items-center gap-1"
          style={{
            fontFamily: F_MONO,
            color: state.fieldSwapped ? C.gold : C.inkLight,
            letterSpacing: '0.2em',
          }}
        >
          {state.fieldSwapped ? '⇅ CAMP' : '— —'}
        </div>
      </div>

      {/* TEAM B */}
      <TeamPanel
        team="B"
        fallbackLabel="BLAU"
        displayName={teamBName}
        color={C.blue}
        colorDark={C.blueDark}
        colorDeep={C.blueDeep}
        score={bLabel}
        isVal={bIsVal}
        games={state.teamB.games}
        isServer={state.server === 'B'}
        position="bottom"
      />

      {/* ACTIONS */}
      <div
        className="px-3 pt-2.5 pb-3 grid grid-cols-3 gap-2"
        style={{
          background: C.paperDark,
          borderTop: `1px solid ${C.line}`,
          flexShrink: 0,
        }}
      >
        <button
          onClick={() => dispatch({ type: 'RALLA_1' })}
          disabled={!canRalla1}
          className="flex flex-col items-center justify-center punt-btn"
          style={{
            background: canRalla1
              ? `linear-gradient(180deg, ${C.red} 0%, ${C.redDark} 100%)`
              : C.paper,
            color: canRalla1 ? C.cream : C.inkLight,
            opacity: canRalla1 ? 1 : 0.45,
            border: `1px solid ${canRalla1 ? C.redDark : C.line}`,
            borderRadius: 6,
            paddingTop: 9,
            paddingBottom: 8,
            boxShadow: canRalla1 ? `0 3px 0 ${C.redDeep}` : 'none',
          }}
        >
          <div
            style={{
              fontFamily: F_DISPLAY,
              fontWeight: 900,
              fontSize: '1.3rem',
              lineHeight: 1,
              letterSpacing: '-0.01em',
            }}
          >
            RALLA 1
          </div>
          <div
            className="mt-0.5 text-[8px] font-bold opacity-80"
            style={{ fontFamily: F_MONO, letterSpacing: '0.22em' }}
          >
            {state.rallas.r1 ? 'MARCADA' : 'MARCAR'}
          </div>
        </button>

        <button
          onClick={() => dispatch({ type: 'RALLA_2' })}
          disabled={!canRalla2}
          className="flex flex-col items-center justify-center punt-btn"
          style={{
            background: canRalla2
              ? `linear-gradient(180deg, ${C.blue} 0%, ${C.blueDark} 100%)`
              : C.paper,
            color: canRalla2 ? C.cream : C.inkLight,
            opacity: canRalla2 ? 1 : 0.45,
            border: `1px solid ${canRalla2 ? C.blueDark : C.line}`,
            borderRadius: 6,
            paddingTop: 9,
            paddingBottom: 8,
            boxShadow: canRalla2 ? `0 3px 0 ${C.blueDeep}` : 'none',
          }}
        >
          <div
            style={{
              fontFamily: F_DISPLAY,
              fontWeight: 900,
              fontSize: '1.3rem',
              lineHeight: 1,
              letterSpacing: '-0.01em',
            }}
          >
            RALLA 2
          </div>
          <div
            className="mt-0.5 text-[8px] font-bold opacity-80"
            style={{ fontFamily: F_MONO, letterSpacing: '0.22em' }}
          >
            {state.rallas.r2 ? 'MARCADA' : 'MARCAR'}
          </div>
        </button>

        <button
          onClick={() => dispatch({ type: 'UNDO' })}
          disabled={!canUndo}
          className="flex flex-col items-center justify-center punt-btn"
          style={{
            background: canUndo ? `linear-gradient(180deg, ${C.ink} 0%, #000 100%)` : C.paper,
            color: canUndo ? C.bg : C.inkLight,
            opacity: canUndo ? 1 : 0.4,
            border: `1px solid ${canUndo ? C.ink : C.line}`,
            borderRadius: 6,
            paddingTop: 9,
            paddingBottom: 8,
            boxShadow: canUndo ? '0 3px 0 #000' : 'none',
          }}
        >
          <div
            style={{
              fontFamily: F_DISPLAY,
              fontWeight: 900,
              fontSize: '1.3rem',
              lineHeight: 1,
              letterSpacing: '-0.01em',
            }}
          >
            ↶ DESF.
          </div>
          <div
            className="mt-0.5 text-[8px] font-bold opacity-80"
            style={{ fontFamily: F_MONO, letterSpacing: '0.22em' }}
          >
            DESHACER
          </div>
        </button>
      </div>

      {/* FLASH */}
      {state.flash && !isFinal && !isSetup && (
        <div
          className="absolute z-20 pointer-events-none flash-anim"
          style={{
            left: '50%',
            top: 78,
            background: C.ink,
            color: C.bg,
            fontFamily: F_MONO,
            fontSize: 11,
            letterSpacing: '0.18em',
            fontWeight: 700,
            padding: '8px 14px',
            borderRadius: 999,
            boxShadow: `0 6px 16px ${C.ink}44, 0 1px 0 ${C.cream}33 inset`,
            maxWidth: '90%',
            textAlign: 'center',
            textTransform: 'uppercase',
          }}
        >
          {state.flash}
        </div>
      )}

      {/* ============ SETUP OVERLAY ============ */}
      {isSetup && (
        <SetupScreen state={state} dispatch={dispatch} />
      )}

      {/* FINAL OVERLAY */}
      {isFinal && state.winner && (
        <FinalScreen state={state} dispatch={dispatch} />
      )}
    </div>
  );
}

/* ============================================================
   SETUP SCREEN
   ============================================================ */

function SetupScreen({ state, dispatch }) {
  const teamA = state.teamA;
  const teamB = state.teamB;

  const canStart = true; // Siempre se puede empezar (campos opcionales)

  return (
    <div
      className="absolute inset-0 z-40 flex flex-col grain setup-in"
      style={{ background: C.bg }}
    >
      {/* Mini header */}
      <div
        className="px-5 pt-5 pb-3 text-center"
        style={{ flexShrink: 0, borderBottom: `1px solid ${C.line}` }}
      >
        <div
          style={{
            fontFamily: F_MONO,
            fontSize: 9.5,
            fontWeight: 700,
            letterSpacing: '0.42em',
            color: C.inkSoft,
          }}
        >
          PILOTA VALENCIANA
        </div>
        <div
          className="mt-1"
          style={{
            fontFamily: F_SERIF,
            fontWeight: 700,
            fontStyle: 'italic',
            fontSize: 'clamp(1.6rem, 8vw, 2.2rem)',
            lineHeight: 1,
          }}
        >
          Llargues
        </div>
        <div
          className="mt-1 italic"
          style={{ fontFamily: F_SERIF, color: C.inkSoft, fontSize: '0.85rem' }}
        >
          <span style={{ color: C.gold }}>·</span> partida a {TARGET_GAMES}{' '}
          <span style={{ color: C.gold }}>·</span>
        </div>
      </div>

      {/* Scrollable content */}
      <div
        className="flex-1 overflow-y-auto setup-scroll px-5 py-5"
        style={{ minHeight: 0 }}
      >
        <TeamSetupCard
          team="A"
          fallbackLabel="ROIG"
          color={C.red}
          colorDark={C.redDark}
          colorDeep={C.redDeep}
          data={teamA}
          dispatch={dispatch}
        />

        <div className="my-5 flex items-center gap-3">
          <div style={{ flex: 1, height: 1, background: C.line }} />
          <span
            style={{
              fontFamily: F_SERIF,
              fontStyle: 'italic',
              color: C.inkLight,
              fontSize: '0.9rem',
            }}
          >
            contra
          </span>
          <div style={{ flex: 1, height: 1, background: C.line }} />
        </div>

        <TeamSetupCard
          team="B"
          fallbackLabel="BLAU"
          color={C.blue}
          colorDark={C.blueDark}
          colorDeep={C.blueDeep}
          data={teamB}
          dispatch={dispatch}
        />

        {/* QUI SACA */}
        <div className="mt-7 mb-3 flex items-center gap-3">
          <div style={{ flex: 1, height: 1, background: C.line }} />
          <span
            style={{
              fontFamily: F_MONO,
              fontSize: 10,
              fontWeight: 700,
              letterSpacing: '0.32em',
              color: C.ink,
              whiteSpace: 'nowrap',
            }}
          >
            QUI SACA?
          </span>
          <div style={{ flex: 1, height: 1, background: C.line }} />
        </div>

        <div className="flex flex-col gap-3 pb-2">
          <SaqueButton
            team="A"
            displayName={(teamA.pueblo || '').trim() || 'ROIG'}
            color={C.red}
            colorDark={C.redDark}
            colorDeep={C.redDeep}
            onClick={() => dispatch({ type: 'CHOOSE_SERVER', team: 'A' })}
            disabled={!canStart}
          />
          <SaqueButton
            team="B"
            displayName={(teamB.pueblo || '').trim() || 'BLAU'}
            color={C.blue}
            colorDark={C.blueDark}
            colorDeep={C.blueDeep}
            onClick={() => dispatch({ type: 'CHOOSE_SERVER', team: 'B' })}
            disabled={!canStart}
          />
        </div>

        <div
          className="mt-5 italic text-center pb-4"
          style={{ fontFamily: F_SERIF, color: C.inkLight, fontSize: '0.78rem' }}
        >
          Toca l'equip que comença al saque
        </div>
      </div>
    </div>
  );
}

function TeamSetupCard({ team, fallbackLabel, color, colorDark, colorDeep, data, dispatch }) {
  const filledCount = data.players.filter((p) => p.trim()).length;
  const canAddMore = data.shownPlayers < MAX_PLAYERS;

  return (
    <div
      className="relative"
      style={{
        background: C.paper,
        border: `1px solid ${C.line}`,
        borderRadius: 12,
        padding: '14px 16px 16px',
        boxShadow: `0 2px 0 ${C.lineSoft}`,
      }}
    >
      {/* Color tag */}
      <div
        className="absolute"
        style={{
          top: -1,
          left: 16,
          right: 16,
          height: 3,
          background: `linear-gradient(90deg, ${color}, ${colorDark})`,
          borderRadius: '0 0 2px 2px',
        }}
      />

      {/* Header */}
      <div className="flex items-center justify-between mb-2">
        <div className="flex items-center gap-2">
          <div style={{ width: 12, height: 12, background: color, borderRadius: 2 }} />
          <span
            style={{
              fontFamily: F_MONO,
              fontSize: 10,
              fontWeight: 700,
              color: C.inkSoft,
              letterSpacing: '0.28em',
            }}
          >
            EQUIP · {team} · {fallbackLabel}
          </span>
        </div>
        {filledCount > 0 && (
          <span
            style={{
              fontFamily: F_MONO,
              fontSize: 9,
              fontWeight: 700,
              color: color,
              letterSpacing: '0.18em',
            }}
          >
            {filledCount} JUG.
          </span>
        )}
      </div>

      {/* Pueblo input */}
      <div className="mb-3">
        <label
          style={{
            display: 'block',
            fontFamily: F_MONO,
            fontSize: 9,
            fontWeight: 700,
            color: C.inkLight,
            letterSpacing: '0.25em',
            marginBottom: 2,
          }}
        >
          PUEBLO
        </label>
        <input
          type="text"
          className="pelota-input pueblo"
          placeholder={`Poble de l'equip ${team}`}
          value={data.pueblo}
          maxLength={24}
          autoComplete="off"
          autoCapitalize="characters"
          onChange={(e) => dispatch({ type: 'UPDATE_PUEBLO', team, value: e.target.value })}
          style={{ borderBottomColor: data.pueblo ? color : C.line }}
        />
      </div>

      {/* Players */}
      <div>
        <label
          style={{
            display: 'block',
            fontFamily: F_MONO,
            fontSize: 9,
            fontWeight: 700,
            color: C.inkLight,
            letterSpacing: '0.25em',
            marginBottom: 4,
          }}
        >
          JUGADORS · MÀX. {MAX_PLAYERS}
        </label>

        <div className="flex flex-col gap-1.5">
          {data.players.slice(0, data.shownPlayers).map((player, i) => (
            <div key={i} className="flex items-center gap-2">
              <span
                style={{
                  fontFamily: F_DISPLAY,
                  fontWeight: 800,
                  fontSize: 18,
                  color: player.trim() ? color : C.inkLight,
                  width: 22,
                  textAlign: 'center',
                  lineHeight: 1,
                  flexShrink: 0,
                }}
              >
                {i + 1}
              </span>
              <input
                type="text"
                className="pelota-input"
                placeholder={`Jugador ${i + 1}${i >= 1 ? ' · opcional' : ''}`}
                value={player}
                maxLength={24}
                autoComplete="off"
                autoCapitalize="words"
                onChange={(e) =>
                  dispatch({ type: 'UPDATE_PLAYER', team, index: i, value: e.target.value })
                }
              />
            </div>
          ))}
        </div>

        {canAddMore && (
          <button
            onClick={() => dispatch({ type: 'ADD_PLAYER_SLOT', team })}
            className="mt-2"
            style={{
              background: 'transparent',
              border: `1px dashed ${C.line}`,
              borderRadius: 6,
              padding: '6px 10px',
              color: C.inkSoft,
              fontFamily: F_MONO,
              fontSize: 10,
              fontWeight: 700,
              letterSpacing: '0.2em',
              cursor: 'pointer',
              width: '100%',
            }}
          >
            + AFEGIR JUGADOR ({data.shownPlayers}/{MAX_PLAYERS})
          </button>
        )}
      </div>
    </div>
  );
}

function SaqueButton({ team, displayName, color, colorDark, colorDeep, onClick, disabled }) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className="setup-btn relative overflow-hidden"
      style={{
        background: `linear-gradient(180deg, ${color} 0%, ${colorDark} 100%)`,
        color: C.cream,
        border: `1px solid ${colorDark}`,
        borderRadius: 10,
        padding: '16px 20px',
        boxShadow: `0 5px 0 ${colorDeep}, inset 0 1px 0 ${C.cream}30`,
        fontFamily: F_DISPLAY,
        transition: 'all 0.1s ease',
        width: '100%',
        textAlign: 'left',
        opacity: disabled ? 0.5 : 1,
        cursor: disabled ? 'not-allowed' : 'pointer',
      }}
    >
      <div className="flex items-center justify-between gap-3">
        <div className="min-w-0 flex-1">
          <div
            style={{
              fontFamily: F_MONO,
              fontSize: 9.5,
              fontWeight: 700,
              letterSpacing: '0.32em',
              opacity: 0.85,
            }}
          >
            EQUIP · {team}
          </div>
          <div
            className="truncate"
            style={{
              fontWeight: 900,
              fontSize: 'clamp(1.6rem, 7vw, 2.1rem)',
              lineHeight: 1,
              letterSpacing: '-0.02em',
              marginTop: 2,
              textTransform: 'uppercase',
            }}
            title={displayName}
          >
            {displayName}
          </div>
        </div>
        <div className="flex items-center gap-2 flex-shrink-0">
          <Pelota size={18} color={C.cream} />
          <span
            style={{
              fontFamily: F_MONO,
              fontSize: 10,
              fontWeight: 700,
              letterSpacing: '0.22em',
              opacity: 0.9,
            }}
          >
            SACA
          </span>
        </div>
      </div>
      <div
        className="absolute inset-0 pointer-events-none opacity-15"
        style={{
          background: `repeating-linear-gradient(45deg, transparent 0 8px, ${C.cream}10 8px 9px)`,
        }}
      />
    </button>
  );
}

/* ============================================================
   FINAL SCREEN
   ============================================================ */

function FinalScreen({ state, dispatch }) {
  const winnerKey = state.winner === 'A' ? 'teamA' : 'teamB';
  const loserKey = state.winner === 'A' ? 'teamB' : 'teamA';
  const winnerName = getTeamName(state, state.winner);
  const winnerPlayers = getFilledPlayers(state[winnerKey]);
  const loserPlayers = getFilledPlayers(state[loserKey]);
  const loserName = getTeamName(state, state.winner === 'A' ? 'B' : 'A');

  const bgColor = state.winner === 'A' ? C.red : C.blue;
  const bgColorDark = state.winner === 'A' ? C.redDark : C.blueDark;

  return (
    <div
      className="absolute inset-0 z-30 flex flex-col items-center justify-between final-in"
      style={{
        background: `radial-gradient(circle at 50% 30%, ${bgColor}, ${bgColorDark})`,
        padding: '32px 24px 24px',
        overflow: 'auto',
      }}
    >
      <div className="text-center w-full">
        <div
          style={{
            color: C.cream + 'AA',
            fontFamily: F_MONO,
            fontSize: 10,
            fontWeight: 700,
            letterSpacing: '0.45em',
          }}
        >
          FINAL DE PARTIDA
        </div>
        <div
          style={{
            fontFamily: F_DISPLAY,
            fontWeight: 900,
            fontSize: 'clamp(2rem, 10vw, 3.4rem)',
            color: C.cream,
            lineHeight: 0.95,
            letterSpacing: '-0.02em',
            marginTop: 14,
          }}
        >
          GUANYA
        </div>
        <div
          className="truncate"
          style={{
            fontFamily: F_DISPLAY,
            fontWeight: 900,
            fontSize: 'clamp(2.8rem, 13vw, 5rem)',
            color: C.gold,
            lineHeight: 0.85,
            letterSpacing: '-0.02em',
            marginTop: 4,
            textShadow: '0 4px 0 #00000033',
            textTransform: 'uppercase',
            maxWidth: '100%',
          }}
        >
          {winnerName}
        </div>
        <div
          className="italic mt-3"
          style={{
            color: C.cream,
            fontFamily: F_SERIF,
            fontSize: '1.3rem',
            fontWeight: 600,
          }}
        >
          {state.teamA.games}{' '}
          <span style={{ opacity: 0.5 }}>·</span> {state.teamB.games}
        </div>
        <div
          className="italic mt-1"
          style={{
            color: C.cream + 'AA',
            fontFamily: F_SERIF,
            fontSize: '0.85rem',
          }}
        >
          contra {loserName}
        </div>

        {/* Players list */}
        {(winnerPlayers.length > 0 || loserPlayers.length > 0) && (
          <div className="mt-6 w-full max-w-xs mx-auto">
            {winnerPlayers.length > 0 && (
              <div
                style={{
                  background: C.cream + '14',
                  border: `1px solid ${C.cream}26`,
                  borderRadius: 8,
                  padding: '10px 14px',
                  textAlign: 'left',
                  backdropFilter: 'blur(2px)',
                }}
              >
                <div
                  style={{
                    fontFamily: F_MONO,
                    fontSize: 9,
                    fontWeight: 700,
                    letterSpacing: '0.28em',
                    color: C.gold,
                    marginBottom: 4,
                  }}
                >
                  GUANYADORS
                </div>
                <ul
                  style={{
                    fontFamily: F_SERIF,
                    color: C.cream,
                    fontSize: 14,
                    fontWeight: 500,
                    listStyle: 'none',
                    padding: 0,
                    margin: 0,
                    lineHeight: 1.5,
                  }}
                >
                  {winnerPlayers.map((p, i) => (
                    <li key={i} className="flex items-baseline gap-2">
                      <span style={{ color: C.gold, fontFamily: F_MONO, fontSize: 10, fontWeight: 700 }}>
                        {i + 1}.
                      </span>
                      <span>{p}</span>
                    </li>
                  ))}
                </ul>
              </div>
            )}
          </div>
        )}
      </div>

      <button
        onClick={() => dispatch({ type: 'RESET' })}
        className="mt-6 transition-all flex-shrink-0"
        style={{
          background: C.cream,
          color: bgColor,
          fontFamily: F_DISPLAY,
          fontWeight: 800,
          fontSize: '1.1rem',
          letterSpacing: '0.15em',
          padding: '12px 26px',
          borderRadius: 999,
          boxShadow: '0 6px 18px #00000033',
        }}
      >
        NOVA PARTIDA
      </button>
    </div>
  );
}
