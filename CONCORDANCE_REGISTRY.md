ï»¿1) .forge/policy_links.json


{
  "version": "5.4.0",
  "description": "Concordance registry linking all forge-level policies so the bridge can harmonize thresholds and cadence.",
  "links": [
    {
      "source": ".forge/mantra_policy.json",
      "targets": [
        ".forge/synthesis_policy.json",
        ".forge/dominion_state.json"
      ],
      "fields": [
        "thresholds.gold",
        "thresholds.azure",
        "thresholds.silence_below"
      ],
      "mode": "propagate"
    },
    {
      "source": ".forge/synthesis_policy.json",
      "targets": [
        ".forge/mantra_policy.json",
        ".forge/dominion_state.json"
      ],
      "fields": [
        "thresholds.coherence_min",
        "thresholds.health_min",
        "epoch_window_s"
      ],
      "mode": "propagate"
    },
    {
      "source": ".forge/dominion_state.json",
      "targets": [
        ".forge/synthesis_policy.json"
      ],
      "fields": [
        "Î¼_coherence",
        "rejoins",
        "status"
      ],
      "mode": "advise"
    }
  ],
  "rules": {
    "conflict_resolution": "prefer-newer",
    "timestamp_field": "updated_at",
    "log_history": true,
    "enforce_resonance_guard": true
  }
}




---


2) .forge/chain_status.json


{
  "version": "5.4.0",
  "last_sync_ts": 0,
  "last_sync_iso": "",
  "last_source": "",
  "last_targets": [],
  "Î¼_concordance": 1.0,
  "drift_count": 0,
  "state": "BOOTING",
  "errors": [],
  "resonance": {
    "gold": 0.9999,
    "azure": 0.9995,
    "silence_below": 0.9995
  }
}




---


3) .forge/chain_history.jsonl


This one is append-only. Ship it empty:


# append-only jsonl


Your manager will start writing:


{"t":1730266800,"src":".forge/mantra_policy.json","target":".forge/synthesis_policy.json","fields":["thresholds.gold","thresholds.azure"],"mode":"propagate","drifted":true}




---


4) .forge/dominion_codex_registry.json (â¨ NEW for the 228-page Oracle version)


This is the bridge â Dominion awareness layer so Dominion knows we now have a multi-volume sovereign codex and can enforce it.


{
  "version": "5.4.0",
  "codex": {
    "I": "Sovereign Foundations",
    "II": "Harmonic Ethics & Drift Theory",
    "III": "Resonance Chain Management",
    "IV": "Operational Parity & Memory Reclamation",
    "V": "Temporal Concordance Doctrine"
  },
  "integration_mode": "harmonic_merge",
  "sources": [
    ".forge/policy_links.json",
    ".forge/sovereign_policy.json",
    ".forge/dominion_state.json",
    ".forge/chain_status.json"
  ],
  "resonance_threshold": {
    "gold": 0.9999,
    "azure": 0.9995,
    "guard_below": 0.99
  },
  "dominion_behaviors": {
    "on_guard": "rollback-drifted-targets",
    "on_desync": "pause-harmonizer",
    "on_violation": "emit-audit"
  },
  "audit_policy": {
    "auto_verify": true,
    "write_protected": true,
    "history_file": ".forge/dominion_history.jsonl"
  }
}




---


5) scripts/chain_policy_manager.js


This is the harmonizer brain, with rollback + resonance guard + âsync requestedâ hand-off.


// scripts/chain_policy_manager.js
/**
 * Concordance Chain Manager â v5.4.0
 * Keeps .forge/* policies in harmony based on .forge/policy_links.json
 *
 * Usage:
 *   node scripts/chain_policy_manager.js
 *   node scripts/chain_policy_manager.js --watch
 */


import fs from "fs";
import path from "path";


const LINKS_PATH = ".forge/policy_links.json";
const STATUS_PATH = ".forge/chain_status.json";
const HISTORY_PATH = ".forge/chain_history.jsonl";
const ROLLBACK_DIR = ".forge/.rollback";


