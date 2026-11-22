# Automated training pipeline: GitLab CI + AWS Step Functions

**Модель:** `churn_prediction`. Прогнозує, які клієнти найближчим часом підуть, щоб команда утримання встигла зробити їм пропозицію.
**Мета документа:** домовитися, як запускається тренування, хто відповідає за кожен крок, які перевірки має пройти модель і коли її можна випускати.

Головна думка всього документа: **pipeline, що завершився зеленим, ще не означає модель, готову до production.** Перше показує, що технічно все відпрацювало. Друге — це рішення людей на основі метрик і перевірок.

## 1. High-level workflow

```
GitLab event / manual trigger        хтось змерджив зміну, натиснув Run або спрацював розклад
        ↓
GitLab CI pipeline                   перевіряє, чи треба тренувати, збирає контекст, запускає AWS
        ↓
AWS Step Functions execution         оркеструє кроки тренування в AWS
        ↓
Data validation                      чи є дані, чи правильна схема, чи немає дірок
        ↓
Training job                         тренування на зафіксованій версії коду й даних
        ↓
Model evaluation                     метрики на test, порівняння з поточною prod-моделлю
        ↓
Logging metrics and artifacts        усе пишеться в MLflow, модель зберігається в S3
        ↓
Model Registry update                нова версія зі статусом candidate або rejected
        ↓
Notification to team                повідомлення в Slack-канал #ml-churn-pipeline
```

**Що запускає pipeline.** Merge у `main` змін у коді тренування чи конфігу моделі, ручний запуск у GitLab, розклад раз на місяць або сигнал з моніторингу про drift (детальніше в розділі 4).

**Роль GitLab CI.** Це «вхідні двері». GitLab CI вирішує, чи треба взагалі тренувати (наприклад, зміна в README тренування не запускає), фіксує, з якого коміту й гілки стартували, і передає цей контекст у Step Functions. Саме тренування в GitLab не виконується: runner лише запускає workflow в AWS і чекає на результат.

**Роль AWS Step Functions.** Це «диспетчер» кроків в AWS. Він запускає їх по черзі, зупиняє процес, якщо крок впав, робить повторні спроби там, де це безпечно, і зберігає історію кожного запуску. В AWS Console видно, на якому кроці зараз execution і де саме він зламався.

**Де фіксуються метрики.** У MLflow: кожен run має метрики, параметри, commit SHA і версію датасету.

**Де зберігається модель-кандидат.** Файли моделі лежать в S3 (artifact store MLflow), а версія зі статусом — у MLflow Model Registry.

**Хто отримує повідомлення.** Slack-канал команди: ML-інженери, MLOps, тех-лід і AI PM. При падінні окремо згадується owner кроку, на якому зламалось.

## 2. Контрольні точки pipeline

| Етап pipeline | Що перевіряє команда | Хто owner | Що є успіхом | Що робити при помилці |
|---|---|---|---|---|
| Trigger і GitLab CI | Чи запуск очікуваний (правильний trigger, гілка), чи передано commit SHA, branch і source | MLOps Engineer | Execution у Step Functions стартував з повним Git-контекстом | Зупинити, перевірити правила запуску в CI. Зайвий запуск — скасувати |
| Data validation | Чи є свіжі дані, чи збігається схема, чи заповнені обов'язкові поля, частка пропусків і баланс класів | Data Engineer | Усі перевірки пройдені, версія датасету зафіксована | Зупинити pipeline до тренування, повідомити Data Engineer |
| Training job | Чи запустилось тренування з правильною версією коду, даних і параметрів, чи не вийшло за ліміт часу | AI/ML Engineer | Job завершився, модель збережена | Подивитися логи. Інфраструктурний збій — один повтор. Помилка в коді — задача на виправлення |
| Model evaluation | Метрики на test, порівняння з поточною production-моделлю, метрики по сегментах | AI/ML Engineer + AI PM | Пройдені quality gates з розділу 3 | Модель не просувати, статус rejected, розбір причин |
| Logging | Чи записані в MLflow метрики, параметри, commit SHA, версія даних і артефакт моделі | MLOps Engineer | Run у MLflow повний, модель можна завантажити | Не реєструвати модель, доки логування не виправлене: без нього модель неможливо відтворити |
| Model Registry | Чи з'явилась нова версія з правильним статусом і посиланням на run | MLOps Engineer | Версія зареєстрована як candidate (або rejected з причиною) | Перевірити доступи до Registry і крок реєстрації |
| Notification | Чи прийшло повідомлення, і чи зрозуміло з нього, що робити далі | AI PM / MLOps Engineer | Повідомлення в каналі з усіма полями з розділу 7 | Додати алерт на «execution без повідомлення», оновити runbook |
| Approval і release | Чи хтось свідомо погодив модель перед staging/production | Tech Lead + AI PM | Статус approved з іменем того, хто погодив | Модель лишається candidate, у production нічого не змінюється |

