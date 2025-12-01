
# CTFd Async Scraper (httpx + FastAPI)

Небольшой асинхронный парсер для CTFd-платформ, который по ссылке на список задач
или конкретную задачу:

- забирает список чанленджей через CTFd API `/api/v1/challenges` :contentReference[oaicite:0]{index=0}  
- для каждого ID подтягивает описание и файлы через `/api/v1/challenges/{id}`  
- сохраняет всё на диск в аккуратную структуру папок  
- генерирует `INDEX.md` с индексом задач  
- собирает ZIP-архив со всем дампом  
- отдаёт результат через простой веб-интерфейс на FastAPI.

Используются:

- **httpx.AsyncClient** для асинхронных HTTP-запросов и конкарренси :contentReference[oaicite:1]{index=1}  
- **FastAPI + Uvicorn** как лёгкий веб-фронтенд  
- **BeautifulSoup4** как HTML-фолбэк, если API недоступен.

> ⚠️ Инструмент предназначен для личного использования: бэкап своих CTF-площадок,
> оффлайн-разбор тасков и т.п. Убедись, что у тебя есть право скрейпить конкретный CTF.

---

## Возможности

- Работает с любым CTFd ≥2.x / 3.x, где доступен REST API `/api/v1/...`. :contentReference[oaicite:2]{index=2}  
- Поддерживаемые способы аутентификации:
  - **Access Token** (`Authorization: Token <токен>`) — стандартный способ для CTFd. :contentReference[oaicite:3]{index=3}  
  - **Логин + пароль** через HTML-форму `/login`.
  - **Сырые cookies** (например, `session=...; site_password=...` для закрытых по паролю инстансов). :contentReference[oaicite:4]{index=4}  
- Понимает URL-ы:
  - список: ``https://ctf.example.com/challenges``
  - конкретная задача: ``https://ctf.example.com/challenges#-<ID>``
- Для каждой задачи:
  - забирает **название, категорию, стоимость, описание, список файлов** из API;
  - при проблемах с API — фолбэк на HTML-страницу задачи.
- Структура сохранения:

  ```text
  <out_dir>/
    Web/
      Роботы/
        description.txt
        page.html        # если включено сохранение HTML
        files/
          robots.zip
    OSINT/
      Первый_CTF/
        description.txt
        files/
          statement.pdf


* Генерирует `INDEX.md` с колонками: `# / Category / Title / URL / Local path / Files`.
* Собирает ZIP-архив всего `<out_dir>` и даёт ссылку `/download?...` в веб-интерфейсе.
* Асинхронная закачка нескольких задач одновременно (ограничивается параметром `concurrency`).

---

## Структура проекта

```text
ctfd_scraper/
├─ scraper_core.py   # Вся логика: httpx, API, парсинг, сохранение, INDEX.md, ZIP
├─ web_app.py        # Веб-интерфейс (FastAPI) поверх ядра
├─ requirements.txt
└─ README.md
```

* **`scraper_core.py`** — чистое ядро без FastAPI. Можно вызывать из CLI, других скриптов и т.п.
* **`web_app.py`** — тонкий FastAPI-слой: HTML-форма + вызов `run_scrape(...)` из ядра.

---

## Требования

* Python 3.10+ (проект асинхронный, заточен под современный Python).
* UNIX-подобная ОС (Linux/macOS) — но в целом код кроссплатформенный.
* Доступ к нужному CTFd-инстансу (в том числе по VPN / туннелю, если нужно).

### `requirements.txt` (пример)

```txt
httpx>=0.27.0
beautifulsoup4>=4.12.0
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
```

Устанавливать лучше в виртуальное окружение.

---

## Установка

```bash
git clone <твой-репозиторий> ctfd_scraper
cd ctfd_scraper

python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

---

## Запуск веб-интерфейса

```bash
uvicorn web_app:app --reload --host 0.0.0.0 --port 8000
```

После этого зайди в браузере на:

```text
http://127.0.0.1:8000/
```

---

## Поля формы и режимы аутентификации

Веб-форма (`GET /`) содержит поля:

* **CTFd URL**

  * Например:

    * `https://ctf.bug-makers.ru/challenges` — скачать *все* задачи.
    * `https://ctf.bug-makers.ru/challenges#-23` — скачать только задачу с ID `23`.