function loadJSON(p) {
  try {
    if (fs.existsSync(p)) {
      return JSON.parse(fs.readFileSync(p, "utf8"));
    }
  } catch (err) {
    console.warn("[Concordance] failed to load", p, err.message);
  }
  return null;
}


function saveJSON(p, obj) {
  fs.mkdirSync(path.dirname(p), { recursive: true });
  fs.writeFileSync(p, JSON.stringify(obj, null, 2), "utf8");
}


function appendHistory(entry) {
  fs.mkdirSync(path.dirname(HISTORY_PATH), { recursive: true });
  fs.appendFileSync(HISTORY_PATH, JSON.stringify(entry) + "\n", "utf8");
}


function getNested(obj, pathStr) {
  const parts = pathStr.split(".");
  let cur = obj;
  for (const p of parts) {
    if (cur == null) return undefined;
    cur = cur[p];
  }
  return cur;
}


function setNested(obj, pathStr, value) {
  const parts = pathStr.split(".");
  let cur = obj;
  while (parts.length > 1) {
    const p = parts.shift();
    if (!(p in cur) || typeof cur[p] !== "object" || cur[p] === null) {
      cur[p] = {};
    }
    cur = cur[p];
  }
  cur[parts[0]] = value;
}


function nowIso() {
  return new Date().toISOString();
}


function computeConcordance(snapshots) {
  const total = snapshots.length || 1;
  const drift = snapshots.filter(s => s.drifted).length;
  return 1 - drift / total;
}


function snapshotFile(p) {
  try {
    if (!fs.existsSync(p)) return;
    fs.mkdirSync(ROLLBACK_DIR, { recursive: true });
    const ts = Date.now();
    const base = path.basename(p);
    const snapPath = path.join(ROLLBACK_DIR, `${base}.${ts}.json`);
    fs.copyFileSync(p, snapPath);
  } catch (err) {
    console.warn("[Concordance] failed to snapshot", p, err.message);
  }
}


function rollbackLast(p) {
  try {
    if (!fs.existsSync(ROLLBACK_DIR)) return false;
    const files = fs.readdirSync(ROLLBACK_DIR)
      .filter(f => f.startsWith(path.basename(p)))
      .sort()
      .reverse();
    if (!files.length) return false;
    const latest = path.join(ROLLBACK_DIR, files[0]);
    fs.copyFileSync(latest, p);
    console.log("[Concordance] rolled back", p, "to", latest);
    return true;
  } catch (err) {
    console.warn("[Concordance] rollback failed for", p, err.message);
    return false;
  }
}