## 3. Quality gates для нової моделі

Для churn-моделі найдорожча помилка — **пропустити клієнта, який справді піде.** Тому головна метрика — recall для класу «піде». Але recall легко «накрутити», якщо позначати як «піде» всіх підряд, і тоді команда утримання витрачатиме бюджет на клієнтів, які й так лишились би. Тому дивимось recall разом із precision і F1.

| Quality gate | Приклад правила | Хто перевіряє | Рішення |
|---|---|---|---|
| Data quality | Немає критичних помилок схеми, пропусків в обов'язкових полях менше 2%, частка churn у даних 5–20% | Data Engineer | Продовжити або зупинити pipeline |
| Технічне тренування | Job завершився, артефакт збережений, run у MLflow повний | MLOps Engineer | Продовжити або failed |
| Model metric | Recall (churn) ≥ 0.75, precision ≥ 0.60, F1 ≥ 0.70 | AI/ML Engineer | Продовжити або rejected |
| Порівняння з baseline | F1 не нижче за поточну production-модель, recall не просів більше ніж на 0.02 | AI/ML Engineer | Продовжити або rejected |
| Business metric / сегменти | Немає погіршення recall для ключових сегментів (VIP-клієнти, нові клієнти до 3 місяців) | AI PM | Погодити або повернути на аналіз |
| Стабільність | Метрики на cross-validation не розкидані більше ніж на ±0.03 | AI/ML Engineer | Продовжити або потрібні додаткові експерименти |
| Approval | Модель має статус approved, у Registry вказано, хто і коли погодив | Tech Lead + AI PM | Дозволити staging. Production — тільки після успішного staging |

**Manual approval потрібен завжди** перед staging і production. Автоматично модель може отримати лише статус candidate або rejected. Рішення ухвалюють тех-лід (технічна сторона) і AI PM (бізнес-сторона).

**Якщо модель навчилась, але не пройшла quality gates,** це нормальний результат, а не аварія. Модель отримує статус rejected з причиною, production-модель лишається тією ж, а ML-інженер розбирає, що пішло не так (дані змінились, неправильні параметри, просів сегмент). Якщо модель не проходить gates кілька запусків поспіль, це сигнал для AI PM: можливо, змінилась сама задача або дані, і треба переглянути пріоритети.

## 4. Правила запуску training pipeline

| Trigger | Коли доречний | Перевага | Ризик | Середовище |
|---|---|---|---|---|
| Push / merge у `main` | Змінився код тренування або feature engineering | Кожна зміна коду одразу перевіряється на реальному тренуванні | Зайві runs і витрати, якщо тригерить будь-яка зміна. Обмежуємо правилами за змінними файлами (`src/training/**`, `configs/**`) | dev |
| Manual trigger | Перевірка гіпотези, повторний запуск після виправлення, тренування релізного кандидата | Повний контроль, жодних випадкових запусків | Залежить від дисципліни: можуть забути запустити, або запускає хто завгодно. Право запуску має обмежене коло людей | staging / prod |
| Schedule (раз на місяць) | Регулярне оновлення на свіжих даних, бо поведінка клієнтів змінюється | Передбачуваність, модель не «старіє» | Тренування без реальної потреби, витрати на runs, які нічого не покращують | staging / prod |
| Зміна model config | Змінились гіперпараметри, набір ознак або поріг | Зміна конфігу одразу перевіряється метриками | Кілька дрібних змін поспіль — кілька тренувань. Групувати зміни в один MR | dev / staging |
| Drift signal | Моніторинг показав, що розподіл даних змінився або recall у production просів | Тренування тоді, коли воно справді потрібне | Потрібен зрілий моніторинг. Помилковий алерт — зайве тренування. Справжній drift може означати, що проблема в даних, а не в моделі | prod (з manual approval на запуск) |

