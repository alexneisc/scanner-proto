<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { scan, cancel, requestPermissions, Format } from "@tauri-apps/plugin-barcode-scanner";

  type Entry = { type: string; text: string; ms: number };

  const LABELS: Record<string, string> = {
    [Format.QRCode]: "QR",
    [Format.UPC_A]: "UPC-A",
    [Format.UPC_E]: "UPC-E",
    [Format.EAN8]: "EAN-8",
    [Format.EAN13]: "EAN-13",
    [Format.Code39]: "Code 39",
    [Format.Code93]: "Code 93",
    [Format.Code128]: "Code 128",
    [Format.Codabar]: "Codabar",
    [Format.ITF]: "ITF",
    [Format.Aztec]: "Aztec",
    [Format.DataMatrix]: "Data Matrix",
    [Format.PDF417]: "PDF417",
    [Format.GS1DataBar]: "GS1 DataBar",
    [Format.GS1DataBarLimited]: "GS1 DataBar Limited",
    [Format.GS1DataBarExpanded]: "GS1 DataBar Expanded",
  };

  let scanning = $state(false);
  let torchOn = $state(false);
  let torchInfo = $state("");
  let cancelled = false;
  let last = $state<Entry | null>(null);
  let message = $state("");
  let history = $state<Entry[]>([]);

  function errText(e: unknown): string {
    if (typeof e === "string") return e;
    if (e instanceof Error) return e.message;
    if (e && typeof e === "object" && "message" in e) return String((e as { message: unknown }).message);
    try { return JSON.stringify(e); } catch { return String(e); }
  }

  async function start() {
    message = "";
    torchInfo = "";
    cancelled = false;
    scanning = true;
    try {
      const perm = await requestPermissions();
      if (perm !== "granted") {
        message = "Camera access denied. Enable it in Settings > Scanner Proto.";
        return;
      }
      const t0 = performance.now();
      const r = await scan({ windowed: true });
      const ms = Math.round(performance.now() - t0);
      const entry = { type: LABELS[r.format] ?? String(r.format), text: r.content, ms };
      last = entry;
      history = [entry, ...history].slice(0, 10);
    } catch (e) {
      const t = errText(e);
      message = cancelled || t === "cancelled" ? "Scan cancelled" : `Error: ${t}`;
    } finally {
      if (torchOn) await setTorch(false);
      scanning = false;
    }
  }

  async function setTorch(on: boolean) {
    try {
      const r = await invoke<{ enabled: boolean }>("plugin:barcode-scanner|set_torch", { enabled: on });
      torchOn = r?.enabled === true;
      torchInfo = `set_torch(${on}) -> ${JSON.stringify(r)}`;
    } catch (e) {
      torchOn = false;
      torchInfo = `set_torch(${on}) error: ${errText(e)}`;
    }
  }

  async function stop() {
    cancelled = true;
    try {
      await cancel();
    } catch (e) {
      message = `Error: ${errText(e)}`;
      scanning = false;
    }
  }
</script>

<svelte:head>
  {#if scanning}
    <style>
      html, body { background: transparent !important; }
    </style>
  {/if}
</svelte:head>

{#if scanning}
  <div class="controls">
    {#if torchInfo}<div class="info">{torchInfo}</div>{/if}
    <button class="flash" onclick={() => setTorch(!torchOn)}>Flash: {torchOn ? "On" : "Off"}</button>
    <button class="cancel" onclick={stop}>Cancel</button>
  </div>
{:else}
  <main>
    <button class="scan" onclick={start}>Scan</button>

    {#if message}<p class="msg">{message}</p>{/if}

    {#if last}
      <section class="result">
        <div class="type">{last.type}</div>
        <div class="text">{last.text}</div>
        <div class="ms">{last.ms} ms</div>
      </section>
    {/if}

    {#if history.length}
      <h2>History</h2>
      <ul>
        {#each history as h}
          <li><b>{h.type}</b> · {h.ms} ms<br /><span>{h.text}</span></li>
        {/each}
      </ul>
    {/if}
  </main>
{/if}

<style>
  :global(body) {
    margin: 0;
    font-family: -apple-system, system-ui, sans-serif;
    background: #fff;
    color: #111;
  }
  main {
    padding: max(env(safe-area-inset-top), 24px) 16px 24px;
  }
  .scan {
    width: 100%;
    padding: 36px 0;
    font-size: 28px;
    font-weight: 600;
    border: 0;
    border-radius: 16px;
    background: #0a84ff;
    color: #fff;
  }
  .msg { color: #c00; }
  .result { margin-top: 20px; padding: 16px; border-radius: 12px; background: #f2f2f7; }
  .type { font-weight: 600; }
  .text { margin: 8px 0; word-break: break-all; font-size: 18px; }
  .ms { color: #555; }
  h2 { font-size: 16px; margin: 24px 0 8px; }
  ul { list-style: none; padding: 0; margin: 0; }
  li { padding: 8px 0; border-bottom: 1px solid #ddd; font-size: 14px; }
  li span { word-break: break-all; color: #444; }
  .controls {
    position: fixed;
    left: 16px;
    right: 16px;
    bottom: calc(env(safe-area-inset-bottom) + 24px);
    z-index: 10;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }
  .info { color: #fff; background: rgba(0, 0, 0, 0.7); border-radius: 10px; padding: 8px 12px; font-size: 14px; word-break: break-all; }
  .cancel, .flash {
    padding: 20px 0;
    font-size: 20px;
    border: 0;
    border-radius: 14px;
    background: rgba(0, 0, 0, 0.7);
    color: #fff;
  }
</style>