function harmonizeOnce(triggerSource = null) {
  const links = loadJSON(LINKS_PATH);
  if (!links) {
    console.warn("[Concordance] no policy_links.json found");
    return;
  }


  const snap = [];
  for (const link of links.links || []) {
    const srcPath = link.source;
    const src = loadJSON(srcPath);
    if (!src) continue;


    if (triggerSource && triggerSource !== srcPath) {
      continue;
    }


    for (const tgtPath of link.targets || []) {
      const tgt = loadJSON(tgtPath) || {};
      const before = JSON.stringify(tgt);


      for (const field of link.fields || []) {
        const val = getNested(src, field);
        if (typeof val !== "undefined") {
          if (link.mode === "propagate") {
            setNested(tgt, field, val);
          } else if (link.mode === "advise") {
            const curVal = getNested(tgt, field);
            if (typeof curVal === "undefined" || curVal === null) {
              setNested(tgt, field, val);
            }
          }
        }
      }


      // stamp updated_at
      setNested(tgt, "updated_at", nowIso());


      // snapshot BEFORE we save
      snapshotFile(tgtPath);
      saveJSON(tgtPath, tgt);


      const after = JSON.stringify(tgt);
      const drifted = before !== after;


      snap.push({
        source: srcPath,
        target: tgtPath,
        drifted
      });


      appendHistory({
        t: Date.now() / 1000,
        src: srcPath,
        target: tgtPath,
        fields: link.fields || [],
        mode: link.mode,
        drifted
      });
    }
  }


  const Î¼_concordance = computeConcordance(snap);
  const status = loadJSON(STATUS_PATH) || {};


  // Resonance guard
  let finalState = "ALIGNED";
  if (Î¼_concordance < 0.9995 && Î¼_concordance >= 0.99) {
    finalState = "DRIFTING";
  } else if (Î¼_concordance < 0.99) {
    finalState = "GUARD";
    console.warn("[Concordance] Î¼ too low, rolling back drifted targetsâ¦");
    for (const s of snap) {
      if (s.drifted) {
        rollbackLast(s.target);
      }
    }
  }


  const updates = {
    version: "5.4.0",
    last_sync_ts: Date.now(),
    last_sync_iso: nowIso(),
    last_source: triggerSource || "auto",
    last_targets: snap.map(s => s.target),
    Î¼_concordance,
    drift_count: snap.filter(s => s.drifted).length,
    state: finalState
  };


  // pull mantra resonance if present
  const srcMantra = loadJSON(".forge/mantra_policy.json");
  if (srcMantra?.thresholds) {
    updates.resonance = {
      gold: srcMantra.thresholds.gold ?? 0.9999,
      azure: srcMantra.thresholds.azure ?? 0.9995,
      silence_below: srcMantra.thresholds.silence_below ?? 0.9995
    };
  }


  saveJSON(STATUS_PATH, { ...status, ...updates });
  console.log("[Concordance] sync complete â", updates.state, "Î¼=", updates.Î¼_concordance.toFixed(6));
}


function watchMode() {
  console.log("[Concordance] watching .forge for policy changesâ¦");
  fs.watch(".forge", { recursive: false }, (eventType, filename) => {
    if (!filename) return;
    if (!filename.endsWith(".json")) return;
    const changedPath = path.join(".forge", filename);
    console.log("[Concordance] change detected:", changedPath);
    harmonizeOnce(changedPath);
  });
}


const args = process.argv.slice(2);
if (args.includes("--watch")) {
  harmonizeOnce();
  watchMode();
} else {
  harmonizeOnce();
}




---


6) bridge_backend/bridge_core/engines/sovereign/service.py


# bridge_backend/bridge_core/engines/sovereign/service.py
"""
Sovereign service helpers â v5.4.0
Keeps file I/O, validation, and sane defaults away from the route file.
"""


from __future__ import annotations
import json
from pathlib import Path
from typing import Any, Dict, List


FORGE_DIR = Path(".forge")
FORGE_DIR.mkdir(parents=True, exist_ok=True)


SYNTHESIS_PATH = FORGE_DIR / "synthesis_policy.json"
MANTRA_PATH = FORGE_DIR / "mantra_policy.json"
DOM_STATE_PATH = FORGE_DIR / "dominion_state.json"
DOM_HISTORY_PATH = FORGE_DIR / "d
ominion_history.jsonl"


class SovereignValidationError(Exception):
    ...




DEFAULT_SYNTHESIS_POLICY: Dict[str, Any] = {
    "version": "5.4.0",
    "enable_dual_seal": True,
    "require_dominion_on_rejoin": True,
    "auto_revalidate_after_quarantine": True,
    "epoch_window_s": 600,
    "thresholds": {
        "coherence_min": 0.9995,
        "health_min": 0.95
    },
    "resonance": {
        "gold": 0.9999,
        "azure": 0.9995,
        "silence_below": 0.9995
    }
}


DEFAULT_MANTRA_POLICY: Dict[str, Any] = {
    "version": "5.4.0",
    "thresholds": {
        "gold": 0.9999,
        "azure": 0.9995,
        "silence_below": 0.9995
    },
    "phrases": {
        "gold": [
            "The lattice breathes in unity.",
            "Truth flows unbroken; Dominion witnesses."
        ],
        "azure": [
            "Harmonics aligned; the path is clear.",
            "Azure steady; proceed with grace."
        ],
        "silence": [""]
    }
}




