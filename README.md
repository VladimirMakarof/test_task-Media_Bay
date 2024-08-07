# Презентация выполнения тестового задания

## Задание

Для выполнения тестового задания предоставлены макеты верстки на платформе [Figma](https://www.figma.com/design/jnm6v6f6erKb93tagPon3p/%D0%A2%D0%B5%D1%81%D1%82%D0%BE%D0%B2%D0%BE%D0%B5-%D0%B7%D0%B0%D0%B4%D0%B0%D0%BD%D0%B8%D0%B5-%2F%2F-%D0%92%D0%B5%D0%B1-%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%87%D0%B8%D0%BA?node-id=0-1&t=R9xeBmf7pwpYvGv5-1). Цель - создать функциональный фильтр для товаров с возможностью фильтрации по брендам и цветам, а также отображение и взаимодействие с выбранными фильтрами.

Доступ к проекту: [ссылка ](http://stan13h6.beget.tech/), логин: `admin`, пароль: `123456`.

## Выполненные шаги

### 1. Инициализация проекта

Создан проект на основе предоставленного макета Figma. Были созданы основные HTML-структуры и стили для отображения фильтров и карточек товаров.

### 2. Получение данных из инфоблока

С помощью Bitrix API были получены данные о брендах и цветах из инфоблока:

```php
$propertyEnums = CIBlockPropertyEnum::GetList(
    Array("SORT" => "ASC"),
    Array("IBLOCK_ID" => $arParams["IBLOCK_ID"], "CODE" => "BRAND")
);
$brandList = [];
while ($enumFields = $propertyEnums->GetNext()) {
    $brandList[] = $enumFields;
}

// Аналогично для цветов
```
### 3. Сортировка и группировка брендов

Бренды отсортированы в алфавитном порядке и сгруппированы по первой букве:

```php

usort($brandList, function($a, $b) {
    return strcmp($a['VALUE'], $b['VALUE']);
});

// Группировка по первой букве
$groupedBrands = [];
foreach ($brandList as $brand) {
    $firstLetter = strtoupper(substr($brand['VALUE'], 0, 1));
    if (!isset($groupedBrands[$firstLetter])) {
        $groupedBrands[$firstLetter] = [];
    }
    $groupedBrands[$firstLetter][] = $brand;
}

```

### 4. Отображение фильтров

Фильтры брендов и цветов выведены на страницу с использованием предоставленных стилей. Для чекбоксов брендов и кругов цветов применены стили из Figma:

```php

// Пример HTML для брендов
<div class="brand-list-container">
    <?php foreach ($groupedBrands as $letter => $brands): ?>
        <div class="brand-group">
            <p class="brand-group-letter"><?= $letter ?></p>
            <?php foreach ($brands as $brand): ?>
                <div class="brand-item" data-brand="<?= $brand['ID'] ?>">
                    <input type="checkbox" id="brand_<?= $brand['ID'] ?>" name="brand[]" value="<?= $brand['ID'] ?>" checked>
                    <label for="brand_<?= $brand['ID'] ?>" class="custom-checkbox"><?= $brand['VALUE'] ?></label>
                </div>
            <?php endforeach; ?>
        </div>
    <?php endforeach; ?>
</div>

```

### 5. Настройка JavaScript для фильтрации

Создан скрипт на JavaScript для фильтрации товаров по выбранным брендам и цветам:

```js

document.addEventListener('DOMContentLoaded', function () {
    const SITE_TEMPLATE_PATH = "<?= SITE_TEMPLATE_PATH ?>";
    const brandCheckboxes = document.querySelectorAll('.brand-item input[type="checkbox"]');
    const colorCircles = document.querySelectorAll('.color-palette-container .color-circle');
    const productCards = document.querySelectorAll('.product-card-layout2');
    const productCount = document.getElementById('product-count');
    const selectedFiltersContainer = document.querySelector('.selected-filters');

    const colorMap = {
        'Белый': '#FFFFFF',
        'Черный': '#141414',
        'Коричневый': '#603E28',
        'Серый': '#808080',
        'Бежевый': '#C9723A',
        'Цвет дерева': `url(${SITE_TEMPLATE_PATH}/images/wood_texture.png)`
    };

    // Устанавливаем цвет круга для каждого товара
    productCards.forEach(card => {
        const colorValue = card.getAttribute('data-color');
        const colorCircle = card.querySelector('.flex-price-info-circle');
        if (colorCircle && colorValue) {
            if (colorMap[colorValue]) {
                if (colorValue === 'Цвет дерева') {
                    colorCircle.style.backgroundImage = colorMap[colorValue];
                    colorCircle.style.backgroundSize = 'cover';
                } else {
                    colorCircle.style.backgroundColor = colorMap[colorValue];
                    if (colorMap[colorValue] === '#FFFFFF') {
                        colorCircle.style.border = '1px solid #000000';
                    }
                }
            } else {
                colorCircle.style.backgroundColor = '#d9c8bb';
            }
        }
    });

    brandCheckboxes.forEach(checkbox => {
        checkbox.addEventListener('change', function() {
            filterProducts();
            updateSelectedFilters();
        });
    });

    colorCircles.forEach(circle => {
        circle.addEventListener('click', function() {
            circle.classList.toggle('selected');
            filterProducts();
            updateSelectedFilters();
        });
    });

    filterProducts();
    updateSelectedFilters();

    function filterProducts() {
        const selectedBrands = Array.from(brandCheckboxes)
            .filter(checkbox => checkbox.checked)
            .map(checkbox => checkbox.value);

        const selectedColors = Array.from(colorCircles)
            .filter(circle => circle.classList.contains('selected'))
            .map(circle => circle.getAttribute('data-color'));

        let visibleCount = 0;

        productCards.forEach(card => {
            const cardBrand = card.getAttribute('data-brand');
            const cardColor = card.getAttribute('data-color');

            const brandMatch = selectedBrands.includes(cardBrand);
            const colorMatch = selectedColors.includes(cardColor);

            // Если есть выбранные бренды или цвета, фильтруем товары
            if ((selectedBrands.length > 0 && brandMatch) || (selectedColors.length > 0 && colorMatch)) {
                card.style.display = 'block';
                visibleCount++;
            } else {
                card.style.display = 'none';
            }
        });

        // Если ни один фильтр не выбран, скрываем все карточки
        if (selectedBrands.length === 0 && selectedColors.length === 0) {
            productCards.forEach(card => {
                card.style.display = 'none';
            });
            visibleCount = 0;
        }

        if (productCount) {
            productCount.textContent = visibleCount;
        }
    }

    function updateSelectedFilters() {
        selectedFiltersContainer.innerHTML = '';

        brandCheckboxes.forEach(checkbox => {
            if (checkbox.checked) {
                const filterItem = document.createElement('div');
                filterItem.className = 'filter-item';
                filterItem.textContent = checkbox.nextElementSibling.textContent;
                const closeButton = document.createElement('span');
                closeButton.className = 'close-btn';
                closeButton.innerHTML = '&times;';
                closeButton.addEventListener('click', function() {
                    checkbox.checked = false;
                    filterItem.remove();
                    filterProducts();
                    updateSelectedFilters();
                });
                filterItem.appendChild(closeButton);
                selectedFiltersContainer.appendChild(filterItem);
            }
        });

        colorCircles.forEach(circle => {
            if (circle.classList.contains('selected')) {
                const filterItem = document.createElement('div');
                filterItem.className = 'filter-item';
                filterItem.textContent = circle.getAttribute('data-color');
                const closeButton = document.createElement('span');
                closeButton.className = 'close-btn';
                closeButton.innerHTML = '&times;';
                closeButton.addEventListener('click', function() {
                    circle.classList.remove('selected');
                    filterItem.remove();
                    filterProducts();
                    updateSelectedFilters();
                });
                filterItem.appendChild(closeButton);
                selectedFiltersContainer.appendChild(filterItem);
            }
        });
    }
});

```

### 6. Динамическое обновление фильтров

Реализовано динамическое обновление выбранных фильтров и их отображение в виде тегов с возможностью удаления:

```js

function updateSelectedFilters() {
    selectedFiltersContainer.innerHTML = '';

    brandCheckboxes.forEach(checkbox => {
        if (checkbox.checked) {
            const filterItem = document.createElement('div');
            filterItem.className = 'filter-item';
            filterItem.textContent = checkbox.nextElementSibling.textContent;
            const closeButton = document.createElement('span');
            closeButton.className = 'close-btn';
            closeButton.innerHTML = '&times;';
            closeButton.addEventListener('click', function() {
                checkbox.checked = false;
                filterItem.remove();
                filterProducts();
                updateSelectedFilters();
            });
            filterItem.appendChild(closeButton);
            selectedFiltersContainer.appendChild(filterItem);
        }
    });

    colorCircles.forEach(circle => {
        if (circle.classList.contains('selected')) {
            const filterItem = document.createElement('div');
            filterItem.className = 'filter-item';
            filterItem.textContent = circle.getAttribute('data-color');
            const closeButton = document.createElement('span');
            closeButton.className = 'close-btn';
            closeButton.innerHTML = '&times;';
            closeButton.addEventListener('click', function() {
                circle.classList.remove('selected');
                filterItem.remove();
                filterProducts();
                updateSelectedFilters();
            });
            filterItem.appendChild(closeButton);
            selectedFiltersContainer.appendChild(filterItem);
        }
    });
}


```

### Итог

В результате проделанной работы был создан функциональный фильтр для товаров с возможностью фильтрации по брендам и цветам, а также динамическое обновление выбранных фильтров с возможностью их удаления. Фильтрация и отображение товаров выполнены в соответствии с предоставленным макетом из Figma. Каждый элемент интерфейса был тщательно стилизован и оптимизирован для лучшего взаимодействия с пользователем.

Выражаю благодарность за предоставленное задание и возможность его выполнить. Если у вас возникнут вопросы или потребуется дополнительная информация, вы можете связаться со мной через Telegram: Vladimir_Makarof.
