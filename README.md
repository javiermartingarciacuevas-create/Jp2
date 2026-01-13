# Jp2
Con mucho cariño para mi gente del colegioarenales de arroyomolinos
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>jp2 — Obtener IP</title>
  <style>
    :root{--bg:#fcfdff;--card:#fff;--muted:#666;--accent:#0b74de}
    body{font-family:system-ui,Arial; background:var(--bg); margin:0; padding:1.25rem; color:#111}
    .wrap{max-width:820px;margin:0 auto}
    header{display:flex;gap:1rem;align-items:center}
    .owner{background:var(--accent);color:#fff;padding:.5rem .75rem;border-radius:8px;font-weight:700}
    .card{background:var(--card);padding:1rem;border-radius:10px;box-shadow:0 6px 18px rgba(10,20,40,.06);margin-top:1rem}
    h1{margin:.25rem 0 0}
    button{background:var(--accent);color:#fff;border:0;padding:.6rem .9rem;border-radius:8px;font-weight:600}
    pre{background:#f5f7fb;padding:1rem;border-radius:8px;overflow:auto}
    .warn{color:#9a2b2b;font-weight:600}
    .small{color:var(--muted);font-size:.95rem}
    .row{display:flex;gap:.75rem;align-items:center;flex-wrap:wrap}
    .kv{display:flex;flex-direction:column}
    @media (max-width:520px){ header{flex-direction:column;align-items:flex-start} }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div>
        <div class="owner">javiermartingaciacuevas@colegioarenales.com</div>
        <h1>Comprobar IP — jp2</h1>
        <div class="small">Página estática (sirve desde GitHub Pages o cualquier hosting HTTPS).</div>
      </div>
    </header>

    <div class="card" id="publicCard">
      <h2>IP pública</h2>
      <div class="row">
        <div class="kv">
          <div id="public-ip" style="font-size:1.25rem">—</div>
          <div class="small">IP pública que ve internet</div>
        </div>
        <div>
          <button id="refresh-public">Obtener/Actualizar</button>
        </div>
      </div>
      <div class="small" style="margin-top:.5rem">
        Usamos servicios públicos (CORS + HTTPS). Esto suele funcionar en iPad aun cuando el dispositivo esté gestionado,
        siempre que la red del colegio no bloquee las peticiones externas.
      </div>
    </div>

    <div class="card" id="localCard">
      <h2>Intento de IP local (WebRTC)</h2>
      <div id="local-ip" style="font-size:1.05rem">—</div>
      <div class="small" style="margin-top:.5rem">
        Nota: navegadores modernos (especialmente Safari en iOS/iPadOS) ocultan o reemplazan la IP local por razones de privacidad.
        En muchos iPad gestionados este método no devolverá la IP local.
      </div>
    </div>

    <div class="card">
      <h2>Dirección MAC</h2>
      <div id="mac" class="warn">No es posible obtener la dirección MAC desde una página web (por diseño de seguridad y privacidad).</div>
      <div class="small" style="margin-top:.5rem">
        Opciones para obtener la MAC: ejecución local por el usuario, app nativa, o herramientas del administrador de red (MDM).
      </div>
    </div>

    <div class="card">
      <h2>Consola</h2>
      <pre id="log">Listo. Pulsa "Obtener/Actualizar" para comprobar la IP pública.</pre>
    </div>

    <footer class="small" style="margin-top:.75rem">
      Si la red del colegio bloquea los servicios externos, las peticiones pueden fallar. Para pruebas rápidas puedes usar datos móviles.
    </footer>
  </div>

<script>
const logEl = document.getElementById('log');
function log(...a){ logEl.textContent += a.join(' ') + '\\n'; }

const services = [
  {name: 'ipify', url: 'https://api.ipify.org?format=json', parse: r => r.ip},
  {name: 'ifconfig.co', url: 'https://ifconfig.co/json', parse: r => r.ip},
];

async function fetchPublicIP() {
  document.getElementById('public-ip').textContent = 'Obteniendo...';
  for (const s of services) {
    try {
      log('Probando servicio:', s.name);
      const res = await fetch(s.url, {cache: 'no-store'});
      if (!res.ok) throw new Error('HTTP ' + res.status);
      const data = await res.json();
      const ip = s.parse(data);
      if (ip) {
        document.getElementById('public-ip').textContent = ip;
        log('IP pública (via ' + s.name + '):', ip);
        return ip;
      }
    } catch (err) {
      log('Fallo servicio', s.name + ':', err.message || err);
    }
  }
  document.getElementById('public-ip').textContent = 'No disponible (posible bloqueo)';
  log('No se pudo obtener IP pública.');
  return null;
}

document.getElementById('refresh-public').addEventListener('click', fetchPublicIP);

function getLocalIPs(timeout = 2500) {
  return new Promise((resolve, reject) => {
    const ips = new Set();
    const RTCPeerConnection = window.RTCPeerConnection || window.webkitRTCPeerConnection || window.mozRTCPeerConnection;
    if (!RTCPeerConnection) return reject(new Error('WebRTC no disponible'));

    const pc = new RTCPeerConnection({iceServers: []});
    try { pc.createDataChannel(''); } catch(e){}
    pc.onicecandidate = ev => {
      if (!ev || !ev.candidate) return;
      const parts = ev.candidate.candidate.split(' ');
      for (let p of parts) {
        if (/^(?:[0-9]{1,3}\\.){3}[0-9]{1,3}$/.test(p) || p.includes('.local')) {
          ips.add(p);
        }
      }
    };

    pc.createOffer()
      .then(offer => pc.setLocalDescription(offer))
      .catch(err => { try{ pc.close(); }catch(e){}; reject(err); });

    setTimeout(() => { try{ pc.close(); }catch(e){}; resolve(Array.from(ips)); }, timeout);
  });
}

(async function tryLocal() {
  const el = document.getElementById('local-ip');
  el.textContent = 'Intentando...';
  try {
    const ips = await getLocalIPs();.
    if (!ips || ips.length === 0) {
      el.textContent = 'No detectado o bloqueado (común en Safari iOS / iPadOS).';
      log('WebRTC: sin IP local.');
    } else {
      el.textContent = ips.join(', ');
      log('IPs locales detectadas:', ips.join(', '));
    }
  } catch (err) {
    el.textContent = 'Error/No disponible: ' + (err && err.message ? err.message : err);
    log('Error en WebRTC:', err);
  }
})();

fetchPublicIP();
</script>
</body>
</html>
# jp2 — Obtener IP

