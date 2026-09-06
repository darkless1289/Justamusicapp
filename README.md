import React, { useState, useRef, useEffect, useCallback, useMemo } from "react";
import {
  Play, Pause, SkipBack, SkipForward, Upload, Volume2, VolumeX,
  Shuffle, Repeat, Disc3, Plus, Pencil, Trash2, Scissors,
  Moon, Image as ImageIcon, X, Check, RotateCcw, RotateCw,
} from "lucide-react";

const fmt = (s) => {
  if (!s || Number.isNaN(s)) return "0:00";
  const m = Math.floor(s / 60);
  const sec = Math.floor(s % 60).toString().padStart(2, "0");
  return `${m}:${sec}`;
};
const uid = () => Math.random().toString(36).slice(2, 10);

const ACCENTS = [
  { name: "Amber", val: "#E8A33D", val2: "#C9536B" },
  { name: "Teal", val: "#3FB6A8", val2: "#2E7D9A" },
  { name: "Violet", val: "#9C7BD9", val2: "#5B4B9E" },
  { name: "Rose", val: "#E8607A", val2: "#B84A6B" },
];

function bufferToWav(buffer) {
  const numCh = buffer.numberOfChannels;
  const len = buffer.length * numCh * 2 + 44;
  const ab = new ArrayBuffer(len);
  const view = new DataView(ab);
  const writeStr = (o, s) => { for (let i = 0; i < s.length; i++) view.setUint8(o + i, s.charCodeAt(i)); };
  writeStr(0, "RIFF"); view.setUint32(4, len - 8, true); writeStr(8, "WAVE");
  writeStr(12, "fmt "); view.setUint32(16, 16, true); view.setUint16(20, 1, true);
  view.setUint16(22, numCh, true); view.setUint32(24, buffer.sampleRate, true);
  view.setUint32(28, buffer.sampleRate * numCh * 2, true); view.setUint16(32, numCh * 2, true);
  view.setUint16(34, 16, true); writeStr(36, "data"); view.setUint32(40, len - 44, true);
  let offset = 44;
  const chans = Array.from({ length: numCh }, (_, i) => buffer.getChannelData(i));
  for (let i = 0; i < buffer.length; i++) {
    for (let c = 0; c < numCh; c++) {
      const s = Math.max(-1, Math.min(1, chans[c][i]));
      view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
      offset += 2;
    }
  }
  return new Blob([ab], { type: "audio/wav" });
}

