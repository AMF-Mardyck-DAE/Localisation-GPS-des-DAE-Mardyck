<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Itinéraire vers le site le plus proche</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; margin: 0; padding: 2rem; background: #f7f7f9; color: #222; }
    main { max-width: 640px; margin: auto; background: #fff; padding: 1.5rem; border-radius: 12px; box-shadow: 0 8px 24px rgba(0,0,0,.06); }
    h1 { margin-top: 0; font-size: 1.4rem; }
    #status { color: #444; }
    ul { padding-left: 1.2rem; }
    li { margin: .4rem 0; }
    a { color: #0b57d0; text-decoration: none; }
    a:hover { text-decoration: underline; }
    .spinner { width: 22px; height: 22px; border: 3px solid #ddd; border-top-color: #0b57d0; border-radius: 50%; animation: spin 1s linear infinite; display:inline-block; vertical-align: -4px; margin-right: .5rem; }
    @keyframes spin { to { transform: rotate(360deg); } }
    footer { margin-top: 1rem; font-size: .85rem; color: #666; }
    .note { background:#f0f4ff; padding: .75rem 1rem; border-radius: 8px; border:1px solid #d9e2ff; }
  </style>
</head>
<body>
  <main id="app">
    <h1>On vous guide vers le site le plus proche…</h1>
    <p class="note">Meilleure expérience en HTTPS (GitHub Pages / Netlify). Si la géolocalisation est bloquée, une liste de liens directs s'affichera.</p>
    <p id="status"><span class="spinner"></span>Veuillez autoriser la géolocalisation.</p>
    <noscript>Activez JavaScript pour continuer.</noscript>
  </main>

  <script>
    // === Vos 6 sites ===
    const LOCATIONS = [
      { name: "DAE Harsco", lat: 50.98065, lng: 2.283231 },
      { name: "DAE Managers postés Couplage", lat: 50.989304, lng: 2.291375 },
      { name: "DAE Local CEDICO", lat: 50.988514, lng: 2.286402 },
      { name: "DAE Tour de contrôle Logistique", lat: 50.987322, lng: 2.280459 },
      { name: "DAE RAC", lat: 50.989142, lng: 2.283731 },
      { name: "DAE BCM", lat: 50.987463, lng: 2.286041 }
    ];

    // === Paramètres de navigation ===
    const TRAVEL_MODE = 'walking';    // 'driving' | 'walking' | 'transit' | 'bicycling'
    const PREFERRED_APP = 'google';   // 'auto' | 'google' | 'apple' | 'waze'
    const REDIRECT_DELAY_MS = 800;    // délai avant l’ouverture de l’app

    // === Calcul distance (Haversine) ===
    const toRad = d => d * Math.PI / 180;
    function haversine(lat1, lon1, lat2, lon2) {
      const R = 6371; // km
      const dLat = toRad(lat2 - lat1);
      const dLon = toRad(lon2 - lon1);
      const a = Math.sin(dLat/2)**2 + Math.cos(toRad(lat1))*Math.cos(toRad(lat2))*Math.sin(dLon/2)**2;
      return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    }
    function nearest(from, places) {
      let best = null, bestD = Infinity;
      for (const p of places) {
        const d = haversine(from.lat, from.lng, p.lat, p.lng);
        if (d < bestD) { bestD = d; best = { ...p, distance_km: d }; }
      }
      return best;
    }

    // === Détection app (honore PREFERRED_APP) ===
    function detectMapApp() {
      if (PREFERRED_APP === 'google' || PREFERRED_APP === 'apple' || PREFERRED_APP === 'waze') return PREFERRED_APP;
      const ua = navigator.userAgent || '';
      const isiOS = /iPhone|iPad|iPod/i.test(ua);
      return isiOS ? 'apple' : 'google';
    }

    // === Liens profonds ===
    function buildDeepLink(app, dest, travelMode='driving') {
      const name = encodeURIComponent(dest.name);
      if (app === 'waze') {
        return `https://waze.com/ul?ll=${dest.lat},${dest.lng}&navigate=yes`;
      }
      if (app === 'apple') {
        const modeFlag = travelMode === 'walking' ? 'w' : (travelMode === 'transit' ? 'r' : 'd');
        return `https://maps.apple.com/?daddr=${dest.lat},${dest.lng}&q=${name}&dirflg=${modeFlag}`;
      }
      // Google Maps par défaut
      const valid = ['driving','walking','transit','bicycling'];
      const mode = valid.includes(travelMode) ? travelMode : 'driving';
      return `https://www.google.com/maps/dir/?api=1&destination=${dest.lat},${dest.lng}&travelmode=${mode}`;
    }

    // === Fallback si géolocalisation refusée/échoue ===
    function showFallback(message, fromPos=null) {
      const app = document.getElementById('app');
      app.innerHTML = `<h1>Choisissez votre destination</h1><p>${message}</p>`;
      const ul = document.createElement('ul');
      const chosenApp = detectMapApp();
      LOCATIONS.forEach(loc => {
        let extra = '';
        if (fromPos) extra = ` — ${haversine(fromPos.lat, fromPos.lng, loc.lat, loc.lng).toFixed(1)} km`;
        const li = document.createElement('li');
        const a = document.createElement('a');
        a.href = buildDeepLink(chosenApp, loc, TRAVEL_MODE);
        a.textContent = `${loc.name}${extra}`;
        a.rel = "noopener";
        li.appendChild(a);
        ul.appendChild(li);
      });
      app.appendChild(ul);
    }

    // === Démarrage ===
    (function run(){
      if (!('geolocation' in navigator)) {
        showFallback("Votre navigateur ne prend pas en charge la géolocalisation.");
        return;
      }
      const status = document.getElementById('status');
      status.textContent = "Localisation en cours…";
      navigator.geolocation.getCurrentPosition(pos => {
        const from = { lat: pos.coords.latitude, lng: pos.coords.longitude };
        const dest = nearest(from, LOCATIONS);
        if (!dest) { showFallback("Aucune destination configurée."); return; }
        const url = buildDeepLink(detectMapApp(), dest, TRAVEL_MODE);
        status.innerHTML = `Destination la plus proche : <strong>${dest.name}</strong> (${dest.distance_km.toFixed(1)} km). Ouverture de Google Maps…`;
        setTimeout(() => { window.location.href = url; }, REDIRECT_DELAY_MS);
      }, err => {
        console.warn(err);
        const reason = err.code === 1 ? "Permission refusée." : "Impossible d'obtenir la position.";
        showFallback(`${reason} Vous pouvez tout de même ouvrir l'itinéraire :`);
      }, { enableHighAccuracy: true, timeout: 8000, maximumAge: 30000 });
    })();
  </script>

  <footer>
    Confidentialité : votre position ne quitte jamais votre appareil. Aucune donnée n’est stockée.
  </footer>
</body>
</html>
``
