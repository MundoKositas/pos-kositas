// Service worker del POS Kositas (Ronda 59) — permite ABRIR el POS aunque no haya internet.
// Va en la MISMA carpeta que pos.html. Qué hace:
//   - pos.html: «primero la red, si falla el caché» → con internet siempre se usa la versión más
//     nueva (y se guarda copia); sin internet se abre la última copia guardada.
//   - Librerías externas (Supabase, Excel, PDF): «primero el caché» → se descargan una vez.
//   - Logotipo público en Supabase Storage: «caché y se renueva en segundo plano».
//   - TODO lo demás (consultas a la base de datos, login, Realtime…) NO se toca: va directo a la red.
// Para forzar que todos los equipos tomen un sw.js nuevo, sube el número de VERSION.
const VERSION = 'mk-pos-v1';
const CACHE = VERSION;
const LIBS = [
  'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2',
  'https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js',
  'https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js',
  'https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.8.2/jspdf.plugin.autotable.min.js',
];
const SUPABASE_ORIGIN = 'https://vsfzkktmdwqkfbecmfcj.supabase.co';
const PAGINA = new URL('pos.html', self.registration.scope).href;

self.addEventListener('install', (event)=>{
  event.waitUntil((async ()=>{
    const cache = await caches.open(CACHE);
    // allSettled: si una librería no se puede bajar ahora, no se aborta toda la instalación.
    await Promise.allSettled([
      ...LIBS.map(u => cache.add(new Request(u, { mode: 'cors' }))),
      cache.add(new Request(PAGINA, { cache: 'reload' })),
    ]);
    self.skipWaiting();
  })());
});

self.addEventListener('activate', (event)=>{
  event.waitUntil((async ()=>{
    const nombres = await caches.keys();
    await Promise.all(nombres.filter(n => n !== CACHE).map(n => caches.delete(n)));
    await self.clients.claim();
  })());
});

async function primeroRed(request, claveCache, esperaMs){
  const cache = await caches.open(CACHE);
  try{
    const ctrl = new AbortController();
    const t = setTimeout(()=> ctrl.abort(), esperaMs);
    const resp = await fetch(request, { signal: ctrl.signal });
    clearTimeout(t);
    if(resp && resp.ok){ cache.put(claveCache, resp.clone()); return resp; }
    const guardada = await cache.match(claveCache);
    return guardada || resp;
  }catch(e){
    const guardada = await cache.match(claveCache);
    if(guardada) return guardada;
    throw e;
  }
}
async function primeroCache(request){
  const cache = await caches.open(CACHE);
  const guardada = await cache.match(request.url);
  if(guardada) return guardada;
  const resp = await fetch(request);
  if(resp && (resp.ok || resp.type === 'opaque')) cache.put(request.url, resp.clone());
  return resp;
}
async function cacheYRenueva(request){
  const cache = await caches.open(CACHE);
  const guardada = await cache.match(request.url);
  const red = fetch(request).then(resp=>{ if(resp && resp.ok) cache.put(request.url, resp.clone()); return resp; }).catch(()=> null);
  return guardada || (await red) || Response.error();
}

self.addEventListener('fetch', (event)=>{
  const req = event.request;
  if(req.method !== 'GET') return;
  const url = new URL(req.url);

  // La página del POS (con o sin parámetros en la dirección)
  if(req.mode === 'navigate' && url.origin === self.location.origin && url.pathname === new URL(PAGINA).pathname){
    event.respondWith(primeroRed(req, PAGINA, 4000));
    return;
  }
  if(LIBS.includes(req.url)){
    event.respondWith(primeroCache(req));
    return;
  }
  if(url.origin === SUPABASE_ORIGIN && url.pathname.startsWith('/storage/v1/object/public/')){
    event.respondWith(cacheYRenueva(req));
    return;
  }
  // todo lo demás: sin intervención
});
