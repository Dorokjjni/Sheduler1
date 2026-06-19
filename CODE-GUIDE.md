'use strict';

/*
 * Логика версии 3.
 *
 * Главное изменение этой версии: данные хранятся не по календарным датам,
 * а по семи постоянным дням недели. Задача, записанная в воскресенье,
 * считается регулярной и отображается каждое воскресенье.
 */

// Новый ключ отделяет регулярное расписание от календарной версии приложения.
const STORAGE_KEY = 'circle-planner-recurring-v3';

// Читаемые названия категорий для списка задач.
const CATEGORY_NAMES = {
  work: 'Работа',
  study: 'Учёба',
  health: 'Здоровье',
  home: 'Дом',
  rest: 'Отдых'
};

// Индексы всегда идут от понедельника (0) до воскресенья (6).
const WEEKDAY_NAMES = [
  'Понедельник',
  'Вторник',
  'Среда',
  'Четверг',
  'Пятница',
  'Суббота',
  'Воскресенье'
];

const WEEKDAY_SHORT = ['Пн', 'Вт', 'Ср', 'Чт', 'Пт', 'Сб', 'Вс'];

// Текущее состояние интерфейса.
const state = {
  data: loadData(),
  activeDayIndex: getCurrentWeekdayIndex(),
  anchorHour: null,
  selectedRange: null
};

// Сохраняем ссылки на элементы один раз, чтобы не искать их при каждой отрисовке.
const elements = {
  activeDayLabel: document.querySelector('#activeDayLabel'),
  todayButton: document.querySelector('#todayButton'),
  selectionHint: document.querySelector('#selectionHint'),
  taskForm: document.querySelector('#taskForm'),
  selectedRange: document.querySelector('#selectedRange'),
  taskTitle: document.querySelector('#taskTitle'),
  taskCategory: document.querySelector('#taskCategory'),
  cancelSelectionButton: document.querySelector('#cancelSelectionButton'),
  formError: document.querySelector('#formError'),
  dayTaskList: document.querySelector('#dayTaskList'),
  dayTaskCount: document.querySelector('#dayTaskCount'),
  weekRange: document.querySelector('#weekRange'),
  clockSegments: Array.from(document.querySelectorAll('.hour-segment')),
  dayColumns: Array.from(document.querySelectorAll('.day-column'))
};

initialize();

function initialize() {
  bindClockEvents();
  bindMainEvents();
  renderAll();
  registerServiceWorker();
}

/* -------------------------------------------------------------------------- */
/* Обработчики событий                                                        */
/* -------------------------------------------------------------------------- */

function bindClockEvents() {
  elements.clockSegments.forEach(segment => {
    const hour = Number(segment.dataset.hour);

    // Обычное нажатие мышью или пальцем.
    segment.addEventListener('click', () => selectHour(hour));

    // То же действие с клавиатуры для доступности.
    segment.addEventListener('keydown', event => {
      if (event.key === 'Enter' || event.key === ' ') {
        event.preventDefault();
        selectHour(hour);
      }
    });
  });
}

function bindMainEvents() {
  // Кнопка выбирает сегодняшний день недели, но не создаёт календарную дату.
  elements.todayButton.addEventListener('click', () => {
    selectWeekday(getCurrentWeekdayIndex());
  });

  // Отмена закрывает форму и снимает синее выделение.
  elements.cancelSelectionButton.addEventListener('click', resetSelection);

  // Отправка формы создаёт регулярную задачу выбранного дня недели.
  elements.taskForm.addEventListener('submit', event => {
    event.preventDefault();
    saveSelectedTask();
  });
}

/* -------------------------------------------------------------------------- */
/* Выбор времени на циферблате                                                */
/* -------------------------------------------------------------------------- */

function selectHour(hour) {
  elements.formError.textContent = '';

  const isFirstClick = state.anchorHour === null;

  if (isFirstClick) {
    /*
     * Одно нажатие сразу выбирает один полный час.
     * Например, сектор 6 означает промежуток 06:00–07:00.
     */
    state.anchorHour = hour;
    state.selectedRange = { start: hour, end: hour + 1 };
  } else {
    /*
     * Следующее нажатие расширяет диапазон от первого выбранного часа.
     * Последний нажатый сектор также включается целиком.
     * Например: сначала 6, затем 8 -> 06:00–09:00.
     */
    state.selectedRange = {
      start: Math.min(state.anchorHour, hour),
      end: Math.max(state.anchorHour, hour) + 1
    };
  }

  showTaskForm();
  renderClocks();

  // После первого нажатия сразу переводим курсор в поле названия задачи.
  if (isFirstClick) {
    window.setTimeout(() => elements.taskTitle.focus({ preventScroll: true }), 60);
  }
}

