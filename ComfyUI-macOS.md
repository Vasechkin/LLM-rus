# ComfyUI на macOS с Apple Silicon

Практический обзор ComfyUI для Mac с чипами Apple Silicon (M1–M4 и новее): установка, модели, ретушь фотографий, workflow и сравнение с Ollama.

> Актуальность проверена по официальным материалам ComfyUI, PyTorch и Homebrew на 20 октября 2026 года. Команды рассчитаны на macOS 13+ и zsh.

## Что такое ComfyUI

ComfyUI — локальный графовый интерфейс и движок для генерации и редактирования изображений, видео и других медиа. Workflow строится из узлов и сохраняется в JSON, поэтому его можно повторить и передать через локальный API.

На Apple Silicon используется backend PyTorch MPS (Metal). Это не CUDA: скорость и совместимость custom nodes отличаются от Windows/Linux с NVIDIA.

## Установка

### Вариант A: Comfy Desktop

Скачайте актуальный установщик для macOS с официальной страницы ComfyUI, откройте .dmg, перетащите приложение в Applications и запустите. При предупреждении Gatekeeper разрешите запуск в System Settings → Privacy & Security.

Desktop удобен для первого запуска и управления несколькими установками. Для воспроизводимых окружений и отладки custom nodes используйте ручную установку.

### Вариант B: ручная установка

Проверьте архитектуру и инструменты разработчика:

```zsh
uname -m                 # ожидается arm64
sw_vers -productVersion
xcode-select --install   # если Command Line Tools ещё не установлены
```

Homebrew устанавливайте официальной командой. Не используйте `sudo brew install` и не добавляйте путь вручную до выполнения `shellenv`:

```zsh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
brew --version
brew install python@3.12 git
python3.12 --version
```

Клонируйте ComfyUI в отдельное окружение:

```zsh
mkdir -p ~/Applications
cd ~/Applications
git clone https://github.com/Comfy-Org/ComfyUI.git
cd ComfyUI
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install torch torchvision torchaudio
python -m pip install -r requirements.txt
```

Проверьте MPS:

```zsh
python - <<'PY'
import torch
print("PyTorch:", torch.__version__)
print("MPS built:", torch.backends.mps.is_built())
print("MPS available:", torch.backends.mps.is_available())
PY
```

Запустите сервер:

```zsh
cd ~/Applications/ComfyUI
source .venv/bin/activate
python main.py --listen 127.0.0.1 --port 8188
```

Откройте http://127.0.0.1:8188. Параметр `--listen 0.0.0.0` не нужен для локальной работы и расширяет поверхность доступа.

При нехватке памяти MPS можно проверить отдельный запуск:

```zsh
PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0 python main.py --listen 127.0.0.1 --port 8188
```

Это не увеличивает физическую память и может привести к swap. Сначала уменьшайте разрешение, batch size и размер модели.

## Модели и каталоги

Основные каталоги внутри `~/Applications/ComfyUI/models/`:

| Тип | Каталог |
|---|---|
| Checkpoint (SD 1.5/SDXL) | `checkpoints/` |
| VAE | `vae/` |
| LoRA | `loras/` |
| ControlNet | `controlnet/` |
| CLIP / text encoder | `clip/` |
| Upscaler | `upscale_models/` |
| IP-Adapter | `ipadapter/` |

Загружайте веса только из доверенных источников и сверяйте формат с описанием workflow. Для SDXL не подставляйте checkpoint SD 1.5 без изменения графа.

- **SD 1.5** — лёгкий старт, много LoRA и ControlNet.
- **SDXL** — универсальное качество, но больше расход памяти.
- **SDXL Inpainting** — предпочтительный вариант для локального исправления областей.
- **FLUX.1** — высокое качество, но тяжёлый вариант для Mac с 16 ГБ.
- **LoRA** — адаптеры стиля, персонажа и детализации; должны соответствовать базовой модели.
- **ControlNet** — контроль позы, контуров, глубины и структуры.
- **IP-Adapter** — перенос визуальных признаков референса, но не гарантия идентичности.

На Apple Silicon учитывайте память для весов, text encoder, latent-тензоров и промежуточных узлов. Начинайте с SD 1.5/SDXL, разрешения 512–768 px и batch size 1.

## Ретушь фотографий

ComfyUI подходит для локальной правки, но не является автоматическим фоторедактором. Качество определяют маска, denoise и проверка результата в масштабе 100%.

### Универсальный inpainting workflow

`Load Checkpoint → Load Image → Mask Editor → VAE Encode (for Inpainting) → CLIP Text Encode → KSampler → VAE Decode → Save Image`

Стартовые параметры:

- складки на ткани: denoise 0.25–0.40;
- трещины на поверхности: denoise 0.35–0.55;
- блики: denoise 0.20–0.35;
- steps 20–35, CFG 4–7, batch size 1.

