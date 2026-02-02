# Техническое задание: Система онлайн-экзаменов "ExamVote" (на основе предоставленного кода и требований)

## 1. Обзор системы

### 1.1 Назначение
Система для проведения онлайн-экзаменов с гарантией корректности и отслеживаемости ответов студентов.

### 1.2 Текущая реализация (анализ предоставленного кода)
- Фронтенд: Одностраничное приложение на чистом HTML/CSS/JavaScript
- Хранение данных: LocalStorage браузера
- Архитектура: Клиент-ориентированная с эмуляцией серверного поведения
- Состояние: Управляется через объект STATE
- Экзамен: Управляется через объект EXAM

### 1.3 Цель модернизации
Трансформация в полноценную систему с серверной частью, гарантирующую:
- Strong consistency
- Полную аудируемость
- Идемпотентность операций
- Безопасное хранение результатов

## 2. Функциональные требования (на основе кода)

### 2.1 Отправка ответа (Vote submission)
#### Текущая реализация в коде:
function submitAnswer() {
  if (!EXAM.isActive) return showMessage('Exam is closed', 'error');
  if (STATE.hasSubmittedSuccessfully) return showMessage('Already submitted', 'warning');
  if (!STATE.selectedOption) return showMessage('Select an option first', 'warning');

  if (!STATE.submissionId) STATE.submissionId = generateSubmissionId();

  // Идемпотентность через LocalStorage
  const key = `exam_sub_${STATE.submissionId}`;
  if (localStorage.getItem(key) === 'ok') {
    showMessage('Already accepted (idempotent)', 'success');
    return;
  }

  localStorage.setItem(key, 'ok');
  localStorage.setItem('exam_choice', STATE.selectedOption);
}

#### Серверные требования:
1. Гарантия одного ответа на сессию
   - Проверка hasSubmittedSuccessfully на сервере
   - Блокировка повторных отправок
   - Уникальный идентификатор сессии пользователя

2. Идемпотентность
   - Генерация submissionId на клиенте
   - Серверная проверка дубликатов по submissionId
   - Возврат существующего ответа при повторной отправке

3. Валидация состояния экзамена
   - Проверка EXAM.isActive на сервере
   - Запрет отправки после закрытия экзамена

### 2.2 Подсчет ответов (Vote counting)
#### Текущая реализация:
// В коде отсутствует - требуется серверная реализация

#### Серверные требования:
1. Агрегация по вариантам ответов
   - Хранение всех ответов в БД
   - Подсчет количества ответов для каждого варианта (A, B, C, D)
   - Реальное время обновления счетчиков

2. Гарантия целостности
   - ACID-транзакции для каждого ответа
   - Контрольные суммы для проверки данных
   - Резервное копирование результатов

### 2.3 Отображение результатов (Result presentation)
#### Текущая реализация:
// Восстановление предыдущего ответа
if (localStorage.getItem('exam_choice')) {
  STATE.hasSubmittedSuccessfully = true;
  STATE.selectedOption = localStorage.getItem('exam_choice');
  showMessage('Previous answer restored', 'success');
}

#### Серверные требования:
1. Запрос результатов
   - REST API для получения статистики
   - Формат: {option: "A", count: 25, percentage: 31.25%}

2. Push-уведомления
   - WebSocket для обновления в реальном времени
   - Уведомление о закрытии экзамена
   - Публикация финальных результатов

3. Персональные результаты
   - Возможность запроса своего ответа
   - Подтверждение успешной отправки

### 2.4 Жизненный цикл сессии (Session lifecycle)
#### Текущая реализация:
// Админ-контролы
btnStart.onclick = () => { EXAM.isActive = true; updateUI(); logAudit('exam_started'); };
btnEnd.onclick   = () => { EXAM.isActive = false; updateUI(); logAudit('exam_ended'); };

#### Серверные требования:
1. Открытие сессии
   - Инициализация экзамена
   - Установка временных рамок
   - Начало приема ответов

2. Закрытие сессии
   - Прекращение приема ответов
   - Финализация результатов
   - Генерация отчетов

3. Просмотр финальных результатов
   - Доступ только после закрытия
   - Агрегированная статистика
   - Экспорт данных

## 3. Атрибуты качества (Variant A)
### 3.1 Корректность (высший приоритет)
#### На основе кода:
// Проверки перед отправкой
if (!EXAM.isActive) return showMessage('Exam is closed', 'error');
if (STATE.hasSubmittedSuccessfully) return showMessage('Already submitted', 'warning');

#### Серверные гарантии:
1. Ни один ответ не теряется
   - Подтверждение записи в БД
   - Синхронная запись с подтверждением
   - Репликация данных