export default function MusicApp() {
  const [library, setLibrary] = useState([]);
  const [playlists, setPlaylists] = useState([{ id: "all", name: "All tracks", trackIds: [], system: true }]);
  const [activePlaylist, setActivePlaylist] = useState("all");
  const [currentId, setCurrentId] = useState(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [progress, setProgress] = useState(0);
  const [duration, setDuration] = useState(0);
  const [volume, setVolume] = useState(0.8);
  const [muted, setMuted] = useState(false);
  const [shuffle, setShuffle] = useState(false);
  const [repeat, setRepeat] = useState(false);
  const [skipSeconds, setSkipSeconds] = useState(10);
  const [editingSkip, setEditingSkip] = useState(false);
  const [accent, setAccent] = useState(ACCENTS[0]);
  const [sleepAt, setSleepAt] = useState(null);
  const [sleepRemaining, setSleepRemaining] = useState(null);
  const [renamingPlaylist, setRenamingPlaylist] = useState(null);
  const [renameValue, setRenameValue] = useState("");
  const [trimTrackId, setTrimTrackId] = useState(null);
  const [trimRange, setTrimRange] = useState([0, 0]);
  const [trimBuffer, setTrimBuffer] = useState(null);
  const [addMenuFor, setAddMenuFor] = useState(null);

  const audioRef = useRef(null);
  const canvasRef = useRef(null);
  const fileInputRef = useRef(null);
  const artInputRef = useRef(null);
  const artForTrack = useRef(null);
  const audioCtxRef = useRef(null);
  const analyserRef = useRef(null);
  const sourceRef = useRef(null);
  const rafRef = useRef(null);
  const sleepIntervalRef = useRef(null);

  const currentPlaylistTracks = useMemo(() => {
    const pl = playlists.find((p) => p.id === activePlaylist);
    if (!pl) return [];
    if (pl.system) return library;
    return pl.trackIds.map((id) => library.find((t) => t.id === id)).filter(Boolean);
  }, [playlists, activePlaylist, library]);

  const current = library.find((t) => t.id === currentId) || null;
  const currentIndexInView = currentPlaylistTracks.findIndex((t) => t.id === currentId);

  const ensureAudioGraph = useCallback(() => {
    if (!audioCtxRef.current) {
      const Ctx = window.AudioContext || window.webkitAudioContext;
      audioCtxRef.current = new Ctx();
      analyserRef.current = audioCtxRef.current.createAnalyser();
      analyserRef.current.fftSize = 128;
      sourceRef.current = audioCtxRef.current.createMediaElementSource(audioRef.current);
      sourceRef.current.connect(analyserRef.current);
      analyserRef.current.connect(audioCtxRef.current.destination);
    }
    if (audioCtxRef.current.state === "suspended") audioCtxRef.current.resume();
  }, []);

  const handleFiles = (e) => {
    const files = Array.from(e.target.files || []);
    if (!files.length) return;
    const newTracks = files.map((f) => ({
      id: uid(), name: f.name.replace(/\.[^/.]+$/, ""), url: URL.createObjectURL(f), artUrl: null, duration: 0,
    }));
    setLibrary((prev) => {
      const combined = [...prev, ...newTracks];
      if (!currentId) setCurrentId(combined[0].id);
      return combined;
    });
    e.target.value = "";
  };

  const playId = (id) => { setCurrentId(id); setIsPlaying(true); };
  const togglePlay = () => { if (!current) return; ensureAudioGraph(); setIsPlaying((p) => !p); };

  const goRelative = useCallback((dir) => {
    if (!currentPlaylistTracks.length) return;
    let idx = currentIndexInView;
    if (idx === -1) idx = 0;
    if (shuffle) idx = Math.floor(Math.random() * currentPlaylistTracks.length);
    else idx = (idx + dir + currentPlaylistTracks.length) % currentPlaylistTracks.length;
    playId(currentPlaylistTracks[idx].id);
  }, [currentPlaylistTracks, currentIndexInView, shuffle]);

  const jump = useCallback((secs) => {
    if (!audioRef.current) return;
    audioRef.current.currentTime = Math.max(0, Math.min(duration, audioRef.current.currentTime + secs));
  }, [duration]);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;
    if (isPlaying) audio.play().catch(() => setIsPlaying(false));
    else audio.pause();
  }, [isPlaying, currentId]);

  useEffect(() => { if (audioRef.current) audioRef.current.volume = muted ? 0 : volume; }, [volume, muted]);

  useEffect(() => {
    if (!("mediaSession" in navigator) || !current) return;
    navigator.mediaSession.metadata = new window.MediaMetadata({
      title: current.name,
      artist: "My Library",
      artwork: current.artUrl ? [{ src: current.artUrl, sizes: "512x512", type: "image/png" }] : [],
    });
    navigator.mediaSession.setActionHandler("play", () => setIsPlaying(true));
    navigator.mediaSession.setActionHandler("pause", () => setIsPlaying(false));
    navigator.mediaSession.setActionHandler("previoustrack", () => goRelative(-1));
    navigator.mediaSession.setActionHandler("nexttrack", () => goRelative(1));
    navigator.mediaSession.setActionHandler("seekbackward", () => jump(-skipSeconds));
    navigator.mediaSession.setActionHandler("seekforward", () => jump(skipSeconds));
  }, [current, skipSeconds, goRelative, jump]);

  useEffect(() => {
    if (!sleepAt) { setSleepRemaining(null); return; }
    sleepIntervalRef.current = setInterval(() => {
      const rem = sleepAt - Date.now();
      if (rem <= 0) {
        setIsPlaying(false);
        setSleepAt(null);
        setSleepRemaining(null);
        clearInterval(sleepIntervalRef.current);
      } else {
        setSleepRemaining(rem);
      }
    }, 1000);
    return () => clearInterval(sleepIntervalRef.current);
  }, [sleepAt]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    const draw = () => {
      rafRef.current = requestAnimationFrame(draw);
      const { width, height } = canvas;
      ctx.clearRect(0, 0, width, height);
      const analyser = analyserRef.current;
      if (analyser && isPlaying) {
        const data = new Uint8Array(analyser.frequencyBinCount);
        analyser.getByteFrequencyData(data);
        const gap = 3;
        const barWidth = width / data.length - gap;
        for (let i = 0; i < data.length; i++) {
          const v = data[i] / 255;
          const barHeight = Math.max(3, v * height);
          const x = i * (barWidth + gap);
          const grad = ctx.createLinearGradient(0, height - barHeight, 0, height);
          grad.addColorStop(0, accent.val);
          grad.addColorStop(1, accent.val2);
          ctx.fillStyle = grad;
          ctx.fillRect(x, height - barHeight, barWidth, barHeight);
        }
      } else {
        ctx.fillStyle = "rgba(245,237,228,0.12)";
        const barCount = 48, gap = 3;
        const barWidth = canvas.width / barCount - gap;
        for (let i = 0; i < barCount; i++) ctx.fillRect(i * (barWidth + gap), canvas.height - 4, barWidth, 4);
      }
    };
    draw();
    return () => cancelAnimationFrame(rafRef.current);
  }, [isPlaying, accent]);

  const newPlaylist = () => {
    const id = uid();
    setPlaylists((p) => [...p, { id, name: `Playlist ${p.length}`, trackIds: [] }]);
    setActivePlaylist(id);
  };
  const deletePlaylist = (id) => {
    setPlaylists((p) => p.filter((pl) => pl.id !== id));
    if (activePlaylist === id) setActivePlaylist("all");
  };
  const renamePlaylist = (id, name) => setPlaylists((p) => p.map((pl) => (pl.id === id ? { ...pl, name } : pl)));
  const addToPlaylist = (plId, trackId) =>
    setPlaylists((p) => p.map((pl) => (pl.id === plId && !pl.trackIds.includes(trackId) ? { ...pl, trackIds: [...pl.trackIds, trackId] } : pl)));
  const removeFromPlaylist = (plId, trackId) =>
    setPlaylists((p) => p.map((pl) => (pl.id === plId ? { ...pl, trackIds: pl.trackIds.filter((t) => t !== trackId) } : pl)));
  const moveInPlaylist = (plId, index, dir) =>
    setPlaylists((p) => p.map((pl) => {
      if (pl.id !== plId) return pl;
      const arr = [...pl.trackIds];
      const j = index + dir;
      if (j < 0 || j >= arr.length) return pl;
      [arr[index], arr[j]] = [arr[j], arr[index]];
      return { ...pl, trackIds: arr };
    }));

  const handleArtFile = (e) => {
    const file = e.target.files?.[0];
    const trackId = artForTrack.current;
    if (file && trackId) {
      const url = URL.createObjectURL(file);
      setLibrary((lib) => lib.map((t) => (t.id === trackId ? { ...t, artUrl: url } : t)));
    }
    e.target.value = "";
  };

  const openTrim = async (track) => {
    setTrimTrackId(track.id);
    setTrimBuffer(null);
    const buf = await fetch(track.url).then((r) => r.arrayBuffer());
    const Ctx = window.AudioContext || window.webkitAudioContext;
    const tmpCtx = new Ctx();
    const decoded = await tmpCtx.decodeAudioData(buf);
    setTrimBuffer(decoded);
    setTrimRange([0, decoded.duration]);
    tmpCtx.close();
  };
  const saveTrim = async () => {
    if (!trimBuffer) return;
    const [start, end] = trimRange;
    const sampleRate = trimBuffer.sampleRate;
    const length = Math.floor((end - start) * sampleRate);
    const OfflineCtx = window.OfflineAudioContext || window.webkitOfflineAudioContext;
    const offline = new OfflineCtx(trimBuffer.numberOfChannels, length, sampleRate);
    const src = offline.createBufferSource();
    src.buffer = trimBuffer;
    src.connect(offline.destination);
    src.start(0, start, end - start);
    const rendered = await offline.startRendering();
    const wav = bufferToWav(rendered);
    const url = URL.createObjectURL(wav);
    const original = library.find((t) => t.id === trimTrackId);
    const newTrack = { id: uid(), name: `${original.name} (trimmed)`, url, artUrl: original.artUrl, duration: end - start };
    setLibrary((lib) => [...lib, newTrack]);
    setTrimTrackId(null);
    setTrimBuffer(null);
  };

  return (
    <div
      style={{
        "--bg": "#1C1220", "--surface": "#241726", "--surface2": "#31202f",
        "--accent": accent.val, "--accent2": accent.val2, "--text": "#F5EDE4", "--muted": "#9C8AA5",
        background: "var(--bg)", color: "var(--text)",
        fontFamily: "Inter, ui-sans-serif, system-ui, sans-serif", minHeight: "640px",
      }}
      className="w-full flex flex-col md:flex-row rounded-xl overflow-hidden relative"
    >
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,600&family=Inter:wght@400;500;600&display=swap');
        .row:hover { background: var(--surface2); }
        .row.active { background: var(--surface2); }
        input[type="range"] { -webkit-appearance:none; height:4px; border-radius:999px; background:rgba(245,237,228,0.15); }
        input[type="range"]::-webkit-slider-thumb { -webkit-appearance:none; width:12px; height:12px; border-radius:50%; background:var(--accent); cursor:pointer; }
        input[type="range"]::-moz-range-thumb { width:12px; height:12px; border:none; border-radius:50%; background:var(--accent); cursor:pointer; }
        input[type="text"], input[type="number"] { background:var(--surface2); color:var(--text); border:1px solid rgba(245,237,228,0.15); border-radius:8px; padding:4px 8px; }
        @keyframes spin { from { transform: rotate(0deg);} to { transform: rotate(360deg);} }
      `}</style>

      <div style={{ background: "var(--surface)", borderColor: "rgba(245,237,228,0.08)" }}
        className="w-full md:w-80 flex-shrink-0 border-b md:border-b-0 md:border-r flex flex-col">
        <div className="p-4 flex items-center justify-between">
          <div className="flex items-center gap-2">
            <Disc3 size={20} color="var(--accent)" />
            <span style={{ fontFamily: "Fraunces, serif" }} className="text-lg">Library</span>
          </div>
          <button onClick={() => fileInputRef.current?.click()}
            style={{ background: "var(--surface2)" }}
            className="flex items-center gap-1.5 text-sm px-3 py-1.5 rounded-full hover:opacity-80">
            <Upload size={14} /> Add tracks
          </button>
          <input ref={fileInputRef} type="file" accept="audio/*" multiple className="hidden" onChange={handleFiles} />
        </div>

        <div className="px-4 flex items-center gap-2 flex-wrap pb-2">
          {playlists.map((pl) => (
            <div key={pl.id} className="flex items-center gap-1">
              {renamingPlaylist === pl.id ? (
                <span className="flex items-center gap-1">
                  <input type="text" value={renameValue} autoFocus
                    onChange={(e) => setRenameValue(e.target.value)}
                    onKeyDown={(e) => e.key === "Enter" && (renamePlaylist(pl.id, renameValue), setRenamingPlaylist(null))}
                    className="text-xs w-24" />
                  <button onClick={() => { renamePlaylist(pl.id, renameValue); setRenamingPlaylist(null); }}><Check size={14} color="var(--accent)" /></button>
                </span>
              ) : (
                <button onClick={() => setActivePlaylist(pl.id)}
                  style={{ background: activePlaylist === pl.id ? "var(--accent)" : "var(--surface2)", color: activePlaylist === pl.id ? "var(--bg)" : "var(--text)" }}
                  className="text-xs px-3 py-1 rounded-full flex items-center gap-1">
                  {pl.name}
                  {!pl.system && (
                    <>
                      <Pencil size={11} onClick={(e) => { e.stopPropagation(); setRenamingPlaylist(pl.id); setRenameValue(pl.name); }} />
                      <Trash2 size={11} onClick={(e) => { e.stopPropagation(); deletePlaylist(pl.id); }} />
                    </>
                  )}
                </button>
              )}
            </div>
          ))}
          <button onClick={newPlaylist} style={{ color: "var(--muted)" }} className="text-xs flex items-center gap-1 px-2 py-1">
            <Plus size={14} /> New
          </button>
        </div>

        <div className="flex-1 overflow-y-auto px-2 pb-4" style={{ maxHeight: "420px" }}>
          {currentPlaylistTracks.length === 0 ? (
            <div style={{ color: "var(--muted)" }} className="px-4 py-8 text-sm leading-relaxed">
              No tracks here yet.
            </div>
          ) : (
            currentPlaylistTracks.map((t, i) => (
              <div key={t.id} className={`row w-full px-2 py-2 rounded-lg flex items-center gap-2 ${t.id === currentId ? "active" : ""}`}>
                <button onClick={() => playId(t.id)} className="flex items-center gap-2 flex-1 min-w-0 text-left">
                  <div style={{ width: 28, height: 28, borderRadius: 6, background: t.artUrl ? `url(${t.artUrl}) center/cover` : "var(--surface2)", flexShrink: 0 }} />
                  <span className="truncate text-sm">{t.name}</span>
                </button>
                {!playlists.find((p) => p.id === activePlaylist)?.system && (
                  <>
                    <button onClick={() => moveInPlaylist(activePlaylist, i, -1)} style={{ color: "var(--muted)" }}>▲</button>
                    <button onClick={() => moveInPlaylist(activePlaylist, i, 1)} style={{ color: "var(--muted)" }}>▼</button>
                    <button onClick={() => removeFromPlaylist(activePlaylist, t.id)} style={{ color: "var(--muted)" }}><X size={13} /></button>
                  </>
                )}
                <div className="relative">
                  <button onClick={() => setAddMenuFor(addMenuFor === t.id ? null : t.id)} style={{ color: "var(--muted)" }}><Plus size={14} /></button>
                  {addMenuFor === t.id && (
                    <div style={{ background: "var(--surface2)" }} className="absolute right-0 top-6 z-10 rounded-lg p-1 text-xs w-36 shadow-lg">
                      {playlists.filter((p) => !p.system).map((p) => (
                        <button key={p.id} onClick={() => { addToPlaylist(p.id, t.id); setAddMenuFor(null); }} className="block w-full text-left px-2 py-1 hover:opacity-80">
                          Add to {p.name}
                        </button>
                      ))}
                    </div>
                  )}
                </div>
                <button onClick={() => { artForTrack.current = t.id; artInputRef.current?.click(); }} style={{ color: "var(--muted)" }} title="Set cover"><ImageIcon size={14} /></button>
                <button onClick={() => openTrim(t)} style={{ color: "var(--muted)" }} title="Trim"><Scissors size={14} /></button>
              </div>
            ))
          )}
        </div>
        <input ref={artInputRef} type="file" accept="image/*" className="hidden" onChange={handleArtFile} />

        <div className="p-4 flex items-center gap-2" style={{ borderTop: "1px solid rgba(245,237,228,0.08)" }}>
          <span style={{ color: "var(--muted)" }} className="text-xs mr-1">Theme</span>
          {ACCENTS.map((a) => (
            <button key={a.name} onClick={() => setAccent(a)}
              style={{ background: a.val, width: 18, height: 18, borderRadius: "50%", outline: accent.name === a.name ? `2px solid ${a.val2}` : "none", outlineOffset: 2 }} />
          ))}
        </div>
      </div>

      <div className="flex-1 flex flex-col items-center justify-between p-6 md:p-10 relative">
        <div className="w-full flex items-center justify-end gap-3 mb-2">
          <div className="flex items-center gap-1 text-xs" style={{ color: "var(--muted)" }}>
            <RotateCcw size={13} />
            <span>Skip</span>
            {editingSkip ? (
              <input type="number" value={skipSeconds} autoFocus min={1} max={60}
                onChange={(e) => setSkipSeconds(Number(e.target.value) || 1)}
                onBlur={() => setEditingSkip(false)}
                onKeyDown={(e) => e.key === "Enter" && setEditingSkip(false)}
                className="w-12 text-xs" />
            ) : (
              <button onClick={() => setEditingSkip(true)} className="underline">{skipSeconds}s</button>
            )}
          </div>
          <SleepButton sleepRemaining={sleepRemaining} onSet={(mins) => setSleepAt(mins ? Date.now() + mins * 60000 : null)} />
        </div>

        <div className="w-full flex flex-col items-center flex-1 justify-center">
          <div style={{
            width: 180, height: 180, borderRadius: "50%",
            background: current?.artUrl ? `url(${current.artUrl}) center/cover` : `radial-gradient(circle at 35% 30%, var(--accent), var(--accent2) 60%, #241726 61%)`,
            boxShadow: "0 20px 40px rgba(0,0,0,0.4)", animation: isPlaying ? "spin 6s linear infinite" : "none",
            border: "6px solid rgba(245,237,228,0.08)",
          }} className="flex items-center justify-center mb-8">
            {!current?.artUrl && <div style={{ width: 26, height: 26, background: "var(--bg)", borderRadius: "50%" }} />}
          </div>

          <h1 style={{ fontFamily: "Fraunces, serif" }} className="text-2xl md:text-3xl text-center px-4 mb-1">
            {current ? current.name : "Nothing playing"}
          </h1>
          <p style={{ color: "var(--muted)" }} className="text-sm mb-8">
            {current ? playlists.find((p) => p.id === activePlaylist)?.name : "Add a track to get started"}
          </p>

          <canvas ref={canvasRef} width={480} height={64} className="w-full max-w-md mb-6" />

          <audio ref={audioRef} src={current?.url} crossOrigin="anonymous"
            onTimeUpdate={(e) => setProgress(e.target.currentTime)}
            onLoadedMetadata={(e) => setDuration(e.target.duration)}
            onEnded={() => (repeat ? playId(currentId) : goRelative(1))} />

          <div className="w-full max-w-md flex items-center gap-3 mb-2">
            <span style={{ color: "var(--muted)", fontFamily: "monospace" }} className="text-xs w-10 text-right">{fmt(progress)}</span>
            <input type="range" min={0} max={duration || 0} value={progress}
              onChange={(e) => { const v = Number(e.target.value); if (audioRef.current) audioRef.current.currentTime = v; setProgress(v); }}
              className="flex-1" />
            <span style={{ color: "var(--muted)", fontFamily: "monospace" }} className="text-xs w-10">{fmt(duration)}</span>
          </div>
        </div>

        <div className="w-full max-w-md flex flex-col items-center gap-5">
          <div className="flex items-center gap-5">
            <button onClick={() => setShuffle((s) => !s)} style={{ color: shuffle ? "var(--accent)" : "var(--muted)" }}><Shuffle size={18} /></button>
            <button onClick={() => jump(-skipSeconds)} style={{ color: "var(--text)" }}><RotateCcw size={20} /></button>
            <button onClick={() => goRelative(-1)} style={{ color: "var(--text)" }}><SkipBack size={22} fill="currentColor" /></button>
            <button onClick={togglePlay} disabled={!current}
              style={{ background: "var(--accent)", color: "var(--bg)" }}
              className="w-14 h-14 rounded-full flex items-center justify-center disabled:opacity-40 hover:opacity-90">
              {isPlaying ? <Pause size={24} fill="currentColor" /> : <Play size={24} fill="currentColor" style={{ marginLeft: 2 }} />}
            </button>
            <button onClick={() => goRelative(1)} style={{ color: "var(--text)" }}><SkipForward size={22} fill="currentColor" /></button>
            <button onClick={() => jump(skipSeconds)} style={{ color: "var(--text)" }}><RotateCw size={20} /></button>
            <button onClick={() => setRepeat((r) => !r)} style={{ color: repeat ? "var(--accent)" : "var(--muted)" }}><Repeat size={18} /></button>
          </div>
          <div className="w-full flex items-center gap-3">
            <button onClick={() => setMuted((m) => !m)} style={{ color: "var(--muted)" }}>
              {muted || volume === 0 ? <VolumeX size={16} /> : <Volume2 size={16} />}
            </button>
            <input type="range" min={0} max={1} step={0.01} value={muted ? 0 : volume}
              onChange={(e) => { setVolume(Number(e.target.value)); setMuted(false); }}
              className="flex-1 max-w-[160px]" />
          </div>
        </div>
      </div>

      {trimTrackId && (
        <div className="absolute inset-0 flex items-center justify-center p-6" style={{ background: "rgba(0,0,0,0.6)" }}>
          <div style={{ background: "var(--surface)" }} className="rounded-xl p-6 w-full max-w-md">
            <div className="flex items-center justify-between mb-4">
              <h3 style={{ fontFamily: "Fraunces, serif" }} className="text-lg">Trim track</h3>
              <button onClick={() => setTrimTrackId(null)}><X size={18} /></button>
            </div>
            {!trimBuffer ? (
              <p style={{ color: "var(--muted)" }} className="text-sm">Loading audio…</p>
            ) : (
              <>
                <p style={{ color: "var(--muted)" }} className="text-xs mb-3">
                  Start {fmt(trimRange[0])} · End {fmt(trimRange[1])} · Length {fmt(trimRange[1] - trimRange[0])}
                </p>
                <label style={{ color: "var(--muted)" }} className="text-xs">Start</label>
                <input type="range" min={0} max={trimBuffer.duration} step={0.1} value={trimRange[0]}
                  onChange={(e) => setTrimRange([Math.min(Number(e.target.value), trimRange[1] - 0.5), trimRange[1]])}
                  className="w-full mb-3" />
                <label style={{ color: "var(--muted)" }} className="text-xs">End</label>
                <input type="range" min={0} max={trimBuffer.duration} step={0.1} value={trimRange[1]}
                  onChange={(e) => setTrimRange([trimRange[0], Math.max(Number(e.target.value), trimRange[0] + 0.5)])}
                  className="w-full mb-4" />
                <div className="flex justify-end gap-2">
                  <button onClick={() => setTrimTrackId(null)} style={{ color: "var(--muted)" }} className="text-sm px-3 py-1.5">Cancel</button>
                  <button onClick={saveTrim} style={{ background: "var(--accent)", color: "var(--bg)" }} className="text-sm px-4 py-1.5 rounded-full">Save as new track</button>
                </div>
              </>
            )}
          </div>
        </div>
      )}
    </div>
  );
}

function SleepButton({ sleepRemaining, onSet }) {
  const [open, setOpen] = useState(false);
  return (
    <div className="relative">
      <button onClick={() => setOpen((o) => !o)} style={{ color: sleepRemaining ? "var(--accent)" : "var(--muted)" }} className="flex items-center gap-1 text-xs">
        <Moon size={14} /> {sleepRemaining ? `${Math.ceil(sleepRemaining / 60000)}m` : "Sleep timer"}
      </button>
      {open && (
        <div style={{ background: "var(--surface2)" }} className="absolute right-0 top-6 z-10 rounded-lg p-2 text-xs w-32 shadow-lg">
          {[10, 20, 30, 45, 60].map((m) => (
            <button key={m} onClick={() => { onSet(m); setOpen(false); }} className="block w-full text-left px-2 py-1 hover:opacity-80">{m} minutes</button>
          ))}
          <button onClick={() => { onSet(null); setOpen(false); }} className="block w-full text-left px-2 py-1 hover:opacity-80" style={{ color: "var(--muted)" }}>Turn off</button>
        </div>
      )}
    </div>
  );
}