Правила запуску треба погодити заздалегідь і записати в README репозиторію. Інакше або тренування запускається на кожен коміт і бюджет тане, або все залежить від того, чи хтось згадав натиснути кнопку.

## 5. Risk register

| # | Ризик | Ймовірність | Вплив | Як зменшити ризик | Owner |
|---|---|---|---|---|---|
| 1 | GitLab CI має надмірні AWS-права (наприклад, AdministratorAccess) | Medium | High | Least privilege: роль, яка може лише запустити конкретний Step Functions workflow. OIDC замість постійних ключів, окремі ролі для dev, staging і prod | DevOps / MLOps |
| 2 | AWS-ключі зберігаються в змінних GitLab і витікають | Medium | High | OIDC з короткоживучими токенами. Якщо ключі все ж потрібні — masked/protected змінні і регулярна ротація | DevOps |
| 3 | Модель не проходить quality gates, але її все одно випускають | Medium | High | Автоматично тільки candidate/rejected, перехід у staging/production лише через approval у Registry | AI/ML Engineer + AI PM |
| 4 | Training запускається занадто часто, витрати ростуть | High | Medium | Правила за змінними файлами, manual trigger для staging і prod, бюджет-алерти в AWS | AI PM |
| 5 | Тренування на неякісних або неповних даних | Medium | High | Обов'язковий data validation перед тренуванням, зупинка pipeline при помилці | Data Engineer |
| 6 | Неможливо відтворити модель (немає commit SHA, версії даних, параметрів) | Medium | High | Git-контекст передається в Step Functions і пишеться в MLflow. Модель без повного логування не реєструється | MLOps Engineer |
| 7 | Pipeline падає «тихо», команда не дізнається | Medium | Medium | Notification на success і failure, алерт на execution без фінального статусу | MLOps Engineer |
| 8 | Тренування зависає й працює годинами | Low | Medium | Timeout на кожен крок у Step Functions, ліміт на тип і кількість інстансів | MLOps Engineer |
| 9 | Dev-тренування пише в production Registry або читає prod-дані без дозволу | Low | High | Окремі AWS-акаунти або ролі для dev і prod, різні Registry/статуси | DevOps / MLOps |
| 10 | Нова модель краща в цілому, але гірша на важливому сегменті | Medium | High | Метрики по сегментах у quality gates, перевірка AI PM перед approval | AI PM |

**До використання в production обов'язково мають бути закриті ризики 1, 2, 3, 6 і 9.** Решту можна закривати паралельно, але з owner-ом і датою.

## 6. Acceptance criteria для Jira-задачі

**Задача:** Інтегрувати GitLab CI з AWS Step Functions для automated training pipeline

**Запуск і контекст**
- [ ] GitLab CI pipeline запускає AWS Step Functions workflow для тренування `churn_prediction`.
- [ ] У Step Functions input передаються `commit_sha`, `branch`, `source="gitlab-ci"`, `trigger_type` (merge / manual / schedule / drift) і хто запустив.
- [ ] Training запускається тільки за погодженими правилами з розділу 4: зміна в README не запускає тренування.
- [ ] GitLab job показує посилання на Step Functions execution і завершується з тим самим статусом, що й execution.

**Видимість і кроки**
- [ ] Execution видно в AWS Console, у назві або input є commit SHA.
- [ ] Pipeline має окремий data validation step перед training job. При помилці валідації тренування не стартує.
- [ ] Кожен крок має timeout, повтори налаштовані лише для інфраструктурних збоїв.

**Логування і Registry**
- [ ] Метрики (recall, precision, F1), параметри, commit SHA і версія датасету записуються в MLflow.
- [ ] Артефакт моделі зберігається в S3 і доступний для завантаження з MLflow.
- [ ] Нова модель отримує у Model Registry статус `candidate` або `rejected` з причиною.

