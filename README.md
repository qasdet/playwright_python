# Playwright Python Automation Framework

## Обзор проекта

Фреймворк автоматизации тестирования веб-приложений с использованием Playwright и pytest.

## Структура проекта

```
playwright_python/
├── pages/                      # Классы Page Objects
│   ├── base_page.py            # Базовый класс для всех страниц
│   ├── playwright_home_page.py # Объект главной страницы
│   └── playwright_languages_page.py
├── page_factory/               # Переиспользуемые классы UI-компонентов
│   ├── component.py           # Абстрактный базовый компонент
│   ├── button.py               # Компонент кнопки
│   ├── input.py               # Компонент поля ввода
│   ├── link.py                # Компонент ссылки
│   ├── title.py               # Компонент заголовка
│   └── list_item.py           # Компонент элемента списка
├── components/                 # Сложные UI-компоненты
│   ├── navigation/
│   │   └── navbar.py          # Компонент навигационной панели
│   └── modals/
│       └── search_modal.py    # Компонент модального окна поиска
├── tests/                      # Тестовые файлы
│   ├── conftest.py            # pytest фикстуры
│   └── test_search.py         # Тесты функциональности поиска
├── settings.py                 # Конфигурация (BASE_URL)
└── requirements.txt           # Зависимости
```

## Зависимости

```
allure-pytest==2.12.0
pytest==7.2.0
playwright==1.28.0
pytest-playwright==0.3.0
```

## Архитектура и паттерны

### 1. Page Object Model (POM)

Страницы представлены в виде Python-классов, оборачивающих объект Playwright `Page`.

**BasePage** (`pages/base_page.py`):
```python
class BasePage:
    def __init__(self, page: Page) -> None:
        self.page = page
        self.navbar = Navbar(page)

    def visit(self, url: str) -> Response | None:
        return self.page.goto(url, wait_until='networkidle')

    def reload(self) -> Response | None:
        return self.page.reload(wait_until='domcontentloaded')
```

**PlaywrightHomePage** наследуется от BasePage:
```python
class PlaywrightHomePage(BasePage):
    def __init__(self, page: Page) -> None:
        super().__init__(page)
```

### 2. Паттерн Component Factory

Абстрактный базовый класс `Component` с специализированными подклассами для различных типов UI-элементов.

**Component** (`page_factory/component.py`):
```python
class Component(ABC):
    def __init__(self, page: Page, locator: str, name: str) -> None:
        self.page = page
        self.name = name
        self.locator = locator

    def get_locator(self, **kwargs) -> Locator:
        locator = self.locator.format(**kwargs)
        return self.page.locator(locator)

    def click(self, **kwargs) -> None:
        locator = self.get_locator(**kwargs)
        locator.click()

    def should_be_visible(self, **kwargs) -> None:
        locator = self.get_locator(**kwargs)
        expect(locator).to_be_visible()

    def should_have_text(self, text: str, **kwargs) -> None:
        locator = self.get_locator(**kwargs)
        expect(locator).to_have_text(text)
```

**Специализированные компоненты** (наследуются от Component):
- `Button` - добавляет методы `hover()`, `double_click()`
- `Input` - добавляет методы `fill()`, `should_have_value()`
- `Link` - без дополнительных методов (наследует базовые)
- `Title` - без дополнительных методов (наследует базовые)
- `ListItem` - без дополнительных методов (наследует базовые)

Каждый специализированный компонент определяет свойство `type_of`:
```python
class Button(Component):
    @property
    def type_of(self) -> str:
        return 'button'
```

### 3. Паттерн Facade

Сложные компоненты агрегируют более простые. **Navbar** содержит `search_modal`, `api_link`, `docs_link`, `search_button`:
```python
class Navbar:
    def __init__(self, page: Page) -> None:
        self.page = page
        self.search_modal = SearchModal(page)
        self.api_link = Link(page, locator="//a[text()='API']", name='API')
        self.docs_link = Link(page, locator="//a[text()='Docs']", name='Docs')
        self.search_button = Button(page, locator="button.DocSearch-Button", name='Search')
```

**SearchModal** агрегирует `search_input`, `search_result`, `empty_results_title`:
```python
class SearchModal:
    def __init__(self, page: Page) -> None:
        self.page = page
        self.empty_results_title = Title(page, locator='p.DocSearch-Help', name='Empty results')
        self.search_input = Input(page, locator='#docsearch-input', name='Search docs')
        self.search_result = ListItem(page, locator='#docsearch-item-{result_number}', name='Result item')
```

### 4. Локаторные шаблоны (Locator Templating)

Локаторы используют Python-строки формата с плейсхолдерами `{param}` для динамических значений:
```python
self.language_title = Title(page, locator='h2#{language}', name='Language title')
self.search_result = ListItem(page, locator='#docsearch-item-{result_number}', name='Result item')
```

Использование с `get_locator(**kwargs)`:
```python
self.language_title.should_be_visible(language=language)
self.search_result.click(result_number=result_number)
```

### 5. pytest Fixtures для внедрения зависимостей

**conftest.py** предоставляет фикстуры с областью видимости function:
```python
@pytest.fixture(scope='function')
def chromium_page() -> Page:
    with sync_playwright() as playwright:
        chromium = playwright.chromium.launch(headless=False)
        yield chromium.new_page()

@pytest.fixture(scope='function')
def playwright_home_page(chromium_page: Page) -> PlaywrightHomePage:
    return PlaywrightHomePage(chromium_page)

@pytest.fixture(scope='function')
def playwright_languages_page(chromium_page: Page) -> PlaywrightLanguagesPage:
    return PlaywrightLanguagesPage(chromium_page)
```

### 6. Интеграция с Allure

Каждое действие оборачивается в `allure.step()` для детальной отчётности:
```python
def click(self, **kwargs) -> None:
    with allure.step(f'Clicking {self.type_of} with name "{self.name}"'):
        locator = self.get_locator(**kwargs)
        locator.click()

def should_be_visible(self, **kwargs) -> None:
    with allure.step(f'Checking that {self.type_of} "{self.name}" is visible'):
        locator = self.get_locator(**kwargs)
        expect(locator).to_be_visible()
```

### 7. Playwright Sync API

Фреймворк использует `playwright.sync_api` (синхронный API) вместо async:
```python
from playwright.sync_api import Page, expect
```

## Иерархия компонентов

```
Component (ABC)
├── Button (hover, double_click)
├── Input (fill, should_have_value)
├── Link
├── Title
└── ListItem
```

## Пример теста

```python
class TestSearch:
    @pytest.mark.parametrize('keyword', ['python'])
    def test_search(self, keyword, playwright_home_page, playwright_languages_page):
        playwright_home_page.visit(BASE_URL)
        playwright_home_page.navbar.open_search()
        playwright_home_page.navbar.search_modal.find_result(keyword, result_number=0)
        playwright_languages_page.language_present(language=keyword)
```

## Ключевые принципы проектирования

1. **Принцип единственной ответственности** - каждый класс компонента обрабатывает один тип UI-элемента
2. **Композиция** - сложные компоненты (Navbar, SearchModal) агрегируют более простые
3. **Наследование** - специализированные компоненты наследуют общее поведение от Component
4. **Внедрение зависимостей** - pytest фикстуры предоставляют экземпляры страниц
5. **Абстракция** - компоненты скрывают детали локаторов Playwright за методами типа `click()`, `should_be_visible()`
