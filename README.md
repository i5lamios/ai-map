# AI Map — демо-сайт «карта ИИ-инструментов компании»

Интерактивная схема: какие ИИ-клиенты, шлюзы и коннекторы (MCP-серверы, Model Context Protocol) есть в компании и как их подключить.
Компания **Nordline Bank**, все люди, сервисы, модели и адреса вымышленные (`*.example`).

## Возможности
- SVG-карта: регионы (Clients, Gateways, Connectors, Sources), узлы и анимированные связи, отдельная раскладка для мобильных.
- Карточка узла по клику: описание, «кто может пользоваться», права, MCP-адрес, команды подключения, кнопки «Copy».
- Адрес страницы указывает на открытую карточку (`/#/n/<id>`), ссылкой можно поделиться.
- Тур по карте при первом визите; кнопка «Pause motion» (учитывает `prefers-reduced-motion`).
- Поиск (Cmd+K) целиком в браузере по `searchindex.json`: стемминг, нечёткий поиск, без сервера.

## Запуск
```bash
cd ai-map-demo
python3 -m http.server 8080      # или: npx serve .
# открыть http://localhost:8080
```
Нужен именно HTTP-сервер, двойной клик по index.html не подойдёт: пути абсолютные (`/css/...`), а поиск загружает JSON.

Через Docker:
```bash
docker run --rm -p 8080:80 -v "$PWD":/usr/share/nginx/html:ro nginx:alpine
```

## Из чего сделано
Это **готовая сборка** (build output), а не исходники. Внутри:

| Файл | Что это |
|---|---|
| `index.html` | вся карта: SVG-узлы с координатами + скрытые карточки `<section class=panel data-panel=<id>>` |
| `search/index.html` | страница полного поиска |
| `searchindex.json` | индекс поиска: `docs[]` с `href`, `title`, `description`, `keywords`, `section` |
| `js/map.js` | логика карты: массив связей `{flow, from, to}`, роутинг `#/n/<id>`, тур, анимация, копирование |
| `js/search.js`, `js/search-worker.js` | клиентский поиск (стеммер Snowball, нечёткий поиск) |
| `css/main.css` | базовые стили (Tailwind CSS v4) |
| `css/map.min.css` | стили карты, анимация, иконки узлов (`.i-<name>` → `icons/<name>.svg`) |
| `fonts/`, `icons/` | шрифты Geist/Inter, иконки |

Оригинал собран генератором статических сайтов (по признакам Hugo) + Tailwind + собственный JS карты. Готовых библиотек «карта MCP» нет: шаблон и `map.js` — своя разработка, библиотеки только у стилей (Tailwind) и поиска.

## Как править данные в этой сборке (быстро, вручную)
1. **Текст карточки:** найти в `index.html` `data-panel=<id>` и править HTML.
2. **Подпись узла:** найти `data-node="<id>"` → `<span class=node__label>`.
3. **Новый узел:** скопировать блок `<g class="node ..." data-node=...>`, поменять `id` и `transform="translate(x y)"`, добавить свою карточку `data-panel`.
4. **Связь:** добавить `{flow:"...",from:"a",id:"eNNN",to:"b"}` в начало `js/map.js`; линию `<path>` в SVG тоже придётся нарисовать.
5. **Поиск:** добавить объект в `docs` в `searchindex.json` и обновить `docCount`.

Ручная правка подходит для мелочей. Для своей карты лучше пересобрать проект из исходников (ниже).

## Как сделать нормальный проект с исходниками
Рекомендуемая структура: данные отдельно, вёрстка генерируется.

```
ai-map/
  data/nodes.yaml      # id, title, kind (client|gateway|connector|source), status (works|dev),
                       # region, description, owner, access, mcp_url, docs_url, icon
  data/edges.yaml      # from, to, flow
  data/layout.yaml     # координаты узлов (или авто-раскладка через ELK.js/dagre)
  src/                 # шаблоны + map.js
  public/icons/
```
Варианты стека:
- **Hugo** (как оригинал): `data/*.yaml` → шаблоны → статический HTML, поиск через Pagefind. Без JS-фреймворков, деплой — любая статика.
- **Vite + TypeScript** (или Astro): данные в JSON, SVG рисует JS, поиск — MiniSearch/Fuse.js.

### Готовый промпт для Claude Code / другого агента
> Сделай статический сайт «AI Map» по образцу из папки `ai-map-demo` (это собранный пример, смотри index.html, js/map.js, css/map.min.css). Стек: Hugo + Tailwind v4, поиск Pagefind. Все данные вынеси в `data/nodes.yaml`, `data/edges.yaml`, `data/layout.yaml`. Сохрани: SVG-карту с регионами и анимированными связями, мобильную раскладку, карточку по клику с deep link `#/n/<id>`, тур, паузу анимации, копирование MCP-адресов и команд, поиск по Cmd+K. Добавление узла = одна запись в YAML, без правки шаблонов. Добавь `Makefile` (dev, build) и Dockerfile на nginx.

## «Бэкенд»: что можно добавить
Сайту сервер не нужен, всё статично. Бэкенд нужен только для живых данных:

| Задача | Как сделать |
|---|---|
| Доступ только сотрудникам | Перед сайтом поставить reverse proxy с SSO: `oauth2-proxy` + nginx/Traefik (OIDC: Keycloak, Entra ID, Okta) |
| Живой статус узлов (Works / Down) | Небольшой сервис (FastAPI/Express) раз в минуту проверяет MCP-адреса (`initialize` / HTTP 200) и отдаёт `GET /api/status` → `map.js` раз в 30–60 с подкрашивает узлы |
| Каталог из реестра, а не из YAML | Сервис отдаёт `GET /api/nodes` из БД/реестра MCP-шлюза; фронт строит карту из JSON, а не из HTML |
| Персональные права | Тот же API по токену SSO отдаёт, к каким коннекторам у пользователя есть доступ, карточка показывает «доступно / запросить» |
| Автообновление | CI (GitLab CI / GitHub Actions): изменение YAML → build → деплой статики |

Минимальный пример статус-API (Python):
```python
# pip install fastapi uvicorn httpx
import asyncio, httpx, yaml
from fastapi import FastAPI
app, status = FastAPI(), {}
NODES = yaml.safe_load(open("data/nodes.yaml"))

async def poll():
    async with httpx.AsyncClient(timeout=5) as c:
        while True:
            for n in NODES:
                if url := n.get("mcp_url"):
                    try: status[n["id"]] = "works" if (await c.get(url)).status_code < 500 else "down"
                    except Exception: status[n["id"]] = "down"
            await asyncio.sleep(60)

@app.on_event("startup")
async def start(): asyncio.create_task(poll())

@app.get("/api/status")
def get_status(): return status
```
Во фронте: `fetch('/api/status')` → для каждого `id` поменять класс `node--works` / `node--down` у `[data-node=id]`.

## Деплой
Любой хостинг статики: nginx, S3 + CloudFront, GitLab/GitHub Pages, Cloudflare Pages. Для внутреннего сайта: nginx + oauth2-proxy в Docker Compose. Закрыть от индексации: в страницах уже стоит `noindex, nofollow`.