* **Логин / Пароль** (опционально)

  * Используются для логина через `/login`, парсер сам ищет имена полей формы.
* **API Token** (опционально, строка вида `ctfd_...`)

  * Добавляется в заголовок:
    `Authorization: Token <твой_токен>` — так требует CTFd API.
  * Можно использовать **только токен**, без логина/пароля.
* **Cookie строка** (опционально)

  * Сырая строка, как в браузере:
    `session=<...>; site_password=<...>;`
  * Нужна, если CTFd защищён site-password или нужна какая-то особая сессия. 
* **Login URL** (опционально)

  * Если пусто — берётся `https://host/login` по хосту из CTFd-URL.
* **Каталог для сохранения**

  * По умолчанию: `./ctf_dump`.
* **Параллелизм** (`concurrency`)

  * Сколько задач качать одновременно.
  * 5–10 — нормальные значения.
* Флаги:

  * **Не скачивать файлы** — только описания и структура.
  * **Не сохранять описания** — только файлы.
  * **Сохранять HTML** — дополнительно сохраняет `page.html` для каждой задачи.

---

## Как всё работает внутри (коротко)

1. **Создаётся httpx.AsyncClient** с нужными заголовками и cookies.

   * В заголовки кладётся `Authorization: Token ...` при наличии токена. 
   * Для запросов к API принудительно ставится `Content-Type: application/json`, чтобы обойти баг, когда токены без этого заголовка могут не приниматься.
2. Если есть логин/пароль — выполняется вход на `/login` (парсер сам подставляет имя и пароль в форму).
3. Если введён URL списка (`.../challenges`):

   * Сначала вызывается `/api/v1/challenges` → берётся **полный список задач с ID**. 
   * Для каждого ID строится человекочитаемый URL вида:
     `https://host/challenges#-<id>`.
4. Для каждой задачи запускается `scrape_ctfd_challenge(...)`:

   * По ID вызывается `/api/v1/challenges/<id>` → **название, категория, стоимость, описание, файлы**.
   * Если API по какой-то причине не сработал — HTML-фолбэк через BeautifulSoup.
   * Задача складывается в папку: `<out_dir>/<Категория>/<Название>/`.
5. После обхода всех задач:

   * Генерируется `INDEX.md` в корне `<out_dir>`.
   * Собирается ZIP-архив `<out_dir>_YYYYmmdd_HHMMSS.zip`.
   * Веб-страница `/run` показывает таблицу задач и ссылку на скачивание ZIP (`/download?path=...`).

---

## Пример: запуск без веб-интерфейса (скриптом)

Если нужно вызвать ядро напрямую:

```python
# cli_example.py
import asyncio
from scraper_core import run_scrape

async def main():
    res = await run_scrape(
        base_urls=["https://ctf.bug-makers.ru/challenges"],
        api_token="ctfd_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",  # твой токен
        out_dir="./ctf_dump",
        concurrency=10,
        save_html=True,
    )
    print("Всего задач:", len(res["results"]))
    print("INDEX:", res["index_path"])
    print("ZIP:", res["zip_path"])

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Ограничения и заметки

* Парсер заточен под **чистый CTFd API**:

  * если инстанс сильно пропатчен, может потребоваться доработка.
* Если API `/api/v1/challenges` отключён или за нестандартной авторизацией, HTML-фолбэк может не увидеть все задачи.
* Не обходит **dynamic JS-рендеринг без API** (например, если организаторы вообще отключили API и отдают всё чистым JS).
* При большом количестве задач и файлов убедись, что:

  * диск не заполнится;
  * скорость сети достаточна;
  * не будешь DOS’ить чужой CTF (не ставь бездумно `concurrency=200`).

---
