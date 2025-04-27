---
created: 25-04-2025T16:40
updated: 2025-04-27T13:43
---
# Ближайшие дедлайны
```dataview
TABLE WITHOUT ID
	file.link AS "Задача",
	dateformat(deadline, "dd.MM.yyyy, HH:mm") AS "Дедлайн",
	substring(priority, 0, 2) AS "Приоритет",
	substring(status, 0, 2) AS "Статус", 
	tags AS "Теги"
FROM "Задачи"
WHERE !completed
WHERE !contains(status, "🔎")
SORT deadline
LIMIT 10
```
---
# Просроченные дедлайны
```dataviewjs
// Текущее время
const now = new Date();

function russianPlural(num, forms) {
    const cases = [2, 0, 1, 1, 1, 2];
    return forms[
        num % 100 > 4 && num % 100 < 20 ? 2 : cases[Math.min(num % 10, 5)]
    ];
}

// Функция форматирования дельты времени с русскими окончаниями
function formatTimeDelta(deadline) {
    const diffMs = deadline - now;
    const absDiffMs = Math.abs(diffMs);
    const isPast = diffMs < 0;

    // Менее часа
    if (absDiffMs < 3600000) {
        return isPast ? "менее часа назад" : "менее часа";
    }

    // Менее суток (часы)
    if (absDiffMs < 86400000) {
        const hours = Math.floor(absDiffMs / 3600000);
        const hoursText = russianPlural(hours, ["час", "часа", "часов"]);
        return isPast ? `${hours} ${hoursText} назад` : `${hours} ${hoursText}`;
    }

    // Дни
    const days = Math.floor(absDiffMs / 86400000);
    const daysText = russianPlural(days, ["день", "дня", "дней"]);
    return isPast ? `${days} ${daysText} назад` : `${days} ${daysText}`;
}

// Функция форматирования даты
function formatDate(dateStr) {
    const date = new Date(dateStr);
    return date.toLocaleString('ru-RU', {
        day: '2-digit',
        month: '2-digit',
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
    }).replace(',', '');
}

function SortByDate(dateStr) {
    const date = new Date(dateStr);
    return date.toLocaleString('ru-RU', {
	    year: 'numeric',
	    month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
    }).replace(',', '');
}

// Получаем задачи
const tasks = dv.pages('"Задачи"')
    .where(t => t.deadline &&
    new Date(t.deadline) < now && 
    !t.completed && 
    !t.status?.includes("🔎"))
    .sort(t => SortByDate(new Date(t.deadline)));

// Создаем таблицу
dv.table(
    ["Задача", "Дедлайн", "Просрочка"],
    tasks.map(t => {
        const deadline = new Date(t.deadline);
        
        return [
            // Задача
            `${t.file.link}`,
            
            // Дедлайн
            formatDate(t.deadline),
            
            // Просрочка
            formatTimeDelta(deadline),
        ];
    })
);
```
---
# Количество слов по дням
```dataviewjs
// Настройки
const DAYS_TO_SHOW = 7;
const FOLDER = "";

// Улучшенная функция для парсинга даты
function parseCustomDate(dateValue) {
    try {
        // Если это уже Date объект
        if (dateValue instanceof Date) return dateValue;
        
        // Если это строка
        if (typeof dateValue === 'string') {
            // Обрабатываем формат dd-MM-yyyyTHH:mm
            if (dateValue.includes('T')) {
                const [datePart, timePart] = dateValue.split('T');
                const [day, month, year] = datePart.split('-').map(Number);
                const [hours, minutes] = timePart.split(':').map(Number);
                return new Date(year, month - 1, day, hours, minutes);
            }
            // Пробуем стандартный парсинг
            return new Date(dateValue);
        }
        
        // Если это Luxon DateTime (используется в Dataview)
        if (dateValue?.toJSDate) return dateValue.toJSDate();
        
        return new Date(dateValue);
    } catch (e) {
        console.error("Ошибка парсинга даты:", dateValue);
        return new Date(); // Возвращаем текущую дату в случае ошибки
    }
}

// Основной код
const now = new Date();
const startDate = new Date(now);
startDate.setDate(now.getDate() - DAYS_TO_SHOW);

// Создаем контейнер для графика
const container = dv.el('div', '');

// Собираем статистику
const dailyStats = {};
const files = dv.pages(FOLDER).where(f => f.updated && !f.file.path.startsWith("Meta/") &&
        !f.file.path.startsWith("Meta\\") && !f.file.path.startsWith(".obsidian/") &&
        !f.file.path.startsWith(".obsidian\\"));

// Используем Promise.all для асинхронной обработки
Promise.all(files.map(async file => {
    try {
        const fileDate = parseCustomDate(file.updated);
        if (isNaN(fileDate.getTime())) {
            console.warn("Некорректная дата в файле:", file.file.name, file.updated);
            return;
        }
        
        if (fileDate >= startDate) {
            const dateKey = fileDate.toISOString().split('T')[0];
            
            if (!dailyStats[dateKey]) {
                dailyStats[dateKey] = { 
                    date: new Date(dateKey), 
                    wordCount: 0 
                };
            }
            
            const content = await dv.io.load(file.file.path);
            dailyStats[dateKey].wordCount += content.split(/\s+/).filter(w => w.length > 0).length;
        }
    } catch (e) {
        console.error(`Ошибка обработки файла ${file.file.name}:`, e);
    }
})).then(() => {
    // Сортируем данные
    const sortedData = Object.values(dailyStats)
        .sort((a, b) => a.date - b.date);
    
    // Подготавливаем данные
    const labels = sortedData.map(d => 
        d.date.toLocaleDateString('ru-RU', { 
            weekday: 'short', 
            day: 'numeric' 
        }));
    const data = sortedData.map(d => d.wordCount);
    
    // Создаем график
    if (sortedData.length > 0) {
        renderChart({
            type: 'line',
            data: {
                labels: labels,
                datasets: [{
                    label: 'Слов в день',
                    data: data,
                    borderColor: 'rgb(75, 192, 192)',
                    backgroundColor: 'rgba(75, 192, 192, 0.1)',
                    tension: 0.1,
                    fill: true
                }]
            },
            options: {
                responsive: true,
                plugins: {
                    title: {
                        display: true,
                        text: `Активность написания за ${DAYS_TO_SHOW} дней`,
                        font: { size: 16 }
                    },
                    tooltip: {
                        callbacks: {
                            label: ctx => `${ctx.raw} слов`
                        }
                    }
                },
                scales: {
                    y: {
                        beginAtZero: true,
                        title: {
                            display: true,
                            text: 'Количество слов'
                        }
                    }
                }
            }
        }, container);
        
        // Выводим статистику
        const total = data.reduce((sum, val) => sum + val, 0);
        dv.paragraph(`**Всего слов:** ${total} | **Среднее в день:** ${Math.round(total/DAYS_TO_SHOW)}`);
    } else {
        dv.paragraph("Нет данных для отображения за выбранный период");
    }
});
```

---
# Радар тегов
```dataviewjs
const pages = dv.pages().where(p => p.file.tags?.length > 0 && !p.file.path.startsWith("Meta/") &&  // Исключаем папку Meta
        !p.file.path.startsWith("Meta\\"));
const tagStats = {};

pages.forEach(page => {
    page.file.tags.forEach(tag => {
        const cleanTag = tag.replace(/^#/, '');
        tagStats[cleanTag] = (tagStats[cleanTag] || 0) + 1;
    });
});

const sortedTags = Object.entries(tagStats)
    .sort((a, b) => b[1] - a[1])

const labels = sortedTags.map(([tag]) => tag);
const data = sortedTags.map(([_, count]) => count);
const maxCount = Math.max(...data);

dv.paragraph(`\`\`\`chart
    type: radar
    labels: [${labels}]
    series:
    - title: Grades
      data: [${data}]
    fill: true
    legend: false
    beginAtZero: true
\`\`\``)
```

---
# Vault info
## 🗄️ Недавние обновления
`$=dv.list(dv.pages('').sort(f=>f.file.mtime.ts,"desc").limit(4).file.link)`
## 〽️ Количество файлов: `$=dv.pages().length`