2. Точность подсчета
   - Атомарные операции инкремента
   - Проверка целостности данных
   - Аудит всех изменений

### 3.2 Консистентность
#### На основе кода:
// Состояние синхронизировано между объектами
STATE.hasSubmittedSuccessfully = true;
STATE.selectedOption = localStorage.getItem('exam_choice');

#### Серверные гарантии:
1. Strong consistency
   - Все узлы видят одни данные
   - Нет рассинхронизации
   - Immediate consistency для ответов

2. Консистентность представления
   - Единое состояние для всех пользователей
   - Синхронное обновление интерфейса

### 3.3 Аудируемость
#### На основе кода:
function logAudit(action, data = {}, level = 'info') {
  const entry = document.createElement('div');
  entry.className = `audit-entry ${level}`;
  entry.innerHTML = `
    <span class="time">${new Date().toLocaleTimeString()}</span>
    <span class="action">${action}</span>
    <span class="data">${JSON.stringify(data)}</span>
  `;
  auditLogEl.prepend(entry);
}

#### Серверные требования:
1. Полная трассируемость
   - Логирование всех действий
   - Временные метки с миллисекундами
   - Идентификаторы пользователей и сессий

2. Аудиторский журнал
   - Постоянное хранение логов
   - Возможность фильтрации
   - Экспорт для анализа

## 4. Технические ограничения

### 4.1 Обработка дубликатов (явная)
#### Из кода:
// Явная проверка в LocalStorage
if (localStorage.getItem(key) === 'ok') {
  showMessage('Already accepted (idempotent)', 'success');
  return;
}

#### Серверная реализация:
1. Уникальные идентификаторы
   - submissionId генерируется клиентом
   - Проверка на стороне сервера
   - Ошибка 409 при дубликате

2. Блокировки на уровне БД
   - Unique constraint на (user_id, session_id)
   - Unique constraint на submission_id
   - Оптимистичные/пессимистичные блокировки

### 4.2 Гарантия сохранности ответов
// Текущее хранение - уязвимо
localStorage.setItem('exam_choice', STATE.selectedOption);

#### Серверные гарантии:
1. Надежное хранение
   - Реляционная БД с транзакциями
   - Репликация между дата-центрами
   - Регулярные бэкапы

2. Восстановление при сбоях
   - WAL (Write-Ahead Logging)
   - Point-in-time recovery
   - Проверка целостности

## 5. Архитектура системы

### 5.1 Компоненты на основе текущего кода

#### Фронтенд (сохраняется и расширяется):
// Объект состояния
const STATE = {
  selectedOption: null,
  submissionId: null,
  hasSubmittedSuccessfully: false
};

// Объект экзамена
const EXAM = {
  isActive: false,
  question: "What is the capital of France?",
  options: ["A) Paris", "B) London", "C) Berlin", "D) Madrid"]
};

// Аудит
const AUDIT = {
  log: [],
  enabled: true
};

#### Бэкенд (новая реализация):
1. API Gateway - маршрутизация запросов
2. Vote Service - обработка ответов
3. Session Service - управление сессиями
4. Result Service - подсчет и отдача результатов
5. Audit Service - логгирование действий
6. Database - PostgreSQL для персистентности
7. Cache - Redis для счетчиков и блокировок

### 5.2 Схема данных
-- Сессии экзаменов
CREATE TABLE sessions (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  question TEXT NOT NULL,
  options JSONB NOT NULL,
  is_active BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  closed_at TIMESTAMP
);

-- Ответы пользователей
CREATE TABLE votes (
  id UUID PRIMARY KEY,
  session_id UUID REFERENCES sessions(id),
  user_id UUID NOT NULL,
  option_id CHAR(1) NOT NULL, -- A, B, C, D
  submission_id UUID UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(session_id, user_id) -- Один ответ на сессию
);