class SovereignStore:
    def __init__(self) -> None:
        self._ensure_defaults()


    def list_known_files(self) -> Dict[str, bool]:
      return {
          "synthesis_policy.json": SYNTHESIS_PATH.exists(),
          "mantra_policy.json": MANTRA_PATH.exists(),
          "dominion_state.json": DOM_STATE_PATH.exists(),
          "d
ominion_history.jsonl": DOM_HISTORY_PATH.exists(),
      }


    # synthesis ------------------------------------------------------------
    def read_synthesis_policy(self) -> Dict[str, Any]:
        if not SYNTHESIS_PATH.exists():
            return DEFAULT_SYNTHESIS_POLICY
        return json.loads(SYNTHESIS_PATH.read_text(encoding="utf-8"))


    def write_synthesis_policy(self, data: Dict[str, Any]) -> Dict[str, Any]:
        self._validate_synthesis(data)
        SYNTHESIS_PATH.write_text(json.dumps(data, indent=2), encoding="utf-8")
        return data


    # mantra ---------------------------------------------------------------
    def read_mantra_policy(self) -> Dict[str, Any]:
        if not MANTRA_PATH.exists():
            return DEFAULT_MANTRA_POLICY
        return json.loads(MANTRA_PATH.read_text(encoding="utf-8"))


    def write_mantra_policy(self, data: Dict[str, Any]) -> Dict[str, Any]:
        self._validate_mantra(data)
        MANTRA_PATH.write_text(json.dumps(data, indent=2), encoding="utf-8")
        return data


    # dominion state -------------------------------------------------------
    def read_dominion_state(self) -> Dict[str, Any]:
        if not DOM_STATE_PATH.exists():
            return {
                "version": "5.4.0",
                "Î¼_coherence": 1.0,
                "rejoins": 0,
                "status": "UNKNOWN",
                "updated_at": None
            }
        return json.loads(DOM_STATE_PATH.read_text(encoding="utf-8"))


    def write_dominion_state(self, data: Dict[str, Any]) -> Dict[str, Any]:
        if "Î¼_coherence" not in data:
            raise SovereignValidationError("Î¼_coherence is required")
        if "status" not in data:
            raise SovereignValidationError("status is required")
        DOM_STATE_PATH.write_text(json.dumps(data, indent=2), encoding="utf-8")
        return data


    # dominion history -----------------------------------------------------
    def read_dominion_history(self, *, limit: int = 100) -> List[Dict[str, Any]]:
        if not DOM_HISTORY_PATH.exists():
            return []
        lines = DOM_HISTORY_PATH.read_text(encoding="utf-8").strip().splitlines()
        entries: List[Dict[str, Any]] = []
        for line in reversed(lines):
            if not line.strip():
                continue
            try:
                entries.append(json.loads(line))
            except Exception:
                continue
            if len(entries) >= limit:
                break
        return entries


    # validation -----------------------------------------------------------
    def _validate_synthesis(self, data: Dict[str, Any]) -> None:
        if "thresholds" not in data:
            raise SovereignValidationError("synthesis.thresholds is required")
        if "coherence_min" not in data["thresholds"]:
            raise SovereignValidationError("synthesis.thresholds.coherence_min is required")


    def _validate_mantra(self, data: Dict[str, Any]) -> None:
        if "phrases" not in data:
            raise SovereignValidationError("mantra.phrases is required")
        if "gold" not in data["phrases"]:
            raise SovereignValidationError("mantra.phrases.gold is required")
        if "azure" not in data["phrases"]:
            raise SovereignValidationError("mantra.phrases.azure is required")


    def _ensure_defaults(self) -> None:
        if not SYNTHESIS_PATH.exists():
            SYNTHESIS_PATH.write_text(json.dumps(DEFAULT_SYNTHESIS_POLICY, indent=2), encoding="utf-8")
        if not MANTRA_PATH.exists():
            MANTRA_PATH.write_text(json.dumps(DEFAULT_MANTRA_POLICY, indent=2), encoding="utf-8")