function showTaskForm() {
  const { start, end } = state.selectedRange;
  const rangeText = formatRange(start, end);

  elements.selectedRange.textContent = rangeText;
  elements.taskForm.hidden = false;
  elements.selectionHint.textContent = `${rangeText}. Введите задачу или нажмите другой час, чтобы расширить промежуток.`;
}

function resetSelection() {
  state.anchorHour = null;
  state.selectedRange = null;
  elements.taskForm.hidden = true;
  elements.taskTitle.value = '';
  elements.formError.textContent = '';
  elements.selectionHint.textContent = 'Одно нажатие выбирает один час. Второе нажатие расширяет промежуток.';
  renderClocks();
}

function saveSelectedTask() {
  if (!state.selectedRange) {
    return;
  }

  const title = elements.taskTitle.value.trim();
  if (!title) {
    elements.formError.textContent = 'Введите название задачи.';
    elements.taskTitle.focus();
    return;
  }

  const dayData = getDayData(state.activeDayIndex);
  const { start, end } = state.selectedRange;

  // Не разрешаем двум регулярным задачам занимать одни и те же часы одного дня.
  const hasOverlap = dayData.tasks.some(task => start < task.end && end > task.start);
  if (hasOverlap) {
    elements.formError.textContent = 'Этот промежуток пересекается с существующей задачей.';
    return;
  }

  dayData.tasks.push({
    id: createId(),
    start,
    end,
    title,
    category: elements.taskCategory.value
  });

  dayData.tasks.sort((first, second) => first.start - second.start);
  saveData();
  resetSelection();
  renderAll();
}

/* -------------------------------------------------------------------------- */
/* Отрисовка интерфейса                                                       */
/* -------------------------------------------------------------------------- */

function renderAll() {
  renderActiveDay();
  renderClocks();
  renderDayTaskList();
  renderWeekTable();
}

function renderActiveDay() {
  elements.activeDayLabel.textContent = `${WEEKDAY_NAMES[state.activeDayIndex]} · повторяется каждую неделю`;
}

function renderClocks() {
  const dayData = getDayData(state.activeDayIndex);

  elements.clockSegments.forEach(segment => {
    const hour = Number(segment.dataset.hour);

    // Сначала возвращаем сектор в нейтральное состояние.
    segment.setAttribute('class', 'hour-segment');

    // Если час занят задачей, окрашиваем его цветом категории.
    const task = dayData.tasks.find(item => hour >= item.start && hour < item.end);
    if (task) {
      segment.classList.add(`is-${task.category}`);
      segment.setAttribute('aria-label', `${formatHour(hour)}: ${task.title}`);
      segment.setAttribute('title', `${formatRange(task.start, task.end)} — ${task.title}`);
    } else {
      segment.setAttribute('aria-label', `Выбрать ${formatHour(hour)}`);
      segment.removeAttribute('title');
    }

    // Текущий создаваемый диапазон всегда выделяем синим поверх категорий.
    if (state.selectedRange && hour >= state.selectedRange.start && hour < state.selectedRange.end) {
      segment.classList.add('is-selected');
    }
  });
}

function renderDayTaskList() {
  const tasks = getDayData(state.activeDayIndex).tasks;
  elements.dayTaskCount.textContent = String(tasks.length);
  elements.dayTaskList.innerHTML = '';

  if (tasks.length === 0) {
    const empty = document.createElement('p');
    empty.className = 'empty-state';
    empty.textContent = `Для дня «${WEEKDAY_NAMES[state.activeDayIndex]}» ещё нет регулярных задач.`;
    elements.dayTaskList.append(empty);
    return;
  }

  tasks.forEach(task => {
    const item = document.createElement('article');
    item.className = 'day-task-item';

    const time = document.createElement('span');
    time.className = `time-chip ${task.category}`;
    time.textContent = formatRange(task.start, task.end);

    const text = document.createElement('div');

    const title = document.createElement('p');
    title.className = 'task-title';
    title.textContent = task.title;

    const category = document.createElement('p');
    category.className = 'task-category';
    category.textContent = `${CATEGORY_NAMES[task.category] || 'Другое'} · каждую неделю`;

    text.append(title, category);

    const removeButton = document.createElement('button');
    removeButton.type = 'button';
    removeButton.className = 'delete-button';
    removeButton.setAttribute('aria-label', `Удалить задачу ${task.title}`);
    removeButton.textContent = '×';
    removeButton.addEventListener('click', () => deleteTask(task.id));

    item.append(time, text, removeButton);
    elements.dayTaskList.append(item);
  });
}

function deleteTask(taskId) {
  const dayData = getDayData(state.activeDayIndex);
  dayData.tasks = dayData.tasks.filter(task => task.id !== taskId);
  saveData();
  renderAll();
}