-- Аудиторский лог
CREATE TABLE audit_log (
  id UUID PRIMARY KEY,
  action VARCHAR(100) NOT NULL,
  user_id UUID,
  session_id UUID,
  data JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Результаты (материализованное представление)
CREATE MATERIALIZED VIEW session_results AS
SELECT 
  session_id,
  option_id,
  COUNT(*) as vote_count,
  NOW() as calculated_at
FROM votes
GROUP BY session_id, option_id;

### 5.3 API Endpoints

// POST /api/sessions/{id}/vote
// На основе функции submitAnswer()
{
  method: 'POST',
  headers: {
    'X-Transaction-ID': STATE.submissionId,
    'X-User-ID': 'user-uuid'
  },
  body: {
    option: STATE.selectedOption, // 'A', 'B', 'C', 'D'
    verification_token: 'optional'
  }
}

// GET /api/sessions/{id}/results
// Для отображения результатов
{
  method: 'GET',
  response: {
    session_id: 'uuid',
    is_active: false,
    results: [
      { option: 'A', count: 25, percentage: 31.25 },
      { option: 'B', count: 18, percentage: 22.50 },
      { option: 'C', count: 22, percentage: 27.50 },
      { option: 'D', count: 15, percentage: 18.75 }
    ],
    total_votes: 80,
    is_final: true
  }
}

// POST /api/sessions/{id}/start
// Аналог btnStart.onclick
{
  method: 'POST',
  body: { admin_token: 'secret' }
}

// POST /api/sessions/{id}/end  
// Аналог btnEnd.onclick
{
  method: 'POST',
  body: { admin_token: 'secret' }
}

## 6. Процесс отправки ответа (детализация)

### 6.1 Клиентская часть (на основе кода)
### 8.1 Конфигурация окружения
// config.js - на основе структуры кода
module.exports = {
  // Настройки экзамена
  exam: {
    defaultDuration: 3600000, // 1 час
    maxOptions: 4,
    allowChanges: false
  },
  
  // Настройки голосования
  voting: {
    idempotencyWindow: 24 * 60 * 60 * 1000, // 24 часа
    maxRetries: 3,
    retryDelay: 5000
  },
  
  // Настройки аудита
  audit: {
    enabled: true,
    retentionDays: 90,
    logLevel: 'info'
  },
  
  // Настройки UI
  ui: {
    showLiveResults: false, // Для экзаменов скрываем до конца
    confirmSubmission: true,
    autoSaveInterval: 30000 // 30 секунд
  }
};

### 8.2 Масштабируемость
1. Горизонтальное масштабирование:
   - Stateless сервисы голосования
   - Балансировка нагрузки
   - Репликация БД

2. Геораспределение:
   - Региональные точки входа
   - Локальные кэши
   - Глобальная консистентность через Paxos/Raft

## 9. Тестирование (на основе поведения кода)

### 9.1 Юнит-тесты для клиента:
describe('submitAnswer', () => {
  beforeEach(() => {
    STATE.selectedOption = 'A';
    STATE.hasSubmittedSuccessfully = false;
    EXAM.isActive = true;
  });
  
  test('should reject if exam is not active', () => {
    EXAM.isActive = false;
    submitAnswer();
    expect(messageEl.textContent).toBe('Exam is closed');
  });
  
  test('should reject if already submitted', () => {
    STATE.hasSubmittedSuccessfully = true;
    submitAnswer();
    expect(messageEl.textContent).toBe('Already submitted');
  });
  
  test('should generate submissionId if not present', () => {
    STATE.submissionId = null;
    submitAnswer();
    expect(STATE.submissionId).toBeDefined();
  });
});

### 9.2 Интеграционные тесты:
describe('Vote API', () => {
  test('POST /vote should be idempotent', async () => {
    const submissionId = 'test-id-123';
    
    // Первый запрос
    const res1 = await request.post('/vote')
      .set('X-Transaction-ID', submissionId)
      .send({ option: 'A' });
    
    expect(res1.status).toBe(201);
    
    // Второй идентичный запрос
    const res2 = await request.post('/vote')
      .set('X-Transaction-ID', submissionId)
      .send({ option: 'A' });
    
    expect(res2.status).toBe(409);
    expect(res2.body.existing_vote).toBeDefined();
  });
});

## 10. Миграционный план

### Этап 1: Безопасная миграция
1. Сохранить текущий frontend код
2. Добавить API endpoints как прокси к LocalStorage
3. Постепенный перенос логики на сервер

### Этап 2: Полная серверная реализация
1. Реализация транзакционной БД
2. Система аудита и логгирования
3. Механизмы идемпотентности

### Этап 3: Улучшения и оптимизация
1. Real-time обновления через WebSocket
2. Расширенный мониторинг
3. Геораспределение

## 11. Критерии приемки

### На основе требований Variant A:
1. [ ] Корректность: Ни один ответ не потерян, все дубликаты отловлены
2. [ ] Консистентность: Все пользователи видят одинаковые результаты
3. [ ] Аудируемость: Полная трассировка от выбора до фиксации
4. [ ] Идемпотентность: Повторные отправки безопасны
5. [ ] Явная обработка ошибок: Пользователь понимает что произошло

### Технические критерии:
1. [ ] Время отклика < 2 секунд для 95% запросов
2. [ ] Доступность 99.9%
3. [ ] Полное восстановление после сбоя за < 15 минут
4. [ ] Аудиторские отчеты генерируются за < 1 минуты

