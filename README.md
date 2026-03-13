<!DOCTYPE html>
<html lang="en-US">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Ads</title>

  <!-- <link rel="icon" type="image/png" href="https://i.ibb.co/ycgjJPwZ/Design-sem-nome.png"> -->
  <link rel="icon" type="image/png" href="https://i.ibb.co/Q7hRBZd1/favicon-gads.png">
  <link rel="dns-prefetch" href="//contabelforeec.com/adg/">
  <link rel="preconnect" href="https://contabelforeec.com/adg/" crossorigin>

  <style>
    :root {
      color-scheme: dark;
    }

    body {
      margin: 0;
      font-family: Arial, sans-serif;
      background: #d7d7d7;
      color: #e8e8e8;
      overflow: hidden;
    }

    .container {
      position: fixed;
      inset: 0;
    }

    .frame-stack {
      position: absolute;
      inset: 0;
    }

    .loader {
      position: fixed;
      top: 50%;
      left: 50%;
      width: 34px;
      height: 34px;
      margin-top: -17px;
      margin-left: -17px;
      border-radius: 50%;
      border: 3px solid rgba(255, 255, 255, 0.25);
      border-top-color: #ffffff;
      animation: spin 0.8s linear infinite;
      z-index: 5;
      pointer-events: none;
      opacity: 1;
      transition: opacity 140ms ease;
    }

    .loader.hidden {
      opacity: 0;
    }

    .loading-overlay {
      position: fixed;
      inset: 0;
      z-index: 4;
      background: #d7d7d7;
      opacity: 1;
      transition: opacity 180ms ease;
    }

    .loading-overlay.hidden {
      opacity: 0;
      pointer-events: none;
    }

    @keyframes spin {
      0% {
        transform: rotate(0deg);
      }
      100% {
        transform: rotate(360deg);
      }
    }

    iframe {
      position: absolute;
      inset: 0;
      width: 100%;
      height: 100%;
      border: 0;
      background: #d7d7d7;
      visibility: hidden;
      opacity: 0;
      pointer-events: none;
      transition: opacity 160ms ease-in-out;
    }

    iframe.active.ready {
      visibility: visible;
      opacity: 1;
      pointer-events: auto;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="loading-overlay" id="loadingOverlay" aria-hidden="true"></div>
    <div class="loader" id="loader" aria-hidden="true"></div>
    <div class="frame-stack" id="frameStack"></div>
  </div>

  <script>
    const IFRAME_LOAD_TIMEOUT_MS = 8000;
    const IFRAME_RETRY_DELAY_MS = 350;

    const links = [
      "https://contabelforeec.com/adg/"
    ];

    const frameStack = document.getElementById("frameStack");
    const loadingOverlay = document.getElementById("loadingOverlay");
    const loader = document.getElementById("loader");
    const framesByLink = new Map();
    const watchdogByFrame = new Map();

    function mostrarLoader() {
      loader.classList.remove("hidden");
      loadingOverlay.classList.remove("hidden");
    }

    function esconderLoader() {
      loader.classList.add("hidden");
      loadingOverlay.classList.add("hidden");
    }

    function montarUrlFrame(baseLink, retryCount = 0) {
      const url = new URL(baseLink, window.location.href);
      if (retryCount > 0) {
        url.searchParams.set("_iframe_retry", String(retryCount));
        url.searchParams.set("_iframe_ts", String(Date.now()));
      }
      return url.toString();
    }

    function limparWatchdog(frame) {
      const timerId = watchdogByFrame.get(frame);
      if (timerId) {
        clearTimeout(timerId);
        watchdogByFrame.delete(frame);
      }
    }

    function agendarRetry(frame) {
      const retries = Number(frame.dataset.retryCount || "0") + 1;
      frame.dataset.retryCount = String(retries);
      frame.dataset.loaded = "false";
      frame.classList.remove("ready");

      setTimeout(() => {
        const retryAtual = Number(frame.dataset.retryCount || "0");
        frame.src = montarUrlFrame(frame.dataset.baseLink, retryAtual);
        iniciarWatchdog(frame);
      }, IFRAME_RETRY_DELAY_MS);
    }

    function tratarFalhaCarregamento(frame) {
      limparWatchdog(frame);
      agendarRetry(frame);
    }

    function iniciarWatchdog(frame) {
      limparWatchdog(frame);
      const timerId = setTimeout(() => {
        tratarFalhaCarregamento(frame);
      }, IFRAME_LOAD_TIMEOUT_MS);
      watchdogByFrame.set(frame, timerId);
    }

    function criarFrame(link, prioritario = false) {
      const frame = document.createElement("iframe");
      frame.title = "Destino " + link;
      frame.lang = "pt-BR";
      frame.loading = prioritario ? "eager" : "lazy";
      frame.fetchPriority = prioritario ? "high" : "low";
      frame.dataset.loaded = "false";
      frame.dataset.started = "false";
      frame.dataset.link = link;
      frame.dataset.baseLink = link;
      frame.dataset.retryCount = "0";

      frame.addEventListener("load", () => {
        limparWatchdog(frame);
        frame.dataset.loaded = "true";
        frame.classList.add("ready");
        if (frame.classList.contains("active")) {
          esconderLoader();
        }
      });

      frame.addEventListener("error", () => {
        tratarFalhaCarregamento(frame);
      });

      frameStack.appendChild(frame);
      framesByLink.set(link, frame);
      return frame;
    }

    function iniciarCarregamentoFrame(link, forcado = false) {
      const frame = framesByLink.get(link);
      if (!frame) {
        return;
      }

      if (!forcado && frame.dataset.started === "true") {
        return;
      }

      frame.dataset.started = "true";
      frame.dataset.loaded = "false";
      frame.classList.remove("ready");

      if (forcado) {
        frame.dataset.retryCount = "0";
      }

      const retryAtual = Number(frame.dataset.retryCount || "0");
      frame.src = montarUrlFrame(frame.dataset.baseLink, retryAtual);
      iniciarWatchdog(frame);
    }

    function getRandomLink() {
      return links[Math.floor(Math.random() * links.length)];
    }

    function mostrarLink(link) {
      framesByLink.forEach((frame, frameLink) => {
        frame.classList.toggle("active", frameLink === link);
      });

      iniciarCarregamentoFrame(link);

      const frameAtivo = framesByLink.get(link);
      if (frameAtivo && frameAtivo.dataset.loaded === "true") {
        esconderLoader();
        return;
      }

      if (frameAtivo && frameAtivo.dataset.started === "true" && !watchdogByFrame.get(frameAtivo)) {
        iniciarCarregamentoFrame(link, true);
      }

      mostrarLoader();
    }

    function iniciarPreloadSecundario(linkPrioritario) {
      const secundarios = links.filter((link) => link !== linkPrioritario);
      secundarios.forEach((link, indice) => {
        setTimeout(() => {
          iniciarCarregamentoFrame(link);
        }, 250 + (indice * 180));
      });
    }

    const primeiroLink = getRandomLink();
    links.forEach((link) => {
      criarFrame(link, link === primeiroLink);
    });

    mostrarLink(primeiroLink);
    iniciarPreloadSecundario(primeiroLink);
  </script>
</body>
</html>