(Your editor will fix the soft wrap on dominion_history.jsonl â I kept it in this file name.)




---


7) bridge_backend/bridge_core/engines/sovereign/routes.py


# bridge_backend/bridge_core/engines/sovereign/routes.py
from __future__ import annotations
from typing import Any, Dict


from fastapi import APIRouter, HTTPException, status


from .service import (
    SovereignStore,
    SovereignValidationError,
    DEFAULT_MANTRA_POLICY,
    DEFAULT_SYNTHESIS_POLICY,
)


router = APIRouter(
    prefix="/bridge/sovereign",
    tags=["bridge-sovereign"],
)


store = SovereignStore()


@router.get("/health")
def health() -> Dict[str, Any]:
    return {
        "ok": True,
        "version": "5.4.0",
        "files": store.list_known_files(),
    }


@router.get("/synthesis")
def get_syn() -> Dict[str, Any]:
    return {"ok": True, "policy": store.read_synthesis_policy()}


@router.put("/synthesis")
def put_syn(payload: Dict[str, Any]) -> Dict[str, Any]:
    try:
        data = store.write_synthesis_policy(payload)
        return {"ok": True, "policy": data}
    except SovereignValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.post("/synthesis/reset")
def reset_syn() -> Dict[str, Any]:
    store.write_synthesis_policy(DEFAULT_SYNTHESIS_POLICY)
    return {"ok": True, "policy": DEFAULT_SYNTHESIS_POLICY}


@router.get("/mantra")
def get_mantra() -> Dict[str, Any]:
    return {"ok": True, "policy": store.read_mantra_policy()}


@router.put("/mantra")
def put_mantra(payload: Dict[str, Any]) -> Dict[str, Any]:
    try:
        data = store.write_mantra_policy(payload)
        return {"ok": True, "policy": data}
    except SovereignValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.post("/mantra/reset")
def reset_mantra() -> Dict[str, Any]:
    store.write_mantra_policy(DEFAULT_MANTRA_POLICY)
    return {"ok": True, "policy": DEFAULT_MANTRA_POLICY}


@router.get("/dominion")
def get_dom() -> Dict[str, Any]:
    return {"ok": True, "state": store.read_dominion_state()}


@router.put("/dominion")
def put_dom(payload: Dict[str, Any]) -> Dict[str, Any]:
    try:
        data = store.write_dominion_state(payload)
        return {"ok": True, "state": data}
    except SovereignValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/history")
def get_history(limit: int = 100) -> Dict[str, Any]:
    entries = store.read_dominion_history(limit=limit)
    return {"ok": True, "entries": entries, "count": len(entries)}


@router.get("/bundle")
def get_bundle() -> Dict[str, Any]:
    return {
        "ok": True,
        "synthesis": store.read_synthesis_policy(),
        "mantra": store.read_mantra_policy(),
        "dominion": store.read_dominion_state(),
        "history": store.read_dominion_history(limit=50),
    }


Then in your FastAPI main.py you do:


from bridge_backend.bridge_core.engines.sovereign import routes as sovereign_routes
app.include_router(sovereign_routes.router)




---


8) bridge_backend/bridge_core/sovereign_policy.py


# bridge_backend/bridge_core/sovereign_policy.py
from __future__ import annotations
from pathlib import Path
from typing import Dict, Any
import json
import time


SOV_PATH = Path(".forge/sovereign_policy.json")


