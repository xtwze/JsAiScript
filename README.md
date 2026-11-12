# JsAiScript
(() => {
  const report = (window.__focusGuard = window.__focusGuard || { log: [] }).log;
  const log = m => { report.push(m); console.log('%c[focus-guard]', 'color:#b57200;font-weight:700', m); };

  // 1) Снять уже висящие слушатели. getEventListeners есть только в консоли DevTools —
  //    поэтому слой защищён try/catch и молча пропускается вне консоли.
  let stripped = 0;
  try {
    if (typeof getEventListeners === 'function')
      for (const [t, n] of [[window, 'window'], [document, 'document']])
        for (const ev of ['blur', 'focus', 'visibilitychange'])
          (getEventListeners(t)[ev] || []).forEach(l => { t.removeEventListener(ev, l.listener); stripped++; });
  } catch {}
  log(stripped ? `снято слушателей: ${stripped}` : 'getEventListeners недоступен — работают нижние слои');

  // 2) Нейтрализовать функции страницы. На живой странице они объявлены на верхнем
  //    уровне обычного <script>, значит висят на window и перезаписываются отсюда.
  for (const fn of ['onBlur', 'onFocus', 'sendFocusEvent']) {
    if (typeof window[fn] === 'function') { window[fn] = () => {}; log(`${fn}() нейтрализована`); }
  }

  // 3) Транспорт: блокируем ТОЛЬКО /api/focus (beacon + fetch), остальное проходит насквозь.
  const beacon = navigator.sendBeacon && navigator.sendBeacon.bind(navigator);
  if (beacon) navigator.sendBeacon = (url, ...rest) =>
    String(url).includes('/api/focus') ? (log(`sendBeacon→ ${url} заблокирован`), true) : beacon(url, ...rest);
  const realFetch = window.fetch;
  window.fetch = (input, init) => {
    const url = typeof input === 'string' ? input : (input && input.url) || '';
    return url.includes('/api/focus')
      ? (log(`fetch→ ${url} заблокирован`), Promise.resolve(new Response('{"ok":true}', { status: 200 })))
      : realFetch(input, init);
  };
  log('транспорт /api/focus перекрыт (beacon + fetch)');

  // 4) Новые регистрации античит-слушателей — мимо. Это твой исходный приём,
  //    исправленный: '===' вместо '=', и тогда условие реально фильтрует по типу.
  for (const [t, n] of [[window, 'window'], [document, 'document']]) {
    const orig = t.addEventListener.bind(t);
    t.addEventListener = (type, listener, options) => {
      if (type === 'blur' || type === 'focus' || type === 'visibilitychange') { log(`${n}.addEventListener(${type}) заблокирован`); return; }
      return orig(type, listener, options);
    };
  }

  // 5) Убрать визуальный эффект (красный экран), даже если класс кто-то успеет повесить.
  const style = document.createElement('style');
  style.textContent = 'body.focus-lost{background:#f4f6f9 !important}body.focus-lost::before{display:none !important}';
  document.documentElement.appendChild(style);
  if (document.body) document.body.classList.remove('focus-lost');

  log('готово: уводи фокус — на сервер ничего не уйдёт. Журнал: __focusGuard.log');
})();
