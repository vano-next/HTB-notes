#Documentation & Reporting

- Модуль: [https://academy.hackthebox.com/app/module/162](https://academy.hackthebox.com/app/module/162)

#Notetaking & Organization

- Урок: [https://academy.hackthebox.com/app/module/162/section/1534](https://academy.hackthebox.com/app/module/162/section/1534)


***

## Тема уроку

Notetaking & Organization: структура нотаток, директорій та логування сесій під час пентест‑ассесментів, включно з рекомендованими інструментами (Obsidian, tmux‑logging), шаблонами для доказів і трекінгу артефактів, а також базовими відповідями до Q1–Q3 лаборатки (tmux, комбінація для вертикального спліту та DONE). [github](https://github.com/tmux-plugins/tmux-logging)

***

## Ціль

- Побудувати **стандартизовану структуру нотаток** (Attack Path, Credentials, Findings, Scan Research, OSINT, AD Enumeration, Administrative/Scoping, Activity Log, Payload Log). [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)
- Налаштувати **директорію проєкту** для зберігання доказів, логів та делівереблів: `PROJECT/{Admin,Deliverables,Evidence/...}`. [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)
- Освоїти **tmux‑logging** як основний інструмент логування термінальної активності та навчитися керувати логами й скріншотами. [tmuxai](https://tmuxai.dev/tmux-logging/)
- Розуміти **кращі практики evidence collection**: що логувати, як робити скріншоти, як форматувати термінальний вивід і трекати payload’и / системні модифікації. [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)
- Відповісти на Q1–Q3 лаборатки:
  - Q1: інструмент для зручного логування сесій – **tmux**.  
  - Q2: вертикальний спліт панелей – **`[Ctrl] + [B] + [Shift] + [%]`**.  
  - Q3: завершення практичних вправ – **DONE**. [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

***

## Список запитань

1. **Q1:** Який інструмент, згаданий у секції, полегшує логування термінальної сесії?  
2. **Q2:** Стів хоче розділити панелі tmux вертикально. Яку комбінацію клавіш ти йому порадиш у форматі `[Key] + [Key] + [Key] + [Key]`?  
3. **Q3:** Після підключення через `xfreerdp` до тестової VM, роботи з assessment‑директурою, Obsidian та tmux‑logging – яке слово потрібно ввести як відповідь?

***

## Відповіді

- **Q1:** `tmux` – саме його з логінг‑плагіном рекомендують як основний засіб для логування сесій. [github](https://github.com/tmux-plugins/tmux-logging)
- **Q2:** `[Ctrl] + [B] + [Shift] + [%]` – стандартна tmux‑комбінація для вертикального розділення панелі. [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)
- **Q3:** `DONE` – вводиться після завершення практичних завдань на VM (структура директорій, Obsidian, tmux‑logging). [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

***

## Конспект по суті уроку

### 1. Структура нотаток (Core Categories)

Рекомендовані основні розділи нотаток: [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

1. **Attack Path** – повний ланцюжок атаки (футхолд → прівеск → домен), із текстом і скріншотами.  
2. **Credentials** – єдиний список усіх знайдених облікових даних, токенів, куків тощо.  
3. **Findings** – окремі вразливості з доказами (скріншоти, вивід команд).  
4. **Vulnerability Scan Research** – дослідження по результатах сканерів (Nessus, OpenVAS, etc.).  
5. **Service Enumeration Research** – нотатки по портах/службах (Nmap, Masscan).  
6. **Web Application Research** – знахідки й тест‑кейси для веб‑додатків (Burp, ZAP).  
7. **AD Enumeration Research** – розслідування AD (BloodHound, PowerView, ldapsearch).  
8. **OSINT** – відкриті джерела: домени, mail, соцмережі, документація.  
9. **Administrative Information** – контакти, RoE, SOW, POC, PM.  
10. **Scoping Information** – діапазони IP, URL, надані креденшли, обмеження.  
11. **Activity Log** – таймлайн дій, корисний для кореляції інцидентів.  
12. **Payload Log** – всі завантажені файли, web‑shell’и, скрипти, з хешами та статусом cleanup.  

***

### 2. Структура директорій (Folder Structure)

Рекомендований шаблон директорій: [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

```bash
mkdir -p PROJECT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}
```

Вигляд:

```text
PROJECT/
├── Admin/                    # SOW, kickoff notes, status reports
├── Deliverables/            # Reports, spreadsheets, presentations
├── Evidence/
│   ├── Findings/           # Per-finding evidence folders
│   ├── Scans/
│   │   ├── Vuln/          # Vuln scanner output
│   │   ├── Service/       # Nmap/Masscan
│   │   ├── Web/           # Burp/ZAP/EyeWitness
│   │   └── AD Enumeration/ # BloodHound, PowerView
│   ├── Notes/              # Markdown/Obsidian/CherryTree notes
│   ├── OSINT/             # OSINT export
│   ├── Wireless/          # WiFi evidence
│   ├── Logging output/    # tmux, script logs
│   └── Misc Files/        # Payloads, scripts, tools
└── Retest/                # Retest evidence
```

***

### 3. Рекомендовані інструменти

**Notetaking:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- Обов’язково локально для клієнтських даних:
  - Obsidian (Markdown‑вощук, локальне сховище).  
  - CherryTree (дерево нотаток).  
  - VS Code (Markdown + Git).  
  - Notion (якщо компанія дозволяє, у режимі локальних воркспейсів).  

**Cloud (лише тренінг):**

- GitBook, Outline, Standard Notes, Evernote – для нетренінгових, не клієнтських даних. [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

**Session logging:** [github](https://github.com/tmux-plugins/tmux-logging)

- tmux + tmux‑logging (рекомендоване рішення).  
- `script` (Unix built‑in).  
- Terminator (GUI‑термінал з логами).  
- Windows Terminal / PowerShell транскрипти.

***

### 4. Tmux Logging Setup

**Встановлення tmux‑plugins/tpm + tmux‑logging:** [github](https://github.com/tmux-plugins/tmux-logging)

```bash
# Clone TPM
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# Створити ~/.tmux.conf
cat > ~/.tmux.conf << 'EOF'
# Plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-logging'

# History limit
set -g history-limit 50000

# Initialize TPM (at bottom)
run '~/.tmux/plugins/tpm/tpm'
EOF

# Застосувати конфіг
tmux source ~/.tmux.conf
```

**Використання:** [github](https://github.com/tmux-plugins/tmux-logging)

```bash
# Новий tmux session
tmux new -s assessment
```

- Встановити плагіни (вперше): `Ctrl+B`, `Shift+I`.  
- Старт / стоп логування: `Ctrl+B`, `Shift+P`.  
- Ретроактивне логування (зберегти весь буфер pane’а): `Ctrl+B`, `Alt+Shift+P`.  
- Screen capture поточного pane’а: `Ctrl+B`, `Alt+P`.  
- Очистити історію pane’а: `Ctrl+B`, `Alt+C`.  

**Ключові комбінації:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- `Ctrl+B`, `Shift+%` – вертикальний split.  
- `Ctrl+B`, `Shift+"` – горизонтальний split.  
- `Ctrl+B`, `O` – перемикання між pane’ами.  
- `Ctrl+B`, `Shift+P` – toggle logging.

***

### 5. Evidence Collection

**Що збирати:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- Вивід усіх ключових команд (сканування, експлуатація, прівеск).  
- Скріншоти GUI (браузер, RDP, AD tools).  
- Результати Nmap, vuln‑сканерів.  
- Успішні й невдалі експлуатації (для повноти).  
- Системну інформацію (версії, конфіги).  
- Всі знайдені креденшли.

**Скріншоти – best practices:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- Обов’язково адрес‑бар у браузері.  
- Кріпати до релевантної області, додавати просту рамку.  
- Мінімальні аннотації (стрілки, бокси).  
- Редакція **чорними блоками**, не пікселізація або CSS‑маскування.  

**Terminal output:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- По можливості не скріншотити, а **копіювати‑вставляти текст**: краще для редагування, редактінгу, копіювання, виглядає професійно.

***

### 6. Artifact Tracking

**Payload Log:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- Log:
  - Timestamp.  
  - Host IP/hostname.  
  - Шлях до payload’а.  
  - Хеш (SHA256/MD5).  
  - Статус cleanup (Removed / Needs cleanup).  
  - Опис (reverse shell, web shell, скрипт).  

Приклад:

```markdown
## Payload Log

| Timestamp           | Host        | Path                     | Hash       | Status     | Notes                     |
|---------------------|------------|--------------------------|------------|-----------|---------------------------|
| 2025-01-15 14:30    | 10.10.10.50| C:\temp\shell.exe        | a1b2c3d4...| Removed    | Reverse shell payload     |
| 2025-01-15 15:45    | 10.10.10.51| /var/www/html/cmd.php    | e5f6g7h8...| Needs cleanup | Web shell               |
```

**Account / system modifications:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- IP/host.  
- Timestamp.  
- Опис змін (користувач створений, доданий до групи, змінений конфіг).  
- Location (шлях / сервіс).  
- Статус (Removed / Reverted / Pending).  

***

### 7. Assessment Workflow

**Перед ассесментом:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

1. Створити директорію `CLIENT-ASSESSMENT/...`.  
2. Ініціалізувати notetaking (Obsidian vault / CherryTree file).  
3. Налаштувати tmux‑logging.  
4. Підготувати шаблони Evidence / Payload Log.

**Під час ассесменту:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- Логувати всі команди й результати.  
- Вести централізований список креденшлів.  
- Скріни ключових подій.  
- Зберігати невдалі спроби експлуатації.  
- Вести Activity Log.  
- Вести Payload Log і системні модифікації.

**Після ассесменту:** [scribd](https://www.scribd.com/document/968574831/HTB-Logging-Evidence)

- Сортувати evidence по Findings.  
- Редагувати чутливу інформацію.  
- Перевірити відтворюваність команд.  
- Прибрати тимчасові файли й payload’и.  
- Архівувати повний набір даних.