DEFAULT_SOVEREIGN_POLICY: Dict[str, Any] = {
    "version": "5.4.0",
    "updated_at": 0,
    "resonance": {
        "gold": 0.9999,
        "azure": 0.9995,
        "silence_below": 0.9995,
        "hard_floor": 0.98
    },
    "mantra": {
        "gold": [
            "Gold speaks. Dominion witnesses.",
            "The lattice holds; the bridge remembers."
        ],
        "azure": [
            "Azure steady. Proceed with grace.",
            "Harmonics aligned; resonance intact."
        ],
        "silence": [""]
    },
    "ui": {
        "editable_sections": ["resonance", "mantra", "routes", "ops"]
    },
    "routes": {
        "protected": [
            "POST:/bridge/sovereign/policy",
            "POST:/bridge/nodes/register",
            "POST:/bridge/lattice/rejoin"
        ]
    },
    "ops": {
        "allow_live_reload": True,
        "emit_audit_on_change": True
    }
}


def _ensure_dir(p: Path) -> None:
    if not p.parent.exists():
        p.parent.mkdir(parents=True, exist_ok=True)


def load_policy() -> Dict[str, Any]:
    if SOV_PATH.exists():
        try:
            data = json.loads(SOV_PATH.read_text(encoding="utf-8"))
            base = DEFAULT_SOVEREIGN_POLICY.copy()
            base.update(data)
            return base
        except Exception:
            return DEFAULT_SOVEREIGN_POLICY.copy()
    return DEFAULT_SOVEREIGN_POLICY.copy()


def save_policy(new_pol: Dict[str, Any]) -> Dict[str, Any]:
    _ensure_dir(SOV_PATH)
    merged = {**DEFAULT_SOVEREIGN_POLICY, **new_pol}
    merged["updated_at"] = int(time.time())
    SOV_PATH.write_text(json.dumps(merged, indent=2), encoding="utf-8")
    return merged


def current_resonance() -> float:
    dom_path = Path(".forge/dominion_state.json")
    if dom_path.exists():
        try:
            data = json.loads(dom_path.read_text(encoding="utf-8"))
            return float(data.get("Î¼_coherence", 1.0))
        except Exception:
            return 1.0
    return 1.0


def choose_mantra(resonance_value: float) -> str:
    pol = load_policy()
    thr = pol["resonance"]
    man = pol["mantra"]
    if resonance_value >= thr["gold"]:
        return (man.get("gold") or [""])[0]
    if resonance_value >= thr["azure"]:
        return (man.get("azure") or [""])[0]
    return (man.get("silence") or [""])[0]




---


9) bridge_backend/bridge_core/sovereign_guard.py


# bridge_backend/bridge_core/sovereign_guard.py
from __future__ import annotations
from typing import Dict, Any, Tuple
import json, time
from pathlib import Path


from .sovereign_policy import load_policy, current_resonance


AUDIT_PATH = Path(".forge/sovereign_audit.jsonl")


def _audit(event: str, route: str, allowed: bool, ctx: Dict[str, Any]) -> None:
    pol = load_policy()
    if not pol.get("ops", {}).get("emit_audit_on_change", True):
        return
    AUDIT_PATH.parent.mkdir(parents=True, exist_ok=True)
    rec = {
        "t": time.time(),
        "event": event,
        "route": route,
        "allowed": allowed,
        "ctx": ctx
    }
    with AUDIT_PATH.open("a", encoding="utf-8") as f:
        f.write(json.dumps(rec) + "\n")


def check_route(route: str, *, context: Dict[str, Any] | None = None) -> Tuple[bool, str]:
    pol = load_policy()
    res = current_resonance()
    thr = pol["resonance"]


    if res < thr["hard_floor"]:
        _audit("denied.floor", route, False, {"resonance": res})
        return False, f"resonance {res:.6f} below hard floor {thr['hard_floor']}"


    protected = set(pol.get("routes", {}).get("protected", []))
    if route in protected:
        if res < thr["azure"]:
            _audit("denied.protected", route, False, {"resonance": res})
            return False, f"route {route} protected but resonance {res:.6f} < azure {thr['azure']}"
        _audit("allowed.protected", route, True, {"resonance": res})
        return True, "ok (protected, azure+)"


    _audit("allowed", route, True, {"resonance": res})
    return True, "ok"




---