Маска должна покрывать дефект с небольшим запасом, но не захватывать здоровые детали. Для positive prompt описывайте материал и свет: `natural fabric texture, same lighting, realistic fine detail`. Для артефактов можно добавить negative prompt: `plastic, smooth skin, repeating pattern, blur, artifacts`.

### Складки и трещины

Делайте отдельную маску на каждый дефект. Для сохранения формы добавляйте ControlNet Tile/Lineart, но учитывайте дополнительный расход памяти. Для лица используйте маленькие маски и низкий denoise: иначе могут измениться черты, возраст и текстура.

### Блики и пересветы

Inpainting не восстановит достоверную информацию из полностью выбеленного участка. Если в блике сохранились цвет и контур, маска с малым denoise часто даёт естественный результат. При клиппинге используйте RAW/оригинал или ручную коррекцию; генеративный результат может быть правдоподобным, но не достоверным.

Для портретов можно добавить reference/IP-Adapter, но проверяйте лицензию модели. Сохраняйте оригинал и помечайте изменённые версии.

## Примеры workflow

### Img2img

`Load Image → VAE Encode → KSampler → VAE Decode → Save Image`

Для мягкой детализации используйте SDXL, denoise 0.15–0.35 и batch size 1. Это не заменяет точечный inpainting.

### Контроль структуры

`Load Image → ControlNet (canny/lineart/depth/tile) → KSampler → VAE Decode`

Уменьшайте strength, если результат становится «перерисованным». На MPS такой граф может потреблять заметно больше памяти.

Workflow можно импортировать перетаскиванием JSON или PNG с метаданными. Если узел отсутствует, проверьте README custom node, установите зависимости в то же `.venv` и перезапустите ComfyUI.

## Custom nodes и обновления

ComfyUI Manager упрощает поиск узлов, но сторонний код запускается локально с правами пользователя. Устанавливайте известные репозитории, фиксируйте версии и делайте резервную копию окружения.

Не копируйте старые инструкции вслепую: они могут требовать nightly PyTorch, `--use-mps` или неподходящую версию Python. Для базового запуска достаточно стандартного PyTorch с MPS и `python main.py`; специальные флаги применяйте только по инструкции конкретного узла или модели.

## Ограничения Apple Silicon

- MPS обычно медленнее современной NVIDIA CUDA-карты в тяжёлых diffusion workflow.
- Унифицированная память общая для macOS и ComfyUI; следите за swap.
- Не все custom nodes тестируются на macOS/MPS; CUDA-зависимые узлы могут не работать.
- Высокие разрешения, batch size, FLUX и многомодельные графы быстро увеличивают расход памяти.
- Ретушь может изменить мелкие детали, лицо, текст, логотипы и фактуру. Для документов и архивной реставрации сохраняйте оригинал.
- Workflow и custom nodes могут обращаться к внешним API. Проверяйте сеть и настройки приватности.

## ComfyUI и Ollama

| Задача | ComfyUI | Ollama |
|---|---|---|
| Основной тип | Графовый движок для изображений и медиа | Runtime для языковых и vision-моделей |
| Интерфейс | Узлы, workflow, маски, sampler, ControlNet | CLI/API и чат-интерфейсы |
| Ретушь фото | Inpainting, img2img, LoRA, ControlNet | Не основной сценарий; может описывать задачу или вызывать внешний pipeline |
| Модели | Checkpoint, VAE, LoRA, ControlNet, IP-Adapter | GGUF/совместимые LLM и vision-модели |
| Лучшее применение | Точный визуальный pipeline и пакетная обработка | Текст, код, OCR/описание и оркестрация |

Их можно использовать вместе: Ollama принимает русское описание задачи и помогает составить prompt, а ComfyUI выполняет воспроизводимую визуальную обработку. Ollama не заменяет ComfyUI для масок, sampler-ов и ControlNet.

## Диагностика

- `command not found: brew` — выполните `eval "$(/opt/homebrew/bin/brew shellenv)"` и добавьте строку в `~/.zprofile`.
- `No module named torch` — активируйте `.venv` и используйте `python -m pip`.
- MPS недоступен — проверьте `uname -m`, macOS и `torch.backends.mps.is_available()`.
- Модель не найдена — проверьте каталог, расширение и совместимость checkpoint.
- Падение по памяти — уменьшите разрешение/batch size, закройте приложения и попробуйте MPS watermark override.
- Не найден custom node — проверьте зависимости, логи и совместимость с macOS/MPS.

## Официальные ссылки

- [ComfyUI: репозиторий и ручная установка](https://github.com/Comfy-Org/ComfyUI)
- [ComfyUI: системные требования](https://docs.comfy.org/installation/system_requirements)
- [ComfyUI Desktop для macOS](https://docs.comfy.org/installation/desktop/macos)
- [Homebrew: официальная установка](https://brew.sh/)
- [PyTorch: MPS backend](https://pytorch.org/docs/stable/notes/mps.html)
- [Ollama: документация](https://docs.ollama.com/)