**Quality gates і release**
- [ ] Quality gates з розділу 3 перевіряються автоматично після evaluation, результат видно в MLflow і в повідомленні.
- [ ] Порівняння з поточною production-моделлю входить у evaluation.
- [ ] Модель не переходить у staging чи production без manual approval, а в Registry фіксується, хто погодив.
- [ ] Pipeline не має технічної можливості сам змінити production-модель.

**Security**
- [ ] GitLab CI отримує доступ до AWS через OIDC, без постійних ключів у змінних.
- [ ] IAM-роль має лише права на запуск конкретного workflow і читання його статусу.
- [ ] Для dev, staging і prod — окремі ролі. Запуск prod-тренування дозволено лише з protected-гілки.

**Notification і документація**
- [ ] Команда отримує повідомлення в Slack при success і failure у форматі з розділу 7.
- [ ] У README описано: як запустити pipeline, правила запуску, де дивитися результати, що робити при падінні (runbook).

**Definition of Done:** команда запустила pipeline тричі (успішний запуск, запуск із зламаними даними, запуск з моделлю нижче порогу), і в кожному випадку отримала правильний статус в AWS, MLflow, Registry і Slack.

## 7. Notification messages

**Успіх:**

```
✅ Training pipeline completed: model is a candidate, NOT released

Pipeline: churn-training (GitLab CI → Step Functions)
Model: churn_prediction, version 14
Trigger: merge to main by o.petrenko
Commit: 7f3e2a1 (branch: main)
Step Functions execution: ChurnTrainingPipeline / exec-7f3e2a1-20261123-1012
Data: customers_2026_11 (validated, 412,380 rows)

Metrics vs production (v12):
  Recall (churn):  0.78  (prod 0.76)  ✅
  Precision:       0.64  (prod 0.63)  ✅
  F1:              0.70  (prod 0.69)  ✅
  VIP segment recall: 0.81 (prod 0.82)  ⚠️ -0.01, within allowed range

Quality gates: 6/6 passed
Registry status: candidate
Details: MLflow run 9c41d…  |  Model Registry → churn_prediction v14

Next action: review the VIP segment and approve/reject for staging by Nov 26
Owner: Tech Lead (technical), AI PM (business sign-off)
```

**Помилка:**

```
❌ Training pipeline failed: no new model

Pipeline: churn-training (GitLab CI → Step Functions)
Model: churn_prediction
Trigger: schedule (monthly retraining)
Commit: 7f3e2a1 (branch: main)
Step Functions execution: ChurnTrainingPipeline / exec-sched-20261201-0300
Failed step: DataValidation
Reason: column "contract_type" missing in customers_2026_12; 18% nulls in "monthly_charges" (limit 2%)

Impact: training did not start. Production model v12 keeps working unchanged.
Details: Step Functions execution → DataValidation logs  |  GitLab job #48211

Next action: check the data export and rerun the pipeline manually after the fix
Owner: Data Engineer (@data-oncall)
```

В обох повідомленнях окремо сказано, що сталося з production. У першому — що модель **ще не випущена**, у другому — що production працює як і раніше. Саме це найперше хоче знати стейкхолдер, коли бачить повідомлення.

## 8. Висновок

Automated training pipeline перетворює тренування з ручного запуску ноутбука на повторюваний процес: кожна модель тренується однаково, з відомим кодом і даними, і її можна відтворити. GitLab CI вирішує, коли тренувати, і прив'язує кожен запуск до конкретного коміту. AWS Step Functions проводить тренування крок за кроком, зупиняється на помилках і зберігає історію виконання. Але зелений статус pipeline означає лише, що всі кроки відпрацювали технічно: модель може успішно натренуватись і при цьому бути гіршою за поточну, особливо на важливому сегменті клієнтів. Тому між «pipeline success» і production стоять quality gates, порівняння з production-моделлю і ручне погодження тех-ліда та AI PM. Перед першим production-релізом потрібно також закрити ключові ризики: доступ GitLab до AWS лише через OIDC з мінімальними правами, повне логування в MLflow, розділення dev і prod та заборона автоматичного релізу.
