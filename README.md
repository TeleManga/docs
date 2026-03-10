# Документация TeleManga

Для генерации статики используется [zensical](https://zensical.org/)

## Структура репозитория

- `docs` - корневая папка документации, в которую смотрит `zensical`
- `zensical.toml` - файл с конфигурацией `zensical`

## Локальный запуск

При необходимости локального изменения модификации, достаточно выполнить следующие команды

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
zensical serve
```