function renderWeekTable() {
  elements.weekRange.textContent = 'Это расписание не зависит от дат и повторяется каждую неделю.';
  const currentDayIndex = getCurrentWeekdayIndex();

  elements.dayColumns.forEach((column, dayIndex) => {
    const dayData = getDayData(dayIndex);

    column.classList.toggle('is-active', dayIndex === state.activeDayIndex);
    column.classList.toggle('is-current', dayIndex === currentDayIndex);

    const header = column.querySelector('.day-header');
    const dayName = column.querySelector('.day-name');
    const dayRepeat = column.querySelector('.day-date');
    const focus = column.querySelector('.day-focus');
    const miniTaskList = column.querySelector('.mini-task-list');

    dayName.textContent = WEEKDAY_SHORT[dayIndex];
    dayRepeat.textContent = dayIndex === currentDayIndex ? 'текущий день · каждую неделю' : 'каждую неделю';

    // При нажатии открываем на циферблатах именно этот регулярный день.
    header.onclick = () => selectWeekday(dayIndex);

    // Заметки каждого дня недели также сохраняются постоянно.
    focus.value = dayData.focus;
    focus.oninput = () => {
      getDayData(dayIndex).focus = focus.value;
      saveData();
    };

    miniTaskList.innerHTML = '';

    if (dayData.tasks.length === 0) {
      const noTasks = document.createElement('span');
      noTasks.className = 'no-mini-tasks';
      noTasks.textContent = 'Нет регулярных задач';
      miniTaskList.append(noTasks);
      return;
    }

    dayData.tasks.forEach(task => {
      const miniTask = document.createElement('div');
      miniTask.className = `mini-task ${task.category}`;

      const time = document.createElement('time');
      time.textContent = formatRange(task.start, task.end);

      const title = document.createElement('span');
      title.textContent = task.title;

      miniTask.append(time, title);
      miniTaskList.append(miniTask);
    });
  });
}

function selectWeekday(dayIndex) {
  state.activeDayIndex = dayIndex;
  resetSelection();
  renderAll();

  // На телефоне после выбора дня возвращаем пользователя к циферблатам.
  document.querySelector('.clocks-card').scrollIntoView({
    behavior: 'smooth',
    block: 'start'
  });
}

/* -------------------------------------------------------------------------- */
/* Хранение данных                                                            */
/* -------------------------------------------------------------------------- */

function loadData() {
  try {
    const saved = localStorage.getItem(STORAGE_KEY);
    const parsed = saved ? JSON.parse(saved) : null;

    if (parsed && Array.isArray(parsed.weekdays) && parsed.weekdays.length === 7) {
      return normalizeData(parsed);
    }
  } catch (error) {
    console.warn('Не удалось прочитать сохранённые данные:', error);
  }

  return createEmptyData();
}

function saveData() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state.data));
  } catch (error) {
    console.warn('Не удалось сохранить данные:', error);
  }
}

function createEmptyData() {
  return {
    weekdays: Array.from({ length: 7 }, () => ({ focus: '', tasks: [] }))
  };
}

function normalizeData(data) {
  return {
    weekdays: data.weekdays.map(day => ({
      focus: typeof day?.focus === 'string' ? day.focus : '',
      tasks: Array.isArray(day?.tasks) ? day.tasks : []
    }))
  };
}

function getDayData(dayIndex) {
  if (!state.data.weekdays[dayIndex]) {
    state.data.weekdays[dayIndex] = { focus: '', tasks: [] };
  }

  return state.data.weekdays[dayIndex];
}

/* -------------------------------------------------------------------------- */
/* Вспомогательные функции                                                    */
/* -------------------------------------------------------------------------- */

function getCurrentWeekdayIndex() {
  // Date.getDay(): воскресенье = 0. Переводим к формату понедельник = 0.
  return (new Date().getDay() + 6) % 7;
}

function formatHour(hour) {
  // Значение 24 допустимо только как конец последнего промежутка.
  return `${String(hour).padStart(2, '0')}:00`;
}

function formatRange(start, end) {
  return `${formatHour(start)}–${formatHour(end)}`;
}

function createId() {
  if (window.crypto && typeof window.crypto.randomUUID === 'function') {
    return window.crypto.randomUUID();
  }

  return `${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

/* -------------------------------------------------------------------------- */
/* PWA                                                                        */
/* -------------------------------------------------------------------------- */

function registerServiceWorker() {
  if (!('serviceWorker' in navigator)) {
    return;
  }

  window.addEventListener('load', () => {
    navigator.serviceWorker.register('./service-worker.js').catch(error => {
      console.warn('Service Worker не зарегистрирован:', error);
    });
  });
}