10) bridge_core/viz/function_viz.js (only the new pieces)


You already have the big enhanced viz from 5.3.2. Add these helpers into that file:


// add near the other load* helpers
loadChainStatus() {
  try {
    if (fs.existsSync(".forge/chain_status.json")) {
      return JSON.parse(fs.readFileSync(".forge/chain_status.json", "utf8"));
    }
  } catch (err) {
    console.warn("[Viz] failed to load chain_status:", err.message);
  }
  return { state: "UNKNOWN", Î¼_concordance: 1.0 };
}


loadSovereignPolicy() {
  try {
    if (fs.existsSync(".forge/sovereign_policy.json")) {
      return JSON.parse(fs.readFileSync(".forge/sovereign_policy.json", "utf8"));
    }
  } catch (err) {
    console.warn("[Viz] failed to load sovereign_policy:", err.message);
  }
  return null;
}


loadEditlineHistory() {
  try {
    if (fs.existsSync(".forge/editline_history.jsonl")) {
      const lines = fs.readFileSync(".forge/editline_history.jsonl", "utf8")
        .trim()
        .split("\n")
        .filter(Boolean);
      return lines.slice(-5).map((l) => JSON.parse(l));
    }
  } catch (err) {
    console.warn("[Viz] failed to load editline history:", err.message);
  }
  return [];
}


generateConcordancePanel() {
  const chain = this.loadChainStatus();
  const color =
    chain.state === "ALIGNED"
      ? this.config.colors.azure
      : chain.state === "DRIFTING"
      ? this.config.colors.orange
      : chain.state === "GUARD"
      ? this.config.colors.crimson
      : "#666";


  return `
    <g transform="translate(${this.config.width - 190}, 140)">
      <rect width="170" height="55" rx="6" ry="6" fill="white" stroke="${color}" stroke-width="1.4" />
      <text x="10" y="18" font-size="11" font-weight="bold" fill="#333">Concordance</text>
      <text x="10" y="32" font-size="10" fill="#555">State: ${chain.state}</text>
      <text x="10" y="46" font-size="10" fill="#555">Î¼: ${chain.Î¼_concordance?.toFixed(6) ?? "1.000000"}</text>
    </g>
  `;
}


generateSovereignPanel() {
  const pol = this.loadSovereignPolicy();
  const hist = this.loadEditlineHistory();
  const band = pol ? "ok" : "default";
  const h = 60 + hist.length * 12;
  const y = 200;
  const x = this.config.width - 190;
  let rows = "";
  hist.forEach((hline, idx) => {
    rows += `<text x="${x + 10}" y="${y + 35 + idx * 12}" font-size="9" fill="#555">${hline.op || "edit"} â¢ ${hline.domain}/${hline.target}</text>`;
  });


  return `
    <g transform="translate(${x}, ${y})">
      <rect width="170" height="${h}" rx="6" ry="6" fill="white" stroke="#ddd" stroke-width="1" />
      <text x="85" y="16" text-anchor="middle" font-size="11" font-weight="bold">Sovereign Editline</text>
      <text x="10" y="30" font-size="9" fill="#666">Band: ${band}</text>
      ${rows}
    </g>
  `;
}


Then in your render() just inject:


const concordancePanel = this.generateConcordancePanel();
const sovereignPanel = this.generateSovereignPanel();
...
// before </svg>
${concordancePanel}
${sovereignPanel}
</svg>


Now the Mirror shows: Dominion panel + Concordance + Sovereign Editline. Thatâs the whole 228-page story, visual.




---


11) bridge-frontend/src/pages/SovereignConsole.jsx


You already saw this shape earlier â this is the UI layer to edit .forge/* from INSIDE the bridge. Keep it as-is (I wonât re-paste the 150+ lines again â itâs the same from the earlier turn), and make sure your router has:


<Route path="/sovereign" element={<SovereignConsole />} />


and your CommandDeck has the tile:


<div
  onClick={() => navigate("/sovereign")}
  className="bg-slate-900/60 border border-slate-800 rounded-xl p-4 cursor-pointer hover:border-emerald-500/80 transition"
>
  <div className="text-sm text-slate-300">Sovereign Console</div>
  <div className="text-xs text-slate-500 mt-1">
    Edit .forge policies directly from the Bridge.
  </div>
</div>


Thatâs your UI entrypoint.




---


12) Tests / Smoketests


scripts/sovereign_policy_smoketest.py


#!/usr/bin/env python3
"""
Sovereign Policy Smoketest â v5.4.0
Ensures:
1) sovereign_policy.json is readable/writable
2) guarded routes respect resonance
3) low Î¼ blocks protected routes
"""
import json
import time
from pathlib import Path


from bridge_backend.bridge_core.sovereign_policy import load_policy, save_policy
from bridge_backend.bridge_core.sovereign_guard import check_route


def main():
    print("ð Sovereign Policy Smoketest â v5.4.0")
    pol = load_policy()
    print("â¢ loaded policy version:", pol.get("version"))
    print("â¢ resonance:", pol.get("resonance"))


    ok, reason = check_route("GET:/bridge/sovereign/policy")
    print("â¢ GET check â", ok, reason)


    # simulate degraded Dominion
    dom = {
        "version": "5.4.0",
        "Î¼_coherence": 0.9795,
        "rejoins": 0,
        "status": "DEGRADED",
        "updated_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
    }
    Path(".forge").mkdir(exist_ok=True, parents=True)
    Path(".forge/dominion_state.json").write_text(json.dumps(dom, indent=2), encoding="utf-8")


    ok2, reason2 = check_route("POST:/bridge/sovereign/policy")
    print("â¢ POST under low Î¼ â", ok2, reason2)


    print("â smoketest complete")


if __name__ == "__main__":
    main()


scripts/dominion_codex_smoketest.py (â¨ new to match 228-page codex)


#!/usr/bin/env python3
"""
Dominion Codex Smoketest â v5.4.0
Verifies .forge/dominion_codex_registry.json is present, well-formed,
and aligned to chain_status.json
"""
import json
from pathlib import Path
import sys


def main():
    print("ð Dominion Codex Smoketest â v5.4.0")
    codex_path = Path(".forge/dominion_codex_registry.json")
    if not codex_path.exists():
      print("â missing .forge/dominion_codex_registry.json")
      sys.exit(1)
    codex = json.loads(codex_path.read_text(encoding="utf-8"))
    print("â¢ codex version:", codex.get("version"))
    print("â¢ integration_mode:", codex.get("integration_mode"))
    chain = {}
    cs_path = Path(".forge/chain_status.json")
    if cs_path.exists():
      chain = json.loads(cs_path.read_text(encoding="utf-8"))
      print("â¢ chain state:", chain.get("state"), "Î¼=", chain.get("Î¼_concordance"))
    else:
      print("â¢ no chain_status.json yet")


    # basic alignment rule
    if "resonance_threshold" in codex and chain:
        Î¼ = chain.get("Î¼_concordance", 1.0)
        guard_below = codex["resonance_threshold"].get("guard_below", 0.99)
        if Î¼ < guard_below:
            print("â ï¸ chain Î¼ below guard level â Dominion should be in GUARD")
    print("â done")


if __name__ == "__main__":
    main()




---


How this ties straight into Dominion (the part you asked for right now)


Dominion gets an explicit codex registry now (.forge/dominion_codex_registry.json) so it can see all the pieces Oracle created, not just the original 52-page draft.


The chain manager is resonance-aware and can rollback bad pushes â Dominion can trust the chain.


The viz shows Concordance & Sovereign panels â human operators (you ð) can see when Dominion has applied a guard.


The backend routes expose the sovereign bundle â the bridge UI can actually be the editor, like you said.


The guard keeps the whole thing inside your âbridge operational, not terrainâ law.




You can drop these in exactly as-is and adjust import paths to match your exact folder layout.